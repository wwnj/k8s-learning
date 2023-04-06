# kubernetes indexer分析
kubernetes indexer是实现了多索引的本地缓存
## 1.背景
以db中学校学生表做本地缓存为例
```go
type Student struct {
	ID uint32
	Name string
	Class uint32
}

// 本地缓存，id和学生的映射
var idCache = map[uint32]Student{}
```
当需要以学生名来取值时，此时没有合适的缓存，可以再添加一份
```go
// 本地缓存，名称和学生的映射
var idCache = map[string]Student{}
```
但这样缓存就存了两份，浪费了内存。可以维护一个学生名和id的索引，在拖过id的缓存取值即可
```go
// 索引，名称和id的映射
var index = map[string]id{}
```
这样要通过名称查缓存，通过idCache[index[name]]即可。kubernetes indexer就是类似的思路，indexer支持任意类型，线程安全
## 2.使用
参考[TestMultiIndexKeys单侧](https://github.com/kubernetes/kubernetes/blob/v1.26.3/staging/src/k8s.io/client-go/tools/cache/index_test.go)
```go
// 取pod中标签foo的值作为索引值
func testIndexFunc(obj interface{}) ([]string, error) {
    pod := obj.(*v1.Pod)
    return []string{pod.Labels["foo"]}, nil
}

func TestMultiIndexKeys(t *testing.T) {
	// 创建索引器
	index := NewIndexer(MetaNamespaceKeyFunc, Indexers{"byUser": testUsersIndexFunc})

	pod1 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "one", Annotations: map[string]string{"users": "ernie,bert"}}}
	pod2 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "two", Annotations: map[string]string{"users": "bert,oscar"}}}
	pod3 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "tre", Annotations: map[string]string{"users": "ernie,elmo"}}}
    // 添加索引
	index.Add(pod1)
	index.Add(pod2)
	index.Add(pod3)

	expected := map[string]sets.String{}
	expected["ernie"] = sets.NewString("one", "tre")
	expected["bert"] = sets.NewString("one", "two")
	expected["elmo"] = sets.NewString("tre")
	expected["oscar"] = sets.NewString("two")
	expected["elmo1"] = sets.NewString()
	{
		for k, v := range expected {
			found := sets.String{}
			// 根据索引名+缓存key获取索引结果
			indexResults, err := index.ByIndex("byUser", k)
			if err != nil {
				t.Errorf("Unexpected error %v", err)
			}
			for _, item := range indexResults {
				found.Insert(item.(*v1.Pod).Name)
			}
			if !found.Equal(v) {
				t.Errorf("missing items, index %s, expected %v but found %v", k, v.List(), found.List())
			}
		}
	}
    // 验证删除索引
	index.Delete(pod3)
	erniePods, err := index.ByIndex("byUser", "ernie")
	if err != nil {
		t.Errorf("unexpected error: %v", err)
	}
	if len(erniePods) != 1 {
		t.Errorf("Expected 1 pods but got %v", len(erniePods))
	}
	for _, erniePod := range erniePods {
		if erniePod.(*v1.Pod).Name != "one" {
			t.Errorf("Expected only 'one' but got %s", erniePod.(*v1.Pod).Name)
		}
	}

	elmoPods, err := index.ByIndex("byUser", "elmo")
	if err != nil {
		t.Errorf("unexpected error: %v", err)
	}
	if len(elmoPods) != 0 {
		t.Errorf("Expected 0 pods but got %v", len(elmoPods))
	}

	copyOfPod2 := pod2.DeepCopy()
	copyOfPod2.Annotations["users"] = "oscar"
	// 验证更新索引
	index.Update(copyOfPod2)
	bertPods, err := index.ByIndex("byUser", "bert")
	if err != nil {
		t.Errorf("unexpected error: %v", err)
	}
	if len(bertPods) != 1 {
		t.Errorf("Expected 1 pods but got %v", len(bertPods))
	}
	for _, bertPod := range bertPods {
		if bertPod.(*v1.Pod).Name != "one" {
			t.Errorf("Expected only 'one' but got %s", bertPod.(*v1.Pod).Name)
		}
	}
}
```
## 3.源码分析
```go
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		// 线程安全的存储
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		// 缓存key计算函数
		keyFunc:      keyFunc,
	}
}
```
```go
// 线程安全的存储
func NewThreadSafeStore(indexers Indexers, indices Indices) ThreadSafeStore {
	return &threadSafeMap{
		// 缓存
		items: map[string]interface{}{},
		index: &storeIndex{
			// Indexers是map[string]IndexFunc类型，保存索引名与索引值计算函数的映射
			indexers: indexers,
			// Indices是map[string]Index类型，通过索引名找到Index，在通过索引值找到缓存key
			indices:  indices,
		},
	}
}
```
```go
func (c *cache) Add(obj interface{}) error {
	// 计算缓存key
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Add(key, obj)
	return nil
}
```
```go
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

func (c *threadSafeMap) Update(key string, obj interface{}) {
	// 更新操作，加锁
    c.lock.Lock()
    defer c.lock.Unlock()
	// 获取旧缓存对象
    oldObject := c.items[key]
	// 设置新缓存对象
    c.items[key] = obj
	// 更新索引
    c.index.updateIndices(oldObject, obj, key)
}
```
```go
func (i *storeIndex) updateIndices(oldObj interface{}, newObj interface{}, key string) {
	var oldIndexValues, indexValues []string
	var err error
	// 遍历所有索引器
	for name, indexFunc := range i.indexers {
		// 在缓存对象第一次添加的时候，oldObj为nil
		if oldObj != nil {
			// 计算旧的索引值
			oldIndexValues, err = indexFunc(oldObj)
		} else {
			oldIndexValues = oldIndexValues[:0]
		}
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}

		if newObj != nil {
            // 计算新的索引值
			indexValues, err = indexFunc(newObj)
		} else {
			indexValues = indexValues[:0]
		}
		if err != nil {
			panic(fmt.Errorf("unable to calculate an index entry for key %q on index %q: %v", key, name, err))
		}
        // 通过索引名找到索引，为空则创建
		index := i.indices[name]
		if index == nil {
			index = Index{}
			i.indices[name] = index
		}
        // 一个小优化，当添加的新值和旧值都为1并且相同时，无需处理
		if len(indexValues) == 1 && len(oldIndexValues) == 1 && indexValues[0] == oldIndexValues[0] {
			// We optimize for the most common case where indexFunc returns a single value which has not been changed
			continue
		}
        // 从索引中删除key
		for _, value := range oldIndexValues {
			i.deleteKeyFromIndex(key, value, index)
		}
        // 从索引中添加key
		for _, value := range indexValues {
			i.addKeyToIndex(key, value, index)
		}
	}
}

func (i *storeIndex) addKeyToIndex(key, indexValue string, index Index) {
    set := index[indexValue]
    if set == nil {
        set = sets.String{}
        index[indexValue] = set
    }
	// 索引名+索引值可能对应多个缓存key，通过set去重
    set.Insert(key)
}

func (i *storeIndex) deleteKeyFromIndex(key, indexValue string, index Index) {
    set := index[indexValue]
    if set == nil {
        return
    }
    // 删除缓存key
    set.Delete(key)
    // 当缓存key的set为空时，删除索引值的映射，避免oom
    if len(set) == 0 {
        delete(index, indexValue)
    }
}
```