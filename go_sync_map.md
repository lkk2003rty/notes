# sync.Map 源码分析

Go 发布了 1.9 版本，其中一个特性就是实现了并发安全的 map。作为一个无知好奇青年本菜鸟就去围观了下它到底是怎么实现的，有啥高明之处。要说如果自己实现一个并发安全的 map 的话，最简单的想法就是在原生的 map 外面加上一个读写锁，然后实现 Set、Get、Delete 方法就差不多齐活了。实际上在没有官方版本的并发安全 map 出现之前 Go 官方也是这么建议的，请戳[这里](https://blog.golang.org/go-maps-in-action)。那么 Go 1.9 里的 sync.Map 是这么实现的么？当然不是，它实际上比这要复杂多了。

## 概览

首先看一下 sync.Map 的结构体定义

```go
type Map struct {
    mu Mutex
    read atomic.Value
    dirty map[interface{}]*entry
    misses int
}

type entry struct {
    p unsafe.Pointer // *interface{}
}
```

其实 sync.Map 结构体并不复杂。原生的 map 肯定是必不可少的一部分（dirty），同时为了支持并发的写所以肯定会有锁（mu）。除此以外多了一个计数器 misses 和初看不知是啥的东东 read。这里可以看到 dirty 的 value 类型是 `*entry`，而 entry 结构体可以认为就是 unsafe.Pointer 类型。因此不难猜测实际上在 sync.Map 内部并没有存 value 的值，而是存了 value 的地址。但是这里也留给我们 2 个问题。

- 为什么 dirty 的 value 类型是一个指针，这样设计有什么用意？
- read 成员变量到底是什么类型的？ 

此外，我们还看到另外一个名为 readOnly 的结构体，它是长这样的。

```go
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}
```

根据注释可以得知 sync.Map 中 read 成员变量的类型就是 readOnly，这也回答我们上面的疑问。注意到 readOnly 结构体中 amended 这一成员变量的注释，再结合 readOnly 这样的结构体名字，我们对 sync.Map 的实际操作可以做个大概的猜测。sync.Map 中有 2 个 map，其中 1 个（dirty）是负责写操作的，另外 1 个（read）是负责读操作的，各司其职。其中只有涉及到操作 dirty 的时候才需要加锁，也就是 sync.Map 中的 mu 干的事儿。由于是 2 个 map，所以必然要涉及数据同步的问题，以保证数据的准确性。因此上面 sync.Map 中的 missess 也好，readOnly 中的 amended 也罢就是用于做数据同步的辅助变量。从后续的代码阅读来看，这个猜测是正确的。

接下来，仔细围观下 sync.Map 支持的操作。

- Load: 获取 key 对应的 value
- Delete: 删除 key
- Store: 设置键值对
- LoadOrStore: 如果 key 存在返回对应的 value，否则就是设置键值对
- Range: 其实就是 map 操作

是不是赶脚少了点什么？对，没有返回 sync.Map 中 key 数量的函数。但是转念一想，这个也容易实现啊。利用 Range 函数搞一把就行了。

下面分别看一下这每一个函数具体是怎么实现的，有何玄机。

## Load

根据上面的分析，取个 key 对应的 value 不麻烦啊，查一遍 read，如果找不到的话从 dirty 找就是了。大体上代码的流程也是这样的没错，但是实际上还有些小地方需要注意的。

```go
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // 当前 read 找不到 key 并且 dirty 已经和 read 不一致的情况下需要去 dirty 找
    if !ok && read.amended {
        m.mu.Lock()
        // Avoid reporting a spurious miss if m.dirty got promoted while we were
        // blocked on m.mu. (If further loads of the same key will not miss, it's
        // not worth copying the dirty map for this key.)
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // Regardless of whether the entry was present, record a miss: this key
            // will take the slow path until the dirty map is promoted to the read
            // map.
            // 增加 misses 计数并决定是否更新 read
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

func (e *entry) load() (value interface{}, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*interface{})(p), true
}
 
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}

```

- 代码中的流程是一旦在 read 找不到对应的 key 的话，加锁之后还要在重新加载一遍 read，这是为啥？
  注释已经给出了说明，下面具体分析下。由于 sync.Map 的目标是并发安全，那么设想这样一种场景：第一次在 read 查找不到，加锁准备从 dirty 里找的时候其它的 goroutine 刚好把这个 key/value 塞到 read 里面了。所以应对这样的场景需要在加锁之后重新加载一次 read。
- 上面描述的场景是否有可能发生呢？ 
  有，只要有其它 goroutine 在 `read, _ := m.read.Load().(readOnly)` 和 `m.mu.Lock()` 之间完成了 sync.Map 的 Store 操作就会出现这样的问题。
- 那么在加锁之后再去从 read 里面找能否保证在下面流程中会不会出现和上面同样的 read 中数据变化的问题呢？
  不会，这就需要我们去查看什么时候会对 read 做删除或插入操作了。通过简单的搜索就能知道，read 不会删除 key，也不会插入新的 key。只会有一种情况，就是被重新赋值。而重新赋值的时机有两处，两处都是在加锁了之后进行的。一处位于 Store 中 `m.read.Store(readOnly{m: read.m, amended: true})`，另外一处则位于 LoadOrStore 中调用 missLocked 方法间接实现。
- 好麻烦的流程啊，为何不直接一上来就加锁再进行查找的流程呢?
  也可以。但是如果一上来就加锁的话，和原始的锁+原生 map 的做法有何区别呢？现在的做法可以充分利用 read 的特性。可以看到 read 要更新就是整体更新，那么也就是说只有当 read 和存储真实情况的 dirty 不一致的时候才有必要去 dirty 去查找一番。而这不一致的情况==有且只能是==查找的 key 在 dirty 而不在 read 中。
  
想明白了这些问题再去看代码就清晰多了。read 中的 amended 变量用来表示 read 和 dirty 是否是一致的。而在 missLocked 函数中就可以看出 sync.Map 中的 misses 是个计数器，统计在 read 中找不到 key 的次数。一旦累计次数和 dirty 中 key 的数量一致的情况下就会去更新 read 并清空 dirty 和 misses。更新 read 中 m 的话就直接把 dirty 复制一份就是了。这里需要注意一点就是 read 中 m 的 value 类型是指针，也就是说==如果把 read 中 m 的 value 指向的内容改了那么对应的 dirty 里的也变了==。

这里还有个地方需要注意，就算在 read 中找到了 key，也有可能实际上这个 key 已经不存在了。这里就涉及到 entry 中值的变化了，请参考后面给出的 entry 值的变迁图。

## Delete

```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    if ok {
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        // 判断是否已经被删除了，不可少
        if p == nil || p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```

这就比较简单了 read 中能找到的就将其对应 entry 中的值设置为 nil，找不到并且 read 和 dirty 不一致的话加锁直捣黄龙把 dirty 中的 key 删了。这里唯一需要注意的是 entry 的 delete 函数，在这个函数中所有的操作都是在一个死循环中的。为啥？还是那句话，因为是并发操作，所以为了保证正确性不得已而为之。那么会死循环么？有可能，但是概率极低。毕竟要保证==每次==循环中在 `atomic.LoadPointer(&e.p)` 之后 `atomic.CompareAndSwapPointer(&e.p, p, nil)` 之前，e.p 的值变了，这基本不可能啊。

## Store

```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            // The entry was previously expunged, which implies that there is a
            // non-nil dirty map and this entry is not in it.
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        e.storeLocked(&value)
    } else {
        if !read.amended {
            // We're adding the first new key to the dirty map.
            // Make sure it is allocated and mark the read-only map as incomplete.
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

// tryStore stores a value if the entry has not been expunged.
//
// If the entry is expunged, tryStore returns false and leaves the entry
// unchanged.
func (e *entry) tryStore(i *interface{}) bool {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return false
    }
    for {
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
    }
}

// unexpungeLocked ensures that the entry is not marked as expunged.
//
// If the entry was previously expunged, it must be added to the dirty map
// before m.mu is unlocked.
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// storeLocked unconditionally stores a value to the entry.
//
// The entry must be known not to be expunged.
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func newEntry(i interface{}) *entry {
    return &entry{p: unsafe.Pointer(&i)}
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```

Store 操作相对来说比较复杂，主要是由于存在并发操作的可能性所以需要处理各种情况。其总体流程是能在 read 里搞的就在 read 里搞，搞不了的就往 dirty 里扔。之前说过 read 不会删 key 也不会增加新的键值对，只会整体更新，所以要直接往 read 里存的前提就是这个 key 已经存在。 因此 Store 一上来就判断 key 是否在 read 里是否存在，存在的话尝试直接在 read 里设置键值对。

看到这样的流程我们自然就有这样的疑问，是否会出现 read 中有这个 key 而 dirty 没有的情况。因为从后面 dirty 的赋值过程我们可以看到 read 和 dirty 中相同 key 的 value 都是指向同样地方的指针，所以无论谁改都无所谓。那么为了检查是否有 read 和 dirty 不一致的情况出现我们就需要查看 dirty 是怎么更新的。dirty 的更新是在 dirtyLocked 这个函数里。主要流程就是把 read 里面 value 靠谱的扔到 dirty 里，如果 value 恰好是 nil 的话则变更成 expunged，表示这个值 dirty 里没有而 read 里面有。在这一过程考虑两种情况。1. 本来 value 是有值的，在函数 tryExpungeLocked 的循环过程中其它 goroutine 调用 Delete 而把其变成了 nil。2. 本来 value 是 nil，在函数 tryExpungeLocked 的循环过程中其它 goroutine 调用 Store 而把其变成了有值的。无论情况 1 还是情况 2 函数 tryExpungeLocked 都会返回 false，从而使得后续这个键值对会扔到 dirty 里面。这样就能保证 dirty 中所有的 value 都不是 expunged 的了。再结合后面的流程我们所以可以的出结论 read 会有 dirty 没有的 key，但是这些 key 对应的 value 都是 expunged 的。而在往这样的 key 存 value 的时候会有加锁往 dirty 的操作，从而保证的不会有 bug。这也是为什么在 Load 和 Range 中可以直接用 dirty 来更新 read 的原因。

## LoadOrStrore

```go
// LoadOrStore returns the existing value for the key if present.
// Otherwise, it stores and returns the given value.
// The loaded result is true if the value was loaded, false if stored.
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool) {
    // Avoid locking if it's a clean hit.
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        actual, loaded, ok := e.tryLoadOrStore(value)
        if ok {
            return actual, loaded
        }
    }

    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            m.dirty[key] = e
        }
        actual, loaded, _ = e.tryLoadOrStore(value)
    } else if e, ok := m.dirty[key]; ok {
        actual, loaded, _ = e.tryLoadOrStore(value)
        m.missLocked()
    } else {
        if !read.amended {
            // We're adding the first new key to the dirty map.
            // Make sure it is allocated and mark the read-only map as incomplete.
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
        actual, loaded = value, false
    }
    m.mu.Unlock()

    return actual, loaded
}

// tryLoadOrStore atomically loads or stores a value if the entry is not
// expunged.
//
// If the entry is expunged, tryLoadOrStore leaves the entry unchanged and
// returns with ok==false.
func (e *entry) tryLoadOrStore(i interface{}) (actual interface{}, loaded, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == expunged {
        return nil, false, false
    }
    if p != nil {
        return *(*interface{})(p), true, true
    }

    // Copy the interface after the first load to make this method more amenable
    // to escape analysis: if we hit the "load" path or the entry is expunged, we
    // shouldn't bother heap-allocating.
    ic := i
    for {
        if atomic.CompareAndSwapPointer(&e.p, nil, unsafe.Pointer(&ic)) {
            return i, false, true
        }
        p = atomic.LoadPointer(&e.p)
        if p == expunged {
            return nil, false, false
        }
        if p != nil {
            return *(*interface{})(p), true, true
        }
    }
}

```

LoadOrStrore 的逻辑其实就是把 Load 和 Store 结合在一起就差不多，没啥要特别注意的。 

## Range

```go
func (m *Map) Range(f func(key, value interface{}) bool) {
    // We need to be able to iterate over all of the keys that were already
    // present at the start of the call to Range.
    // If read.amended is false, then read.m satisfies that property without
    // requiring us to hold m.mu for a long time.
    read, _ := m.read.Load().(readOnly)
    if read.amended {
        // m.dirty contains keys not in read.m. Fortunately, Range is already O(N)
        // (assuming the caller does not break out early), so a call to Range
        // amortizes an entire copy of the map: we can promote the dirty copy
        // immediately!
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        if read.amended {
            read = readOnly{m: m.dirty}
            m.read.Store(read)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }

    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue
        }
        if !f(k, v) {
            break
        }
    }
}
```

Range 实际就是一个 map 操作，主要流程其实就是保证下 read 的数据准确，然后遍历 read 就是了。

## entry 值的变迁

由于 sync.Map 的读写分离设计导致了负责存储 value 的 entry 多了些其它的变化，需要引入 nil 和 expunged 这两个特殊的值来标识状态（expunged 的定义如下所示）。也就是意味这其实 value 是存在一个状态机的，其状态变迁图如下。

```go
// expunged is an arbitrary pointer that marks entries which have been deleted
// from the dirty map.
var expunged = unsafe.Pointer(new(interface{}))
```

```
    
v ----> nil ----> expunged 
         /|\         |
          |__________|           
```

这里 v 表示正常的键值对的 value 值。

v -> nil： 只是调用 Delete 操作触发
nil -> expunged：更新 dirty 标识该值对应的 key 是 read 中有而 dirty 没有
expunged -> nil：调用 Store 或者 LoadOrStore 的时候将该键值对写入 dirty，虽然这里是改成了 nil 但是后续操作会把真正的值赋上的。

# 总结

整个 sync.Map 的分析到此就可以告一个段落了，可以看到要实现一个并发安全并且高效的 map 还是很麻烦的。由于很多时候没有加锁导致在很多地方需要写死循环，而后在死循环里面进行 CAS 操作从而保证原子性。而且就算加锁了也要对不是并发安全的地方进行 double check。




