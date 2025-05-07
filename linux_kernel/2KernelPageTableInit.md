# 内核页表初始化
> 1. 暂时用 `2.6.11` x86 平台的内核来做讲解。
> 2. 从 `bois` 中获取各种配置信息，我也不懂，所以仅介绍结果
> 3. 该篇文章已假设你了解了内核的内存总体的分布，如果没有请参考我的这篇文章
> 4. 我们以常见的非连续性内存模型来讲解，即 `CONFIG_DISCONTIGMEM` 已定义的代码。

## 物理内存分布

首先可用的物理内存（地址）不是连续分布的，有很多空洞。因为这些空洞要么不可用，要么被其他外设占用，所以在内核初始化时，必须从bois中获取可用于物理内存。另外由于内存只识别 4KB 对齐的内存，所以那些处于被对齐位置的小于 4KB 的内存也不能为内核所用。相关代码如下：

1. `e820` 映射表

内存把可用的物理地址分布存放在此数据结构中

```c
struct e820map {
    int nr_map;
    struct e820entry {
	unsigned long long addr;	/* start of memory segment */
	unsigned long long size;	/* size of memory segment */
	unsigned long type;		/* type of memory segment */
    } map[32];
};
/*内核内存映射表，将根据其中的内存分布初始化内核页表*/
struct e820map e820;
```

2. 从 `bois` 参数中获取物理内存

内核完成实模式的启动后，在开启分页机制需要获取物理内存的分布，函数如下：

```c
static char * __init machine_specific_memory_setup(void)
{
    /* 一般情况 copy_e820_map() 从bois参数中获取内存分布回成功，
     * 如果是失败则会使用默认的内存分布进行初始化 */
	if (copy_e820_map(E820_MAP, E820_MAP_NR) < 0) {
		unsigned long mem_size;
		if (ALT_MEM_K < EXT_MEM_K) {
			mem_size = EXT_MEM_K;
		} else {
			mem_size = ALT_MEM_K;
		}

		e820.nr_map = 0;
		add_memory_region(0, 0x9f000, E820_RAM);
		add_memory_region(1024*1024, mem_size << 10, E820_RAM);
  	}
	return who;
}

static int __init copy_e820_map(struct e820entry * biosmap, int nr_map)
{
	do {
		unsigned long long start = biosmap->addr;
		unsigned long long size = biosmap->size;
		unsigned long long end = start + size;
		unsigned long type = biosmap->type;

		/* 地址溢出64位就会失败 */
		if (start > end)
			return -1;
        /*添加到映射表中*/
		add_memory_region(start, size, type);
	} while (biosmap++,--nr_map);
	return 0;
}

static void __init add_memory_region(unsigned long long start,
                                  unsigned long long size, int type)
{
	int x = e820.nr_map;
	e820.map[x].addr = start;
	e820.map[x].size = size;
	e820.map[x].type = type;
	e820.nr_map++;
}
```

3. 查找最大页帧号

在内核很多地方都需要知道最大的可用物理地址，我们可以从上面得到内存分布中计算出最大可用的物理地址，相比物理地址，内核更喜欢使用页帧号来表示物理地址。计算出的页帧号存储在 `max_pfn` 中。

```c
void __init find_max_pfn(void)
{
	int i;
	max_pfn = 0;
	for (i = 0; i < e820.nr_map; i++) {
		unsigned long start, end;
		start = PFN_UP(e820.map[i].addr);
		end = PFN_DOWN(e820.map[i].addr + e820.map[i].size);
		if (start >= end)
			continue;
		if (end > max_pfn)
			max_pfn = end;
	}
}
```

4. 最小页帧号

最小页帧号与内核镜像的映射有直接关系，由于这一部分与硬件、内核的编译、汇编等有直接的关系，本人能力有限，先仅给出一个结果。内核将从 物理内存 `1MB` 的位置出开始放置内核镜像的数据（代码段和数据段），这些数据之后的物理内存才能被用于其他（包括后续初始化以及运行中的内存分配）。启用分页后，所有寻址都通过页表进行，这部分内核镜像数据也不例外，所以需要为了分页下访问这部分数据的而建立初始化页表。从 `vmlinux.lds.S` 中我们可以看到镜像的分布：

```
pfn = 0
+----------+-------------------------------+-----+-----+-----+
|  hole    | text | data | init data | ... | bss | pg0 | pg1 |
+----------+-------------------------------+-----+-----+-----+
                                           |     |
                                 +---------+     +------------+
								 | ... | swapper_pg_dir | ... |
								 +----------------------------+
```

以上的分布，`hole` 是空洞部分，这部分物理地址可能被其他硬件使用，为了将内核镜像连续的映射，所以选择跳过；后续就是内核镜像的各个数据段，直到 `pg0` 结束。其中 `bss` 段中包含了 全局目录页的空间 `swapper_pg_dir`，`pg0` 被用作建立初始化页表的 中间目录页，如果内核的镜像需要 `2张` 中间目录页，也就是大于 `4MB` 小于等于 `8MB`，那么还需要一页的数据 `pg1`，具体使用多少张中间目录页，以此类推。用作页表初始化的中间目录页之后的物理地址就是在*内核初始化起初*可以最小页帧的起始位置，使用 `init_pg_tables_end` 来记录。后续这个值会因为其他初始化变化。

5. 最大直接映射页帧号

我们知道32位平台内存只能最多映射 `896MB` 的内存，一切大于这个值的物理内存都视为高端内存，内核通过对物理内存大小的判断来计算直接映射的结束位置，并存储在 `max_low_pfn` 中。

