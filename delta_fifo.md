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

// literalListerGetter实现了KeyListerGetter接口
type literalListerGetter func() []testFifoObject

var _ KeyListerGetter = literalListerGetter(nil)

func (kl literalListerGetter) ListKeys() []string {
    result := []string{}
    for _, fifoObj := range kl() {
        result = append(result, fifoObj.name)
    }
    return result
}

func (kl literalListerGetter) GetByKey(key string) (interface{}, bool, error) {
    for _, v := range kl() {
        if v.name == key {
            return v, true, nil
        }
    }
    return nil, false, nil
}

func TestDeltaFIFO_ReplaceMakesDeletions(t *testing.T) {
    f := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
        KeyFunction: testFifoObjectKeyFunc,
        KnownObjects: literalListerGetter(func() []testFifoObject {
            return []testFifoObject{mkFifoObj("foo", 5), mkFifoObj("bar", 6), mkFifoObj("baz", 7)}
        }),
    })
	// 删除
    f.Delete(mkFifoObj("baz", 10))
	// 替换，f.emitDeltaTypeReplaced为false时action为Sync，否则action为Replace
    f.Replace([]interface{}{mkFifoObj("foo", 5)}, "0")
    // 期望的列表
    expectedList := []Deltas{
        {{Deleted, mkFifoObj("baz", 10)}},
        {{Sync, mkFifoObj("foo", 5)}},
        {{Deleted, DeletedFinalStateUnknown{Key: "bar", Obj: mkFifoObj("bar", 6)}}},
    }
    
    for _, expected := range expectedList {
        cur := Pop(f).(Deltas)
        if e, a := expected, cur; !reflect.DeepEqual(e, a) {
            t.Errorf("Expected %#v, got %#v", e, a)
        }
    }
    
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
        {{Deleted, DeletedFinalStateUnknown{Key: "bar", Obj: mkFifoObj("bar", 6)}}},
    }
    
    for _, expected := range expectedList {
        cur := Pop(f).(Deltas)
        if e, a := expected, cur; !reflect.DeepEqual(e, a) {
            t.Errorf("Expected %#v, got %#v", e, a)
        }
    }
    
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
func NewDeltaFIFOWithOptions(opts DeltaFIFOOptions) *DeltaFIFO {
    if opts.KeyFunction == nil {
        opts.KeyFunction = MetaNamespaceKeyFunc
    }
    
    f := &DeltaFIFO{
        items:        map[string]Deltas{},
        queue:        []string{},
        keyFunc:      opts.KeyFunction,
        knownObjects: opts.KnownObjects,
        
        emitDeltaTypeReplaced: opts.EmitDeltaTypeReplaced,
    }
    f.cond.L = &f.lock
    return f
}
```
```go
// 计算obj对应的key
func (f *DeltaFIFO) KeyOf(obj interface{}) (string, error) {
	// 如果obj为Deltas类型
    if d, ok := obj.(Deltas); ok {
		// 如果没有值，抛err
        if len(d) == 0 {
            return "", KeyError{obj, ErrZeroLengthDeltasObject}
        }
		// 取最新的obj
        obj = d.Newest().Object
    }
	// 如果obj为DeletedFinalStateUnknown类型，则直接返回DeletedFinalStateUnknown.Key
    if d, ok := obj.(DeletedFinalStateUnknown); ok {
        return d.Key, nil
    }
	// 否则，使用keyFunc
    return f.keyFunc(obj)
}

func (d Deltas) Newest() *Delta {
    if n := len(d); n > 0 {
        return &d[n-1]
    }
    return nil
}

// Delete方法添加Deleted类型的Delta，如果f.knownObjects为nil并且obj不存在时，不做处理；如果f.knownObjects不为nil，且f.knownObjects.GetByKey(id)不存在并且f.items[id]不存在，不做处理
func (f *DeltaFIFO) Delete(obj interface{}) error {
	// 计算obj对应的key
    id, err := f.KeyOf(obj)
    if err != nil {
        return KeyError{obj, err}
    }
    f.lock.Lock()
    defer f.lock.Unlock()
    f.populated = true
    if f.knownObjects == nil {
		// 如果f.items不存在则不处理
        if _, exists := f.items[id]; !exists {
            return nil
        }
    } else {
        _, exists, err := f.knownObjects.GetByKey(id)
        _, itemsExist := f.items[id]
		// 如果f.knownObjects.GetByKey(id)和f.items[id]都不存在，则不处理
        if err == nil && !exists && !itemsExist {
            return nil
        }
    }
    // Deleted类型入队
    return f.queueActionLocked(Deleted, obj)
}

func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
    // 计算obj对应的key
    id, err := f.KeyOf(obj)
    if err != nil {
        return KeyError{obj, err}
    }
    oldDeltas := f.items[id]
    newDeltas := append(oldDeltas, Delta{actionType, obj})
    newDeltas = dedupDeltas(newDeltas)
    
    if len(newDeltas) > 0 {
        if _, exists := f.items[id]; !exists {
            f.queue = append(f.queue, id)
        }
        f.items[id] = newDeltas
        f.cond.Broadcast()
    } else {
        // This never happens, because dedupDeltas never returns an empty list
        // when given a non-empty list (as it is here).
        // If somehow it happens anyway, deal with it but complain.
        if oldDeltas == nil {
            klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
            return nil
        }
        klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
        f.items[id] = newDeltas
        return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
    }
    return nil
}

// re-listing and watching can deliver the same update multiple times in any
// order. This will combine the most recent two deltas if they are the same.
func dedupDeltas(deltas Deltas) Deltas {
    n := len(deltas)
    if n < 2 {
        return deltas
    }
    a := &deltas[n-1]
    b := &deltas[n-2]
    if out := isDup(a, b); out != nil {
        deltas[n-2] = *out
        return deltas[:n-1]
    }
    return deltas
}

// If a & b represent the same event, returns the delta that ought to be kept.
// Otherwise, returns nil.
// TODO: is there anything other than deletions that need deduping?
func isDup(a, b *Delta) *Delta {
    if out := isDeletionDup(a, b); out != nil {
        return out
    }
    // TODO: Detect other duplicate situations? Are there any?
    return nil
}

// keep the one with the most information if both are deletions.
func isDeletionDup(a, b *Delta) *Delta {
    if b.Type != Deleted || a.Type != Deleted {
        return nil
    }
    // Do more sophisticated checks, or is this sufficient?
    if _, ok := b.Object.(DeletedFinalStateUnknown); ok {
        return a
    }
    return b
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