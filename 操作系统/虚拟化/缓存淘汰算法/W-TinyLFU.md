@[toc]
## 1. 背景与概述

### 1.1 什么是缓存驱逐算法

缓存系统的核心问题是当缓存空间有限时，需要决定哪些数据应该被保留，哪些数据应该被移除。这就是缓存驱逐算法要解决的问题。常见的缓存驱逐算法包括LRU（Least Recently Used，最近最少使用）和LFU（Least Frequently Used，最不经常使用）等。

### 1.2 W-TinyLFU 的定义与价值

W-TinyLFU（Window Tiny Least Frequently Used）是一种高性能的缓存驱逐算法，旨在以较低的内存开销实现高缓存命中率。它巧妙地结合了LRU和LFU两种策略的优点，同时规避了它们各自的缺点。

W-TinyLFU特别适用于需要高性能缓存且内存资源受限的场景，例如CDN、数据库缓存、Web缓存等。

## 2. 核心思想与设计理念

W-TinyLFU的核心思想可以概括为以下几点：

### 2.1 时间局部性与频率局部性的结合

- **利用LRU处理新对象和一次性访问的对象**：它认为最近访问的对象很可能在不久的将来再次被访问（时间局部性）。
- **利用LFU保护长期流行的对象**：它认为访问频率高的对象更有价值，应该长时间保留在缓存中（频率局部性）。

