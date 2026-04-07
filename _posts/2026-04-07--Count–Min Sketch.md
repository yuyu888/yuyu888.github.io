---
layout: mypost
title: Count–Min Sketch
categories: [算法]
---


 偶然发现了 Count–Min Sketch 感觉非常实用， 可以解决很多实际问题， 记录下

==================以下内容为ai生成==================


# Count–Min Sketch：用极小内存统计海量数据的频率

在大数据场景下，我们经常面临一个经典问题：**如何用有限的内存，统计海量数据中每个元素出现的频率？**

例如：
- 统计过去1小时访问量最大的URL（热点榜单）
- 统计一个视频网站上最火的100个视频
- 统计某个API接口被哪些IP高频调用

直接用一个哈希表记录每个元素的计数器，是最直观的方法。但当数据量达到数亿甚至数十亿级别时，内存会成为瓶颈。

Count–Min Sketch（简称CMS）是一种**概率型数据结构**，用极小的内存（通常几KB到几MB）近似统计频率，允许一定误差，但能以极高的效率完成计算。

---

## 核心思想：多个哈希函数 + 二维数组

CMS的结构非常简单：

- 一个二维数组：宽度 `w`，深度 `d`
- `d` 个独立的哈希函数：`h₁, h₂, ..., h_d`

每个哈希函数将输入元素映射到 `[0, w-1]` 范围的一个位置。

```
         w 列
      ┌─────────────┐
d 行  │  计数器数组   │
      └─────────────┘
```

### 更新元素

当元素 `x` 到来时：
- 对每个哈希函数 `hᵢ(x)` 计算位置
- 对应位置的计数器 **加1**

```python
for i in range(d):
    position = hash_i(x) % w
    count[i][position] += 1
```

### 查询元素频率

查询元素 `x` 的频率时：
- 同样计算 `d` 个哈希位置
- 取这些位置中的**最小值**

```python
def query(x):
    min_count = infinity
    for i in range(d):
        pos = hash_i(x) % w
        min_count = min(min_count, count[i][pos])
    return min_count
```

**为什么取最小值？** 因为不同元素可能被哈希到同一个位置（哈希冲突），导致计数器被“污染”而偏大。最小值最接近真实值。

---

## 实例演示：统计网页访问量

假设我们有：
- 宽度 `w = 5`
- 深度 `d = 3`
- 3个哈希函数（为简化，用简单取模代替）

### 初始状态

```
      col0 col1 col2 col3 col4
row0:   0    0    0    0    0
row1:   0    0    0    0    0
row2:   0    0    0    0    0
```

### 插入元素 A

假设：
- `h₀(A) = 1`
- `h₁(A) = 3`
- `h₂(A) = 0`

```
row0 col1: 0→1
row1 col3: 0→1
row2 col0: 0→1
```

### 插入元素 B

假设：
- `h₀(B) = 1`  ← 和A冲突
- `h₁(B) = 2`
- `h₂(B) = 4`

```
row0 col1: 1→2   (冲突，累加)
row1 col2: 0→1
row2 col4: 0→1
```

### 插入元素 A 第二次

```
row0 col1: 2→3
row1 col3: 1→2
row2 col0: 1→2
```

### 查询 A 的频率

- row0 col1 = 3
- row1 col3 = 2
- row2 col0 = 2

取最小值：`min(3, 2, 2) = 2`

真实频率：2 ✅ 正确

### 查询 B 的频率

- row0 col1 = 3（被A污染）
- row1 col2 = 1
- row2 col4 = 1

取最小值：`min(3, 1, 1) = 1`

真实频率：1 ✅ 正确

### 查询一个从未出现过的元素 C

假设：
- `h₀(C) = 1` → row0 col1 = 3
- `h₁(C) = 2` → row1 col2 = 1
- `h₂(C) = 4` → row2 col4 = 1

取最小值：`min(3, 1, 1) = 1`

**问题来了**：C从未出现，但我们返回了1。这就是**假阳性**——CMS永远不会低估频率，但可能高估。

---

## 误差分析

CMS的精度由 `w` 和 `d` 控制。

- 宽度 `w` 决定**哈希冲突概率**，误差与 `1/w` 成正比
- 深度 `d` 决定**多个哈希函数取最小值的可靠性**

**数学结论**（假设总元素数为N）：

> 查询结果误差超过 `εN` 的概率 ≤ `e^{-d}`

其中 `ε = 2/w`（更常见的公式是 `ε = e/w`，取决于具体实现）。