```c
unsigned long __init find_max_low_pfn(void)
{
	unsigned long max_low_pfn;

	max_low_pfn = max_pfn;
	/*MAXMEM_PFN(保留高端内存后的内存大小，一般情况下是896MB)可以直接映射的内存页帧最大编号*/
	if (max_low_pfn > MAXMEM_PFN) {
        /*如果系统可用物理内存大于896mb
		 *减去 MAXMEM_PFN 就得到高端页数
		 */
		highmem_pages = max_pfn - MAXMEM_PFN;
		max_low_pfn = MAXMEM_PFN;
	} else {
		/*如果系统可用物理内存小于等于896MB，就没有高端页框*/
		highmem_pages = 0;
	}
	return max_low_pfn;
}
```

我们不关心启动项参数 `highmem=` 带来的影响，所以移除了一些判断代码。

## `NUMA` 与物理内存

`NUMA` 全称 `Non-Uniform Memory Access`，译为 `非一致性内存访问`，简单的来说就是将内存分组，多 `CPU` 系统可以在访问离它越近（电路上）的内存时，速度越快。离它最近的称为本地内存，这些分组称为 `node` 节点并且被编号 `nid`，内核一般情况只要是编号都使用位图来实现。

1. 明确节点管理页帧范围

内存被分组，但是物理地址是统一的，所以内核在抽象这些 `节点` 前，必须先搞清楚每个节点的物理地址起止，同样内核使用页帧号来表述，相关函数如下：

```c
static inline void get_memcfg_numa(void)
{
    /*初始化node和其中的页帧起止号*/
	if (get_memcfg_numaq())
		return;

    /* 如果失败则强制使用平坦模式，
     * 即使有多真是的node，我们也只视为仅有一个node，管理所有的页帧 */
	get_memcfg_numa_flat();
}

int __init get_memcfg_numaq(void)
{
    int node;
	struct eachquadmem *eq;
	struct sys_cfg_data *scd =
		(struct sys_cfg_data *)__va(SYS_CFG_DATA_PRIV_ADDR);

    /*
     * 1. 查明哪些node可用
     * 2. 查明可用node对应的页帧起止号
     */
	nodes_clear(node_online_map);
	for_each_node(node) {
		if (scd->quads_present31_0 & (1 << node)) {
			node_set_online(node);
			eq = &scd->eq[node];
			/* 将 MB 为单位的地址 转换为 4K 为单位的地址 （页帧号）*/
			node_start_pfn[node] = MB_TO_PAGES(
				eq->hi_shrd_mem_start - eq->priv_mem_size);
			node_end_pfn[node] = MB_TO_PAGES(
				eq->hi_shrd_mem_start + eq->hi_shrd_mem_size);
		}
	}
}

int __init get_memcfg_numa_flat(void)
{
    /*找到最大页帧号*/
	find_max_pfn();
	node_start_pfn[0] = 0;
	node_end_pfn[0] = max_pfn;
    /* 将所有内存都视为同一个节点，就可以使用同一个内存抽象模型*/
	nodes_clear(node_online_map);
	node_set_online(0);
	return 1;
}
```

2. 建立页帧号与节点号的映射

内核在物理内存的管理上都使用页帧号，在使用过程中，需要在给出页帧号时能快速的得到其所属的节点号，从而得到后面将介绍的 节点抽象数据结构 `struct pglist_data` 的描述符，来进行相关的操作。

节点从 `0` 开始编号，且每个节点能管理内存最小值必须大于等于 `256MB`，而且一定是这个值的倍数，即不存在一个连续的、按 `256MB` 对齐的物理内存属于2个节点的情况。那么对于当前版本内核最多能管理 `64GB` 的内存来说，最多可以有 `64*1024/256` 个节点，即为 `256` 个，所以节点号恰好能使用 单字节 来表示，内核使用 `s8 physnode_map[256]` 来存储这些编号。

```c
#define MAX_NR_PAGES 16777216 /*最大页帧数量，即 64GB /4K 的值*/
#define MAX_ELEMENTS 256 /*映射表槽位数*/
#define PAGES_PER_ELEMENT (MAX_NR_PAGES/MAX_ELEMENTS) /*每个槽位能映射的页帧数*/

unsigned long __init setup_memory(void)
{
    unsigned long pfn;
    ...
    /*遍历在线节点*/
    for_each_online_node(nid) {
        /*按每个槽位能映射页帧数划分节点管理的节点*/
		for (pfn = node_start_pfn[nid]; pfn < node_end_pfn[nid]; pfn += PAGES_PER_ELEMENT) {
            /*建立映射，pfn/PAGES_PER_ELEMENT 就得到了页帧在映射表中的下标，其中存储的就是节点号*/
			physnode_map[pfn / PAGES_PER_ELEMENT] = nid;
		}
	}
    ...
}
```

3. 内存与节点分布图

经过上面的映射后，你会得到这样的一个结果，假设我们有2个 `NUMA` 节点，其中一个节点管理了 `512MB` 的空间，另一个管理了 `768MB` 的空间，分布如下：

```
        +---------------+--------------------------+
        |  node 0       |         node 1           | 节点分布
        +-------+-------+--------+--------+--------+
        | 256MB | 256MB | 256 MB | 256 MB | 256 MB | 物理地址分布
        +-------+-------+--------+--------+--------+-------+-------+-----+
        |  0    |   0   |   1    |   1    |   1    |  -1   |  -1   | ... |  physnode_map[] 的结果
        +-------+-------+--------+--------+--------+-------+-------+-----+
        |<--------------------------- 256 ------------------------------>|
```

