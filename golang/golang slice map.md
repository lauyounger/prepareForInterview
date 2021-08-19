### 数组
数组是由一组相同类型的元素组成的数据结构，计算机会为其分配一块联系的内存空间，我们可以通过下标快速访问。

#### 初始化
数组的初始化，分为字面常量和变量。
字面常量值，在编译期间，就可以确定数组的大小。也可以通过静态类型检查判断数组越界

常量初始化时，golang做了一些特殊处理，会根据数组大小做不同的策略：   
1.当元素数量小于或者等于4个时，会直接将数组中的元素放置在栈上；    
2.当元素数量大于4个时，会将数组中的元素放置到静态区并在运行时取出；    

总结起来，在不考虑逃逸分析的情况下，如果数组中元素的个数小于或者等于 4 个，那么所有的变量会直接在栈上初始化，如果数组元素大于 4 个，变量就会在静态存储区初始化然后拷贝到栈上，那为什么这么做呢，个人的理解是在栈上初始化需要对变量一个一个赋值，静态区的话可以直接把整片内存复制过去，在元素多的时候会节省很多时间。

变量初始化或者访问就必须在运行期间，由运行时检测

---


### 切片

切片是动态数组

#### 数据结构

```
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
- Data 是指向数组的指针;
- Len 是当前切片的长度；
- Cap 是当前切片的容量，即 Data 数组的大小


#### 切片扩容
当添加的元素数量大于当前容量时，切片需要扩容，具体规则如下：
- 新容量大于当前容量的两倍，直接使用新容量
- 新容量小于当前容量的两倍
    - 小于1024的，双倍扩容
    - 大于1024的，循环递增当前容量的1/4，直到大于当前容量

除此之外，go的内存分配规则，为了避免内存碎片还需要内存对齐，所以会按照内存分配规则的class_to_size常量列表向上取整新容量，获得新的内存大小。

###### 面试题：
1.切片的自动扩容，大概是什么逻辑   
2.Append单个元素和多个元素，策略有区别的吗   

回答：append单个元素，或者append少量的多个元素，这里的少量指double之后的容量能容纳，这样就会走以下扩容流程，不足1024，双倍扩容，超过1024的，1.25倍扩容。若是append多个元素，且double后的容量不能容纳，直接使用预估的容量。敲重点！！！！此外，以上两个分支得到新容量后，均需要根据slice的类型size，算出新的容量所需的内存情况capmem，然后再进行capmem向上取整，得到新的所需内存，除上类型size，得到真正的最终容量,作为新的slice的容量。

---

### map==哈希表


hash表是一种key-values结构的重要数据结构，存取理想情况下都在O(1)复杂度，要想实现理想的hash表结构，需要关注两个点，哈希函数和冲突解决方法。

###### hash函数
实现哈希表的关键点在于哈希函数的选择，哈希函数的选择在很大程度上能够决定哈希表的读写性能。在理想情况下，哈希函数应该能够将不同键映射到不同的索引上，这要求哈希函数的输出范围大于输入范围，但是由于键的数量会远远大于映射的范围，所以在实际使用时，这个理想的效果是不可能实现的。


###### 解决hash冲突的方法：

- 开放地址寻址法

开放地址法读取数据
当需要查找某个键对应的值时，就会从索引的位置开始对数组进行线性探测，找到目标键值对或者空内存就意味着这一次查询操作的结束。   
**装载因子**：
开放寻址法中对性能影响最大的就是装载因子，它是数组中元素的数量与数组大小的比值，随着装载因子的增加，线性探测的平均用时就会逐渐增加，这会同时影响哈希表的读写性能，当装载率超过 70% 之后，哈希表的性能就会急剧下降，而一旦装载率达到 100%，整个哈希表就会完全失效，这时查找和插入任意元素的时间复杂度都是 𝑂(𝑛) 的，它们可能需要遍历数组中全部的元素，所以在实现哈希表时一定要时刻关注装载因子的变化。

- 拉链法

拉链法也有其对应装载因子的计算方式：装载因子=元素数量÷桶数量   

Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。
哈希在每一个桶中存储键对应哈希的高 8 位，当对哈希进行操作时，这些 tophash 就成为了一级缓存帮助哈希快速遍历桶中元素，每一个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会被存储到哈希的溢出桶中。
随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。

```go
type hmap struct {
	count     int       //map中的key-value对个数
	flags     uint8     //是否扩容标志，用于并发是是否正在写入的判断
	B         uint8     //已创建桶的个数为2^B
	noverflow uint16    //已使用的溢出桶个数
	hash0     uint32    //hash seed种子

	buckets    unsafe.Pointer   //当前桶
	oldbuckets unsafe.Pointer   //扩容时的旧桶
	nevacuate  uintptr          //接下来要迁移的桶的编号

	extra *mapextra     //额外的
}

