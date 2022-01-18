本文是对往年所有压实策略的一个整体分析，并将策略区分成四个方面进行讨论，通过对工作负载和调优旋钮的调整，对比每个方面的策略的多个性能指标的变化，展示给我们如果寻找最适合自己的压实策略。算是一篇很好的实验文。

压实策略的四个设计原语：
1. 压实触发器：何时重新组织数据布局？When
2. 数据布局：如何在物理存储上组织数据布局？How
3. 压实粒度：布局重组每次移动多少数据？How much
4. 数据移动策略（文件选择策略）：重组是移动哪个数据块？Which
	
### 压实触发器策略
1. 层级饱和度达到阈值
2. 排序 run 的数量达到阈值
3. 文件脏程度（在一层呆了太久）
4. 空间放大程度达到阈值
5. 文件到达过期时间

### 数据分布策略
通常分为 leveling 和 tiering

压实和写入之间会竞争磁盘带宽，因此我们可能会在压实热度大小进行权衡（比如 sam or wam)

数据分布策略由需要的压实热度驱动（我们的压实需求有多急切）
对数据分布的效果是控制每层的 sort run 数量

1. leveling: 每层一个 sort run
2. tiering: 每层多个 sort run
3. 1-leveing: L1: tiering 其他 leveling
4. L-leveing: Lmax leveling 其他 tiering
5. Hybrid: 每一次都可以独立的使用 tiering 或 leveling

### 压实粒度策略

随着单次压实粒度的变大，空间放大程度减小，延迟峰值加剧。

部分压实不能在根本上改变压实导致的数据移动总量，但是会随着时间均匀的移动数据，防止不必要的延迟峰值。

1. Level: 两层的所有数据
2. Sorted runs: 一层的所有 sort runs
3. Soerted file: 一次一个 sort file
4. Several sorted files: 多个 sort file

### 数据移动策略 （文件选择策略）

1. Round-robin: 轮询方式选择文件
2. Least overlapping parent: 最小和 LN+1 层重叠的文件
3. Least overlapping  grandparent: 最小和 LN+2 层重叠的文件
4. Codest: 最小最近访问文件
5. Oldest: 一层中的最老文件
6. Tombstone density: 文件中的墓碑数量达到阈值
7. Tombstone-TTL: 文件有过期墓碑（为了及时删除）

### 性能指标
1. 压实延迟
	1. 选择压实文件
	2. 读取指定文件到内存中
	3. 归并排序
	4. 将结果写回磁盘
	5. 使旧文件失效
	6. 更新元数据和 manifest 文件
2. 写放大
	* 一个条目在其生命周期内被重新写入的次数。
3. 写延迟
	* 设备带宽利用率驱动，取决于：
	1. 压实导致的写延迟
	2. 持续的设备带宽
4. 读放大：点查而读取的页面总数与理想情况下应当读取的页面之间的比率。
5. 点查延迟：文件的位置会影响点查。
6. 范围查询延迟：范围查询选择性，数据分布会影响它。
7. 空间放大：数据分布，压实粒度，数据移动策略会影响它。
8. 删除性能：时间限制内持续删除条目的程度。关乎隐私安全。

### 基准测试方法
改变输入参数的方法：
1. 调整工作负载
	1. 插入，更新，点查（空，非空），范围查找（选择性），删除。
	2. 数据分布：均匀分布，正态分布，Zipfian 分布。
	3. 基准测试方式：YCSB 基准测试，插入基准测试
2. 使用调优旋钮
	1. 内存缓冲区大小
	2. 块缓存大小
	3. 树的大小比例

### 性能影响

#### 空数据库读写工作负载

* 写	
1. 压实导致大量数据移动。
2. 部分压实可以减少数据移动但会增加压实次数。
3. 全量整层压实会有更高的平均压实延时。
4. Tiering 会有最高的尾部写延时（写延时高峰）
	* 因为当多个连续级别接近饱和的时候，缓冲区刷新可能导致级联压实。
* 读
1.  Tiering 的点查延迟最高，Full-Level 最低。
	1.  Tiering 非零查的延迟会相对更差。
	2.  缓存大小和缓存策略会影响（利于） Tiering 的点查延迟。
	3.  数据移动策略不会显著影响点查延迟，受益于布隆过滤器和块缓存。
	4.  空查询和非空查询比率达到平衡时，性能更差。	
	5.  读放大受到块缓存大小和文件结构的影响。tiering 更严重。
	6.  压实策略对范围扫描的影响也很大。tiering 更严重。	

#### 混合工作负载

改变插入负载
改变点查分布
改变更新比率
改变删除比率
改变插入数量
改变条目大小

都出现了一些对应的性能变化，暂且不列出来，，，有需求了再回来瞅一瞅。

### 调优旋钮影响

1. 缓冲区越大，磁盘上文件的大小越大，每次压实移动会需要更多的数据，压实延迟增加。但 Tier 的平均压实延迟反而越来越好。
2. 改变逻辑页面的大小会改变每个页面的条目数，每个文件的索引块大小会随着索引页面的增多而增加，而导致 i/o 增加。逻辑页面大小的增加会减少平均压实的延迟。页面大小增大，文件的索引块大小减小，点查找获取元数据块的平均 i/o 次数减少。

### 总结
了解你自己的压实策略，寻找工作负载适应的策略，探索新压实策略。