有了这个映射表页帧号到节点号的转换就可以是

```c
static inline int pfn_to_nid(unsigned long pfn)
{
	/*按256取下边界的倍数既可以*/
	return((int) physnode_map[(pfn) / PAGES_PER_ELEMENT]);
}
```

## 内存管理模型的抽象

内核通过按页大小来管理内存，上面我们介绍了 `NUMA` 的概念，页属于不同的物理节点，为了更好的高效实用内存，我们要对页和`NUMA`节点进行抽象，用于描述他们的结构和状态。

1. 页框

内核必须记录物理页（页框）的当前状态。比如，内核必须知道哪些页已经被使用，还区分这些已分配的页中哪些页已经用于用户态进程的数据存储，哪些又用于了内核本身的数据存储；哪些页是空闲的。虽然页表上有一些标志为可以用于表示状态，但页的状态很多，这些位是不够；而且为了高效的分配释放页，页表的数据结构也不适合。所以我们为了对页进行管理，需要对页抽象一个描述符。

```c
struct page {
	page_flags_t flags; /*原子状态*/
	atomic_t _count; /* 计数器 -1时空闲 page_count() = _count + 1 即为使用者的数目 */
	atomic_t _mapcount; /**< 1.页框被页表引用的次数：初始值 -1，非共享0， 共享>0 。 */
	unsigned long private;	/**< 1.如果PagePrivate被设置，运用到buffer_heads。
					* 2.如果PageSwapCache被设置，运用到swp_entry_t。
					* 3.如果页框未被分配，连续页的首页中，指示伙伴系统中的阶数。
					* 4.如果是组合页，则指示头页的描述符。
					*/
	struct address_space *mapping;	/**< 1.为NULL，可能属于交换分区
					* 2.非空且最低位为1，表明为匿名页，（去掉最低位）存放的是anon_vma的指针
					* 3.非空且最低位为0，表面为映射页，存放的是address_space指针
					*/
	pgoff_t index;	/**< 1. 匿名页或映射页所在进程线性区起始地址(vma也是页对齐的）的页数偏移
					 *   2. 如果用于进程的页表目录页mm_struct->pgd，存储的是进程的mm_struct
					 *   3. 如果是组合页，则第二个页描述符的此字段为组合页的 order 值
					 */
	struct list_head lru;		/* 连接到当前页属于的容器中的节点
					* 1. 当页被分配给slab时， next 为 kmeme_cache 描述符
					*    prev 为 slab 描述符
					*/
}
```

2. 节点描述符

上面介绍过 `NUMA` 的概念，在这个节点抽象之中，还存在另一层抽象。那就是内存区域的抽象，因为计算机结构体系的关系，很多与外设交换数据的物理地址必须是位于低地址处的内存，比如内核分配一张物理页，然后与老式的网卡交换数据（收发网络数据包），这个物理页必须位于 低 `16MB` 之下，因为网卡只能对 `16MB` 的物理地址寻址，称为 `直接内存存取`(`DMA`)，然后从其中读写数据。所以现在我们将抽象 `3` 个内存管理区，记着 `ZONE`，分别为 `ZONE_DMA`、`ZONE_NORMAL` 和 `ZONE_HIGHMEM`。其中 `ZONE_DMA` 就是低 `16MB` 的内存管理区，`ZONE_NORMAL` 是普通的直接映射的内存管理区，在内核内存分布文章中，我们介绍过，32位平台由于内核段的虚地址只有 `1GB` 其中的高端的 `128MB` 用着了动态的映射和其他映射，只有 `896MB` 的内存可以直接的映射在内核页表中，所以当物理内存大于 `896MB` 时就需要一个专门的区域来管理，这部分内存没有被映射到内核页表，访问他们的时候必须映射到动态映射的虚地址上（后续我们会逐步介绍），`ZONE_HIGHMEM` 就是管理高端内存的管理区。

内存管理区数据结构，仅列出了内框基本管理（伙伴系统）的主要字段：

```c
struct zone {
	unsigned long		free_pages;/**< 总的空闲页数量*/
	struct per_cpu_pageset	pageset[NR_CPUS]; /**<每个CPU上的高速页分配缓存cache a page on per cpu*/

	spinlock_t		lock;/**<该描述本身的保护锁*/
	struct free_area	free_area[MAX_ORDER];/**< 标记伙伴系统中的空闲页，这个数组的
            * 每一个元素中的链表由2^k个连续页的起始描述符
            * 串联组成，其中k对应着数组的下标
            * mark free page on buddy system
            */

	struct pglist_data	*zone_pgdat; /**<所属内存节点*/
	struct page		*zone_mem_map; /**<属于该区域的页框数组起始地址*/
	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;/**<属于该区域的起始页框号*/
};
```

`numa` 节点描述符数据结构，同样列出主要字段：

```c
typedef struct pglist_data {
	struct zone node_zones[MAX_NR_ZONES];/**<节点区域描述符数组*/
	struct zonelist node_zonelists[GFP_ZONETYPES];/**<
			*分配区使用的链表优先顺序，
			*包含所有numa上的节点中的区域
			* @see build_zonelists()
			*/
	int nr_zones;/**<节点区域描述符的个数
				  * 表示 node_zones[] 数组前几个被初始化过
				  */
	struct page *node_mem_map;/**<属于该节点页框数组起始地址*/
	struct bootmem_data *bdata;/**<初始化时的分配器*/
	unsigned long node_start_pfn;/**<节点内在全局页框中第一个页框值*/
	int node_id;/**<节点标识符*/
	struct pglist_data *pgdat_next;/**<下一个节点
									* @see contig_page_data 单节点的架构
									*/
} pg_data_t;
```

