```go
type Map struct {
    //互斥锁，用于锁定dirty map
    mu Mutex    
    
    //优先读map,支持原子操作，注释中有readOnly不是说read是只读，而是它的结构体。read实际上有写的操作
    read atomic.Value 
    
    // dirty是一个当前最新的map，允许读写
    dirty map[interface{}]*entry 
    
    // 主要记录read读取不到数据加锁读取read map以及dirty map的次数，当misses等于dirty的长度时，会将dirty复制到read
    misses int 
}

// readOnly 主要用于存储，通过原子操作存储在 Map.read 中元素。
type readOnly struct {
    // read的map, 用于存储所有read数据
    m       map[interface{}]*entry
    
    // 如果数据在dirty中但没有在read中，该值为true,作为修改标识
    amended bool 
}

// entry 为 Map.dirty 的具体map值
type entry struct {
    // nil: 表示为被删除，调用Delete()可以将read map中的元素置为nil
    // expunged: 也是表示被删除，但是该键只在read而没有在dirty中，这种情况出现在将read复制到dirty中，即复制的过程会先将nil标记为expunged，然后不将其复制到dirty
    //  其他: 表示存着真正的数据
    p unsafe.Pointer // *interface{}
}
```

sync.Map的原理很简单，使用了空间换时间策略，通过冗余的两个数据结构(read、dirty),实现加锁对性能的影响。
通过引入两个map将读写分离到不同的map，其中read map提供并发读和已存元素原子写，而dirty map则负责读写。
这样read map就可以在不加锁的情况下进行并发读取,当read map中没有读取到值时,再加锁进行后续读取,并累加未命中数。
当未命中数大于等于dirty map长度,将dirty map上升为read map。
从结构体的定义可以发现，虽然引入了两个map，但是底层数据存储的是指针，指向的是同一份值。

1. `sync.map` 是线程安全的，读取，插入，删除也都保持着常数级的时间复杂度。
2. 通过读写分离，降低锁时间来提高效率，适用于读多写少的场景。
3. Range 操作需要提供一个函数，参数是 `k,v`，返回值是一个布尔值：`f func(key, value interface{}) bool`。
4. 调用 Load 或 LoadOrStore 函数时，如果在 read 中没有找到 key，则会将 misses 值原子地增加 1，当 misses 增加到和 dirty 的长度相等时，会将 dirty 提升为 read。以期减少“读 miss”。
5. 新写入的 key 会保存到 dirty 中，如果这时 dirty 为 nil，就会先新创建一个 dirty，并将 read 中未被删除的元素拷贝到 dirty。
6. 当 dirty 为 nil 的时候，read 就代表 map 所有的数据；当 dirty 不为 nil 的时候，dirty 才代表 map 所有的数据。

总结

- sync.Map采用空间换时间的思想， 通过冗余的两个数据结构(read、dirty),减少加锁对性能的影响
- 使用只读数据(read)，避免读写冲突。ready也会有cas更新数据
- 优先从read读取、更新、删除，因为对read的读取不需要锁
- 采用了延迟删除，删除一个键值只是打标记（会将key对应value的pointer置为nil，但read中仍然有这个key:key;value:nil的键值对）
- 只有在提升dirty的时候才清理删除的数据
- 采用动态调整，当misses次数过多时，将dirty map提升为read map
  



store

1.判断ready中是否存在，存在则用cas操作直接更新值

2，加锁，

是否存在dirty中。存在直接更新数据。不存在为新值，则将ready中的未被标记为删除的值复制到dirty。新插入的值也放入dirty

3.解锁



load

read中读数据，未被读出读出则返回。

未读出，加锁去dirty中查找数据 查到及返回，miss加一。判断条件是否升级dirty为ready map。

不存在返回默认值、



delete

判断ready中是否存在数据，存在标记为删除。数据指针指为nil

不存在，加锁。dirty中查找。找到删除。没找到也不返回任何失败。解锁