**典型配置**：
- 误差率 `ε = 0.01`（1%误差）→ `w = 200`（因为 `2/w ≤ 0.01`）
- 置信概率 `1 - δ = 0.999` → `d = ln(1/δ) ≈ 7`

总内存 = `w × d × 计数器大小`。用4字节整数：
`200 × 7 × 4 ≈ 5.6 KB`

5.6KB就能处理**任意数量**的元素！这是CMS最惊人的地方。

---

## 实战：用CMS实现实时热点榜单

需求：实时统计过去1小时内访问量最高的前10个URL。

### 方案：CMS + 最小堆

CMS只告诉我们频率，不保存具体元素。要找出热点，需要另一层结构。

#### 数据结构
1. **CMS**：记录每个URL的访问频率
2. **最小堆**：维护当前前K个候选热点（K=10）
3. **哈希表**：记录堆中每个URL的当前频率（用于更新）

#### 流程

**每来一次URL访问**：
1. CMS对URL计数 +1
2. 查询CMS得到该URL的最新近似频率 `f`
3. 如果URL已在堆中：更新堆内频率
4. 如果URL不在堆中：
   - 堆大小 < K：直接加入
   - 堆大小 = K：如果 `f > 堆顶频率`，弹出堆顶，加入新URL

**每过一段时间（如1分钟）**：
- 由于时间是滑动窗口，需要让旧数据衰减
- 简单方案：每60秒新建一个CMS，保留最近60个（1小时）的CMS，查询时求和
- 更高效方案：使用**时间衰减CMS**（每个计数器附带时间戳，或定期除以2）

#### 代码框架

```python
import heapq
from collections import defaultdict

class HotList:
    def __init__(self, k=10, width=2000, depth=7):
        self.cms = CountMinSketch(width, depth)
        self.heap = []          # 最小堆 (频率, url)
        self.map = {}           # url -> 频率
        self.k = k
    
    def record(self, url):
        # 1. 更新CMS
        self.cms.add(url)
        
        # 2. 获取最新频率
        freq = self.cms.query(url)
        
        # 3. 更新热点榜单
        if url in self.map:
            # 已在堆中，更新频率
            self.map[url] = freq
            heapq.heapify(self.heap)  # 简化，实际需用lazy update
        else:
            if len(self.heap) < self.k:
                heapq.heappush(self.heap, (freq, url))
                self.map[url] = freq
            else:
                min_freq, min_url = self.heap[0]
                if freq > min_freq:
                    heapq.heapreplace(self.heap, (freq, url))
                    del self.map[min_url]
                    self.map[url] = freq
    
    def top_k(self):
        # 返回按频率降序的热点
        return sorted(self.heap, key=lambda x: -x[0])
```

### 为什么这样可行？

- CMS用5-10KB内存处理任意流量
- 堆只用K个空间（K=10时微不足道）
- 误差率可控：假设真实访问量第10名是1000，误差1%意味着30左右的波动，对热点排名影响很小

---

## CMS vs 其他方案

| 方案 | 内存 | 精确性 | 适用场景 |
|------|------|--------|----------|
| 哈希表 | O(唯一元素数) | 精确 | 小数据 |
| Bloom Filter | 极小 | 只能判断存在性 | 去重/缓存过滤 |
| **Count–Min Sketch** | **固定极小** | 近似（不低估） | 频率统计、热点、异常检测 |
| Count–Mean–Min Sketch | 略大 | 更精确 | 需要更好精度时 |

---

## 生产环境中的CMS

CMS已被广泛应用：

- **Redis Stack** 内置了CMS模块（`redisbloom`）
- **Twitter** 用CMS统计高频话题标签
- **Cloudflare** 用CMS做DDoS攻击检测
- **Flink/Spark** 流处理框架中作为聚合算子

```bash
# Redis中使用示例
> CMS.INITBYPROB my_cms 0.001 0.01   # 误差率0.1%，置信度99%
> CMS.INCRBY my_cms user:123 1
> CMS.QUERY my_cms user:123
(integer) 42
```

---

## 总结

Count–Min Sketch的精髓在于：

1. **固定内存**：与数据量无关，只取决于预设的精度
2. **单向误差**：只高估不低估，适合“超过阈值报警”的场景
3. **极简实现**：几十行代码就能写出可用的版本
4. **高度并行**：每行独立，天然支持分布式累加

它不是万能的——当你需要精确值时不要用它。但在海量数据的实时统计中，用几KB内存换1%的误差，往往是工程上最明智的选择。
