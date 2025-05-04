
![optimized-lfu-cache-flowchart.svg](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/optimized-lfu-cache-flowchart.svg)

## 1. `min_freq` - 最小频率

### 访问时机

- 在`put`方法中处理缓存已满情况时，用于确定哪个频率链表中的节点应被淘汰
- 在`get_node`方法中，需要决定是否更新最小频率时

### 更新时机

- **初始化**：在构造函数中初始化为0（或1，取决于实现）
- **增加**：在`get_node`方法中，当一个频率为`min_freq`的节点的频率增加后，如果该频率链表变为空，则`min_freq++`
- **重置**：在`put`方法中添加新节点时，将`min_freq`设为1，因为新节点的初始频率为1

### 更新示例

```cpp
// get_node方法中，当频率为min_freq的列表变空时
if (dummy->prev == dummy) { // 链表为空
    freq_to_dummy.erase(node->freq);
    delete dummy;
    if (min_freq == node->freq) {
        min_freq++; // 增加min_freq
    }
}

// put方法中，添加新节点时
key_to_node[key] = node = new Node(key, val);
push_front(1, node);
min_freq = 1; // 重置min_freq
```

## 2. `capacity` - 缓存容量

### 访问时机

- 在`put`方法中，决定是否需要淘汰节点前检查当前节点数是否达到容量
- 在`get`和`put`方法开始时，检查容量是否大于0

### 更新时机

- **初始化**：仅在构造函数中设置，之后不再更改
- 该变量通常是只读的，缓存容量在创建后不会改变

### 访问示例

```cpp
// 检查容量是否为正数
if (capacity <= 0) return -1; // 或 return;

// 检查是否达到容量上限
if (key_to_node.size() == capacity) {
    // 执行淘汰逻辑
}
```

## 3. `key_to_node` - 键到节点的映射

### 访问时机

- 在`get_node`方法中，查找指定key对应的节点
- 在`put`方法中，检查key是否已存在
- 在`put`方法中淘汰节点时，从映射中移除被淘汰节点的key

### 更新时机

- **插入**：在`put`方法中添加新节点时
- **删除**：在`put`方法中淘汰节点时
- **注意**：`get`方法只读取不修改此映射

### 更新示例

```cpp
// 查找节点
auto it = key_to_node.find(key);
if (it == key_to_node.end()) {
    return nullptr;
}
Node* node = it->second;

// 添加新节点
key_to_node[key] = node = new Node(key, val);

// 删除节点
key_to_node.erase(back_node->key);
```

## 4. `freq_to_dummy` - 频率到链表哑节点的映射

### 访问时机

- 在`get_node`方法中，获取节点当前频率对应的链表头
- 在`push_front`方法中，获取指定频率的链表头
- 在`put`方法中淘汰节点时，获取`min_freq`对应的链表头

### 更新时机

- **插入**：在`push_front`方法中发现指定频率的链表不存在时，创建新链表
- **删除**：在`get_node`方法中，当频率链表变为空时，删除该频率的映射
- **删除**：在`put`方法淘汰节点后，如果频率链表变为空，删除该频率的映射

### 更新示例

```cpp
// 创建新频率链表
auto it = freq_to_dummy.find(freq);
if (it == freq_to_dummy.end()) {
    it = freq_to_dummy.emplace(freq, new_list()).first;
}

// 删除空链表的映射
if (dummy->prev == dummy) { // 链表为空
    freq_to_dummy.erase(node->freq);
    delete dummy;
}
```

## 变量访问的并发安全考虑

在多线程环境中，这些变量的访问和更新需要特别注意：

1. **锁的粒度**：
    
    - 在Go实现中，我们使用单个读写锁保护所有变量
    - 对于高性能需求，可以考虑更细粒度的锁策略
2. **读写分离**：
    
    - `get`操作主要是读操作，但会修改节点频率和链表结构
    - `put`操作既读又写
    - 可以使用读写锁提高并发性能
3. **原子性**：
    
    - 多个变量的更新需要保持原子性和一致性
    - 例如，从一个频率链表移动到另一个频率链表的过程必须是原子的

