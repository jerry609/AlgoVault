
![optimized-lfu-cache-flowchart.svg](https://obsidian-1311563466.cos.ap-guangzhou.myqcloud.com/obsidian/optimized-lfu-cache-flowchart.svg)

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