内核通过 `struct pglist_data *node_data[]` 数组在作为节点描述符的入口寻找，置于为何使用指针，而不是对象实例，是为了高效性考虑的，各个节点对应的描述以及其中的页描述符都会使用相应节点的内存来初始化。

3. 带有内存管理区域的内存分布图

现在我们必须明确带管理区的抽象模型的内存分布图，上面我们说过内存管理区是将物理地址按用途和是否直接映射划分的，这个划分是线性的，即不存在所有节点都拥有3个区域，如果将其与上面介绍的内存与节点分布图融合会得到如下的分布图：

```

		+----------------+-------------------------+
		|     node 0     |         node 1          |
		+----------+-----+-----+-------------------+
		|   16Mb   |496Mb|384Mb|      128MB        |
		+----------+-----------+-------------------+
		| ZONE_DMA |ZONE_NORMAL|   ZONE_HIGHMEM    |
		+----------+-----------+-------------------+

```

所以 `node0` 上有 `ZONE_DMA` 和部分 `ZONE_NORMAL` 的内存，而 `node1` 上有剩余部分的 `ZONE_NORMAL` 的内存和整个 `ZONE_HIGHMEM` 的内存。

4. 计算页描述和节点描述符消耗的内存

内存初始化的第一步就是要逐步计算出各种描述内存模型数据结构的内存大小，逐步扣除这部分内存的消耗后，最终得到可以用于后续运行时的页框数量。首先要明确有了分页的机制，通过页表我们很容易的安排各个 `NUMA` 节点以及页描述符的虚地址，因为页表可以将物理地址映射到不同的虚地址上。其次上面我就介绍过，内核为了效率，希望各个节点的管理数据结构使用节点自身的内存来存储，所以 `node_data[]` 才是指针数组。内核通过 `calculate_numa_remap_pages()` 和 `allocate_pgdat()` 来建立内存管理数据结构的虚地址模型，我们先给这些数据结构的虚地址分布图，为了更清楚的描述，我们选择使用4个节点：

```
                             paddr                                   vaddr
highstart_pfn+reserve_pages <= 0xf8000000 <----+-----------+-----------------------------------+
                                               |           |                             ^  ^  ^
                                               |           |                             |  |  |
                                               |   NODE1   |           node_remap_offset[1] |  |
                                               |           |                             |  |  |
                                               |           |                             v  |  |
                                               +-----------+-> node_remap_start_vaddr[1]-+  |  |
                                               |           |                                |  |
                                               |           |                                |  |
                                               |   NODE2   |              node_remap_offset[2] |
                                               |           |                                |  |
                                               |           |                                v  |
                                               +-----------+-> node_remap_start_vaddr[2]----+  |
                                               |           |                                   |
                                               |           |                                   |
                                               |   NODE3   |                 node_remap_offset[3] 
                                               |           |                                   |
                                               |           |                                   v
            var:max_low_pfn/highstart_pfn <----+-----------+-> node_remap_start_vaddr[3]-------+
                                               |           |
                                               |direct map |
                                               |           |
											   +-----------+
                                               |tmp bootmem| used by bootmem alloctor
                          var:min_low_pfn(new)-+-----------+
                                               |   NODE0   | only size of pg_data_t
                              min_low_pfn(old) +-----------+ (pg_data_t *)(__va(min_low_pfn << PAGE_SHIFT))
```

简单的描述一下，就是内核使用最低直接映射页框 `min_low_pfn` 后续一些物理页来存储 `node0` 的节点描述符，这个位置相对固定，因为内核镜像不超过 `256MB` 的大小，而节点管理的最小内存范围至少 `256MB`。然后从最高直接映射虚地址向低地址，分配一些虚地址给除0号节点的各个节点描述符和其管理的页框的页框描述符使用，这里只划分了虚地址，之后还需要从各自的内存区域分配物理页，用这些物理页初始化对应的页表，才能初始化其中的数据（才能寻址）。在这个过程中会反复的调整 `min_low_pfn` 和 `max_low_pfn` (最大直接映射的页帧号)，这两个值中间的物理页框才是运行时使用的物理内存。

计算运行时可用页框起止的总代码片段：

```c
unsigned long __init setup_memory(void)
{
	...
	/*计算管理每个节点（除0号节点描述符）需要的页帧描述符和节点描述符的内存空间，并计算相关的数据*/
	reserve_pages = calculate_numa_remap_pages();
	/* 计算启动后的直接映射的最低页帧号，向上对齐init_pg_tables_end （启动时的最低可用页号）*/
	system_start_pfn = min_low_pfn = PFN_UP(init_pg_tables_end);

	/* 计算最大页帧号max_pfn*/
	find_max_pfn();
	/* 计算启动后直接映射的最高页帧号（除去高端内存的页帧数和内存管理的页帧数），
	 * 最大直接映射的高虚地址用于 NODE 的页描述符和 node 描述符的数据结构内存。
	 * 虽然用于页和页管理的内存物理上不连续，来自各个node的尾部，
	 * 当时都被映射到直接映射区的尾部（0xf8000000向地址方向）
	 */
	system_max_low_pfn = max_low_pfn = find_max_low_pfn() - reserve_pages;
	/*高端页框的结束页框一定为最高可用页框*/
	highstart_pfn = highend_pfn = max_pfn;
	/* 如果存在高端内存那么条件成立，调整高端页框的起始页框号
	 * 只要是非直接映射的可用内存都是高端，即使是用于页帧和节点管理的页（这些页被标记为不可用）*/
	if (max_pfn > system_max_low_pfn)
		highstart_pfn = system_max_low_pfn;
}
```

