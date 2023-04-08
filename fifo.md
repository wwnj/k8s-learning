# kubernetes fifo源码解析
# 1.介绍
kubernetes fifo是一个先入先出队列，实现了Add、Update、Delete、Get、Pop等基本API，以及Replace、HasSynced等API，具体如下：
```go
type FIFO struct {
	lock sync.RWMutex
	cond sync.Cond
	// key和obj的映射
	items map[string]interface{}
	// key的队列，去重
	queue []string

	// 当Delete/Add/Update被首先调用，或Replace()的items全部被pop时populated为true
	populated bool
	// Replace()首先被调用时的objs的数量
	initialPopulationCount int

	// keyFunc是用来将obj生成key的
	keyFunc KeyFunc

	// 队列是否关闭，用在Pop方法内的循环控制中
	closed bool
}
```
> func NewFIFO(keyFunc KeyFunc) *FIFO
>> 创建一个先入先出队列
> 
> func (f *FIFO) Add(obj interface{}) error 
>> 添加一个obj，当f.queue中已存在对应的key时，f.queue不再添加
> 
> func (f *FIFO) AddIfNotPresent(obj interface{}) error
>> 当f.items不存在obj对应的key时才添加，这在单一生产者/消费者有用，消费者可以安全的重试，避免与生产者竞争以及重入队已消费的item
> 
> func (f *FIFO) Close()
>> 关闭队列
> 
> func (f *FIFO) Delete(obj interface{}) error
>> 删除不存在f.queue中的item，因为这个实现假设使用者只关心对象，而不关心创建/添加对象的顺序
> 
> func (f *FIFO) Get(obj interface{}) (item interface{}, exists bool, err error)
>> 返回请求的item，不存在时exists为false
> 
> func (f *FIFO) GetByKey(key string) (item interface{}, exists bool, err error)
>> 返回请求的item，不存在时exists为false 
> 
> func (f *FIFO) HasSynced() bool
>> 当Add/Update/Delete/AddIfNotPresent先被调用，或者先被Replace()插入的items都被Pop时，HasSynced返回true
> 
> func (f *FIFO) IsClosed() bool
>> 检车队列是否关闭
> 
> func (f *FIFO) List() []interface{}
>> 返回所有items.
> 
> func (f *FIFO) ListKeys() []string 
>> 返回当前FIFO中所有的key
> 
> func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error)
>> Pop会等到f.queue中有对象，并且会调用PopProcessFunc处理item。如果f.queue中有多个待处理的对象，则将按照Add/Update的顺序返回。在调用PopProcessFunc之前，会从队列(和存储)中删除item。如果PopProcessFunc返回ErrRequeue，会使用AddIfNotPresent()将其添加回来，因此保证可重复消费。PopProcessFunc是在锁定状态下调用的，因此在PopProcessFunc中操作FIFO的数据结构是安全的。
> 
> func (f *FIFO) Replace(list []interface{}, resourceVersion string) error
>> 会根据list重新生成一个map，并将f.items指向新的map，依据该map重新入队f.queue，所以f.queue是无序的 
> 
> func (f *FIFO) Resync() error
>> Resync会保证f.items中的key全部存在f.queue中，一般不应该调用该方法，因为其他api应当维持关联关系
> 
> func (f *FIFO) Update(obj interface{}) error
>> 与Add实现一致
## 2.使用
参考[TestFIFO_requeueOnPop](https://github.com/kubernetes/kubernetes/blob/v1.26.3/staging/src/k8s.io/client-go/tools/cache/fifo_test.go#L75)
```go
// 取testFifoObject中name作为key
func testFifoObjectKeyFunc(obj interface{}) (string, error) {
    return obj.(testFifoObject).name, nil
}

type testFifoObject struct {
    name string
    val  interface{}
}

func mkFifoObj(name string, val interface{}) testFifoObject {
    return testFifoObject{name: name, val: val}
}

func TestFIFO_requeueOnPop(t *testing.T) {
	// 创建FIFO实例
    f := NewFIFO(testFifoObjectKeyFunc)
    // 添加obj
    f.Add(mkFifoObj("foo", 10))
	// Pop操作，但返回ErrRequeue，这时会重入队
    _, err := f.Pop(func(obj interface{}) error {
        if obj.(testFifoObject).name != "foo" {
            t.Fatalf("unexpected object: %#v", obj)
        }
        return ErrRequeue{Err: nil}
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
	// GetByKey，还在队列中
    if _, ok, err := f.GetByKey("foo"); !ok || err != nil {
        t.Fatalf("object should have been requeued: %t %v", ok, err)
    }
    
    _, err = f.Pop(func(obj interface{}) error {
        if obj.(testFifoObject).name != "foo" {
            t.Fatalf("unexpected object: %#v", obj)
        }
        return ErrRequeue{Err: fmt.Errorf("test error")}
    })
    if err == nil || err.Error() != "test error" {
        t.Fatalf("unexpected error: %v", err)
    }
    if _, ok, err := f.GetByKey("foo"); !ok || err != nil {
        t.Fatalf("object should have been requeued: %t %v", ok, err)
    }
    // Pop操作，返回nil，不在队列中了
    _, err = f.Pop(func(obj interface{}) error {
        if obj.(testFifoObject).name != "foo" {
            t.Fatalf("unexpected object: %#v", obj)
        }
        return nil
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
	// GetByKey，不在队列中
    if _, ok, err := f.GetByKey("foo"); ok || err != nil {
        t.Fatalf("object should have been removed: %t %v", ok, err)
    }
}
```
## 3.源码解析
```go
func NewFIFO(keyFunc KeyFunc) *FIFO {
    f := &FIFO{
		// key和obj的映射
        items:   map[string]interface{}{},
		// key的队列，先入先出
        queue:   []string{},
		// obj和key的映射函数
        keyFunc: keyFunc,
    }
	// f.cond.L持有f.lock
    f.cond.L = &f.lock
return f
}
```
```go
func (f *FIFO) Add(obj interface{}) error {
    id, err := f.keyFunc(obj)
    if err != nil {
        return KeyError{obj, err}
    }
    f.lock.Lock()
    defer f.lock.Unlock()
    f.populated = true
	// items中不存在时，才入队
    if _, exists := f.items[id]; !exists {
        f.queue = append(f.queue, id)
    }
    f.items[id] = obj
	// 唤醒所有等待在f.cond的协程，其实就是Pop在等待f.cond
    f.cond.Broadcast()
    return nil
}

```
```go
func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error) {
    f.lock.Lock()
    defer f.lock.Unlock()
    for {
        for len(f.queue) == 0 {
            // 当队列为空时, 避免只有item入队时Pop才可以退出；当f.Close()调用时，Pop也可以退出
            if f.closed {
                return nil, ErrFIFOClosed
            }
            // 等待条件变量唤醒
            f.cond.Wait()
        }
		// 从对头取，先入先出
        id := f.queue[0]
        f.queue = f.queue[1:]
		// 当Replace先被调用时，initialPopulationCount才可能大于0
        if f.initialPopulationCount > 0 {
            f.initialPopulationCount--
        }
        item, ok := f.items[id]
        if !ok {
            // item有可能随后被删除，当被删除时不进行后续操作
            continue
        }
		// 删除item
        delete(f.items, id)
		// 调用item处理函数，如果返回ErrRequeue时，重入队，以便重复消费
        err := process(item)
        if e, ok := err.(ErrRequeue); ok {
            f.addIfNotPresent(id, item)
            err = e.Err
        }
        return item, err
    }
}
```
```go
func (f *FIFO) addIfNotPresent(id string, obj interface{}) {
    f.populated = true
    if _, exists := f.items[id]; exists {
        return
    }
    
    f.queue = append(f.queue, id)
    f.items[id] = obj
    f.cond.Broadcast()
}
```
## 4.总结
kubernetes fifo在实现先入先出队列上，值得我们学习借鉴