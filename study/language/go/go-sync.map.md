###  sync.Map 源码走读
````
syncMap是一个并发安全的map
1. 那么它的底层代码是怎么实现的? 
2. 相比读写锁+map，它有什么优势呢?  
带着这样的问题，我们一起来看一下sync.Map的底层实现
````
#### 相关结构体
````
type Map struct {
	mu Mutex //dirty的锁 
	read atomic.Value // 值存在的地方 , 读取它的值是线程安全的，但是写入必须要借助锁。存储在read中的值可能是无锁的(map中存在并且未删除)，在dirty中删除过的数据一定要同步过去。
	dirty map[interface{}]*entry //写入map
	misses int //map key miss的次数
}

 //原子的存读map的地方
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // dirty是否存在m中没有的值
}


var expunged = unsafe.Pointer(new(interface{})) //r

type entry struct {
	//存值
	p unsafe.Pointer // *interface{}
}
````

store:
````
func (m *Map) Store(key, value interface{}) {
	read, _ := m.read.Load().(readOnly) //拿到对应的map
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
````

load
````

````