## 变量操作时序图

![lfu-variable-flowchart.svg](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/lfu-variable-flowchart.svg)
```
【GET操作】
1. 访问 key_to_node 查找节点
2. 如果找到节点:
   a. 从原频率链表移除节点
   b. 访问并可能更新 freq_to_dummy
   c. 检查并可能更新 min_freq
   d. 增加节点频率并添加到新频率链表
   e. 返回节点值

【PUT操作 - 更新已有节点】
1. 访问 key_to_node 查找节点
2. 如果找到节点:
   a. 执行类似GET的频率更新操作
   b. 更新节点值

【PUT操作 - 添加新节点，缓存已满】
1. 检查 capacity 与 key_to_node.size()
2. 访问 min_freq 确定要淘汰的频率
3. 访问 freq_to_dummy[min_freq] 获取链表头
4. 移除链表尾部节点
5. 更新 key_to_node 删除被淘汰节点
6. 可能更新 freq_to_dummy 删除空链表
7. 创建新节点并添加到 key_to_node
8. 更新 freq_to_dummy 确保频率1的链表存在
9. 将新节点添加到频率1的链表
10. 重置 min_freq = 1
```
- c++
```c++
//
// Created by 17720 on 25-5-4.
//

#include <unordered_map>

using namespace std;
class Node {
public:
    int key;
    int value;
    int freq = 1;
    Node* prev;
    Node* next;
    Node(int k = 0,int v = 0):key(k),value(v){}
};

class LFUCache {
private:
    int min_freq;
    int capacity;
    unordered_map<int,Node*> key_to_node;
    unordered_map<int,Node*> freq_to_dummy;

    Node* get_node(int key) {
        auto it = key_to_node.find(key);
        if(it==key_to_node.end()) {
            return nullptr;
        }
        Node* node = it->second;
        remove(node);
        Node* dummy = freq_to_dummy[node->freq];
        if(dummy->prev == dummy) {
            freq_to_dummy.erase(node->freq);
            delete dummy; // 修改：删除dummy而不是node
            if(min_freq==node->freq) {
                min_freq++;
            }
        }
        push_front(++node->freq,node);
        return node;
    }

    // 创建一个新的双向链表
    Node* new_list() {
        Node* dummy = new Node();
        dummy->next = dummy;
        dummy->prev = dummy;
        return dummy;
    }

    // 在链表头添加一个节点（把一本书放在最上面）
    void push_front(int freq, Node *x) {
        auto it = freq_to_dummy.find(freq);
        if (it==freq_to_dummy.end()) {
            it = freq_to_dummy.emplace(freq,new_list()).first;
        }
        Node* dummy = it->second;
        x->prev = dummy;
        x->next = dummy->next;
        x->prev->next = x;
        x->next->prev = x;
    }

    // 删除一个节点（抽出一本书）
    void remove(Node *x) {
        x->prev->next = x->next;
        x->next->prev = x->prev;
    }

public:
    LFUCache(int capacity) : capacity(capacity), min_freq(0) {} // 初始化min_freq为0

    int get(int key) {
        if (capacity <= 0) return -1; // 处理容量为0的情况
        Node* node = get_node(key);
        return node ? node->value : -1;
    }

    void put(int key, int val) {
        if (capacity <= 0) return; // 处理容量为0的情况
        
        Node* node = get_node(key);
        if(node) {
            node->value = val;
            return;
        }
        
        if(key_to_node.size() == capacity) {
            Node* dummy = freq_to_dummy[min_freq];
            Node* back_node = dummy->prev;
            key_to_node.erase(back_node->key);
            remove(back_node);
            delete back_node;
            
            if(dummy->prev == dummy) {
                freq_to_dummy.erase(min_freq);
                delete dummy;
            }
        }
        
        node = new Node(key, val);
        key_to_node[key] = node;
        push_front(1, node);
        min_freq = 1;
    }
};
```