首先是计算除0号节点外给页描述符和节点描述符使用的内存空间：

```c
static unsigned long calculate_numa_remap_pages(void)
{
	int nid;
	unsigned long size, reserve_pages = 0;

	for_each_online_node(nid) {
        /*不计算启动时的node，这个node的描述符固定位置*/
		if (nid == 0)
			continue;
		/*
		 * 计算管理每个节点需要的页帧描述符和节点描述符的内存空间
         * 每个节点的尾部页帧用于管理页帧
		 */
		size = (node_end_pfn[nid] - node_start_pfn[nid] + 1) 
			* sizeof(struct page) + sizeof(pg_data_t);
		/*
		 * 将这个大小对齐为一个 pmd 表示的虚地址空间大小的倍数(4MB 的倍数)
		 * 使这段用于管理页帧的数据结构的虚地址可以映射在一个或多个PMD中
		 * 并将对应的物理内存作为4MB的巨型页进行映射
		 * LARGE_PAGE_BYTES 为 4MB
		 */
		size = (size + LARGE_PAGE_BYTES - 1) / LARGE_PAGE_BYTES;
		/* 再将这个数量转换为页帧数*/
		size = size * PTRS_PER_PTE;
		node_remap_size[nid] = size;
		/*累加保留地址的大小*/
		reserve_pages += size;
		node_remap_offset[nid] = reserve_pages;
		/*修改该节点可用的页帧结束编号*/
		node_end_pfn[nid] -= size;
		/*记录管理内存空间的起始页帧号*/
		node_remap_start_pfn[nid] = node_end_pfn[nid];
	}
	return reserve_pages;
}
```

简单的说明，以上就是遍历各个节点，然后从各个节点管理的物理页保留需要大小的空间来作为各个管理数据结构描述符的内存，而且是从各个节点的尾部保留的。同样内核使用页框数和页框号来描述这些值。`node_remap_size[]` 数组是保存各自节点为此保留的页框数量；`node_remap_start_pfn[]` 是各自保留页框的起始页号；`node_remap_offset[]` 是保留页框数量的累加值。在计算的过程中或动态的调整各个节点能被运行时使用的页框结束页帧号，`node_end_pfn[]` 被调整了。下面给出简单的示意图：

```
阴影部分就是划分管理数据结构使用的页框：

                                      old node_end_pfn[1]
									     ^
                                         |
           +-----------------------------+----------------+-----------------+
           |           Node1             |   Node2        |    Node3        |
           +-----------------------------+----------------+-----------------+
           |          |//////////////////|        |///////|         |///////|
           +----------+------------------+----------------+-----------------+
           |          |node_remap_size[1]|
           v          |
   node_start_pfn[1]  |
                      |
					  v
                   new node_end_pfn[1]
                   node_remap_start_pfn[1]

假设每个节点都管理一样的内存大小，管理数据结构占用 50 个页，那么：
   node_remap_offset[1] = 50
   node_remap_offset[2] = 100
   node_remap_offset[3] = 150
```

然后初始化 `min_low_pfn` ，刚开始时它等于 初始化页表目录页使用后的 物理地址，然后初始化 `max_low_pfn`，他是高端物理内存地址之下除去保留给节点描述符合页描述符空间后的值，然后是 `highstart_pfn` 高端内存的起始页号。需要注意的是被保留的页其实不是连续的，但是他占用的虚地址是连续的（内核虚地址与物理地址仅有一个偏移），原本可以用于直接映射的物理地址得不到映射，所以减去这些保留页数量。

现在我们初始化 `struct pglist_data *node_data[]` 数组的虚地址了，注意映射在高端虚地址的节点管理描述符 还不能寻址，因为还没有初始化那里的页表，我们进行了一些代码的合并，片段如下。

```c
unsigned long __init setup_memory(void)
{
	...
	for_each_online_node(nid) {
		if (nid) {
			/*非零号节点的虚地址不是固定的，而是根据管理的页框数量和起始虚地址计算出来的*/
			node_remap_start_vaddr[nid] = pfn_to_kaddr(
				(highstart_pfn + reserve_pages) - node_remap_offset[nid]);
			node_data[nid] = (pg_data_t *)node_remap_start_vaddr[nid];
		} else {
			/*上面我介绍过，min_low_pfn 之后一些页用于了 0号节点 pg_data_t 描述符*/
			node_data[nid] = (pg_data_t *)(__va(min_low_pfn << PAGE_SHIFT));
			/*必须对齐调整最低可用的直接映射页帧号*/
			min_low_pfn += PFN_UP(sizeof(pg_data_t));
			/*可以寻址？但是我没有找到相关的代码，我们假设这部分空间以被初始化页表包含*/
			memset(node_data[nid], 0, sizeof(pg_data_t));
		}
	}
	...
}
```

## 初始化临时内存分配器
> 暂时不列出初始化分配器的实现代码，因为这个分配器功能是否有限，而且不支持按NUMA节点分配内存。

