# kubernetes delta_fifo源码解析
# 1.介绍
kubernetes delta_fifo是一个先入先出队列，相较于fifo，有两点不同：
- 与key相关联的不直接是obj，而是Deltas，它是一个切片，Delta不仅包含了obj，还包含了DeltaType
- 当Deltas最后一个元素Delta.DeltaType已经是Deleted类型时，再添加一个Deleted类型的Delta，Deltas不再新增 
delta_fifo的API与fifo类型，不再具体分析
## 2.使用
参考[TestDeltaFIFO_ReplaceMakesDeletions](https://github.com/kubernetes/kubernetes/blob/v1.26.3/staging/src/k8s.io/client-go/tools/cache/delta_fifo_test.go#L333)
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

// literalListerGetter is a KeyListerGetter that is based on a
// function that returns a slice of objects to list and get.
// The function must list the same objects every time.
type literalListerGetter func() []testFifoObject

var _ KeyListerGetter = literalListerGetter(nil)

// ListKeys just calls kl.
func (kl literalListerGetter) ListKeys() []string {
    result := []string{}
    for _, fifoObj := range kl() {
        result = append(result, fifoObj.name)
    }
    return result
}

// GetByKey returns the key if it exists in the list returned by kl.
func (kl literalListerGetter) GetByKey(key string) (interface{}, bool, error) {
    for _, v := range kl() {
        if v.name == key {
            return v, true, nil
        }
    }
    return nil, false, nil
}

func TestDeltaFIFO_ReplaceMakesDeletions(t *testing.T) {
    // We test with only one pre-existing object because there is no
    // promise about how their deletes are ordered.
    
    // Try it with a pre-existing Delete
    f := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
        KeyFunction: testFifoObjectKeyFunc,
        KnownObjects: literalListerGetter(func() []testFifoObject {
            return []testFifoObject{mkFifoObj("foo", 5), mkFifoObj("bar", 6), mkFifoObj("baz", 7)}
        }),
    })
    f.Delete(mkFifoObj("baz", 10))
    f.Replace([]interface{}{mkFifoObj("foo", 5)}, "0")
    
    expectedList := []Deltas{
        {{Deleted, mkFifoObj("baz", 10)}},
        {{Sync, mkFifoObj("foo", 5)}},
        // Since "bar" didn't have a delete event and wasn't in the Replace list
        // it should get a tombstone key with the right Obj.
        {{Deleted, DeletedFinalStateUnknown{Key: "bar", Obj: mkFifoObj("bar", 6)}}},
    }
    
    for _, expected := range expectedList {
        cur := Pop(f).(Deltas)
        if e, a := expected, cur; !reflect.DeepEqual(e, a) {
            t.Errorf("Expected %#v, got %#v", e, a)
        }
    }
    
    // Now try starting with an Add instead of a Delete
    f = NewDeltaFIFOWithOptions(DeltaFIFOOptions{
    KeyFunction: testFifoObjectKeyFunc,
    KnownObjects: literalListerGetter(func() []testFifoObject {
            return []testFifoObject{mkFifoObj("foo", 5), mkFifoObj("bar", 6), mkFifoObj("baz", 7)}
        }),
    })
    f.Add(mkFifoObj("baz", 10))
    f.Replace([]interface{}{mkFifoObj("foo", 5)}, "0")
    
    expectedList = []Deltas{
        {{Added, mkFifoObj("baz", 10)},
        {Deleted, DeletedFinalStateUnknown{Key: "baz", Obj: mkFifoObj("baz", 7)}}},
        {{Sync, mkFifoObj("foo", 5)}},
        // Since "bar" didn't have a delete event and wasn't in the Replace list
        // it should get a tombstone key with the right Obj.
        {{Deleted, DeletedFinalStateUnknown{Key: "bar", Obj: mkFifoObj("bar", 6)}}},
    }
    
    for _, expected := range expectedList {
        cur := Pop(f).(Deltas)
        if e, a := expected, cur; !reflect.DeepEqual(e, a) {
            t.Errorf("Expected %#v, got %#v", e, a)
        }
    }
    
    // Now try starting without an explicit KeyListerGetter
    f = NewDeltaFIFOWithOptions(DeltaFIFOOptions{KeyFunction: testFifoObjectKeyFunc})
    f.Add(mkFifoObj("baz", 10))
    f.Replace([]interface{}{mkFifoObj("foo", 5)}, "0")
    
    expectedList = []Deltas{
        {{Added, mkFifoObj("baz", 10)},
        {Deleted, DeletedFinalStateUnknown{Key: "baz", Obj: mkFifoObj("baz", 10)}}},
        {{Sync, mkFifoObj("foo", 5)}},
    }
    
    for _, expected := range expectedList {
        cur := Pop(f).(Deltas)
        if e, a := expected, cur; !reflect.DeepEqual(e, a) {
            t.Errorf("Expected %#v, got %#v", e, a)
        }
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