- go语言线程安全版
```go
package cache

import (
	"sync"
)

// Node 表示缓存中的一个节点
type Node struct {
	key   int
	value int
	freq  int
	prev  *Node
	next  *Node
}

// LFUCache 是一个并发安全的LFU缓存实现
type LFUCache struct {
	capacity    int
	minFreq     int
	keyToNode   map[int]*Node
	freqToDummy map[int]*Node
	mutex       sync.RWMutex // 读写锁用于并发控制
}

// NewLFUCache 创建一个新的LFU缓存
func NewLFUCache(capacity int) *LFUCache {
	return &LFUCache{
		capacity:    capacity,
		minFreq:     0,
		keyToNode:   make(map[int]*Node),
		freqToDummy: make(map[int]*Node),
	}
}

// newList 创建一个新的双向链表并返回哑节点
func (c *LFUCache) newList() *Node {
	dummy := &Node{}
	dummy.prev = dummy
	dummy.next = dummy
	return dummy
}

// pushFront 在链表头部添加一个节点
func (c *LFUCache) pushFront(freq int, node *Node) {
	dummy, ok := c.freqToDummy[freq]
	if !ok {
		dummy = c.newList()
		c.freqToDummy[freq] = dummy
	}
	
	node.prev = dummy
	node.next = dummy.next
	node.prev.next = node
	node.next.prev = node
}

// remove 从链表中移除一个节点
func (c *LFUCache) remove(node *Node) {
	node.prev.next = node.next
	node.next.prev = node.prev
}

// getNode 获取并更新节点的频率
func (c *LFUCache) getNode(key int) *Node {
	node, ok := c.keyToNode[key]
	if !ok {
		return nil
	}
	
	// 从当前频率列表中移除节点
	c.remove(node)
	
	// 检查并处理空链表
	dummy := c.freqToDummy[node.freq]
	if dummy.prev == dummy { // 链表为空
		delete(c.freqToDummy, node.freq)
		
		// 更新最小频率
		if c.minFreq == node.freq {
			c.minFreq++
		}
	}
	
	// 增加节点频率并添加到新频率列表
	node.freq++
	c.pushFront(node.freq, node)
	
	return node
}

// Get 获取缓存中的值
func (c *LFUCache) Get(key int) int {
	if c.capacity <= 0 {
		return -1
	}
	
	c.mutex.Lock()
	defer c.mutex.Unlock()
	
	node := c.getNode(key)
	if node == nil {
		return -1
	}
	
	return node.value
}

// Put 设置缓存中的值
func (c *LFUCache) Put(key, value int) {
	if c.capacity <= 0 {
		return
	}
	
	c.mutex.Lock()
	defer c.mutex.Unlock()
	
	// 如果key已存在，更新值
	if node := c.getNode(key); node != nil {
		node.value = value
		return
	}
	
	// 如果缓存已满，移除LFU项
	if len(c.keyToNode) >= c.capacity {
		dummy := c.freqToDummy[c.minFreq]
		backNode := dummy.prev
		
		// 从映射和链表中移除
		delete(c.keyToNode, backNode.key)
		c.remove(backNode)
		
		// 如果链表为空，从freq映射移除
		if dummy.prev == dummy {
			delete(c.freqToDummy, c.minFreq)
		}
	}
	
	// 创建新节点并添加到频率为1的链表
	node := &Node{
		key:   key,
		value: value,
		freq:  1,
	}
	c.keyToNode[key] = node
	c.pushFront(1, node)
	c.minFreq = 1
}

// Size 返回当前缓存中的元素数量
func (c *LFUCache) Size() int {
	c.mutex.RLock()
	defer c.mutex.RUnlock()
	
	return len(c.keyToNode)
}

// Clear 清空缓存
func (c *LFUCache) Clear() {
	c.mutex.Lock()
	defer c.mutex.Unlock()
	
	c.keyToNode = make(map[int]*Node)
	c.freqToDummy = make(map[int]*Node)
	c.minFreq = 0
}
```

## 参考

 1. [灵神题解](https://leetcode.cn/problems/lfu-cache/solutions/2457716/tu-jie-yi-zhang-tu-miao-dong-lfupythonja-f56h)
