leveled compaction 开销大

Lazy level 

去除除了最大合并操作外的其他合并操作。

改善了最坏情况的更新开销的复杂性。同时保证点查，长范围查找，存储空间成本的变化不大。

Fluid lsm-tree 对 lsm-tree 设计空间的泛化 
1. 可调参
2. 通过在最大级别合并更少的数据来进行优化更新操作
3. 通过在所有其他（非最大）级别合并更多的数据来优化短范围查询。

Dostoevsky 自适应删除多余合并


### tiering 策略和 leveling 策略比较

T: 层级之间的大小比
L: 层数
B: 每个块的条目数

1. 更新 Update
	* 更新的主要开销在 merge，可以简单理解为 leveling 需要两层一起 mrege，每次更新的大小涉及 `(T + 1)` * X 个 sstable，开销自然比 tiering 一层 `X` 更大。leveling 写放大会比 tiering 更严重一点。
	* tiering `O(L / B) `
	* leveling  `O(L * T / B)`
	* 更新成本来自所有层。
2. 点查
	* 点查的主要开销原因在查询不存在的键，而且正比于布隆过滤器的假正率的期望值。读取的话由于 tiering 每一层 run 的数量是 leveling 的 T 倍，tiering 读放大比 leveling 更加严重。
	* tiering  `O(L * e^(-M/N) * T / B)`
	* leveling `O(L * e^(-M/N) / B) `
	* 在 Monkey 论文表明，高层配置比低层更低 bit/entry，可以让最大层的假正率常数增长，在更低层则指数递减（我目前还不知道为什么），最终可以让 leveling 的复杂度降低到`O(e^(-M/N) / B) `并且 tiering 复杂度降低到 `O(e^(-M/N) * T / B)`。
	* 点查成本的主要分布在最大的层。
3. 空间放大
	* leveling  `O(1 / T)`
	* tiering `O(T)`
	* 空间放大成本的主要分布在最大的层。
4. 范围查询
	* 选择性 s = 在目标范围内的所有 runs 中唯一条目项数
	* 短范围查询
		* leveling  `O(T)`
		* tiering `O(T * L)`
		* 短范围查询的开销来自所有层。
	* 长范围查询
		* leveling  `O(s / B)`
		* tiering `O(T * s / B)`
		* 长范围查询的开销来自最大层。
	
	|测量点|主要分布|
	|-|-|
	|更新|所有层|
	|点查（文中更倾向零查）|最大层|
	|空间放大|最大层|
	|短范围查询|所有层|
	|长范围查询|最大层|
	
	
### lazy leveling 
	* 配置:
		*  最大层是 leveling 其他层是 tiering。
		*  调整每一层的假正率。
	* 复杂度：
		1. 零查开销（布隆过滤器命中，但不存在的数据） `O(e^(- M / N))`  == leveling 
		2. 点查开销 `O(1)` == leveling
		3. 短范围查询开销 `O(1 + (L - 1) * T)` > leveling && << tiering 
		4. 长范围查询开销 `O(s / B)` = leveing
		5. 更新开销 `O(L + T) / B` << leveling && > tiering 
		6. 空间放大开销 `O(1 / T)` = leveling
	* 适合由更新，点查，长范围查找的工作负载。

### Fluid LSM-tree
支持切换和合并策略

较小层级 阈值 K 个 run
最大的层级 阈值 Z 个 run

leveling K=1 Z=1
tiering K=T-1 Z=T-1
lazy-leveling K= T-1 Z=1

lazy-leveling K 减少 -> leveling  (merge more)
lazy-leveling Z 增加 -> tiering (merge less)

问题是如何通过调整 K,Z,T 和其他一些参数, 去找到最佳调优。

### Dostoevsky

Dostoevsky 就迭代的调参 K,Z,T

自变量
* K Z T
中间变量
* 更新代价 W
* 零点查代价 R
* 非零点查代价 V
* 范围查找代价 Q
因变量
* 最坏吞吐量 r

r 通过对 WRVQ 加权算出来

用来寻找最佳 Fluid LSM-tree 调优参数

扩展性强
改善访问倾斜
低空间放大
稳定范围查找
跨内存预算稳定性能