通过上面的操作我们已经分配好了节点管理描述符和页描述符的虚地址空间，但是还不能对这些虚地址寻址，因为没有分配物理地址。没有这些数据结构的支持，就不能使用伙伴系统来分配页，而现在我们又需要分配页，这就引出了我们需要建立一个简单的页分配器。

1. 位图模式的也分配器

我们抽象一个简单的基于位图的分配器来记录页的使用状态，数据结构如下：

```c
typedef struct bootmem_data {
	unsigned long node_boot_start; /**< 可分配的起始物理地址*/
	unsigned long node_low_pfn; /**< 可分配最后一个页帧号*/
	void *node_bootmem_map; /**<
							 * 用于管理位图的起始虚拟地址，
							 * 该段地址可以在启动完成后重用于buddy系统，
							 * 位图：0表示未分配，1表示已分配
							 */
	unsigned long last_offset; /**< 上次分配的页帧中最后一页中，已使用内存的偏移（后面的空间未使用）*/
	unsigned long last_pos; /**< 上次分配的最后一个页帧号*/
	unsigned long last_success;	/**< 上一次分配的第一个页帧号，提高搜索速度 */
} bootmem_data_t;
```

PS:如果你不知道位图是怎么回事请自行百度~

2. 分配器的初始化

内核将在计算了保留页后，使用0号节点的 `bdata` 字段来存储临时分配器的描述符，这个描述符是在镜像的 `bss` ，所以可以寻址。

```c
unsigned long __init setup_memory(void)
{
	...
	/*将0号节点用于启动内存分配器，所以几乎只能使用0号节点上的内存，一个致命的缺陷*/
	node_data[0]->bdata = &node0_bdata;

	/*
     * 将直接映射的页都初始化到 bootmem 内存管理器中
	 */
	bootmap_size = init_bootmem_node(node_data[0], min_low_pfn, 0, system_max_low_pfn);
	...
}
```

`init_bootmem_node()` 初始化 位图页分配器 描述符，*最小直接映射页框号作为后的一些页作为位图的空间，这些物理地址已经被映射到初始化页表中了，所以可以被寻址*。

**最重要的就是第一步初始化后，分配器内没有可以分配的页，然后进一步初始化：将可用页释放到其中。后面最终的伙伴系统页框分配器也是这个思路。**

3. 收集可以分配页

现在将可以分配的也释放到分配器中，用于后续的分配。

```c
unsigned long __init setup_memory(void)
{
	...
	/*将可用的页帧号映射到bootmem页帧管理位图中：0表示未分配，1表示已分配*/
	register_bootmem_low_pages(system_max_low_pfn);
	...
}
```

暂时将所有能直接映射的页都视为可以分配的，但是从上面的介绍，我们知道有很多页都是已使用，或根本不能使用的，后续将修正这个问题。

```c
static void __init register_bootmem_low_pages(unsigned long max_pfn)
{
	int i;

	for (i = 0; i < e820.nr_map; i++) {
		unsigned long curr_pfn, last_pfn, size;
		/*非 e820_ram 不是可用物理内存*/
		if (e820.map[i].type != E820_RAM)
			continue;
		/*排队一个区间中可用的物理内存，按页大小对齐*/
		curr_pfn = PFN_UP(e820.map[i].addr);
		if (curr_pfn >= max_pfn)
			continue;
		/*向下收缩，成为了比*/
		last_pfn = PFN_DOWN(e820.map[i].addr + e820.map[i].size);
		if (last_pfn > max_pfn)
			last_pfn = max_pfn;
		/*相等，则没有可用的页*/
		if (last_pfn <= curr_pfn)
			continue;
		size = last_pfn - curr_pfn;
		/*是否当分配器中*/
		free_bootmem_node(node_data[0], PFN_PHYS(curr_pfn), PFN_PHYS(size));
	}
}
```

我们通过扫描 `e820` 映射表中的所有可以使用的内存，然后使用 `free_bootmem_node()` 将其释放回分配器中，这样分配器就有可以使用的页了。

现在我们需要保留我们已经使用了的页，包括被分配自身位图使用了的页、初始化页表使用的页、内核镜像使用了的页、为硬件保留的物理地址（可以当着保留页）。

```c
unsigned long __init setup_memory(void)
{
	...
	/*
	 * 计算第一部分保留页的起止时，考虑如下：
	 * 内核镜像从1MB开始映射，包括初始化页表使用的页，直到 min_low_pfn 处，bootmap_size 是位图使用的页空间
	 * 加上 (PAGE_SIZE - 1) 是为了对齐。
	 */
	reserve_bootmem_core(node_data[0], HIGH_MEMORY, (PFN_PHYS(min_low_pfn) +
		 bootmap_size + PAGE_SIZE-1) - (HIGH_MEMORY));
	...

	/*
	 * 其他被硬件占用的物理地址，也转换为页，然后被保留。
	 * 仅作说明，就不一一列出了。
	 */
	reserve_bootmem_core(node_data[0], 0, PAGE_SIZE);
	...
}
```

保留页 `reserve_bootmem_core()` 就是 `free_bootmem_node()` 的反向操作。

4. 分配内存

初始化页分配器的功能有限，它能分配小于1页的内存，但是不释放小于1页的数据（这导致了一些内存的浪费）；为了节约宝贵的 `dma` 内存区域，它支持跳过一段指定区域的内存进行分配；为了配合高速缓存，还支持按某种大小对齐进行分配。该分配器最重要的缺陷是不能按 `NUAM` 节点分配页，这就导致了虽然各个节点的描述符和页描述符的虚地址分离的，但是物理页却全是0号节点的，内存亲和性没有保证。但是这个抽象模型是支持的，只要我们优化这个分配器和其他一些不多的耦合环节，就可以达到最初的目的。现仅给出分配接口：