type mapextra struct {
	overflow    *[]*bmap    //已经被使用的所有溢出桶的数组地址
	oldoverflow *[]*bmap    //扩容时，旧桶已使用的溢出桶数组地址
	nextOverflow *bmap      //下一个空闲溢出桶
}

```

![image-20210819122201016](/Users/liuyangyang/Library/Application Support/typora-user-images/image-20210819122201016.png)

###### 扩容机制

map插入元素的时候，会触发扩容机制，有两种扩容方式：
- 当前元素的数量/桶的个数 > 6.5的时候，触发翻倍扩容。
- 使用了太多的溢出桶时（溢出桶使用的太多会导致map处理速度降低）   
    - B <=15，已使用的溢出桶个数 >= $2^B$ 时，引发等量扩容。
    - B > 15，已使用的溢出桶个数 >= $2^{15}$ 时，引发等量扩容。

当扩容之后：

- 第一步：B会根据扩容后新桶的个数进行增加（翻倍扩容新B=旧B+1，等量扩容 新B=旧B）。
- 第二步：oldbuckets指向原来的桶（旧桶）。
- 第三步：buckets指向新创建的桶（新桶中暂时还没有数据）。
- 第四步：nevacuate设置为0，表示如果数据迁移的话，应该从原桶（旧桶）中的第0个位置开始迁移。
- 第五步：noverflow设置为0，扩容后新桶中已使用的溢出桶为0。
- 第六步：extra.oldoverflow设置为原桶（旧桶）已使用的所有溢出桶。即：h.extra.oldoverflow = h.extra.overflow
- 第七步：extra.overflow设置为nil，因为新桶中还未使用溢出桶。
- 第八步：extra.nextOverflow设置为新创建的桶中的第一个溢出桶的位置。

![image-20210819122747396](/Users/liuyangyang/Library/Application Support/typora-user-images/image-20210819122747396.png)

**翻倍扩容**   
翻倍扩容就是将旧桶的数据，分流到两个新桶中，比例不定。
首先，我们要知道，翻倍扩容容量会是之前的2倍，所以hmap的B会加一。扩容后还需要进行数据迁移，就要便利旧桶里面的数据(包含溢出桶的)，然后重新hash，得到新key。具体分配到哪个桶是由key的低B位决定的，新B已经加一，所以需要多一bit位来确定新桶的编号。

- 当新增的位的值为 0，则数据会迁移到与旧桶编号一致的位置。
- 当新增的位的值为 1，则数据会迁移到翻倍后对应编号位置。

这就是代码精髓的地方了。

![image-20210819124254578](/Users/liuyangyang/Library/Application Support/typora-user-images/image-20210819124254578.png)


```
旧桶个数为32个，翻倍后新桶的个数为64。
在重新计算旧桶中的所有key哈希值时，红色位只能是0或1，所以桶中的所有数据的后B位只能是以下两种情况：
	- 000111【7】，意味着要迁移到与旧桶编号一致的位置。
	- 100111【39】，意味着会迁移到翻倍后对应编号位置。
	
特别提醒：同一个桶中key的哈希值的低B位一定是相同的，不然不会放在同一个桶中，所以同一个桶中黄色标记的位都是相同的。
```

**等量扩容**
场景：当我们频繁大量的插入、删除数据时，会建立大量的溢出桶，这不仅会造成内存泄漏，也会极大的影响map的读写效率。因此需要等量扩容。

如果是等量扩容（溢出桶太多引发的扩容），那么数据迁移机制就会比较简单，就是将旧桶（含溢出桶）中的值迁移到新桶中。

这种扩容和迁移的意义在于：当溢出桶比较多而每个桶中的数据又不多时，可以通过等量扩容和迁移让数据更紧凑，从而减少溢出桶。
