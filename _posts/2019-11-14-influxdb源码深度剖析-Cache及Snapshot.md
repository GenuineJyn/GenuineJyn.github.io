# influxdb源码深度剖析 --- Cache及Snapshot

> 本文涉及的代码基于[influxdb v1.7.8](https://github.com/influxdata/influxdb/tree/v1.7.8).
> 
> ***关键词***： TSM Cache，Snapshot(快照落盘)
> 

本文对influxdb TSM引擎的重要组成部分：Cache进行详细介绍，并对Snapshot落盘到TSM file加以描述。


## 1. Cache

Engine创建的时候创建了Cache，默认大小1GB, 内存缓存。

```
// file: tsdb/engine/tsm1/engine.go
216     cache := NewCache(uint64(opt.Config.CacheMaxMemorySize))
```
看看Cache具体实现，结构定义如下。

```
179 // Cache maintains an in-memory store of Values for a set of keys.
180 type Cache struct {
181     // Due to a bug in atomic  size needs to be the first word in the struct, as
182     // that's the only place where you're guaranteed to be 64-bit aligned on a
183     // 32 bit system. See: https://golang.org/pkg/sync/atomic/#pkg-note-BUG
184     size         uint64		// 当前已经使用掉的大小
185     snapshotSize uint64
186
187     mu      sync.RWMutex
188     store   storer			  	// 实际存储数据的地方
189     maxSize uint64				// 就是创建时候的那个大小，默认1GB
190
191     // snapshots are the cache objects that are currently being written to tsm files
192     // they're kept in memory while flushing so they can be queried along with the cache.
193     // they are read only and should never be modified
194     snapshot     *Cache		// 数据落盘的时候需要用到
195     snapshotting bool
196
197     // This number is the number of pending or failed WriteSnaphot attempts since the last successful one.
198     snapshotAttempts int
199
200     stats         *CacheStatistics     // 各种统计信息
201     lastSnapshot  time.Time			   // 上一次snapshot时间
202     lastWriteTime time.Time
203
204     // A one time synchronization used to initial the cache with a store.  Since the store can allocate a
205     // a large amount memory across shards, we lazily create it.
206     initialize       atomic.Value
207     initializedCount uint32
208 }
```

其中，Cache.store的实际类型为ring, 默认大小为`const ringShards = 16`, paritions数量最大为16，ring的定义如下。

```
// file: tsdb/engine/tsm1/ring.go
 34 type ring struct {
 35     // Number of keys within the ring. This is used to provide a hint for
 36     // allocating the return values in keys(). It will not be perfectly accurate
 37     // since it doesn't consider adding duplicate keys, or trying to remove non-
 38     // existent keys.
 39     keysHint int64
 40
 41     // The unique set of partitions in the ring.
 42     // len(partitions) <= len(continuum)
 43     partitions []*partition		// 会根据int(xxhash.Sum64(key)%partitions)来取partition
 											// key 为 key(measurement,tag-set）+ #!~# + FieldKey
 44 }
 
219 // partition provides safe access to a map of series keys to entries.
220 type partition struct {
221     mu    sync.RWMutex
222     store map[string]*entry
223 }

// file: tsdb/engine/tsm1/cache.go
 35 // entry is a set of values and some metadata.
 36 type entry struct {
 37     mu     sync.RWMutex
 38     values Values // All stored values.
 39
 40     // The type of values stored. Read only so doesn't need to be protected by
 41     // mu.
 42     vtype byte
 43 }
```

Value是一个Interface，influxdb具体实现支持的类型包括：`IntegerValue`, `UnsignedValue`, `FloatValue`, `BooleanValue`, `StringValue`, `EmptyValue`，Value定义如下，具体实现类型的实现都可以在`tsdb/engine/tsm1/encoding.go`中找到，并且每一种具体类型都包含一个时间戳字段：`unixnano int64`。

```
// file: tsdb/engine/tsm1/encoding.go
  96 // Value represents a TSM-encoded value.
  97 type Value interface {
  98     // UnixNano returns the timestamp of the value in nanoseconds since unix epoch.
  99     UnixNano() int64
 100
 101     // Value returns the underlying value.
 102     Value() interface{}
 103
 104     // Size returns the number of bytes necessary to represent the value and its timestamp.
 105     Size() int
 106
 107     // String returns the string representation of the value and its timestamp.
 108     String() string
 109
 110     // internalOnly is unexported to ensure implementations of Value
 111     // can only originate in this package.
 112     internalOnly()
 113 }
```

整个Cache的内存布局总结如下图所示。

![Cache](https://raw.githubusercontent.com/GenuineJyn/GenuineJyn.github.io/master/pictures/influxdb/influxdb_cache.png)

## 2. Snapshot

Cache缓存的内存数据，能够提高查询效率，最终要持久化存储，也就是我们常说的快照落盘。

### 2.1 哪里启动的？什么条件下进行？

Snapshot调用链流程大致概括如下图所示。

![](https://raw.githubusercontent.com/GenuineJyn/GenuineJyn.github.io/master/pictures/influxdb/influxdb_cache_schedule.png)

其中，启动定时进行Cache snapshot的地方是在Engine.compactCache()，相关代码如下。

```
// file: tsdb/engine/tsm1/engine.go
1911 // compactCache continually checks if the WAL cache should be written to disk.
1912 func (e *Engine) compactCache() {
1913     t := time.NewTicker(time.Second)
1914     defer t.Stop()
1915     for {
1916         e.mu.RLock()
1917         quit := e.snapDone
1918         e.mu.RUnlock()
1919
1920         select {
1921         case <-quit:
1922             return
1923
1924         case <-t.C:
1925             e.Cache.UpdateAge()
1926             if e.ShouldCompactCache(time.Now()) {
1927                 start := time.Now()
1928                 e.traceLogger.Info("Compacting cache", zap.String("path", e.path))
1929                 err := e.WriteSnapshot()
1930                 if err != nil && err != errCompactionsDisabled {
1931                     e.logger.Info("Error writing snapshot", zap.Error(err))
1932                     atomic.AddInt64(&e.stats.CacheCompactionErrors, 1)
1933                 } else {
1934                     atomic.AddInt64(&e.stats.CacheCompactions, 1)
1935                 }
1936                 atomic.AddInt64(&e.stats.CacheCompactionDuration, time.Since(start).Nanoseconds())
1937             }
1938         }
1939     }
1940 }
```
Engine每秒钟判断一次是否进行Cache Snapshot，接下来看一下是否进行相关操作的判断：`e.ShouldCompactCache(time.Now())`.

```
// file: tsdb/engine/tsm1/engine.go
1942 // ShouldCompactCache returns true if the Cache is over its flush threshold
1943 // or if the passed in lastWriteTime is older than the write cold threshold.
1944 func (e *Engine) ShouldCompactCache(t time.Time) bool {
1945     sz := e.Cache.Size()
1946
1947     if sz == 0 {
1948         return false
1949     }
1950
1951     if sz > e.CacheFlushMemorySizeThreshold {
1952         return true
1953     }
1954
1955     return t.Sub(e.Cache.LastWriteTime()) > e.CacheFlushWriteColdDuration
1956 }
```
从代码上看两种条件满足一个即可：

* Cache大小超过阈值，CacheFlushMemorySizeThreshold有配置项`cache-snapshot-memory-size`设定，默认值为25MB;
* 距离上次snapshot时间超过一定阈值时间，`DefaultCacheSnapshotWriteColdDuration = time.Duration(10 * time.Minute)`

### 2.2 Cache Snapshot细节

顺着Engine.WriteSnapshot()看看Cache Snapshot的细节，主要代码如下。

```
1789 // WriteSnapshot will snapshot the cache and write a new TSM file with its contents, releasing the snapshot when done.
1790 func (e *Engine) WriteSnapshot() (err error) {
......
1796     defer func() {
1797         elapsed := time.Since(started)
1798         e.Cache.UpdateCompactTime(elapsed)
......
1804     }()
1805
1806     closedFiles, snapshot, err := func() (segments []string, snapshot *Cache, err error) {
1807         e.mu.Lock()
1808         defer e.mu.Unlock()
1809
1810         if e.WALEnabled {
1811             if err = e.WAL.CloseSegment(); err != nil {
1812                 return
1813             }
1814
1815             segments, err = e.WAL.ClosedSegments()
1816             if err != nil {
1817                 return
1818             }
1819         }
1820
1821         snapshot, err = e.Cache.Snapshot()
......
1837
1838     // The snapshotted cache may have duplicate points and unsorted data.  We need to deduplicate
1839     // it before writing the snapshot.  This can be very expensive so it's done while we are not
1840     // holding the engine write lock.
1841     dedup := time.Now()
1842     snapshot.Deduplicate()
......
1846
1847     return e.writeSnapshotAndCommit(log, closedFiles, snapshot)
1848 }
```
实现上很清楚：`Cache.Snapshot() -> snapshot.Deduplicate() -> writeSnapshotAndCommit->defer func() {e.Cache.UpdateCompactTime(elapsed)}`，顺着这个思路看看相关内容，最后落盘成功后会更新相关状态。

在Cache.Snapshot()中返回Cache中存储的数据，c.snapshot是一个指向Cache的指针, shore存储实际的数据，每次要Snapshot()把包含实际数据的*Cache传递出来，reset store继续存新的数据，这里 有点双buffer的意思，主要代码如下。

```
// file: tsdb/engine/tsm1/cache.go
407     c.snapshot.store, c.store = c.store, c.snapshot.store
......
415     // Reset the cache's store.
416     c.store.reset()
417     atomic.StoreUint64(&c.size, 0)
418     c.lastSnapshot = time.Now()
......
423     return c.snapshot, nil
```

接下来Deduplicate()去重，store的实际类型是ring（tsdb/engine/tsm1/ring.go, 具体数据结构进行参见上一节的介绍)，去重逻辑代码如下。

```
426 // Deduplicate sorts the snapshot before returning it. The compactor and any queries
427 // coming in while it writes will need the values sorted.
428 func (c *Cache) Deduplicate() {
429     c.mu.RLock()
430     store := c.store
431     c.mu.RUnlock()
432
433     // Apply a function that simply calls deduplicate on each entry in the ring.
434     // apply cannot return an error in this invocation.
435     _ = store.apply(func(_ []byte, e *entry) error { e.deduplicate(); return nil })
436 }
```

对store apply一个去重的closure：e.deduplicate(), deduplicate()对每个entry中的数据判断是否要进行排序，如果需要排序按照timestamp进行排序，有序之后很容去重，apply的主要代码如下，对每个partitions中的entry进行去重，特别要注意的是：`在去重的过程后才变的有序的`。

```
// file: tsdb/engine/tsm1/ring.go
145 func (r *ring) apply(f func([]byte, *entry) error) error {
......
152     for _, p := range r.partitions {
153         wg.Add(1)
154
155         go func(p *partition) {
156             defer wg.Done()
157
158             p.mu.RLock()
159             for k, e := range p.store {
160                 if err := f([]byte(k), e); err != nil {
161                     res <- err
162                     p.mu.RUnlock()
163                     return
164                 }
165             }
166             p.mu.RUnlock()
167         }(p)
168     }
169
170     go func() {
171         wg.Wait()
172         close(res)
173     }()
......
181     return nil
182 }
```

接下来看看如何把Cache落盘到tsm文件的。

```
1868 // writeSnapshotAndCommit will write the passed cache to a new TSM file and remove the closed WAL segments.
1869 func (e *Engine) writeSnapshotAndCommit(log *zap.Logger, closedFiles []string, snapshot *Cache) (err error) {
1870     defer func() {
1871         if err != nil {
1872             e.Cache.ClearSnapshot(false)
1873         }
1874     }()
1875
1876     // write the new snapshot files
1877     newFiles, err := e.Compactor.WriteSnapshot(snapshot)
1878     if err != nil {
1879         log.Info("Error writing snapshot from compactor", zap.Error(err))
1880         return err
1881     }
1882
1883     e.mu.RLock()
1884     defer e.mu.RUnlock()
1885
1886     // update the file store with these new files
1887     if err := e.FileStore.Replace(nil, newFiles); err != nil {
1888         log.Info("Error adding new TSM files from snapshot. Removing temp files.", zap.Error(err))
1889
1890         // Remove the new snapshot files. We will try again.
1891         for _, file := range newFiles {
1892             if err := os.Remove(file); err != nil {
1893                 log.Info("Unable to remove file", zap.String("path", file), zap.Error(err))
1894             }
1895         }
1896         return err
1897     }
1898
1899     // clear the snapshot from the in-memory cache, then the old WAL files
1900     e.Cache.ClearSnapshot(true)
1901
1902     if e.WALEnabled {
1903         if err := e.WAL.Remove(closedFiles); err != nil {
1904             log.Info("Error removing closed WAL segments", zap.Error(err))
1905         }
1906     }
1907
1908     return nil
1909 }
```
大致过程：把snapshot落盘tsm文件，更新新创建的文件到FileStore(tsm files维护)，清理之前的快照及恢复状态，因为已经落盘了WAL也没必要了。


### 2.3 snapshot数据落盘

#### 2.3.1 要并发，先把Cache切块

```
// file: tsdb/engine/tsm1/compact.go
 807 // WriteSnapshot writes a Cache snapshot to one or more new TSM files.
 808 func (c *Compactor) WriteSnapshot(cache *Cache) ([]string, error) {
......
 818     start := time.Now()
 819     card := cache.Count()
 820
 821     // Enable throttling if we have lower cardinality or snapshots are going fast.
 822     throttle := card < 3e6 && c.snapshotLatencies.avg() < 15*time.Second
 823
 824     // Write snapshost concurrently if cardinality is relatively high.
 825     concurrency := card / 2e6
 826     if concurrency < 1 {
 827         concurrency = 1
 828     }
 829
 830     // Special case very high cardinality, use max concurrency and don't throttle writes.
 831     if card >= 3e6 {
 832         concurrency = 4
 833         throttle = false
 834     }
 835
 836     splits := cache.Split(concurrency)
```
首先，获取cache缓存point的数量，如果数量比较少或者快照频率非常快，则要限流；如果数据量比较大，则高优先级的4并发不限流，最后把Cache切分成concurrency个小Cache, Split把16partition散列到不同的Cache的不同的位置.

#### 2.3.2 并发写，cacheKeyIterator登场

```
 843     resC := make(chan res, concurrency)
 844     for i := 0; i < concurrency; i++ {
 845         go func(sp *Cache) {
 846             iter := NewCacheKeyIterator(sp, tsdb.DefaultMaxPointsPerBlock, intC)
 847             files, err := c.writeNewFiles(c.FileStore.NextGeneration(), 0, nil, iter, throttle)
 848             resC <- res{files: files, err: err}
 849
 850         }(splits[i])
 851     }
```

这里的实现很巧妙，通过cacheKeyIterator在划分为大小1000个点的Block（partition中是一个map，每个桶上再分块，整体上就是一块一块的迭代器操作了），详细而具体探究这个巧妙的实现，先看一下cacheKeyIterator，定义如下。

```
// file: tsdb/engine/tsm1/compact.go
1862 type cacheKeyIterator struct {
1863     cache *Cache			// 迭代器作用的对象
1864     size  int				// block的大小：1000，迭代器每次操作的“步长”
1865     order [][]byte       // 存所有key值
1866
1867     i         int			// 游标
1868     blocks    [][]cacheBlock			//编码后的block具体数据，每个key一个[]cacheBlock
1869     ready     []chan struct{}
1870     interrupt chan struct{}
1871     err       error
1872 }
```
创建一个KeyIterator迭代器，cacheKeyIterator实现了这个interface

```
1881 // NewCacheKeyIterator returns a new KeyIterator from a Cache.
1882 func NewCacheKeyIterator(cache *Cache, size int, interrupt chan struct{}) KeyIterator {
1883     keys := cache.Keys()
1884
1885     chans := make([]chan struct{}, len(keys))
1886     for i := 0; i < len(keys); i++ {
1887         chans[i] = make(chan struct{}, 1)
1888     }
1889
1890     cki := &cacheKeyIterator{
1891         i:         -1,
1892         size:      size,
1893         cache:     cache,
1894         order:     keys,
1895         ready:     chans,
1896         blocks:    make([][]cacheBlock, len(keys)),
1897         interrupt: interrupt,
1898     }
1899     go cki.encode()
1900     return cki
1901 }
```
从上面blocks看每个key对应最后的压缩数据[]cacheBlock, 然后go出去到cki.encode(), cacheBlock定义如下。

```
1874 type cacheBlock struct {
1875     k                []byte
1876     minTime, maxTime int64
1877     b                []byte
1878     err              error
1879 }
```
在cki.encode()中完成数据的压缩到blocks，然后迭代器就可以一块一块的写到文件中了，Next()迭代器必备武器: i为游标（其实为-1很重要，先有Next()才能写第0块），阻塞在 <-c.ready[c.i]，数据encode完成才能继续，通过channel来保证同步的。

```
1993 func (c *cacheKeyIterator) Next() bool {
1994     if c.i >= 0 && c.i < len(c.ready) && len(c.blocks[c.i]) > 0 {
1995         c.blocks[c.i] = c.blocks[c.i][1:]
1996         if len(c.blocks[c.i]) > 0 {
1997             return true
1998         }
1999     }
2000     c.i++
2001
2002     if c.i >= len(c.ready) {
2003         return false
2004     }
2005
2006     <-c.ready[c.i]
2007     return true
2008 }
```

接下来cki.encode()怎么产生板砖的。

#### 2.3.3 cki.encode() 板砖的产生器

```
1911 func (c *cacheKeyIterator) encode() {
1912     concurrency := runtime.GOMAXPROCS(0)
1913     n := len(c.ready)
1914
1915     // Divide the keyset across each CPU
1916     chunkSize := 1
1917     idx := uint64(0)
1918
1919     for i := 0; i < concurrency; i++ {
1920         // Run one goroutine per CPU and encode a section of the key space concurrently
1921         go func() {
1922             tenc := getTimeEncoder(tsdb.DefaultMaxPointsPerBlock)
1923             fenc := getFloatEncoder(tsdb.DefaultMaxPointsPerBlock)
1924             benc := getBooleanEncoder(tsdb.DefaultMaxPointsPerBlock)
1925             uenc := getUnsignedEncoder(tsdb.DefaultMaxPointsPerBlock)
1926             senc := getStringEncoder(tsdb.DefaultMaxPointsPerBlock)
1927             ienc := getIntegerEncoder(tsdb.DefaultMaxPointsPerBlock)
1928
1929             defer putTimeEncoder(tenc)
1930             defer putFloatEncoder(fenc)
1931             defer putBooleanEncoder(benc)
1932             defer putUnsignedEncoder(uenc)
1933             defer putStringEncoder(senc)
1934             defer putIntegerEncoder(ienc)
1935
1936             for {
		             //很重要，隔离并发写，没有互斥的问题
1937                 i := int(atomic.AddUint64(&idx, uint64(chunkSize))) - chunkSize
1938
1939                 if i >= n {
1940                     break
1941                 }
1942
1943                 key := c.order[i]
1944                 values := c.cache.values(key)
1945
1946                 for len(values) > 0 {
1947
1948                     end := len(values)
1949                     if end > c.size {
1950                         end = c.size
1951                     }
1952
1953                     minTime, maxTime := values[0].UnixNano(), values[end-1].UnixNano()
1954                     var b []byte
1955                     var err error
1956
1957                     switch values[0].(type) {
1958                     case FloatValue:
1959                         b, err = encodeFloatBlockUsing(nil, values[:end], tenc, fenc)
1960                     case IntegerValue:
1961                         b, err = encodeIntegerBlockUsing(nil, values[:end], tenc, ienc)
1962                     case UnsignedValue:
1963                         b, err = encodeUnsignedBlockUsing(nil, values[:end], tenc, uenc)
1964                     case BooleanValue:
1965                         b, err = encodeBooleanBlockUsing(nil, values[:end], tenc, benc)
1966                     case StringValue:
1967                         b, err = encodeStringBlockUsing(nil, values[:end], tenc, senc)
1968                     default:
1969                         b, err = Values(values[:end]).Encode(nil)
1970                     }
1971
1972                     values = values[end:]
1973
1974                     c.blocks[i] = append(c.blocks[i], cacheBlock{
1975                         k:       key,
1976                         minTime: minTime,
1977                         maxTime: maxTime,
1978                         b:       b,
1979                         err:     err,
1980                     })
1981
1982                     if err != nil {
1983                         c.err = err
1984                     }
1985                 }
1986                 // Notify this key is fully encoded
1987                 c.ready[i] <- struct{}{}
1988             }
1989         }()
1990     }
1991 }
```
* 1. 先从不同类型的EncodePool中取encoder，用完放回去；
* 2. 从cache中根据key取数据size个（1000）数据，不同类型的用对应的encoder进行编码成一个cacheBlock，放到blocks对应的位置；
* 3. 迭代进行，直到把当前key的板砖都搬完；
* 4. c.ready[i] <- struct{}{}编码完成，迭代器才能Next()下去。

#### 2.3.4 encode，如何制造不同类型板砖的？

初始化的不同类型的EncoderPool，传递进去的一个closure，用于创建对应类型的Encoder，如下。

```
// file: tsdb/engine/tsm1/encoding.go
  61     timeEncoderPool = pool.NewGeneric(runtime.NumCPU(), func(sz int) interface{} {
  62         return NewTimeEncoder(sz)
  63     })
  64     integerEncoderPool = pool.NewGeneric(runtime.NumCPU(), func(sz int) interface{} {
  65         return NewIntegerEncoder(sz)
  66     })
  67     floatEncoderPool = pool.NewGeneric(runtime.NumCPU(), func(sz int) interface{} {
  68         return NewFloatEncoder()
  69     })
  70     stringEncoderPool = pool.NewGeneric(runtime.NumCPU(), func(sz int) interface{} {
  71         return NewStringEncoder(sz)
  72     })
  73     booleanEncoderPool = pool.NewGeneric(runtime.NumCPU(), func(sz int) interface{} {
  74         return NewBooleanEncoder(sz)
  75     })
```
探究一下实现，看到代码就一目了然了，代码如下。

```
// file: influxdb/pkg/pool/generic.go
  3 // Generic is a pool of types that can be re-used.  Items in
  4 // this pool will not be garbage collected when not in use.
  5 type Generic struct {
  6     pool chan interface{}
  7     fn   func(sz int) interface{}
  8 }
  9
 10 // NewGeneric returns a Generic pool with capacity for max items
 11 // to be pool.
 12 func NewGeneric(max int, fn func(sz int) interface{}) *Generic {
 13     return &Generic{
 14         pool: make(chan interface{}, max),
 15         fn:   fn,
 16     }
 17 }
 18
 19 // Get returns a item from the pool or a new instance if the pool
 20 // is empty.  Items returned may not be in the zero state and should
 21 // be reset by the caller.
 22 func (p *Generic) Get(sz int) interface{} {
 23     var c interface{}
 24     select {
 25     case c = <-p.pool:
 26     default:
 27         c = p.fn(sz)
 28     }
 29
 30     return c
 31 }
 32
 33 // Put returns an item back to the pool.  If the pool is full, the item
 34 // is discarded.
 35 func (p *Generic) Put(c interface{}) {
 36     select {
 37     case p.pool <- c:
 38     default:
 39     }
 40 }
```

通过runtime.NumCPU()个channel控制量，Get()调用closure创建对应类型的Encoder。本文前面提到influxdb支持类型：`IntegerValue`, `UnsignedValue`, `FloatValue`, `BooleanValue`, `StringValue`，每种类型中包含了时间戳，从EncoderPool看创建了对应的Encoder，包括TimerEncoder，integer和unsigned使用同一种InterEncoder。对应的Encoder实现可以在如下文件找到：

Encoder       | file
--------------|------------------------------
TimerEncoder  |tsdb/engine/tsm1/timestamp.go|
IntegerEncoder|tsdb/engine/tsm1/int.go      |
FloatEncoder  |tsdb/engine/tsm1/float.go    |
BooleanEncoder|tsdb/engine/tsm1/bool.go     |
StringEncoder |tsdb/engine/tsm1/string.go   |

decoder也能找到，这里不详细讲解encode/decode，不过influxdb考虑的压缩还是考虑非常细的。例如：按照timestamp是有序的，压缩的时候就从大到小算差值，这样压缩率就高了，如果是等差数列或者相等的值，采用RLE(Run Length Encoding)压缩，效率更高了，等差数列差值会变成[block] [block] [block] [block] [block]压缩完了就变成了[5][block]。再例如对整形的压缩，负数会采用ZigZag变成正整数（不然的话，负数采用补码方式来存）提升压缩率，这在protobuffer中也有，这里不详细介绍了，后续可以单独整理一遍文章。

#### 2.3.5 落盘 不同类型板砖开始搬到一起

这是我们TSM文件最后的格式, 上面介绍的迭代器就是造Blocks的材料：

```
+--------+------------------------------------+-------------+--------------+
| Header |               Blocks               |    Index    |    Footer    |
|5 bytes |              N bytes               |   N bytes   |   4 bytes    |
+--------+------------------------------------+-------------+--------------+
```


```
// file: tsdb/engine/tsm1/compact.go
 845         go func(sp *Cache) {
 846             iter := NewCacheKeyIterator(sp, tsdb.DefaultMaxPointsPerBlock, intC)
 847             files, err := c.writeNewFiles(c.FileStore.NextGeneration(), 0, nil, iter, throttle)
 848             resC <- res{files: files, err: err}
 849
 850         }(splits[i])
```

tsm文件命名是这个格式的：000000001-000000001.tsm，generation-sequence.tsm这种形式，tsm也会不断把小的compact成大的tsm（上图另一个分支），看看847行如何具体操作的。

```
1021 func (c *Compactor) writeNewFiles(generation, sequence int, src []string, iter KeyIterator, throttle      bool) ([]string, error) {
1022     // These are the new TSM files written
1023     var files []string
1024
1025     for {
1026         sequence++
1027
1028         // New TSM files are written to a temp file and renamed when fully completed.                1029         fileName := filepath.Join(c.Dir, c.formatFileName(generation, sequence)+"."+TSMFileExtension+     "."+TmpTSMFileExtension)
1030
1031         // Write as much as possible to this file
1032         err := c.write(fileName, iter, throttle)
1034         // We've hit the max file limit and there is more to write.  Create a new file
1035         // and continue.
1036         if err == errMaxFileExceeded || err == ErrMaxBlocksExceeded {
1037             files = append(files, fileName)
1038             continue
......
1064         files = append(files, fileName)
1065         break
1066     }
1067
1068     return files, nil
1069 }
```
先写到generation-sequence.tsm.tmp文件中，写满一个文件或者文件中的Blocks够了(代码1036行)，表示当前tsm文件写完了，再写下一个新的文件，直到把文件都写完了。

```
// file: tsdb/engine/tsm1/compact.go
1071 func (c *Compactor) write(path string, iter KeyIterator, throttle bool) (err error) {
1072     fd, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR|os.O_EXCL, 0666)
......
1095     // Use a disk based TSM buffer if it looks like we might create a big index
1096     // in memory.
1097     if iter.EstimatedIndexSize() > 64*1024*1024 {
1098         w, err = NewTSMWriterWithDiskBuffer(limitWriter)
1099         if err != nil {
1100             return err
1101         }
1102     } else {
1103         w, err = NewTSMWriter(limitWriter)
1104         if err != nil {
1105             return err
1106         }
1107     }
```
先预估一下Index的大小，如果过大则采用硬盘来缓存，不然直接在内存中操作，如果大量使用内存的话可能会增加内核OOME(OutOfMemoryException)风险。

```
1128     for iter.Next() {
......
1136         // Each call to read returns the next sorted key (or the prior one if there are
1137         // more values to write).  The size of values will be less than or equal to our
1138         // chunk size (1000)
1139         key, minTime, maxTime, block, err := iter.Read()
......
1148         // Write the key and value
1149         if err := w.WriteBlock(key, minTime, maxTime, block); err == ErrMaxBlocksExceeded {
1150             if err := w.WriteIndex(); err != nil {
1151                 return err
1152             }
1153             return err
......
1158         // If we have a max file size configured and we're over it, close out the file
1159         // and return the error.
1160         if w.Size() > maxTSMFileSize {
1161             if err := w.WriteIndex(); err != nil {
1162                 return err
1163             }
......
1174     // We're all done.  Close out the file.
1175     if err := w.WriteIndex(); err != nil {
1176         return err
1177     }
1178     return nil
1179 }
```
迭代器总算发挥作用了，不断的把block写到TSM文件中，最后写index，WriteBlock中会先写Header，
最后Blocks，Blocks布局如下图所示。

```
+--------------------------------------------------------------------+
│                           Blocks                                   │
+---------------------+-----------------------+----------------------+
|       Block 1       |        Block 2        |       Block N        |
+---------------------+-----------------------+----------------------+
|   CRC    |  Data    |    CRC    |   Data    |   CRC    |   Data    |
| 4 bytes  | N bytes  |  4 bytes  | N bytes   | 4 bytes  |  N bytes  |
+---------------------+-----------------------+----------------------+
```
WriteBlock写Block过程中，TSMWriter要不断的创建索引数据，内存索引或者磁盘索引，如果是磁盘索引会在文件系统中产生这样的临时文件`*.idx.tmp`。

```
// file: tsdb/engine/tsm1/writer.go
238 // NewIndexWriter returns a new IndexWriter.
239 func NewIndexWriter() IndexWriter {
240     buf := bytes.NewBuffer(make([]byte, 0, 1024*1024))
241     return &directIndex{buf: buf, w: bufio.NewWriter(buf)}
242 }
243
244 // NewIndexWriter returns a new IndexWriter.
245 func NewDiskIndexWriter(f *os.File) IndexWriter {
246     return &directIndex{fd: f, w: bufio.NewWriterSize(f, 1024*1024)}
247 }
```

WriteIndex最后Blocks后面追加Index数据。

```
706 // WriteIndex writes the index section of the file.  If there are no index entries to write,
707 // this returns ErrNoValues.
708 func (t *tsmWriter) WriteIndex() error {
709     indexPos := t.n
710
711     if t.index.KeyCount() == 0 {
712         return ErrNoValues
713     }
714
715     // Set the destination file on the index so we can periodically
716     // fsync while writing the index.
717     if f, ok := t.wrapped.(syncer); ok {
718         t.index.(*directIndex).f = f
719     }
720
721     // Write the index
722     if _, err := t.index.WriteTo(t.w); err != nil {
723         return err
724     }
725
726     var buf [8]byte
727     binary.BigEndian.PutUint64(buf[:], uint64(indexPos))
728
729     // Write the index index position
730     _, err := t.w.Write(buf[:])
731     return err
732 }
```
在写完Index后在最后添加一个8字节的Footer，用于表示index开始的位置。

```
+---------+
│ Footer  │
+---------+
│Index Ofs│
│ 8 bytes │
+---------+
```

这样TSM文件就写完了。

#### 2.3.6 directIndex/indirectIndex

> 本节把directIndex/indirectIndex内容详细描述一下

在写Block的过程伴随着索引的构建不断积累，写完Block后才开始写Index，实际类型是：`directIndex`, 存储布局如下所示，具体来看一下最后如何落盘的。

```
+-----------------------------------------------------------------------------+
│                                   Index                                     │
+-----------------------------------------------------------------------------+
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │ ...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │    │
+-----------------------------------------------------------------------------+
```

在WriteBlock的时候，不断记录Index的相关内容。

```
// file: tsdb/engine/tsm1/writer.go
685     // Record this block in index
686     t.index.Add(key, blockType, minTime, maxTime, t.n, uint32(n))
```
先看一下directIndex定义及相关数据结构。

```
// file: tsdb/engine/tsm1/writer.go
254 // directIndex is a simple in-memory index implementation for a TSM file.  The full index
255 // must fit in memory.
256 type directIndex struct {
257     keyCount int              // 记录key的数量
258     size     uint32           // 记录当前累计的索引大小，不包含type
259
260     // The bytes written count of when we last fsync'd
261     lastSync uint32
262     fd       *os.File         // DiskIndex        	
263     buf      *bytes.Buffer    // MemoryIndex
264
265     f syncer                  // 指向要写入的tsmfile，每写入25MB会执行Sync()一次
266
267     w *bufio.Writer           // wrap fd or buffer 的 Writer
268
269     key          []byte       // 每次缓存一个key的内容，新的key来了则会把上一个key的flush到内存或者disk
270     indexEntries *indexEntries // 当前key的索引数据
271 }

// file: tsdb/engine/tsm1/reader.go
1561 type indexEntries struct {
1562     Type    byte
1563     entries []IndexEntry
1564 }

// file: tsdb/engine/tsm1/writer.go
178 // IndexEntry is the index information for a given block in a TSM file.
179 type IndexEntry struct {
180     // The min and max time of all points stored in the block.
181     MinTime, MaxTime int64
182
183     // The absolute position in the file where this block is located.
184     Offset int64
185
186     // The size in bytes of the block in the file.
187     Size uint32
188 }
```

把索引的内容写到indexEntries中，知道新key到来，把数据flush内存或者disk，记录key的个数和数据size。
```
273 func (d *directIndex) Add(key []byte, blockType byte, minTime, maxTime int64, offset int64, size uint    32) {
274     // Is this the first block being added?
275     if len(d.key) == 0 {
276         // size of the key stored in the index
277         d.size += uint32(2 + len(key))
278         // size of the count of entries stored in the index
279         d.size += indexCountSize
280
281         d.key = key
282         if d.indexEntries == nil {
283             d.indexEntries = &indexEntries{}
284         }
285         d.indexEntries.Type = blockType
286         d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
287             MinTime: minTime,
288             MaxTime: maxTime,
289             Offset:  offset,
290             Size:    size,
291         })
292
293         // size of the encoded index entry
294         d.size += indexEntrySize
295         d.keyCount++
296         return
297     }
298
299     // See if were still adding to the same series key.
300     cmp := bytes.Compare(d.key, key)
301     if cmp == 0 {
302         // The last block is still this key
303         d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
304             MinTime: minTime,
305             MaxTime: maxTime,
306             Offset:  offset,
307             Size:    size,
308         })
309
310         // size of the encoded index entry
311         d.size += indexEntrySize
312
313     } else if cmp < 0 {
314         d.flush(d.w)
315         // We have a new key that is greater than the last one so we need to add
316         // a new index block section.
317
318         // size of the key stored in the index
319         d.size += uint32(2 + len(key))
320         // size of the count of entries stored in the index
321         d.size += indexCountSize
322
323         d.key = key
324         d.indexEntries.Type = blockType
325         d.indexEntries.entries = append(d.indexEntries.entries, IndexEntry{
326             MinTime: minTime,
327             MaxTime: maxTime,
328             Offset:  offset,
329             Size:    size,
330         })
331
332         // size of the encoded index entry
333         d.size += indexEntrySize
334         d.keyCount++
335     } else {
336         // Keys can't be added out of order.
337         panic(fmt.Sprintf("keys must be added in sorted order: %s < %s", string(key), string(d.key)))
338     }
339 }
```
flush刷新当前key的相关索引到内存或者disk，创建的内存原始大小是1M，超出由bytes.NewBuffer保证。

```
// file: tsdb/engine/tsm1/writer.go
706 // WriteIndex writes the index section of the file.  If there are no index entries to write,
707 // this returns ErrNoValues.
708 func (t *tsmWriter) WriteIndex() error {
709     indexPos := t.n
710
711     if t.index.KeyCount() == 0 {
712         return ErrNoValues
713     }
714
715     // Set the destination file on the index so we can periodically
716     // fsync while writing the index.
717     if f, ok := t.wrapped.(syncer); ok {
718         t.index.(*directIndex).f = f
719     }
720
721     // Write the index
722     if _, err := t.index.WriteTo(t.w); err != nil {
723         return err
724     }
725
726     var buf [8]byte
727     binary.BigEndian.PutUint64(buf[:], uint64(indexPos))
728
729     // Write the index index position
730     _, err := t.w.Write(buf[:])
731     return err
732 }

413 func (d *directIndex) WriteTo(w io.Writer) (int64, error) {
414     if _, err := d.flush(d.w); err != nil {
415         return 0, err
416     }
417
418     if err := d.w.Flush(); err != nil {
419         return 0, err
420     }
421
422     if d.fd == nil {
423         return copyBuffer(d.f, w, d.buf, nil)
424     }
425
426     if _, err := d.fd.Seek(0, io.SeekStart); err != nil {
427         return 0, err
428     }
429
430     return io.Copy(w, bufio.NewReaderSize(d.fd, 1024*1024))
431 }
```
只有Block写完了才能写index，因此最后把缓存在内存或者硬盘上的索引数据追加到TSM的Blocks后面，每25MB Sync()一次，到此index数据落盘完成。

最后TSM File落盘完了，要更新到FileStore供后续的查询使用，在这个过程中会创建indirectIndex用于提高查询效率，入口代码如下。

```
// file: tsdb/engine/tsm1/engine.go
1887     if err := e.FileStore.Replace(nil, newFiles); err != nil {
// file: tsdb/engine/tsm1/file_store.go
 763         tsm, err := NewTSMReader(fd, WithMadviseWillNeed(f.tsmMMAPWillNeed))

 233     t.accessor = &mmapAccessor{
 234         f:            f,
 235         mmapWillNeed: t.madviseWillNeed,
 236     }
 237
 238     index, err := t.accessor.init()
 239     if err != nil {
 240         return nil, err
 241     }
 242
 243     t.index = index
```
接下来从`t.accessor.init()`看一下indirectIndex如何构建的, 主要代码如下。

```
1315 func (m *mmapAccessor) init() (*indirectIndex, error) {
......
         //读Footer，找到索引开始的位置
1351     indexOfsPos := len(m.b) - 8
1352     indexStart := binary.BigEndian.Uint64(m.b[indexOfsPos : indexOfsPos+8])
1353     if indexStart >= uint64(indexOfsPos) {
1354         return nil, fmt.Errorf("mmapAccessor: invalid indexStart")
1355     }
1356
1357     m.index = NewIndirectIndex()
         // 构造IndirectIndex
1358     if err := m.index.UnmarshalBinary(m.b[indexStart:indexOfsPos]); err != nil {
1359         return nil, err
1360     }
1361
1362     // Allow resources to be freed immediately if requested
1363     m.incAccess()
1364     atomic.StoreUint64(&m.freeCount, 1)
1365
1366     return m.index, nil
}
```

```
tsdb/engine/tsm1/reader.go
 687 // indirectIndex is a TSMIndex that uses a raw byte slice representation of an index.  This
 688 // implementation can be used for indexes that may be MMAPed into memory.
 689 type indirectIndex struct {
 724     // b is the underlying index byte slice.  This could be a copy on the heap or an MMAP
 725     // slice reference
 726     b []byte    // 指向index的开始位置，通过与偏移量的配合取数据
 727
 728     // offsets contains the positions in b for each key.  It points to the 2 byte length of
 729     // key.
 730     offsets []byte
 731
 732     // minKey, maxKey are the minium and maximum (lexicographically sorted) contained in the
 733     // file
 734     minKey, maxKey []byte
 735
 736     // minTime, maxTime are the minimum and maximum times contained in the file across all
 737     // series.
 738     minTime, maxTime int64
 739
 740     // tombstones contains only the tombstoned keys with subset of time values deleted.  An
 741     // entry would exist here if a subset of the points for a key were deleted and the file
 742     // had not be re-compacted to remove the points on disk.
 743     tombstones map[string][]TimeRange
 744 }
 
 
 
1198 // UnmarshalBinary populates an index from an encoded byte slice
1199 // representation of an index.
1200 func (d *indirectIndex) UnmarshalBinary(b []byte) error {
1201     d.mu.Lock()
1202     defer d.mu.Unlock()
1203
1204     // Keep a reference to the actual index bytes
1205     d.b = b
......
1213     // To create our "indirect" index, we need to find the location of all the keys in
1214     // the raw byte slice.  The keys are listed once each (in sorted order).  Following
1215     // each key is a time ordered list of index entry blocks for that key.  The loop below
1216     // basically skips across the slice keeping track of the counter when we are at a key
1217     // field.
1218     var i int32
1219     var offsets []int32
1220     iMax := int32(len(b))
1221     for i < iMax {
             // 添加一个offset
1222         offsets = append(offsets, i)
1223
1224         // Skip to the start of the values
1225         // key length value (2) + type (1) + length of key
1226         if i+2 >= iMax {
1227             return fmt.Errorf("indirectIndex: not enough data for key length value")
1228         }
1229         i += 3 + int32(binary.BigEndian.Uint16(b[i:i+2]))
1230
1231         // count of index entries
1232         if i+indexCountSize >= iMax {
1233             return fmt.Errorf("indirectIndex: not enough data for index entries count")
1234         }
1235         count := int32(binary.BigEndian.Uint16(b[i : i+indexCountSize]))
1236         i += indexCountSize
1237
1238         // Find the min time for the block   找到当前block的MinTime
1239         if i+8 >= iMax {
1240             return fmt.Errorf("indirectIndex: not enough data for min time")
1241         }
1242         minT := int64(binary.BigEndian.Uint64(b[i : i+8]))
1243         if minT < minTime {
1244             minTime = minT
1245         }
1246
1247         i += (count - 1) * indexEntrySize
1248
1249         // Find the max time for the block   找到当前Block的MaxTime
1250         if i+16 >= iMax {
1251             return fmt.Errorf("indirectIndex: not enough data for max time")
1252         }
1253         maxT := int64(binary.BigEndian.Uint64(b[i+8 : i+16]))
1254         if maxT > maxTime {
1255             maxTime = maxT
1256         }
1257
1258         i += indexEntrySize
1259     }
1260
1261     firstOfs := offsets[0]
1262     _, key := readKey(b[firstOfs:]) 
1263     d.minKey = key
1264
1265     lastOfs := offsets[len(offsets)-1]
1266     _, key = readKey(b[lastOfs:])
1267     d.maxKey = key
1268
1269     d.minTime = minTime
1270     d.maxTime = maxTime
1271
1272     var err error   
         // mmap 申请内存放所有的offsets
1273     d.offsets, err = mmap(nil, 0, len(offsets)*4)
1274     if err != nil {
1275         return err
1276     }
1277     for i, v := range offsets {
1278         binary.BigEndian.PutUint32(d.offsets[i*4:i*4+4], uint32(v))
1279     }
1280
1281     return nil
1282 }
```

总结一下整个索引数据相关内容如下图。

![](https://raw.githubusercontent.com/GenuineJyn/GenuineJyn.github.io/master/pictures/influxdb/influxdb_index.png)

到此Cache所有内容详细的过了一遍了，从源码中可以学到很多东西，channel和迭代器的巧妙使用。