```c
/**
 * @param bdata 默认只有node0上的全局分配器描述符
 * @param size 分配大小，可以小于一页
 * @param align 按每个大小对齐
 * @param goal 分配这个物理地址之上的内存
 */
void * __init __alloc_bootmem_core(struct bootmem_data *bdata, unsigned long size,
		unsigned long align, unsigned long goal);
```

## 初始化最终页表

现在我们有个简单的页分配器了，可以使用它继续完成后续的工作，这个步骤叫着分页初始化，入口函数为 `pagetable_init()`。

```c
void __init paging_init(void)
{
	/*
	 * 页表初始化
	 * 1. 直接映射页表
	 * 2. 临时映射页表
	 * 3. 固定映射页表
	 * 4. 永久映射页表
	 * 5. 节点管理数据页表
	 */
	pagetable_init();

	/*将内核页表装载到cr3，并刷新TLB*/
	load_cr3(swapper_pg_dir);
	__flush_tlb_all();

	...
	/*初始化节点的内存区域管理数据*/
	zone_sizes_init();
}
```

1. 直接映射页表初始化

我们必须先初始化最终的页表，才能继续后续的启动，否则从 `bootmem` 分配的页不能使用，就不能继续初始化动态的数据结构。

```c
static void __init pagetable_init (void)
{
	unsigned long vaddr;
	pgd_t *pgd_base = swapper_pg_dir;
	/*如果CPU了一些特性，则修改内核页表目录页的标志*/
	if (cpu_has_pge) {
		set_in_cr4(X86_CR4_PGE);
		__PAGE_KERNEL |= _PAGE_GLOBAL;
		__PAGE_KERNEL_EXEC |= _PAGE_GLOBAL;
	}
	/*初始化直接映射的页表，并且挂页*/
	kernel_physical_mapping_init(pgd_base);
	/*初始化用于节点管理的页表，并且挂页*/
	remap_numa_kva();

	/*
	 * 初始化固定映射的页表，没有挂页
	 */
	vaddr = __fix_to_virt(__end_of_fixed_addresses - 1) & PMD_MASK;
	page_table_range_init(vaddr, 0, pgd_base);
	/* 初始化永久映射页表，没有挂页 */
	permanent_kmaps_init(pgd_base);
}
```

首先初始化直接映射部分，这样 `bootmem` 分配器才能真正的工作。为了简化，我们只考虑没有开启 `PAE` 的2级页表。并且分两种情况来考虑，一种是CPU支持 `PSE`，另一种是不支持。

`PSE` 特性就是在直接映射的页表区域可以使用 4MB 的连续页来寻址，在CPU支持的情况下，当我们开启全局目录页项的 `_PAGE_PSE` 标志，那么在CPU在对一个虚地址寻址时，遇到这个标志就会停止继续对下级页表目录的解析，因为它知道当前目录项中的页号就是一个巨型的页的起始页号。在物理上CPU少解析了一次页表目录页，所以这个特效加速了寻址。在最新的CPU上，支持高达1GB的巨型页寻址。

首先看不支持 `PSE` 的初始化：

```c
static void __init kernel_physical_mapping_init(pgd_t *pgd_base)
{
	unsigned long address;
	unsigned long pfn;
	pgd_t *pgd;
	pmd_t *pmd;
	pte_t *pte;
	int pgd_idx, pmd_idx, pte_ofs;
	/*内核页表项起始位置 768下标*/
	pgd_idx = pgd_index(PAGE_OFFSET);
	/*找到对应的目录项*/
	pgd = pgd_base + pgd_idx;
	/*尽管我们不会访问前1MB的空间，但是我们还是对其相应的虚地址的页表初始化，简化了操作*/
	pfn = 0;
	for (; pgd_idx < PTRS_PER_PGD && pfn < max_low_pfn; pgd++, pgd_idx++) {
		/*二级页表 pud 和 pmd 是同一个*/
		pmd = (pmd_t*)pgd;
		address = __va(pfn * PAGE_SIZE);
    	/*如果需要分配一个目录页，如果已经分配，则是内核初始化页表设置的*/
		if (pmd_none(*pmd)) {
			/*分配页表页，并设置到中间目录页的表项中*/
			pte = (pte_t*) alloc_bootmem_low_pages(PAGE_SIZE);
			set_pmd(pmd, __pmd(__pa(pte) | _PAGE_TABLE));
		} else {
			pte = pte_offset_kernel(pmd, address);
		}
		/*初始化页表页中的每一项*/
		for (pte_ofs = 0; pte_ofs < PTRS_PER_PTE && pfn < max_low_pfn; pte++, pfn++, pte_ofs++) {
			/*设置页表项对应的页框和标志位，内核文本段是可执行的*/
			if (is_kernel_text(address))
				set_pte(pte, pfn_pte(pfn, PAGE_KERNEL_EXEC));
			else
				set_pte(pte, pfn_pte(pfn, PAGE_KERNEL));
		}
	}
}
```

不支持 `PSE` 内核就需要分配4K的单页作为页表页，然后转换成页号，填充到目录页的表项的高20位，并在低12位设置为内核态才能寻址的页表标志，防止用户态的进程非法寻址内核数据或执行内核代码。

