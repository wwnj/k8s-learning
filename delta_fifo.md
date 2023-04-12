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
	// delete类型是否重复了
    newDeltas = dedupDeltas(newDeltas)
    
    if len(newDeltas) > 0 {
        if _, exists := f.items[id]; !exists {
            f.queue = append(f.queue, id)
        }
        f.items[id] = newDeltas
        f.cond.Broadcast()
    } else {
        // 正常情况，不应该走到这个分支
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

func isDup(a, b *Delta) *Delta {
	// 是否删除类型重复
    if out := isDeletionDup(a, b); out != nil {
        return out
    }
    return nil
}

func isDeletionDup(a, b *Delta) *Delta {
    if b.Type != Deleted || a.Type != Deleted {
        return nil
    }
    // 都为delete类型，并且b.Object是DeletedFinalStateUnknown类型，则保留a，否则保留b
    if _, ok := b.Object.(DeletedFinalStateUnknown); ok {
        return a
    }
    return b
}
```
```go
// Replace逻辑如下: (1) 添加Sync或Replace Delta类型对象
// (2) 删除操作：对于每个已经存在的keys，但不存在于list中的对象，添加Delete(DeletedFinalStateUnknown{K, O})对象，其中O是K关联的对象；
// 如果f.knownObjects为空， 已经存在的keys是f.items，O是K关联的Deltas.Newest()；
// 如果f.knownObjects不为空，已经存在的keys是f.knownObjects，O是f.knownObjects.GetByKey(K)的返回值
func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
    f.lock.Lock()
    defer f.lock.Unlock()
    keys := make(sets.String, len(list))
    
    // 兼容老版本的客户端
    action := Sync
    if f.emitDeltaTypeReplaced {
        action = Replaced
    }
	
    for _, item := range list {
        key, err := f.KeyOf(item)
        if err != nil {
            return KeyError{item, err}
        }
        keys.Insert(key)
		// 每个list中的item添加Sync/Replaced类型
        if err := f.queueActionLocked(action, item); err != nil {
            return fmt.Errorf("couldn't enqueue object: %v", err)
        }
    }
    
    if f.knownObjects == nil {
        // Do deletion detection against our own list.
        queuedDeletions := 0
        for k, oldItem := range f.items {
            if keys.Has(k) {
                continue
            }

            var deletedObj interface{}
			// 取最新的一个obj
            if n := oldItem.Newest(); n != nil {
                deletedObj = n.Object
            }
            queuedDeletions++
            if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
                return err
            }
        }
    
        if !f.populated {
            f.populated = true
            f.initialPopulationCount = keys.Len() + queuedDeletions
        }
        
        return nil
    }
    
    knownKeys := f.knownObjects.ListKeys()
    queuedDeletions := 0
    for _, k := range knownKeys {
        if keys.Has(k) {
        continue
    }
	// 取f.knownObjects.GetByKey的返回值
    deletedObj, exists, err := f.knownObjects.GetByKey(k)
    if err != nil {
        deletedObj = nil
        klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
    } else if !exists {
        deletedObj = nil
        klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
    }
    queuedDeletions++
    if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
        return err
        }
    }
    
    if !f.populated {
        f.populated = true
        f.initialPopulationCount = keys.Len() + queuedDeletions
    }
    
    return nil
}
```
```go
Add、Pop
```
## 4.总结
kubernetes fifo在实现先入先出队列上，值得我们学习借鉴