### 2.2 高效的频率统计
![CountMinSketch示意图](https://i-blog.csdnimg.cn/img_convert/0936a27283732c3a25e002f62ba67483.png)

- **使用极小的内存开销来近似LFU**：传统的LFU需要为每个缓存项维护一个精确的访问计数器，内存开销很大。TinyLFU使用Count-Min Sketch这种概率数据结构来_估计_对象的访问频率，极大地降低了内存需求。

### 2.3 窗口机制的引入

- **结合窗口（Window）机制**：W-TinyLFU引入了一个小的"Window"缓存，主要基于LRU策略运行，用于处理新进入的对象和近期访问的对象。这有助于算法快速适应访问模式的变化。

## 3. 架构设计与组件

### 3.1 整体架构

W-TinyLFU由三个主要组件构成：窗口缓存（Window Cache）、主缓存（Main Cache）和频率过滤器（TinyLFU Filter）。

![W-TinyLFU整体架构](https://i-blog.csdnimg.cn/img_convert/97c6d3a7350bafc5caba6a5c9d573e8f.png)

### 3.2 窗口缓存（Window Cache）

窗口缓存是一个相对较小的LRU缓存，通常占据整个缓存空间的约1%。它用于快速接纳新对象并捕捉短期热点数据。

```cpp
// 窗口缓存 (LRU) 的核心实现
template<typename K, typename V>
class WindowCache {
private:
    size_t capacity;  // 窗口缓存容量
    std::list<CacheItem<K, V>> items;  // LRU 列表，最近使用的在前面
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> itemMap;  // 快速查找表
    
public:
    // 构造函数设置窗口容量
    explicit WindowCache(size_t cap) : capacity(cap) {}
    
    // 获取缓存项，如果存在则移动到 MRU 位置（最近最常使用）
    std::optional<V> get(const K& key) {
        auto it = itemMap.find(key);
        if (it == itemMap.end()) {
            return std::nullopt;  // 不存在该键
        }
        
        // 找到了，将它移到列表前端（MRU位置）
        auto listIt = it->second;
        CacheItem<K, V> item = *listIt;
        items.erase(listIt);
        items.push_front(item);
        itemMap[key] = items.begin();
        
        return item.value;  // 返回值
    }
    
    // 添加新的缓存项到MRU位置
    bool add(const K& key, const V& value) {
        // 如果已存在，则更新值并移到MRU位置
        auto it = itemMap.find(key);
        if (it != itemMap.end()) {
            auto listIt = it->second;
            listIt->value = value;
            items.splice(items.begin(), items, listIt);
            return false;  // 没有发生驱逐
        }
        
        // 检查是否需要驱逐
        bool evicted = false;
        if (items.size() >= capacity) {
            evicted = true;
        }
        
        // 添加新项到MRU位置
        items.push_front(CacheItem<K, V>(key, value));
        itemMap[key] = items.begin();
        
        return evicted;  // 返回是否需要驱逐
    }
    
    // 驱逐LRU位置的项（最久未使用的）
    std::optional<CacheItem<K, V>> evict() {
        if (items.empty()) {
            return std::nullopt;
        }
        
        // 获取并移除LRU项
        CacheItem<K, V> victim = items.back();
        itemMap.erase(victim.key);
        items.pop_back();
        
        return victim;  // 返回被驱逐的项
    }
    
    // 其他实用方法...
    bool empty() const { return items.empty(); }
    void clear() { items.clear(); itemMap.clear(); }
    size_t size() const { return items.size(); }
};
```

窗口缓存的主要特点：
- 完全按照**LRU**策略运行
- 所有新的缓存项首先进入窗口缓存
- 主要作用：
    - 快速接纳新对象
    - 捕捉短期热点
    - 作为进入主缓存的过滤器

### 3.3 主缓存（Main Cache）

主缓存占据大部分缓存空间（通常约99%），采用分段LRU策略，分为保护段和试用段两部分。

```cpp
// 主缓存 (分段 LRU: Protected + Probationary) 的核心实现
template<typename K, typename V>
class MainCache {
private:
    size_t capacity;  // 总容量
    size_t protectedRatio;  // Protected段所占比例 (0-100)
    
    // Protected段(保护段)和Probationary段(试用段)
    std::list<CacheItem<K, V>> protectedItems;  // 保护段的LRU列表
    std::list<CacheItem<K, V>> probationaryItems;  // 试用段的LRU列表
    
    // 快速查找表
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> protectedMap;
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> probationaryMap;
    
    // 获取Protected段的最大容量
    size_t getProtectedCapacity() const {
        return capacity * protectedRatio / 100;
    }
    
public:
    // 构造函数，设置容量和保护段比例
    MainCache(size_t cap, size_t protRatio = 80) 
        : capacity(cap), protectedRatio(protRatio) {}
    
    // 获取缓存项
    std::optional<V> get(const K& key) {
        // 先查找Protected段
        auto protIt = protectedMap.find(key);
        if (protIt != protectedMap.end()) {
            // 找到了，移到Protected段的MRU位置
            auto listIt = protIt->second;
            CacheItem<K, V> item = *listIt;
            protectedItems.erase(listIt);
            protectedItems.push_front(item);
            protectedMap[key] = protectedItems.begin();
            return item.value;
        }
        
        // 再查找Probationary段
        auto probIt = probationaryMap.find(key);
        if (probIt != probationaryMap.end()) {
            // 找到了，从Probationary段移除，提升到Protected段
            auto listIt = probIt->second;
            CacheItem<K, V> item = *listIt;
            probationaryItems.erase(listIt);
            probationaryMap.erase(key);
            
            // 提升到Protected段
            return promoteItem(item);
        }
        
        return std::nullopt;  // 未找到
    }
    
    // 提升项从Probationary段到Protected段
    V promoteItem(const CacheItem<K, V>& item) {
        // 如果Protected段已满，需要将一个项降级到Probationary段
        if (protectedItems.size() >= getProtectedCapacity() && !protectedItems.empty()) {
            // 将Protected段LRU端的项降级到Probationary段的MRU位置
            CacheItem<K, V> demoted = protectedItems.back();
            protectedItems.pop_back();
            protectedMap.erase(demoted.key);
            
            probationaryItems.push_front(demoted);
            probationaryMap[demoted.key] = probationaryItems.begin();
        }
        
        // 将项添加到Protected段的MRU位置
        protectedItems.push_front(item);
        protectedMap[item.key] = protectedItems.begin();
        
        return item.value;
    }
    
    // 获取牺牲者（即Probationary段LRU端的项）
    std::optional<CacheItem<K, V>> getVictim() const {
        if (probationaryItems.empty()) {
            return std::nullopt;
        }
        return probationaryItems.back();  // 返回试用段最久未使用的项
    }
    
    // 添加新项到缓存（为简洁起见省略了部分代码）
    void add(const K& key, const V& value) {
        // 检查是否已在缓存中，如果是则更新
        // ...
        
        // 新项添加到Probationary段的MRU位置
        probationaryItems.push_front(CacheItem<K, V>(key, value));
        probationaryMap[key] = probationaryItems.begin();
        
        // 确保总容量不超过限制
        maintainSize();
    }
    
    // 其他方法...
};
```

主缓存的主要特点：
- 占据大部分缓存空间（通常约99%）
- 采用**分段LRU（Segmented LRU, SLRU）**策略
- 分为两个段：
    - **Protected Segment（保护段）**：存放被多次访问的对象
    - **Probationary Segment（试用段）**：存放新加入或只被访问过一次的对象

### 3.4 频率过滤器（TinyLFU Filter）

频率过滤器是实现低开销频率统计的核心，使用Count-Min Sketch数据结构来估计对象访问频率。

```cpp
// TinyLFU 频率估计器，使用 Count-Min Sketch 实现
template<typename K>
class TinyLFU {
private:
    // Count-Min Sketch 参数
    static constexpr int HASH_COUNT = 4;       // 哈希函数数量
    int width;                                 // 每行的计数器数量
    std::vector<std::vector<uint8_t>> counters; // 计数器矩阵
    std::vector<uint64_t> seeds;               // 哈希函数种子
    
    // 重置窗口，用于处理计数饱和
    int resetThreshold;
    int itemCount;
    
    // 哈希函数
    size_t hash(const K& key, int seed) const {
        size_t h = std::hash<K>{}(key) ^ seed;
        return h % width;
    }
    
public:
    // 构造函数，初始化Count-Min Sketch
    explicit TinyLFU(int counterSize) {
        // 根据预期的缓存容量设置计数器大小
        width = counterSize * 10;  // 经验法则：宽度 ≈ 容量的10倍
        
        // 初始化 Count-Min Sketch
        counters.resize(HASH_COUNT, std::vector<uint8_t>(width, 0));
        
        // 初始化哈希函数种子
        seeds.resize(HASH_COUNT);
        for (int i = 0; i < HASH_COUNT; i++) {
            seeds[i] = i * 1099511628211ULL;  // 使用大质数
        }
        
        // 设置重置阈值
        resetThreshold = width * 10;  // 当观察到的项达到此阈值时重置
        itemCount = 0;
    }
    
    // 增加一个键的频率计数
    void increment(const K& key) {
        for (int i = 0; i < HASH_COUNT; i++) {
            size_t index = hash(key, seeds[i]);
            // 避免计数器溢出
            if (counters[i][index] < 255) {
                counters[i][index]++;
            }
        }
        
        // 检查是否需要重置计数器
        itemCount++;
        if (itemCount >= resetThreshold) {
            reset();
        }
    }
    
    // 估计一个键的频率
    uint8_t frequency(const K& key) const {
        uint8_t min_count = 255;
        for (int i = 0; i < HASH_COUNT; i++) {
            size_t index = hash(key, seeds[i]);
            min_count = std::min(min_count, counters[i][index]);
        }
        return min_count;
    }
    
    // 周期性重置：所有计数器减半（门限衰减）
    void reset() {
        for (auto& row : counters) {
            for (auto& count : row) {
                count = count >> 1;  // 除以2（位移操作）
            }
        }
        itemCount = 0;
    }
};
```

#### 3.4.1 Count-Min Sketch原理

Count-Min Sketch是一种概率数据结构，用于在有限空间内估计数据流中元素的频率。其原理如下：

- **结构**：它是一个二维数组，配合多个独立的哈希函数
- **插入操作**：
  ```cpp
  // 对应increment方法的核心
  for (int i = 0; i < HASH_COUNT; i++) {
      size_t index = hash(key, seeds[i]);  // 计算哈希值
      counters[i][index]++;  // 增加计数器
  }
  ```

- **查询操作**：
  ```cpp
  // 对应frequency方法的核心
  uint8_t min_count = 255;
  for (int i = 0; i < HASH_COUNT; i++) {
      size_t index = hash(key, seeds[i]);
      min_count = std::min(min_count, counters[i][index]);  // 取最小值作为估计
  }
  return min_count;
  ```

- **衰减机制**：
  ```cpp
  // 对应reset方法的核心
  for (auto& row : counters) {
      for (auto& count : row) {
          count = count >> 1;  // 所有计数器除以2
      }
  }
  ```

频率过滤器的主要特点：
- 不存储实际的对象数据，只存储频率计数的估计值
- 使用多个哈希函数减小冲突概率
- 通过周期性衰减来适应访问模式的变化

## 4. 工作流程详解

![W-TinyLFU工作流程图](https://i-blog.csdnimg.cn/img_convert/cb9d8cad186ed7589ed2aa3aae4a5d83.png)

### 4.1 缓存访问（Cache Hit）流程

![缓存命中流程](https://i-blog.csdnimg.cn/img_convert/d5906913edc90999922b2bfce60590b7.png)

W-TinyLFU的缓存访问流程是通过WTinyLFUCache类的get方法实现的：

```cpp
// 获取缓存项
std::optional<V> get(const K& key) {
    // 尝试从窗口缓存获取
    auto windowResult = windowCache.get(key);
    if (windowResult) {
        frequencySketch.increment(key);  // 增加频率计数
        return windowResult;
    }
    
    // 尝试从主缓存获取
    auto mainResult = mainCache.get(key);
    if (mainResult) {
        frequencySketch.increment(key);  // 增加频率计数
        return mainResult;
    }
    
    return std::nullopt;  // 不存在于缓存中
}
```

#### 4.1.1 Window Cache命中

当访问的数据位于Window Cache中时：

1. 通过`windowCache.get(key)`找到数据
2. 数据被移动到Window Cache的MRU位置（在WindowCache类内部完成）
3. 通过`frequencySketch.increment(key)`增加TinyLFU中该数据的频率计数
4. 返回找到的数据值

WindowCache中的get方法实现了LRU更新：

```cpp
std::optional<V> get(const K& key) {
    auto it = itemMap.find(key);
    if (it == itemMap.end()) {
        return std::nullopt;  // 不存在
    }
    
    // 移动到 MRU 位置
    auto listIt = it->second;
    CacheItem<K, V> item = *listIt;
    items.erase(listIt);
    items.push_front(item);  // 放到列表前端（MRU位置）
    itemMap[key] = items.begin();
    
    return item.value;
}
```

#### 4.1.2 Main Cache命中

当访问的数据位于Main Cache中（无论是试用段或保护段）时：

1. 通过`mainCache.get(key)`找到数据
2. 数据移动到对应段的MRU位置（在MainCache类内部完成）
3. 如果数据在试用段，它会被提升到保护段（如果保护段有空间）
4. 通过`frequencySketch.increment(key)`增加TinyLFU中该数据的频率计数
5. 返回找到的数据值

MainCache中的get方法实现了SLRU的访问和提升逻辑：

```cpp
std::optional<V> get(const K& key) {
    // 先查找Protected段
    auto protIt = protectedMap.find(key);
    if (protIt != protectedMap.end()) {
        auto listIt = protIt->second;
        CacheItem<K, V> item = *listIt;
        
        // 移动到Protected段的MRU位置
        protectedItems.erase(listIt);
        protectedItems.push_front(item);
        protectedMap[key] = protectedItems.begin();
        
        return item.value;
    }
    
    // 再查找Probationary段
    auto probIt = probationaryMap.find(key);
    if (probIt != probationaryMap.end()) {
        auto listIt = probIt->second;
        CacheItem<K, V> item = *listIt;
        
        // 从Probationary段移除
        probationaryItems.erase(listIt);
        probationaryMap.erase(key);
        
        // 提升到Protected段
        return promoteItem(item);  // 这里实现了从试用段到保护段的提升
    }
    
    return std::nullopt;  // 不存在
}
```

### 4.2 缓存未命中与驱逐决策流程

![缓存未命中与接纳/驱逐决策](https://i-blog.csdnimg.cn/img_convert/13c3d981ac48f2f76c09bbf75959463e.png)

当缓存中不存在请求的数据时，W-TinyLFU通过put方法处理：

```cpp
// 添加/更新缓存项
void put(const K& key, const V& value) {
    // 先尝试直接获取(用于更新)
    if (get(key)) {
        // 如果在窗口缓存中，更新窗口缓存
        auto windowResult = windowCache.get(key);
        if (windowResult) {
            windowCache.add(key, value);
            return;
        }
        
        // 如果在主缓存中，更新主缓存
        mainCache.add(key, value);
        return;
    }
    
    // 新项，增加频率
    frequencySketch.increment(key);
    
    // 添加到窗口缓存
    bool evicted = windowCache.add(key, value);
    
    // 如果窗口驱逐了一个项，执行策略决定
    if (evicted) {
        auto evictedItem = windowCache.evict();
        if (evictedItem) {
            handleWindowEviction(*evictedItem);
        }
    }
}
```

#### 4.2.1 缓存未命中处理

当发生缓存未命中时：

1. 首先增加新数据的频率计数：`frequencySketch.increment(key);`
2. 新数据被添加到Window Cache：`windowCache.add(key, value);`
3. 如果Window Cache已满，会返回`evicted = true`
4. 此时需要从Window Cache驱逐一个项：`evictedItem = windowCache.evict();`
5. 被驱逐的项成为"候选者"，进入驱逐决策流程：`handleWindowEviction(*evictedItem);`

#### 4.2.2 驱逐决策机制

当Window Cache已满需要驱逐项时，W-TinyLFU需要决定是否将窗口缓存中驱逐出的候选者接纳到主缓存中：

```cpp
// 处理窗口缓存的驱逐
void handleWindowEviction(const CacheItem<K, V>& candidate) {
    // 获取主缓存中的牺牲者（从试用段LRU端）
    auto victimOpt = mainCache.getVictim();
    
    // 如果主缓存没有牺牲者(空的)，直接添加候选项
    if (!victimOpt) {
        mainCache.add(candidate.key, candidate.value);
        return;
    }
    
    const CacheItem<K, V>& victim = *victimOpt;
    
    // 比较候选项与牺牲者的频率
    uint8_t candidateFreq = frequencySketch.frequency(candidate.key);
    uint8_t victimFreq = frequencySketch.frequency(victim.key);
    
    // 接纳策略：候选项频率大于牺牲者频率
    if (candidateFreq > victimFreq) {
        // 驱逐牺牲者
        mainCache.evict(victim.key);
        // 接纳候选项
        mainCache.add(candidate.key, candidate.value);
    }
    // 否则丢弃候选项（不做任何操作）
}
```

驱逐决策的具体步骤：

1. 从Main Cache的试用段LRU端找到"牺牲者"：`victimOpt = mainCache.getVictim();`
2. 使用TinyLFU Filter查询候选者和牺牲者的访问频率：
   ```cpp
   candidateFreq = frequencySketch.frequency(candidate.key);
   victimFreq = frequencySketch.frequency(victim.key);
   ```
3. 决策逻辑：
   - 如果候选者频率 > 牺牲者频率：接纳候选者进入Main Cache，驱逐牺牲者
   ```cpp
   if (candidateFreq > victimFreq) {
       mainCache.evict(victim.key);
       mainCache.add(candidate.key, candidate.value);
   }
   ```
   - 如果候选者频率 ≤ 牺牲者频率：丢弃候选者，牺牲者保留在缓存中

## 5. 算法优势与应用场景

### 5.1 W-TinyLFU的优势

- **扫描保护**：能有效过滤一次性访问的数据
- **频率感知**：通过频率统计保留真正"热"的数据
- **空间效率**：使用概率数据结构降低内存开销
- **时间效率**：主要操作都具有O(1)时间复杂度
- **平衡新旧**：在保留热点数据的同时，为新数据提供机会
- **适应性**：通过窗口和衰减机制适应访问模式变化

### 5.2 潜在缺点与考虑因素

- **实现复杂度**：相较于简单的LRU，实现更为复杂
- **参数调优**：性能受多个参数影响，需要针对具体场景调优
- **概率性**：频率统计是估计值而非精确值，理论上存在误差可能

### 5.3 适用场景

- 数据库缓存层
- 内容分发网络（CDN）
- Web应用缓存
- 分布式缓存系统
- 任何有明显访问模式的大规模缓存应用

## 6. 实现与接口设计

### 6.1 核心抽象与API设计

#### 6.1.1 主要缓存接口

```cpp
template<typename K, typename V>
class WTinyLFUCache {
public:
    // 获取缓存项，如果存在返回值，否则返回null
    std::optional<V> get(const K& key);
    
    // 存储一个新的缓存项
    void put(const K& key, const V& value);
    
    // 清空缓存
    void clear();
    
    // 当前缓存大小
    size_t size() const;
    
    // 缓存容量
    size_t capacity() const;
};
```

#### 6.1.2 组件接口

- `WindowCache`: 窗口缓存（LRU）
    - `add(key, value)`: 添加项到窗口
    - `evict()`: 驱逐项并返回
- `MainCache`: 主缓存（分段LRU）
    - `add(key, value)`: 添加项到缓存
    - `getVictim()`: 获取牺牲者
    - `promoteProbationaryItem(key)`: 提升试用项到受保护区
- `TinyLFU`: 频率估计器
    - `increment(key)`: 增加键的频率计数
    - `frequency(key)`: 获取键的估计频率
    - `reset()`: 周期性重置计数器

## 7. 性能考量与优化

### 7.1 时间复杂度分析

- **get操作**：平均O(1)时间复杂度
- **put操作**：平均O(1)时间复杂度
- **驱逐决策**：O(1)时间复杂度

### 7.2 空间复杂度分析

- **缓存项存储**：O(n)，其中n为缓存容量
- **TinyLFU过滤器**：O(m)，其中m通常远小于n，Count-Min Sketch结构使用的空间比传统LFU少得多
- **额外元数据**：O(n)用于存储映射关系和链表指针

### 7.3 实际应用优化建议

- **窗口大小调整**：根据工作负载特性调整窗口缓存与主缓存的比例
  ```cpp
  // 调整窗口缓存大小的示例
  WTinyLFUCache<std::string, std::string> cache(1000, 0.01);  // 1%窗口大小
  WTinyLFUCache<std::string, std::string> cache(1000, 0.05);  // 5%窗口大小，更适合变化快的负载
  ```

- **分段比例优化**：调整保护段与试用段的容量比例
  ```cpp
  // 调整主缓存中保护段大小的示例
  WTinyLFUCache<std::string, std::string> cache(1000, 0.01, 80);  // 保护段80%
  WTinyLFUCache<std::string, std::string> cache(1000, 0.01, 60);  // 保护段60%，更容易接纳新项
  ```

- **Count-Min Sketch参数**：根据缓存大小调整哈希函数数量和计数器矩阵大小
  ```cpp
  // Count-Min Sketch参数调整示例
  class TinyLFU {
  private:
      static constexpr int HASH_COUNT = 4;  // 调整哈希函数数量
      // width = counterSize * 10;  // 调整宽度倍数
  };
  ```

- **衰减周期**：根据访问模式变化频率调整计数器衰减周期
  ```cpp
  // 调整衰减周期示例
  // resetThreshold = width * 10;  // 默认设置
  resetThreshold = width * 5;  // 更频繁地衰减，适应快速变化的模式
  resetThreshold = width * 20;  // 减少衰减频率，适应稳定的访问模式
  ```

## 8. 完整实现示例

以下是W-TinyLFU缓存驱逐算法的完整实现：

```cpp
#include <iostream>
#include <unordered_map>
#include <list>
#include <vector>
#include <cstdint>
#include <algorithm>
#include <memory>
#include <cmath>
#include <optional>

// 缓存项结构
template<typename K, typename V>
struct CacheItem {
    K key;
    V value;
    
    CacheItem(const K& k, const V& v) : key(k), value(v) {}
};

// TinyLFU 频率估计器，使用 Count-Min Sketch 实现
template<typename K>
class TinyLFU {
private:
    // Count-Min Sketch 参数
    static constexpr int HASH_COUNT = 4;       // 哈希函数数量
    int width;                                 // 每行的计数器数量
    std::vector<std::vector<uint8_t>> counters; // 计数器矩阵
    std::vector<uint64_t> seeds;               // 哈希函数种子
    
    // 重置窗口，用于处理计数饱和
    int resetThreshold;
    int itemCount;
    
    // 哈希函数
    size_t hash(const K& key, int seed) const {
        size_t h = std::hash<K>{}(key) ^ seed;
        return h % width;
    }
    
public:
    explicit TinyLFU(int counterSize) {
        // 根据预期的缓存容量设置计数器大小
        width = counterSize * 10;  // 经验法则：宽度 ≈ 容量的10倍
        
        // 初始化 Count-Min Sketch
        counters.resize(HASH_COUNT, std::vector<uint8_t>(width, 0));
        
        // 初始化哈希函数种子
        seeds.resize(HASH_COUNT);
        for (int i = 0; i < HASH_COUNT; i++) {
            seeds[i] = i * 1099511628211ULL;  // 使用大质数
        }
        
        // 设置重置阈值
        resetThreshold = width * 10;  // 当观察到的项达到此阈值时重置
        itemCount = 0;
    }
    
    // 增加一个键的频率计数
    void increment(const K& key) {
        for (int i = 0; i < HASH_COUNT; i++) {
            size_t index = hash(key, seeds[i]);
            // 避免计数器溢出
            if (counters[i][index] < 255) {
                counters[i][index]++;
            }
        }
        
        // 检查是否需要重置计数器
        itemCount++;
        if (itemCount >= resetThreshold) {
            reset();
        }
    }
    
    // 估计一个键的频率
    uint8_t frequency(const K& key) const {
        uint8_t min_count = 255;
        for (int i = 0; i < HASH_COUNT; i++) {
            size_t index = hash(key, seeds[i]);
            min_count = std::min(min_count, counters[i][index]);
        }
        return min_count;
    }
    
    // 周期性重置：所有计数器减半（门限衰减）
    void reset() {
        for (auto& row : counters) {
            for (auto& count : row) {
                count = count >> 1;  // 除以2（位移操作）
            }
        }
        itemCount = 0;
    }
};

// 窗口缓存 (LRU)
template<typename K, typename V>
class WindowCache {
private:
    size_t capacity;
    std::list<CacheItem<K, V>> items;  // LRU 列表
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> itemMap;  // 查找表
    
public:
    explicit WindowCache(size_t cap) : capacity(cap) {}
    
    // 获取缓存项，如果存在则移动到 MRU 位置
    std::optional<V> get(const K& key) {
        auto it = itemMap.find(key);
        if (it == itemMap.end()) {
            return std::nullopt;  // 不存在
        }
        
        // 移动到 MRU 位置
        auto listIt = it->second;
        CacheItem<K, V> item = *listIt;
        items.erase(listIt);
        items.push_front(item);
        itemMap[key] = items.begin();
        
        return item.value;
    }
    
    // 添加新的缓存项
    bool add(const K& key, const V& value) {
        // 如果已存在，则更新
        auto it = itemMap.find(key);
        if (it != itemMap.end()) {
            auto listIt = it->second;
            listIt->value = value;
            
            // 移动到 MRU 位置
            items.splice(items.begin(), items, listIt);
            return false;  // 没有发生驱逐
        }
        
        // 检查容量
        bool evicted = false;
        if (items.size() >= capacity) {
            evicted = true;
        }
        
        // 添加新项到 MRU 位置
        items.push_front(CacheItem<K, V>(key, value));
        itemMap[key] = items.begin();
        
        return evicted;
    }
    
    // 驱逐 LRU 项
    std::optional<CacheItem<K, V>> evict() {
        if (items.empty()) {
            return std::nullopt;
        }
        
        // 获取 LRU 项
        CacheItem<K, V> victim = items.back();
        itemMap.erase(victim.key);
        items.pop_back();
        
        return victim;
    }
    
    // 是否为空
    bool empty() const {
        return items.empty();
    }
    
    // 清空缓存
    void clear() {
        items.clear();
        itemMap.clear();
    }
    
    // 当前大小
    size_t size() const {
        return items.size();
    }
};

// 主缓存 (分段 LRU: Protected + Probationary)
template<typename K, typename V>
class MainCache {
private:
    size_t capacity;
    size_t protectedRatio;  // Protected段所占比例 (0-100)
    
    // Protected段和Probationary段
    std::list<CacheItem<K, V>> protectedItems;
    std::list<CacheItem<K, V>> probationaryItems;
    
    // 查找表
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> protectedMap;
    std::unordered_map<K, typename std::list<CacheItem<K, V>>::iterator> probationaryMap;
    
    // 获取Protected段的最大容量
    size_t getProtectedCapacity() const {
        return capacity * protectedRatio / 100;
    }
    
public:
    MainCache(size_t cap, size_t protRatio = 80) 
        : capacity(cap), protectedRatio(protRatio) {}
    
    // 获取缓存项
    std::optional<V> get(const K& key) {
        // 先查找 Protected 段
        auto protIt = protectedMap.find(key);
        if (protIt != protectedMap.end()) {
            auto listIt = protIt->second;
            CacheItem<K, V> item = *listIt;
            
            // 移动到 Protected MRU 位置
            protectedItems.erase(listIt);
            protectedItems.push_front(item);
            protectedMap[key] = protectedItems.begin();
            
            return item.value;
        }
        
        // 再查找 Probationary 段
        auto probIt = probationaryMap.find(key);
        if (probIt != probationaryMap.end()) {
            auto listIt = probIt->second;
            CacheItem<K, V> item = *listIt;
            
            // 从 Probationary 移除
            probationaryItems.erase(listIt);
            probationaryMap.erase(key);
            
            // 添加到 Protected (如果有空间)
            return promoteItem(item);
        }
        
        return std::nullopt;  // 不存在
    }
    
    // 添加新的缓存项（总是添加到 Probationary）
    void add(const K& key, const V& value) {
        // 检查是否已在 Protected
        auto protIt = protectedMap.find(key);
        if (protIt != protectedMap.end()) {
            auto listIt = protIt->second;
            listIt->value = value;
            
            // 移动到 Protected MRU
            protectedItems.splice(protectedItems.begin(), protectedItems, listIt);
            return;
        }
        
        // 检查是否已在 Probationary
        auto probIt = probationaryMap.find(key);
        if (probIt != probationaryMap.end()) {
            auto listIt = probIt->second;
            listIt->value = value;
            
            // 移动到 Probationary MRU
            probationaryItems.splice(probationaryItems.begin(), probationaryItems, listIt);
            return;
        }
        
        // 添加到 Probationary
        probationaryItems.push_front(CacheItem<K, V>(key, value));
        probationaryMap[key] = probationaryItems.begin();
        
        // 确保总容量不超过限制
        maintainSize();
    }
    
    // 提升 Probationary 项到 Protected
    V promoteItem(const CacheItem<K, V>& item) {
        // 如果 Protected 已满，尝试淘汰一些项
        if (protectedItems.size() >= getProtectedCapacity() && !protectedItems.empty()) {
            // 将 Protected LRU 降级到 Probationary MRU
            CacheItem<K, V> demoted = protectedItems.back();
            protectedItems.pop_back();
            protectedMap.erase(demoted.key);
            
            probationaryItems.push_front(demoted);
            probationaryMap[demoted.key] = probationaryItems.begin();
        }
        
        // 添加到 Protected MRU
        protectedItems.push_front(item);
        protectedMap[item.key] = protectedItems.begin();
        
        return item.value;
    }
    
    // 获取牺牲者（Probationary LRU）
    std::optional<CacheItem<K, V>> getVictim() const {
        if (probationaryItems.empty()) {
            return std::nullopt;
        }
        return probationaryItems.back();
    }
    
    // 驱逐一个指定的项
    bool evict(const K& key) {
        // 尝试从 Probationary 移除
        auto probIt = probationaryMap.find(key);
        if (probIt != probationaryMap.end()) {
            probationaryItems.erase(probIt->second);
            probationaryMap.erase(probIt);
            return true;
        }
        
        // 尝试从 Protected 移除（不常见）
        auto protIt = protectedMap.find(key);
        if (protIt != protectedMap.end()) {
            protectedItems.erase(protIt->second);
            protectedMap.erase(protIt);
            return true;
        }
        
        return false;  // 项不存在
    }
    
    // 确保缓存大小不超过容量
    void maintainSize() {
        size_t totalSize = protectedItems.size() + probationaryItems.size();
        while (totalSize > capacity && !probationaryItems.empty()) {
            // 从 Probationary LRU 端驱逐
            K victimKey = probationaryItems.back().key;
            probationaryItems.pop_back();
            probationaryMap.erase(victimKey);
            totalSize--;
        }
        
        // 如果仍然超出容量(罕见情况)，从 Protected 驱逐
        while (totalSize > capacity && !protectedItems.empty()) {
            K victimKey = protectedItems.back().key;
            protectedItems.pop_back();
            protectedMap.erase(victimKey);
            totalSize--;
        }
    }
    
    // 当前大小
    size_t size() const {
        return protectedItems.size() + probationaryItems.size();
    }
    
    // 清空缓存
    void clear() {
        protectedItems.clear();
        probationaryItems.clear();
        protectedMap.clear();
        probationaryMap.clear();
    }
};

// W-TinyLFU 主缓存类
template<typename K, typename V>
class WTinyLFUCache {
private:
    size_t totalCapacity;
    float windowRatio;  // 窗口缓存的比例 (0.0-1.0)
    
    WindowCache<K, V> windowCache;
    MainCache<K, V> mainCache;
    TinyLFU<K> frequencySketch;
    
public:
    WTinyLFUCache(size_t capacity, float windowRatio = 0.01, size_t protectedRatio = 80)
        : totalCapacity(capacity), 
          windowRatio(windowRatio),
          windowCache(static_cast<size_t>(capacity * windowRatio)),
          mainCache(capacity - static_cast<size_t>(capacity * windowRatio), protectedRatio),
          frequencySketch(capacity) {}
    
    // 获取缓存项
    std::optional<V> get(const K& key) {
        // 尝试从窗口缓存获取
        auto windowResult = windowCache.get(key);
        if (windowResult) {
            frequencySketch.increment(key);
            return windowResult;
        }
        
        // 尝试从主缓存获取
        auto mainResult = mainCache.get(key);
        if (mainResult) {
            frequencySketch.increment(key);
            return mainResult;
        }
        
        return std::nullopt;  // 不存在
    }
    
    // 添加/更新缓存项
    void put(const K& key, const V& value) {
        // 先尝试直接获取(用于更新)
        if (get(key)) {
            // 如果在窗口缓存中，更新窗口缓存
            auto windowResult = windowCache.get(key);
            if (windowResult) {
                windowCache.add(key, value);
                return;
            }
            
            // 如果在主缓存中，更新主缓存
            mainCache.add(key, value);
            return;
        }
        
        // 新项，增加频率
        frequencySketch.increment(key);
        
        // 添加到窗口缓存
        bool evicted = windowCache.add(key, value);
        
        // 如果窗口驱逐了一个项，执行策略决定
        if (evicted) {
            auto evictedItem = windowCache.evict();
            if (evictedItem) {
                handleWindowEviction(*evictedItem);
            }
        }
    }
    
    // 处理窗口缓存的驱逐
    void handleWindowEviction(const CacheItem<K, V>& candidate) {
        // 获取主缓存中的牺牲者
        auto victimOpt = mainCache.getVictim();
        
        // 如果主缓存没有牺牲者(空的)，直接添加候选项
        if (!victimOpt) {
            mainCache.add(candidate.key, candidate.value);
            return;
        }
        
        const CacheItem<K, V>& victim = *victimOpt;
        
        // 比较候选项与牺牲者的频率
        uint8_t candidateFreq = frequencySketch.frequency(candidate.key);
        uint8_t victimFreq = frequencySketch.frequency(victim.key);
        
        // 接纳策略：候选项频率大于牺牲者频率
        if (candidateFreq > victimFreq) {
            // 驱逐牺牲者
            mainCache.evict(victim.key);
            // 接纳候选项
            mainCache.add(candidate.key, candidate.value);
        }
        // 否则丢弃候选项（不做任何操作）
    }
    
    // 清空缓存
    void clear() {
        windowCache.clear();
        mainCache.clear();
    }
    
    // 当前缓存大小
    size_t size() const {
        return windowCache.size() + mainCache.size();
    }
    
    // 缓存容量
    size_t capacity() const {
        return totalCapacity;
    }
};
```

## 9. 示例应用：Web页面缓存

下面是一个使用W-TinyLFU的简单Web页面缓存示例：

```cpp
#include <iostream>
#include <string>
#include <chrono>
#include <thread>

// 假设我们已经包含了W-TinyLFU的实现...

// 模拟网页内容
struct WebPage {
    std::string url;
    std::string content;
    long timestamp;
    
    WebPage() : timestamp(0) {}
    WebPage(const std::string& u, const std::string& c) 
        : url(u), content(c), 
          timestamp(std::chrono::system_clock::now().time_since_epoch().count()) {}
};

// 模拟从网络加载页面（昂贵操作）
WebPage fetchFromNetwork(const std::string& url) {
    std::cout << "Fetching " << url << " from network..." << std::endl;
    
    // 模拟网络延迟
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    
    // 返回一个模拟的网页
    return WebPage(url, "Content of " + url + " fetched at " + 
                   std::to_string(std::chrono::system_clock::now().time_since_epoch().count()));
}

int main() {
    // 创建一个容量为100的W-TinyLFU缓存
    WTinyLFUCache<std::string, WebPage> pageCache(100);
    
    // 模拟访问模式
    std::vector<std::string> urls = {
        "https://example.com/home",
        "https://example.com/about",
        "https://example.com/products",
        "https://example.com/blog/post1",
        "https://example.com/blog/post2"
    };
    
    // 模拟热点页面（多次访问）
    std::vector<std::string> hotUrls = {
        "https://example.com/home",
        "https://example.com/products"
    };
    
    // 第一次访问所有页面（填充缓存）
    for (const auto& url : urls) {
        std::cout << "First visit to " << url << std::endl;
        
        // 尝试从缓存获取
        auto cachedPage = pageCache.get(url);
        
        if (!cachedPage) {
            // 缓存未命中，从网络获取
            WebPage page = fetchFromNetwork(url);
            
            // 存入缓存
            pageCache.put(url, page);
            std::cout << "Page cached: " << url << std::endl;
        } else {
            std::cout << "Cache hit for " << url << std::endl;
        }
        
        std::cout << "-------------------" << std::endl;
    }
    
    // 模拟重复访问热点页面（增加频率）
    for (int i = 0; i < 5; i++) {
        for (const auto& url : hotUrls) {
            std::cout << "Repeated visit to hot page: " << url << std::endl;
            
            // 尝试从缓存获取
            auto cachedPage = pageCache.get(url);
            
            if (cachedPage) {
                std::cout << "Cache hit for hot page: " << url << std::endl;
            } else {
                // 理论上不应该发生，因为热点页面应该在缓存中
                std::cout << "Unexpected cache miss for hot page: " << url << std::endl;
                
                // 重新加载并缓存
                WebPage page = fetchFromNetwork(url);
                pageCache.put(url, page);
            }
            
            std::cout << "-------------------" << std::endl;
        }
    }
    
    // 模拟一次性访问（扫描模式）
    for (int i = 0; i < 10; i++) {
        std::string scanUrl = "https://example.com/scan/page" + std::to_string(i);
        
        std::cout << "Scanning: " << scanUrl << std::endl;
        
        // 尝试从缓存获取
        auto cachedPage = pageCache.get(scanUrl);
        
        if (!cachedPage) {
            // 缓存未命中，从网络获取
            WebPage page = fetchFromNetwork(scanUrl);
            
            // 存入缓存
            pageCache.put(scanUrl, page);
        } else {
            std::cout << "Cache hit for " << scanUrl << " (unlikely)" << std::endl;
        }
        
        std::cout << "-------------------" << std::endl;
    }
    
    // 验证热点页面是否仍在缓存中
    for (const auto& url : hotUrls) {
        std::cout << "Final check for hot page: " << url << std::endl;
        
        auto cachedPage = pageCache.get(url);
        
        if (cachedPage) {
            std::cout << "SUCCESS: Hot page still in cache: " << url << std::endl;
        } else {
            std::cout << "FAILURE: Hot page evicted: " << url << std::endl;
        }
        
        std::cout << "-------------------" << std::endl;
    }
    
    // 验证扫描页面是否已被驱逐（随机选择几个）
    for (int i = 0; i < 3; i++) {
        std::string scanUrl = "https://example.com/scan/page" + std::to_string(i);
        
        std::cout << "Final check for scan page: " << scanUrl << std::endl;
        
        auto cachedPage = pageCache.get(scanUrl);
        
        if (cachedPage) {
            std::cout << "Scan page still in cache: " << scanUrl << std::endl;
        } else {
            std::cout << "As expected, scan page evicted: " << scanUrl << std::endl;
        }
        
        std::cout << "-------------------" << std::endl;
    }
    
    return 0;
}
```


## 10. 参考资料

1. [TinyLFU: A Highly Efficient Cache Admission Policy](https://arxiv.org/abs/1512.00727)
2. [w-TinyLFU: A Cost-efficient, Admission-constrained Cache](https://github.com/ben-manes/caffeine/wiki/Design)
3. [Caffeine: A Java implementation of W-TinyLFU](https://github.com/ben-manes/caffeine)
4. [Count-Min Sketch: An improved data structure for finding frequent elements in data streams](https://sites.google.com/site/countminsketch/)
5. [W-TinyLFU缓存淘汰策略](https://juejin.cn/post/7144327955353698334)