支持 `PSE` 时，不用分配4K的单页作为页表页，不但节约了内存，而且加速了寻址，唯一遗憾的是这个版本的内存没有记录初始化页表时，使用的哪些页表页，从而浪费了一点点内存。代码片段如下：

```c
static void __init kernel_physical_mapping_init(pgd_t *pgd_base)
{
	...
	for (; pgd_idx < PTRS_PER_PGD && pfn < max_low_pfn; pgd++, pgd_idx++) {
		/*二级页表 pud 和 pmd 是同一个*/
		pmd = (pmd_t*)pgd;
		...
		/*不用分配页 而是设置巨型页的起始页号
		 * address2 是检验如果巨型页尾部的最后一个单页是内核文本段，那么也需要置位可执行页表标志
		 */
		unsigned int address2 = (pfn + PTRS_PER_PTE - 1) * PAGE_SIZE + PAGE_OFFSET + PAGE_SIZE-1;
		if (is_kernel_text(address) || is_kernel_text(address2))
			set_pmd(pmd, pfn_pmd(pfn, PAGE_KERNEL_LARGE_EXEC));
		else
			set_pmd(pmd, pfn_pmd(pfn, PAGE_KERNEL_LARGE));
		pfn += PTRS_PER_PTE;
	}
}
```

**页表的直接映射部分就是这些，内核为了简化操作，将很多不可用的物理地址也进行了映射，但是只要使用通过伙伴系统分配的页，就不会出现问题。因为在初始化伙伴系统时，会过滤掉不能使用的页**。

2. 内存管理虚地址页表初始化

该版本的内核使用了另一种途径来实现各个节点管理数据内存的节点亲和性，就是将节点各自的保留的页挂载到相应的内存管理数据结构对应的虚地址的页表中。需要注意的是，这部分的虚拟地址和页表挂载的物理地址不是一一映射的，所以虚地址减去一个 `PAGE_OFFSET` 是得不到物理地址的。

如果支持 `PSE` 那么我们使用巨型页初始化这部分的页表。

```c
void __init remap_numa_kva(void)
{
	void *vaddr;
	unsigned long pfn;
	int node;
	pgd_t *pgd;
	pmd_t *pmd;

	for_each_online_node(node) {
		if (node == 0)
			continue;
		/*
		 *每个NODE管理内存的起始虚地址都是按PMD大小对齐的
		 *按PMD大小（4MB）偏移页帧号，并将页帧号映射为4MB的巨型页
		 */
		for (pfn=0; pfn < node_remap_size[node]; pfn += PTRS_PER_PTE) {
			/*计算该节点管理数据结构的起始虚地址*/
			vaddr = node_remap_start_vaddr[node]+(pfn<<PAGE_SHIFT);
			pgd = swapper_pg_dir + pgd_index(vaddr);
			pmd = (pmd_t*)pgd;
			/*该虚拟地址挂载的页帧号为起始页号加偏移*/
			set_pmd(pmd, pfn_pmd(node_remap_start_pfn[node] + pfn, PAGE_KERNEL_LARGE));
			__flush_tlb_one(vaddr);	
		}
	}
}
```

否则使用 `bootmem` 来分配页表页，然后挂页。

```c
void __init remap_numa_kva(void)
{
	...
	for_each_online_node(node) {
		...
		for (pfn=0; pfn < node_remap_size[node];pfn+=PTRS_PER_PTE) {
			...
			pmd = (pmd_t*)pgd;
			BUG_ON(!pmd_none(*pmd));
			/*分配页表页，并设置到中间目录页的表项中*/
			pte = (pte_t*) alloc_bootmem_low_pages(PAGE_SIZE);
			set_pmd(pmd, __pmd(__pa(pte) | _PAGE_TABLE));
			/*初始化页表页中的每一项*/
			for (pte_ofs = 0; pte_ofs < PTRS_PER_PTE; pte++, pte_ofs++, vaddr+=PAGE_SIZE) {
				set_pte(pte, pfn_pte(node_remap_start_pfn[node] + pfn + pte_ofs, PAGE_KERNEL));
				__flush_tlb_one(vaddr);	
			}
		}
	}
}
```

以上这个代码片段是我加的，因为如果CPU不支持巨型页寻址，就不能按照巨型页初始化页表。

3. 非连续性和固定映射的页表初始化

这部分一样的是为了初始化页表，但是不会挂载数据页。因为这部分 `128MB` 的虚地址的页表的数据页可以是任何可用的物理页，这种非线性的映射使内核可以在运行时按需动态的变换映射的物理页，从而达到可以使用超过 `896MB` 的物理地址。由于篇幅所限，并且现在所有体系架构都是64位的，有几乎用不完的虚地址（高达256TB），这种算法基本没有实际的意义了。我们关注的重点应该是内存模型初始化的过程和注意事项，所以暂时就不讲解了。

## 总结

自此我们已经完成了所有相关页表的初始化，此后内核就可以动态的初始化其他的数据结构。在此页表初始化的过程中的关键要考虑如下的问题：

1. 物理内存是怎样分布的，内核是从什么地方得知物理内存的分布。
2. 为了使用物理内存，内核是如何规划虚地址。
3. 为了初始化页表需要使用物理内存，但是如果要使用物理内存又必须使用页表，内核是如何解决鸡与蛋问题的。

带着上面的问题去读内核源码才能有收获，当然首先必须要知道虚地址、物理地址与分页的基本原理。接下来我将介绍伙伴系统 ———— 内核最终的页分配器。