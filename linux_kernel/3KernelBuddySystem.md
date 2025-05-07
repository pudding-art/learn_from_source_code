# 内核物理页管理 —— 伙伴系统
> 在阅读此文章之前，你必须了解内核的页与页表的概念，而且最好能了解虚地址的分布。
> 内核版本依然是 `2.6.11` 的32位平台

## 内存管理模型的抽象

这部分内容在 [内核页表的初始化](https://blog.csdn.net/CrazyHeroZK/article/details/105008124) 已经介绍过。这里仅给出结构体，方便阅读下面代码参考：

```c
struct zone {
	unsigned long		free_pages;/**< 总的空闲页数量*/
	struct per_cpu_pageset	pageset[NR_CPUS]; /**<每个CPU上的高速页分配缓存*/

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

伙伴系统依附于 节点描述符，换句话说每个节点都有一个页分配器入口。下面先从相关数据结构的初始化开始介绍，然后再是伙伴系统的算法介绍。

## 节点描述符初始化

内核在完成页表的初始化后，就可以初始化节点描述符了，因为这部分虚地址自此才可以被寻址，否则将缺页异常。

```c
void __init paging_init(void)
{
    ...
    /*初始化节点的内存区域管理数据*/
	zone_sizes_init();
}
```

1. 节点描述符链表构建

内核为了在内存耗尽后，构建一种后备分配优先级机制，不仅需要使用节点作为下标定位描述符，还需要使用以链表的方式遍历节点描述符和其中的内存管理区。比如遍历所有的内存管理区：

```c
#define for_each_zone(zone) \
	for (zone = pgdat_list->node_zones; zone; zone = next_zone(zone))

static inline struct zone *next_zone(struct zone *zone)
{
	pg_data_t *pgdat = zone->zone_pgdat;
    /*非常巧妙的运用了指针的特性*/
	if (zone < pgdat->node_zones + MAX_NR_ZONES - 1)
		zone++;
	else if (pgdat->pgdat_next) {
		pgdat = pgdat->pgdat_next;
		zone = pgdat->node_zones;
	} else
		zone = NULL;

	return zone;
}
```

构建链表的代码片段如下：

```c
void __init zone_sizes_init(void)
{
	int nid;

	/*
	 * 按节点升序将pg_data_t连接到pgdat_list链表中
     * 这里使用了栈的特点倒序的构建栈，那么遍历时就是顺序的。
     * 遍历最大可能数目的节点，如果不节点在线则不链接到链表中
	 */
	pgdat_list = NULL;
	for (nid = MAX_NUMNODES - 1; nid >= 0; nid--) {
		if (!node_online(nid))
			continue;
		if (nid)
			memset(node_data[nid], 0, sizeof(pg_data_t));
		node_data[nid]->pgdat_next = pgdat_list;
		pgdat_list = node_data[nid];
	}
    ...
}
```

需要注意的是 除0号节点，其他的节点描述符要到此处才能清零，因为这段虚地址页表刚被建立。

2. 内存管理区的页分布

页最终是由每个内存管理区来管理的，所以接着内核需要了解各个节点的每个内存管理区的页的分布情况。我们时刻需要知道节点、区和整个物理地址叠加起来的分布示意图：

```
		+----------------+-------------------------+
		|     node 0     |         node 1          |
		+----------+-----+-----+-------------------+
		|   16Mb   |496Mb|384Mb|      128Mb        |
		+----------+-----------+-------------------+
		| ZONE_DMA |ZONE_NORMAL|   ZONE_HIGHMEM    |
		+----------+-----------+-------------------+
```

计算代码片段如下：

```c
void __init zone_sizes_init(void)
{
    ...
    for_each_online_node(nid) {
        /*该数组记录了各个区的内存页数量，加上该节点的起始页号就知道哪些页属于哪些区*/
		unsigned long zones_size[MAX_NR_ZONES] = {0, 0, 0};
		unsigned long *zholes_size;
		unsigned int max_dma;

		unsigned long low = max_low_pfn;
		unsigned long start = node_start_pfn[nid];
		unsigned long high = node_end_pfn[nid];
		
		/*__PAGE_OFFSET 起最多16MB的连续空间可以作为 DMA*/
		max_dma = virt_to_phys((char *)MAX_DMA_ADDRESS) >> PAGE_SHIFT;

		/*根据该节点所持有的页帧起止编号，初始化该节点的各个内存区域中的页帧数*/
		if (start > low) {
			/*该节点的所有页位于直接映射内存之上，所以对于这个节点只有HIGH内存区有可用页，其他区都是空的*/
			zones_size[ZONE_HIGHMEM] = high - start;
		} else {
            /*该节点的页与直接映射页区有交集*/
			if (low < max_dma)
				/*直接映射内存最大页号都小于16MB，说明整个机器内存很小，
                 *根据节点管理页的数量不能小于256MB的原则，说明不可能存在其他的节点
                 *所以整个机器只有0号节点的DMA区域有内存可用
                 */
				zones_size[ZONE_DMA] = low;
			else {
                if (start < max_dma) {
                    /* 如果起始页号小于最大DMA区域，
                     * 则DMA可以全部赋予该节点DMA区域，而且肯定还有一部分 普通区 内存*/
				    zones_size[ZONE_DMA] = max_dma;
				    zones_size[ZONE_NORMAL] = min(high,low) - max_dma;
                } else if (low > start) {
                    /*normal可以跨两个节点*/
				    zones_size[ZONE_NORMAL] = low - start;
                    /*还需进一步修正*/
                    if (low > high)
				        zones_size[ZONE_NORMAL] = high - start;
                }
                if (high > low)
				    zones_size[ZONE_HIGHMEM] = high - low;
			}
		}
        ...
	}
    ...
}
```

源代码在计算 `zones_size[]` 认为 `ZONE_NORMAL` 不可能横跨2个节点，仅存在 `ZONE_HIGHMEM` 横跨2个节点的情况：

```c
void __init zone_sizes_init(void)
{
    ...
    for_each_online_node(nid) {
        ...
        if (start > low) {
			zones_size[ZONE_HIGHMEM] = high - start;
		} else {
			if (low < max_dma)
				/*直接映射内存小于16MB*/
				zones_size[ZONE_DMA] = low;
			else {
				zones_size[ZONE_DMA] = max_dma;
				zones_size[ZONE_NORMAL] = low - max_dma;
				zones_size[ZONE_HIGHMEM] = high - low;
			}
		}
    }
    ...
}
```

但是根据 `PAGES_PER_ELEMENT` 定义处的原文注释：

```
/*
 * generic node memory support, the following assumptions apply:
 *
 * 1) memory comes in 256Mb contigious chunks which are either present or not
 * 1) 内存来自256MB的连续块，这些块要么是当前节点，要么不是（即节点管理的内存至少是256MB的整数倍）
 * 2) we will not have more than 64Gb in total
 *
 * for now assume that 64Gb is max amount of RAM for whole system
 *    64Gb / 4096bytes/page = 16777216 pages
 * physnode_map每一个元素需要表达256mb的页数，那么总共需要 16777216/((256*1024*1024)/4096) = 256 个元素
 */
#define MAX_NR_PAGES 16777216
#define MAX_ELEMENTS 256
#define PAGES_PER_ELEMENT (MAX_NR_PAGES/MAX_ELEMENTS)
```

而直接映射区域的内存可以接近 896MB （根据页表初始化得知有一部分直接映射的虚地址被用于节点描述符和页描述符），而且节点管理的物理地址可以是不相等的（一般都是主板上一组或多组插槽一个节点，如果没有插满就会出现这样的情况），那么我上面给出的示意图就完全可以在现实中存在。

3. 页描述符空间分配

各个节点的内的页描述符由各个节点的物理内存来存放，内核在使用一个极其简单的 `bootmem` 分配器也保证了这部分内存的节点亲和性，具体算法见《页表初始化》文章，现仅给出源代码片段，如下：

```c
void __init zone_sizes_init(void)
{
    ...
    for_each_online_node(nid) {
        ...
        if (!nid) {
			free_area_init_node(nid, node_data[nid],
				zones_size, start, zholes_size);
		} else {
			/*
			 * 将保留用于节点管理的内存，前半部分用于管理描述符，后半部分用于页描述符数组，
			 * 该部分内存以及映射到高端内存的起始虚拟地址上。
			 */
			unsigned long lmem_map = node_remap_start_vaddr[nid];
			lmem_map += sizeof(pg_data_t) + PAGE_SIZE - 1;
            /*向上按页对齐，保证了按缓存线对齐，保证了效率*/
			lmem_map &= PAGE_MASK;
			node_data[nid]->node_mem_map = (struct page *)lmem_map;
			free_area_init_node(nid, node_data[nid],
				zones_size, start, zholes_size);
		}
    }
}

void __init node_alloc_mem_map(struct pglist_data *pgdat)
{
	unsigned long size;

	size = (pgdat->node_spanned_pages + 1) * sizeof(struct page); /*多分配一页？*/
	pgdat->node_mem_map = alloc_bootmem_node(pgdat, size);
}

void __init free_area_init_node(int nid, struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long node_start_pfn,
		unsigned long *zholes_size)
{
	pgdat->node_id = nid;
	pgdat->node_start_pfn = node_start_pfn;
    ...

	/*（0号节点）还有没有映射管理页帧到虚拟地址，进行了等价的修改*/
	if (!node_data[nid]->node_mem_map)
		node_alloc_mem_map(pgdat);

	free_area_init_core(pgdat, zones_size, zholes_size);
}
```

这个片段的重点是需要知道 `node_remap_start_vaddr[]` 数组存储其它节点(除0号节点外)描述符与管理页的描述符的虚地址，前面部分用于节点描述符，后面用于页描述符，这个在页表初始化的文章中有较详细的介绍；然后 `pglist_data.node_mem_map` 字段是存储的页描述符的起始虚地址，再次强调这部分页表对应的物理地址不是直接映射的，不能使用 页帧号表示。

我们着重介绍 `free_area_init_core()`之前，由于这个版本的内存代码还不怎么规范，特别是 `zone_sizes_init()` 调用 `free_area_init_node()` 我将展示这部分后续版本的源代码片段结构，并进行适当修改以符合当前版本的初始化：

```c
void __init zone_sizes_init(void)
{
    ...
    for_each_online_node(nid) {
        ...
        /*统一条件的调用此函数，不再做过多的判断*/
        free_area_init_node(nid, node_data[nid], zones_size, start, zholes_size);
    }
}

void *alloc_remap(int nid, unsigned long size)
{
    /*0号Node 使用 bootmem 分配*/
	if (!nid)
		return NULL;
	unsigned long lmem_map = node_remap_start_vaddr[nid];
    lmem_map += sizeof(pg_data_t) + PAGE_SIZE - 1;
	lmem_map &= PAGE_MASK
    ...
	memset(lmem_map, 0, size);
	return (void*)lmem_map;
}

static void __init alloc_node_mem_map(struct pglist_data *pgdat)
{
    ...
	unsigned long size = (pgdat->node_spanned_pages + 1) * sizeof(struct page);
	struct page *map = alloc_remap(pgdat->node_id, size);
	if (!map)
		map = alloc_bootmem_node(pgdat, size);
	pgdat->node_mem_map = map;
}

void __init free_area_init_node(int nid, struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long node_start_pfn,
		unsigned long *zholes_size)
{
    ...
	alloc_node_mem_map(pgdat);
	free_area_init_core(pgdat, zones_size, zholes_size);
}
```

虽然有些啰嗦，但是希望你能从中得到启发，将自己的代码实现得很简洁合理。

`free_area_init_core()` 主要完成页描述符的初始化、内存管理区的与页描述符的映射（因为要通过页快速的找到对应的 `zone`）。

```c
static void __init free_area_init_core(struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long *zholes_size)
{
	unsigned long i, j;
	int cpu, nid = pgdat->node_id;
	pgdat->nr_zones = 0;
	unsigned long zone_start_pfn = pgdat->node_start_pfn;
	
	/*初始化节点的各个内存区域，会使用bootmem分配一些内存*/
	for (j = 0; j < MAX_NR_ZONES; j++) {
		struct zone *zone = pgdat->node_zones + j;
		unsigned long size, realsize;
		unsigned long batch;
		
		/*将所有的zone映射到一个全局zone表中*/
		zone_table[NODEZONE(nid, j)] = zone;
		zone->present_pages = zones_size[j];

        /*初始化冷热页高速缓存集合*/
		for (cpu = 0; cpu < NR_CPUS; cpu++) {
			struct per_cpu_pages *pcp;
			pcp = &zone->pageset[cpu].pcp[0];	/* hot */
            ...
			pcp = &zone->pageset[cpu].pcp[1];	/* cold */
            ...
		}

        /*如果没有管理的页，以下的字段就不需初始化了*/
		if (!size)
			continue;

        /*拥有页的最大的管理区数量*/
		pgdat->nr_zones = j+1;
        /*管理区管理的页的起始页号*/
		zone->zone_start_pfn = zone_start_pfn;
        /*管理区管理的页的起始页描述符*/
		zone->zone_mem_map = pfn_to_page(zone_start_pfn);
		
		/*初始化页框 @see memmap_init_zone()*/
		memmap_init(size, nid, j, zone_start_pfn);

        /*增加偏移，得到下一个区域管理的页起始页号*/
		zone_start_pfn += size;
		/*初始化伙伴系统的管理链表 @see zone_init_free_lists()*/
        for (int order = 0; order < MAX_ORDER; order++) {
		    INIT_LIST_HEAD(&zone->free_area[order].free_list);
		    zone->free_area[order].nr_free = 0;
	    }
	}
}
```

伙伴系统识别的是页描述符，当页从伙伴系统中被分配出来时，我们需要将页转换成虚地址才能使用；当使用完毕释放页时，如果仅知道这段内存的虚地址，那么我们必须要经过一个逆转换才能得到页，然后释放到伙伴系统中。我们先给出这些转换的相关函数：

```c
/*最多64个node*/
#define MAX_NODES_SHIFT 6

/*最多4个zone*/
#define MAX_ZONES_SHIFT 2 

/*表示用于node和zone编码的位偏移*/
#define NODEZONE_SHIFT (sizeof(page_flags_t)*8 - MAX_NODES_SHIFT - MAX_ZONES_SHIFT)

/*从page得到其所在的node的编号*/
static inline unsigned long page_to_nid(struct page *page)
{
	/*高6位放置的是node的编码*/
	return (page->flags >> (NODEZONE_SHIFT + ZONES_SHIFT));
}

/*从page得到其所在的zone*/
static inline struct zone *page_zone(struct page *page)
{
	return zone_table[page->flags >> NODEZONE_SHIFT];
}

/*page到pfn的转换*/
static inline unsigned long page_to_pfn(struct page *pg)
{
	struct zone * zone = page_zone(pg);
	return (unsigned long)(pg - zone->zone_mem_map) + zone->zone_start_pfn;
}
/*pfn到page*/
static inline struct page *pfn_to_page(unsigned long pfn)
{
	int nid = pfn_to_nid(pfn);
    unsigned long idx = pfn - node_data[nid]->node_start_pfn;
	return &node_data[nid]->node_mem_map[idx];
}
/*直接映射区的 va到page*/
#define virt_to_page(kaddr)	pfn_to_page(((kaddr)-0xC0000000UL) >> PAGE_SHIFT)

/*直接映射区的 page到va*/
static inline void *lowmem_page_address(struct page *page)
{
	return (page_to_pfn(page) << PAGE_SHIFT) + 0xC0000000UL;
}
```

这些转换是内核按页操作内存的基础设施，它可以独立于伙伴系统外来讲解。首先我们来回顾一下节点和其管理的页的规划过程：

```
假设有3个节点，都是512MB的空间，每个 node 物理页的 分区并按字母编号
    1. A 区 用于可以分配的物理页
    2. B区用于抽象的节点描述符
    3. C区用于抽象的页描述符
    4. 字母后的数字表示所属节点号
    5. 中间的虚地址中的字母标记就是对物理页的映射
    6. N/O mlp表示 新旧的 max_low_pfn 直接映射的最大页帧号位置
    7. 省略号表示被其他设置的初始化使用，置于node0的内存分布为何这样参考页表初始化的文章

                                     +----------------------------+-----------------------+
                                     | A1                   |B1|C1| A2              |B2|C2|
                                     +----------------------------+-----------------------+
kernel virtaddr +-------------------------+===========+-----------+
                |...|A0|B0|...|C0|A0.| A1 |B2|C2|B1|C1|   128MB   |
                +-------------------------+===========+-----------+
               3GB                      N mlp       O mlp        4GB
                +--------------------+
                |...|A0|B0|...|C0|A0.|
                +--------------------+
```

可以看出来用于页描述符的物理内存与虚拟内存要么不知道确切的位置，要么不是线性映射的，所以必须为各个节点记录其页描述符的起始虚地址和起始页帧号才能从页描述符得到页帧号。对于一个节点前半部分是可分配的页，后半部分是页描述符，他们是一一映射的，如下：

```
+---------+---------+---------+
|p0|p1|...|pg_data_t|P0|P1|...|
+---------+---------+---------+
```

所以我们将 `p0` 页帧号记录在 `pg_data_t.node_start_pfn`，将 `P0` 页描述符数组首地址记录在 `pg_data_t.node_mem_map`。只要给出的页帧属于 `p0~pn`范围内，页描述符虚地址属于 `P0~Pn` 范围内，通过指针运算或整数运行得到的步长加上起始地址或页帧号就可以得到需要转换的结果。至于页帧到节点号的转换，已经在页表初始化中介绍过。而页描述符到管理区的转换稍稍绕点，每个节点的管理区页数都是不定的，所以没有办法直接通过上面介绍的步长同步方式来转换，内核通过将节点号和管理区的下标编码到页描述符的标志字段 `struct page.flags` 最高位的方式来定位页所属哪个管理区。

```
+---------------+---------------+
|     node0     |     node1     |
+---------------+---------------+
|dma|normal|high|dma|normal|high|
+---------------+---------------+
```

如果要将上面的所有 `zone` 放入一个数组中，因为每个 `node` 步长是 3 ，又有 2 个节点，要定位其中的任何 `zone` ，按照二维数组定位槽位就可以了。所以 将 `node`的编号编码倒最高6位，相当于二维数组的第一维，，然后接下来2位编码 `zone` 的编号，相当于第二维。为了配置位运算将其转换为一维数组操作，所以将使用第二维的元素数量定为4，最后一个是哑元。内核使用如下宏和函数编码并在 `struct page.flags` 中设置这几位：

```c
#define NODEZONE(node, zone)	((node << ZONES_SHIFT) | zone)

static inline void set_page_zone(struct page *page, unsigned long nodezone_num)
{
	page->flags &= ~(~0UL << NODEZONE_SHIFT);
	page->flags |= nodezone_num << NODEZONE_SHIFT;
}
```

上面的代码片段就是主要初始化页和页描述符转换所需要的基础数据，现在我们可以初始化页描述符了。

```c
void __init memmap_init_zone(unsigned long size, int nid, unsigned long zone,
		unsigned long start_pfn)
{
	struct page *start = pfn_to_page(start_pfn);
	struct page *page;

	for (page = start; page < (start + size); page++) {
		/*在页标志字段的高位编码node和zone的ID*/
		set_page_zone(page, NODEZONE(nid, zone));
		/*页引用计数（实际起始值为-1）*/
		set_page_count(page, 0);
        ...
		/*设置为保留*/
		SetPageReserved(page);
		INIT_LIST_HEAD(&page->lru);
        ...
	}
}
```

这里仅需要注意，初始化期间我们将所有的页都视为不可分配，包括其中有些页被 `bootmem` 分配出去了，因为这些页被初始化数据结构所使用，不能用于伙伴系统，后面会细讲。

## 伙伴系统算法简介

从页表初始化和节点数据结构初始化后，可以得知，内核可用的物理页大部分都是连续的，如果我们以如下简单的链表方式来管理可用页，在内核运行一段时间之后原本连续的物理页会因为丢失其起初的位置信息，从而使分配器不再能分配连续的页。

```
1. 刚开始时 end > start 我们从该区间分配页
2. 每次释放时都缓存到 free_page_list_head 如果是连续页，仅将头页连接到链表上，比如 gh 页，并在头页 g 中标识
3. 运行一段时间后 end == start，我们只能从 空闲链表中查找，如果没有满足数量的连续页存储，分配只能返回失败
4. 空闲链表中有很多连续的物理页，比如 fghi 页是连续的，但有丢失了连续的信息而不能快速合并

  start                end
    |                   |
    v                   v
    +-----------------------------------+
    | a | b | c | d | e | f | g | h | i |
    +-----------------------------------+


free_page_list_head
       |    
       v
    +----+
    |Next|-->[g]-->[i]-->[f]
    +----+   [h]
```

内核对上述的情况称为产生了 `外碎片` 。内核其实可以通过将非连续的页映射到连续虚地址对应的页表中，可以部分解决一部分请求连续分配连续内存的问题，但是这会带来刷新TLB额外的性能损失，而某些外设硬件在与物理内存交换数据是必须要求物理内存是连续的，比如磁盘I/O时会忽略分页直接访问地址总线，所以不如开发一种适当的技术能记录连续页，并在分配或释放页的时候切割或合并连续页，以减少碎片。伙伴系统的算法孕育而生，这是一个“古老”的算法，但是非常经得起考验。

伙伴系统使用多个链表来管理页，而不是上面介绍的单链表，每个链表管理相应阶数（注意“阶数”一词）连续页，先阶段所有版本的内存都使用11个这样的链表，分别管理 1、2、4、8、16、32、64、128、256、512、1024 都是 `2^n` 次方，`n` 称为阶数。所以内核最大可分配 1024 连续页，最大为 4MB。

在 `struct zone` 中 `free_area[MAX_ORDER]` 字段就是这个链表数组的抽象，结构如下：

```c
struct free_area {
	struct list_head	free_list;/**<
								   * 每个链表元素指针都指向page.lru字段.
								   */
	unsigned long		nr_free;/**<链表中元素的数量 = 连续伙伴页的数量*/
};
```

当其中盛放了可用页时，示意图如下：

```
        
     +-----+ 
     | ... |
     +-----+ 
     |  4  | -----------[16]--->[16]--->[16]---...
     +-----+
     |  3  | -----------[8]---->[8]---->[8]---...
     +-----+
     |  2  | -----------[4]---->[4]---->[4]---...
     +-----+
     |  1  | -----------[2]---->[2]---->[2]---...
     +-----+
     |  0  | -----------[1]---->[1]---->[1]---...
     +-----+
```

核心算法就是，请求 `2^n` 个连续页时，会从 `n` 下标的链表中查找是否有可以分配的页，如果有就直接从链表中弹出返回，如果没有就上跳到 `n+1` 继续查找，如果没有再上跳，直到找到一组页，假设当前阶数是 `N` （`N`>`n`），然后二分切割该组连续的页，后半部分放入 `N-1` 链表中，如果此时 `N=N-1` 等于 `n` 那么停止分割，前半部分可以直接返回，否则继续上面的分割方式，直到 `N-1` 等于 `n`，该分配过程还是相对简单的；释放 `2^n` 个连续页时，会计算该页的相邻 `2^n` 连续页（或被释放页组的前面或在其后面）是否空闲（通过页描述符标志字段得知）———— 这个相邻的页组就是伙伴页，也是该算法命名的由来，如果空闲肯定在对应阶数的链表中，首先从链表中弹出，然后将2个 `2^n` 合并成 `2^(n+1)` 的连续页，然后以同样的方式查看他的伙伴页组，如果没有了就插入到下标为 `n=n+1` 链表中，如果它的伙伴页组也空闲，则以同样的方式合并，直到不能再合并为止。

可以看出上面描述的过程除了找到某个页组的伙伴页组不是很明了，其他的过程都很明了。下示意图是一个分配的分裂过程，释放的合并过程是方向反的：

```
星号 `*` 中间的页组就是分裂合并的路径

high +-----+ 
     |  4  | --------------------*[16]*-->[16]----[16]---...
     +-----+                     /    \    new-head
     |  3  | -----------------*[8]*--->[8]
     +-----+                  /   \     new-head
     |  2  | -------------*[4]*--->[4]
low  +-----+              /   \     new-head
     |  1  | ----------*[2]*->[2]
     +-----+             ^     new-head
     |  0  |             | return-page
     +-----+
```

## 初始化伙伴系统

我已多次提过，内核以一种非常简单的方式就完成了伙伴系统中所有页的初始化，那就是在完成数据结构初始化之后，将 `bootmem` 未使用的页当作从 伙伴系统 中分配出来的页，再通过伙伴系统的 `__free_pages()` 函数释放，这些被释放的页就会按伙伴系统的算法自主的合并（如果能），装填了伙伴系统。

该过程的入口函数在 `mem_init()` 片段如下：

```c
void __init start_kernel(void)
{
    ...
    mem_init();
    ...
}

void __init mem_init(void)
{
    ...
    /* 将所有节点中未使用的(直接映射的）页帧归还到各自的buddy数据结构中*/
	totalram_pages += __free_all_bootmem();
    ...
    /* 将所有节点中的非直接映射的页帧（高端内存）归还到各自的buddy数据结构中*/
	set_highmem_pages_init(bad_ppro);
    ...
}
```

1. 归还直接映射页

`bootmem` 中剩余的未使用页是低端内存页，直接`__free_all_bootmem()` 最终调用 `free_all_bootmem_core()`，关键代码片段如下：

```c
/**返回释放到伙伴系统中的页数 ———— 未被初始化使用的页帧数*/
static unsigned long __init free_all_bootmem_core(pg_data_t *pgdat)
{
	struct page *page;
	bootmem_data_t *bdata = pgdat->bdata;
	unsigned long i, count, total = 0;
	unsigned long idx;
	unsigned long *map; 
	int gofast = 0;

	/*直接映射的第一个页的描述符*/
	page = virt_to_page(phys_to_virt(bdata->node_boot_start));
	/*总的页数，即位图的总位数*/
	idx = bdata->node_low_pfn - (bdata->node_boot_start >> PAGE_SHIFT);
	/*bootmem的页位图*/
	map = bdata->node_bootmem_map;
	if (bdata->node_boot_start == 0 ||
		/*至少有32张页，则使用整体（32张页）释放路径*/
	    ffs(bdata->node_boot_start) - PAGE_SHIFT > ffs(BITS_PER_LONG))
		gofast = 1;
	/*将所有启动时未使用的页帧放入到对应的节点管理结构中（buddy系统中） */
	for (i = 0; i < idx; ) {
        /*原位图被置位的位表示已分配，所以取反后变成1表示页帧可用*/
		unsigned long v = ~map[i / BITS_PER_LONG];
		if (gofast && v == ~0UL) {
            /*有连续的页，使用伙伴系统多页释放接口*/
            ...
            /*跳过同步的步长*/
			i += BITS_PER_LONG;
			page += BITS_PER_LONG;
		} else if (v) {
			/*非连续的页*/
			unsigned long m;
			/* i < idx 这个位图集可能越界，因为位图的分配大于实际使用的位*/
			for (m = 1; m && i < idx; m<<=1, page++, i++) {
                /*单页释放*/
                ...
			}
		} else {
			i+=BITS_PER_LONG;
			page += BITS_PER_LONG;
		}
	}
    ...
	/* 释放位图本身 */
	/*位图的起始页帧*/
	page = virt_to_page(bdata->node_bootmem_map);
	/*将位图数转换为页数*/
	idx = ((bdata->node_low_pfn-(bdata->node_boot_start >> PAGE_SHIFT))/8 + PAGE_SIZE-1)/PAGE_SIZE;
	for (i = 0; i < idx; i++, page++) {
		__ClearPageReserved(page);
		set_page_count(page, 1);
		__free_page(page);
	}
    ...
}
```

`bootmem` 的位图中置位表示已使用，那么我们对齐求反，再遍历其中的置位的位就得到未使用的页了，并同步的遍历这些位代表的页的页描述符，然后释放即可。注意为了遍历位图，内核是按 机器字节 来遍历大位图的，即 按 `sizeof(long) * 8 = 32` 一跳来遍历大的位图，然后再遍历一个 `long` 数代表的 `32` 位的值。

如果这些页是单页，或不是连续的 `2^n` 且头页的*页偏移*不是 `2^n` 的倍数（至于为何，会在合并算法中介绍），简单的遍历既可以：

```c
static unsigned long __init free_all_bootmem_core(pg_data_t *pgdat)
{
    ...
    for (i = 0; i < idx; ) {
        if (gofast && v == ~0UL) {
            ...
        } else if (v) {
            /*非连续的页*/
			unsigned long m;
			/* i < idx 这个位图集可能越界，因为位图的分配大于实际使用的位*/
			for (m = 1; m && i < idx; m<<=1, page++, i++) {
				if (v & m) {
                    /*移除保留标志*/
					__ClearPageReserved(page);
					set_page_refs(page, 0);
					__free_page(page);
				}
			}
        }
        ...
    }
    ...
}
```

`m` 是一个与 `i` 位图下标同步变化的最多 `0` 到 `31` 位的掩码（`0x00001`、`0x00002`...），用来辅助 `v` 中哪些位是被置位一的。

`bootmem`毕竟只有少数页被分配作为初始化内核使用，为了加速将连续的空闲页释放到伙伴系统中，内核优化了这个过程。

```c
static unsigned long __init free_all_bootmem_core(pg_data_t *pgdat)
{
    ...
    for (i = 0; i < idx; ) {
        if (gofast && v == ~0UL) {
            /*这个32位的图表示页都未被分配的连续页*/
			int j, order;
            /*清理这32张连续页的保留标志*/
			__ClearPageReserved(page);
			set_page_refs(page, 0);
			for (j = 1; j < BITS_PER_LONG; j++) {
                ...
				__ClearPageReserved(page + j);
			}
			/*释放到伙伴系统中，32张页，其阶数为 5*/
			__free_pages(page, 5);
			i += BITS_PER_LONG;
			page += BITS_PER_LONG;
        } else if (v) {
            ...
        }
        ...
    }
    ...
}
```

需要注意的，从此刻开始伙伴系统页分配器可以工作了，那么 `bootmem` 分配器就不在使用了，它持有的页就可以解放出来了，所以还应当注意释放此部分的页。

2. 归还高端内存页

对于超过 `896MB` 的计算机，内核还包含了高端内存，我们在初始化节点的 `ZONE_HIGHMEM` 就在其中记录的这些内存的起止，遍历他们并释放到伙伴系统中即可。

```c
void __init set_highmem_pages_init(int bad_ppro) 
{
	struct zone *zone;
	struct page * page;
	unsigned long pfn, start_pfn, end_pfn;

    /*遍历所有 节点的 zone*/
	for_each_zone(zone) {
    
        /*忽略非高端内存*/
		if (zone != &zone->zone_pgdat->node_zones[ZONE_HIGHMEM])
			continue;

		/*页号的起止与页描述符同步长遍历*/
		page = zone->zone_mem_map;
		start_pfn = zone->zone_start_pfn;
		end_pfn = zone->zone_start_pfn + zone->spanned_pages;

        /*遍历这些页*/
		for (pfn = start_pfn; pfn < end_pfn; pfn++, page++) {
            if (page_is_ram(pfn)) {
		        ClearPageReserved(page);
		        set_bit(PG_highmem, &page->flags);
		        set_page_count(page, 1);
		        __free_page(page);
	        } else {
		        SetPageReserved(page);
            }
        }
	}
    ...
}
```

过程很简单，就是按 `ZONE_HIGHMEM` 来遍历所有高端内存，需要注意的是，我们使用的是 `spanned_pages` 这个页数，这个页数包含了可能空洞，所以还需要 `page_is_ram()` 来确定是否是可用的内存。

```c
static inline int page_is_ram(unsigned long pagenr)
{
	int i;
	unsigned long addr, end;

    ...
	for (i = 0; i < e820.nr_map; i++) {
		if (e820.map[i].type != E820_RAM)	/* not usable memory */
			continue;
		addr = (e820.map[i].addr+PAGE_SIZE-1) >> PAGE_SHIFT;
		end = (e820.map[i].addr+e820.map[i].size) >> PAGE_SHIFT;
		if  ((pagenr >= addr) && (pagenr < end))
			return 1;
	}
    ...
	return 0;
}
```

因为 `node` 的起止页号是另一种硬件信息得到的（见内核页表初始化 `get_memcfg_numaq()`），非 `e820` 表，从 `e820` 映射表可以补充来判断是否页可用，但其实这个版本对有空洞的硬件架构不能完全的支持。自此我们就完成了伙伴系统的初始化。

## 分配和释放页

伙伴系统最外层的分配释放是非常复杂的，包括了冷热页缓存、分配策略、回收策略等，我们先把握核心的分配和释放算法。

1. 分配页

内核在分配连续页是，都是用 `2^n` 的 `n` 阶数来求取页数，所以只能分配 `2^n` 倍数的页数。硬件不仅将页从物理上按节点分割，而且内核还在此基础上将其按使用分区管理，最终页是从某个节点的某个区分配而来，所以最核心的分配函数才如下定义。

```c
static struct page *__rmqueue(struct zone *zone, unsigned int order)
{
	struct free_area * area;
	unsigned int current_order;
	struct page *page;
	/*
	 * 从分配阶链表向高级阶链表请求连续的页框
	 */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = zone->free_area + current_order;
        /*如果为空就继续向更高阶的链表请求一张更大的连续页*/
		if (list_empty(&area->free_list))
			continue;
        /*找到满足条件的连续页后，从管理链表上移除*/
		page = list_entry(area->free_list.next, struct page, lru);
		list_del(&page->lru);
		/*移除private中存的order值*/
		rmv_page_order(page);
		area->nr_free--;
		zone->free_pages -= 1UL << order;
		/*当满足的阶大于当前的请求的阶时，除开满足的连续页框后，剩余的连续页框按阶进行分裂*/
		return expand(zone, page, order, current_order, area);
	}
	return NULL;
}
```

内核使用页描述符在记录页的状态，页使用 `PG_private` 标志标记 `page.private` 数据有效，根据页所在的容器不同，含义就不同，如果被伙伴系统管理，存储的就是此连续页的 `order` 值，从伙伴系统中移除时也应该改变这些状态，`rmv_page_order()` 和 `set_page_order()` 完成该状态的移除和设定。

上面的简介里已介绍过，分配时从请求页数 *对应阶数* 的链表开始查询是否有满足的连续页，如果没有就向 *更高阶数* 的链表请求，直到找到或超过 *最大阶数*，找到满足条件的页后，如果此页数大于请求数目，还需要分割，如下：

```c
static inline struct page * expand(struct zone *zone, struct page *page,
 	int low, int high, struct free_area *area)
{
	unsigned long size = 1 << high;
	while (high > low) {/*如果high>low，连续的页框可以被分裂*/
		area--;/*递减阶和对应的空闲链表数组指针*/
		high--;/*阶数递减*/
		size >>= 1;/* 每递减一次，连续的页框数减半，（二分）
					* 前半部分用于下次分裂，后半部分插入紧挨的低阶空闲链表
					* 此处计算相应的后半部分的页偏移
					*/
		/*后半部分插入伙伴系统中，如果需要前半部分继续分裂，否则在循环结束时，作为满足分配条件的页而返回*/
		list_add(&page[size].lru, &area->free_list);
		area->nr_free++;
		set_page_order(&page[size], high);/*设置每组连续页框的头框的private字段为对应的阶值*/
	}
	return page;
}
```

现在我们假设 `high=4` 、 `low=2`，上面的代码由一个简单的示意图来表示：

```
1. order 2~3 的 free_list 肯定为空，不然就能满足分配了
2. page.lru 被作为 free_list 的一个节点

    page 被分割的连续 size == 16 页起始页框
    +--------------------+
    |0|...|4|...|8|...|15|
    ++-----+-----+-------+
     |     |     | 
     |     |     |              order
     |     |    +++ page[8].lru +---+
     |     |    |+|<------------| 3 | 循环的第一次 high-- == 4，从 (size>>=1)==8 下标处切割
     |     |    +-+             +---+
     |    +++ page[4].lru       +---+ 
     |    |+|<------------------| 2 | 循环的第二次 high-- == 3，从 (size>>=1)==4 下标处切割
     |    +-+                   +---+
     | &page[0]
     +-------------------------> ret  循环结束 high == low 得到满足分配条件的页
```

2. 释放页

同样我们暂时仅给出释放页的核心函数 ———— 如何查找同阶数的伙伴页，并向上合并。

```c
/**
 * 释放块
 * @param page 被释放的连续页首页
 * @param base 接收释放页的起始页
 * @param zone 接收释放页的管理区
 * @param order 被释放页的阶数
 */
static inline void __free_pages_bulk (struct page *page, struct page *base,
		struct zone *zone, unsigned int order)
{
	unsigned long page_idx;
	struct page *coalesced;
	int order_size = 1 << order;
    
    ...
    /*被释放页到接收释放页起始页的索引*/
	page_idx = page - base;

    /*最高阶连续页没有伙伴，故减1（好孤独？高处不胜寒！。。。）*/
	while (order < MAX_ORDER-1) {
		struct free_area *area;
		struct page *buddy_page;
		unsigned long buddy_idx;
		
        /*计算伙伴页的下标*/
		buddy_idx = (page_idx ^ (1 << order));
		buddy_page = base + buddy_idx;
    
        /*判断伙伴页的起始页号是否有效*/
        if (bad_range(zone, buddy))
			break;
		/*
		 * 判断是否是伙伴并空闲，如果是，则开始从当前阶空闲链表剥离，然后升阶合并
		 */
		if (!page_is_buddy(buddy, order))
			break;

        /*空闲的，在链表中，操作前记得从链表中移除*/
		list_del(&buddy_page->lru);
		area = zone->free_area + order;
		area->nr_free--;
        /*合并后，移除保存在private中的阶值*/
		rmv_page_order(buddy_page);
        /*获取合并后可以与同阶连续块互为伙伴的第一个页框的下标*/
		page_idx &= buddy_idx;
		order++;
	}
	/*跳出循环后，表示该释放的连续页没有空闲的伙伴可以用以合并了，插入到对应阶的空闲链表中*/
	coalesced = base + page_idx;
	set_page_order(coalesced, order);
	list_add(&coalesced->lru, &zone->free_area[order].free_list);
	zone->free_area[order].nr_free++;
}
```

合并的过程就是分裂的反过程，就不细讲了，这里着重讲述怎么计算给定 `order` 连续页的伙伴页。首先必须要知道，伙伴系统的页是按管理区 `zone` 来管理的，所以一组连续页的伙伴页必须是属于同一个管理区 `zone` 的。先从伙伴页的定义开始，我们使用示意图来讲解，*其实上面的所有数据结构，都是根据下面的理论抽象而来*，不要搞反了。

3. 核心原理

**首先明显要明确，为了高效的分裂和合并连续的页，内核使用二分法，所以伙伴系统以一组 `2^n` 张来管理页**，先看一组单页，伙伴页示意图：

```
互为伙伴页的相对页号
0<>1
2<>3
4<>5
    +----------------------------+----------+
    |0|1|2|3|4|5|6|7|8|9|...|1023|0|...|1023|
    +----------------------------+----------+
```

全是单页 `2^order,order=0` 时，假设这一长串可用页恰好是 `1024` 张页的倍数，那么每张页都有伙伴，并且可以合并成 `2^order,order=1`。这里可以很容易的得出这样的结果，`0` 页和 `1` 页互为伙伴（伙伴是相互），当他们都空闲时就可以合并成页数为2的连续页；`1` 页和 `2` 页不是互为伙伴的，因为这样划分伙伴，那么 `0` 页和 `1023` 页就被单出来，不能合并而使分配器自身产生碎片，如果阶数越大就会参数越多的碎片，比如 `1022` 张单页按这样错误的伙伴划分合并成双页，那么有 `511` 张双页，再合并，有 `255` 张四页，但是剩下两张，这不仅仅不合理，而且没有规律而行。

一组双页、四页，伙伴页示意图：

```
互为伙伴页的相对页号
0<>2
4<>6
8<>10
    +----------------------------+----------+
    |0| |2| |4| |6| |8| ... |1023|0|...|1023|
    +----------------------------+----------+
互为伙伴页的相对页号
0<>4
8<>12
16<>20
24<>28
    +----------------------------+----------+
    |0| |4| |8| |12| |16|...|1023|0|...|1023|
    +----------------------------+----------+
```

现在我们总结上面当一对页数为 `2^order` 的连续页互为伙伴是，他们的页号有如下的规则：

1) 互为伙伴的两组连续页，他们的起始相对页号必须是所在 `2^order` 的倍数
2) 互为伙伴的两组连续页，他们的起始相对页号的差值就是 `2^order`
3) 互为伙伴的两组连续页，无论合并多少次后，相邻页组起始相对页号也应该满足条件 1）

对于条件 3）有点难理解，换句话说，如果按 `2^order` 分割的所有伙伴页合并后，一组连续的页就是按 `2^(order+1)` 分割的了，每个分割点的相对页号也应该是 `2^(order+1)` 的倍数，现在我们用个错误例子来释放，错误的合并将导致不对齐的现象：

```
假设在将4页分组合并成八页分组时，我们错误的将 20 号页与 24 号页合并，那么：

    8           16         20          24          28          32           36
    +-----------+-----------+-----------+-----------+-----------+-----------+
    |     4     |     4     |     4     |     4     |     4     |     4     |
    +-----------+-----------+-----------+-----------+-----------+-----------+
    |           8           |           8           |           8           |
    +-----------+-----------+-----------+-----------+-----------+-----------+
    8                      20                      28                      36

合并后 8、20、28、36 号变成了伙伴页的头页，并且按 8 页分组，但是很明显 20、28 号不是 8 的倍数，所以合并错误。

```

现在就可以给出计算 `2^order` 张连续页的相对页号 `relative_pfn` 的伙伴页号 `buddy_relative_pfn` 了的伪代码了：

```c
int calculate_buddy_relative_pfn(int relative_pfn, int order)
{
    /*使用-1表示计算失败，给出的页不是伙伴页的页号*/
    int size = 1<<order;
    int next_size = size * 2;
    /*首先根据上面的条件一判断给定的相对页号是否所属指定连续页数的伙伴页*/
    if (relative_pfn % size != 0)
        return -1;
    /*现在 relative_pfn 确实是一组伙伴页的其中一个，只需要加上或减去页数就可以得到，
     *至于应该加还是减，根据条件三来判断*/

    /*如果当前是伙伴页组的首页组，那么给定页号加上一下个合并的页数就应该是下一个合并页数的倍数*/
    if ((relative_pfn + next_size) % next_size == 0)
        return relative_pfn + size;
    /*否则应该减去*/
    assert(relative_pfn > size);
    assert((relative_pfn - size) % next_size == 0);
    return relative_pfn - size;
}
```

合并后，还需要获取到该组伙伴页的首页组的页号，自然页号小的才是首页组的。内核对上面的函数使用 *异或* 操作做了优化，通过C语言表达式 `relative_pfn ^ (1<<order)` 就简单的获取到了伙伴页的相对页号，并通过 `relative_pfn & buddy_relative_pfn` 就得到了该组首页的页号。伙伴页组必须处于释放状态才能合并，判断函数 `：

```c
static inline int page_is_buddy(struct page *page, int order)
{
       if (PagePrivate(page)           &&
           (page_order(page) == order) && /*释放状态的页（页组的首页） order 存在 private 字段中*/
           !PageReserved(page)         && /*不是保留的页*/
            page_count(page) == 0) /*由于 PG_private 被标记时，还可能处于其他容器中，所以还得判断引用计数*/
               return 1;
       return 0;
}
```

上面一直强调 **`相对页号`**，即相对于页所属 `zone` 起始页帧号的偏移值，这里必须使用 `相对页号`，因为即使通过绝对页帧号计算出伙伴页，不是绝对属于同一个 `zone` ，`zone` 是一个软件层的概念呢，我们完全可以将 `zone` 按非 `4MB` 的大小划分，比如：

```
将原来的 zone_normal 切割成 18 + 862 MB

     | ZONE_NORMAL |
+----+------+------+-----+
|16MB| 18MB |862MB |128MB|
+----+------+------+-----+

那么 ZONE_NORMAL_862MB 的初始化后的 `free_area[]` 链表应该是：

    +---+
 10 | + |->[4MB]->...->[4MB]
    +---+
 9  | + |->[2MB]
    +---+
    |...|
    +---+

对于多出的 2MB 应该是尾部的那 2MB，而不是头部的。

```

内核使用 `bad_range()` 来判断给定的伙伴页是否属于同一个 `zone` 并页帧在该 `zone` 有效的区间内：

```c
static int bad_range(struct zone *zone, struct page *page)
{
    /*是否属于同一个 zone */
	if (zone != page_zone(page))
		return 1;
    /*是否在有效的区间内*/
	if (page_to_pfn(page) >= zone->zone_start_pfn + zone->spanned_pages)
		return 1;
	if (page_to_pfn(page) < zone->zone_start_pfn)
		return 1;
	return 0;
}
```

现在分配和释放函数都有了，只需要加上互斥就可以安全的多CPU操作了。

## 冷热缓存

内核在大多数时间都使用单页，所以内核选择缓存单页，从而降低分裂合并以及多CPU互斥的成本，提高内核的效率。冷热页与缺页有关，这里暂时不展开讲述了，只需要知道冷热页是抽象概念，和物理内存无关。

1. 初始化缓存

每个 `zone` 中都有这样的缓存，关于 `per-cpu` 可以将其比作线程私有数据，可以使多个CPU无锁操作的一种数据结构，数据结构如下：

```c
struct per_cpu_pages {
	int count;		/*链表中的页数量 */
	int low;		/* 低水平位，需要填充 */
	int high;		/* 高水平位，需要清空 */
	int batch;		/* 批量增加或删除的数量 */
	struct list_head list;	/* 缓存链表*/
};

struct per_cpu_pageset {
	struct per_cpu_pages pcp[2];	/* 0: hot.  1: cold */
	...
} ____cacheline_aligned_in_smp;
```

注意使用 `per-cpu` 结构时，必须禁用调度才能保证多CPU安全，如果次函数还在（软）中断上下文使用，还必须禁用中断。几个水平值在 `free_area_init_core()` 被初始化：

```c
static void __init free_area_init_core(struct pglist_data *pgdat,
		unsigned long *zones_size, unsigned long *zholes_size)
{
	...
	for (j = 0; j < MAX_NR_ZONES; j++) {
		...
		
		batch = zone->present_pages / 1024;
		if (batch * PAGE_SIZE > 256 * 1024)
			batch = (256 * 1024) / PAGE_SIZE;
		batch /= 4;
		if (batch < 1)
			batch = 1;

		for (cpu = 0; cpu < NR_CPUS; cpu++) {
			struct per_cpu_pages *pcp;

			pcp = &zone->pageset[cpu].pcp[0];	/* hot */
			pcp->count = 0;
			pcp->low = (int)(2 * batch);
			pcp->high = (int)(6 * batch);
			pcp->batch = (int)(1 * batch);
			INIT_LIST_HEAD(&pcp->list);

			pcp = &zone->pageset[cpu].pcp[1];	/* cold */
			pcp->count = 0;
			pcp->low = 0;
			pcp->high = (int)(2 * batch);
			pcp->batch = (int)(1 * batch);
			INIT_LIST_HEAD(&pcp->list);
		}
	}
}
```

核心是计算 `batch` 数量，它不能超过zone总页数的四千分之一且不能超过16张页。

1. 从缓存分配

内核通过分配标志来控制分配的行为，后面会详述。当分配单页时，内核查看标志确定使用者是分配冷页还是热页，然后从相应的 `per-cpu` 单页缓存分配；如果缓存中没有页，则批量分配一些单页缓存起来，再从缓存中分配一页。

```c
/*批量分配 count 个 2^order 的连续页，存储 list 链表中*/
static int rmqueue_bulk(struct zone *zone, unsigned int order, 
			unsigned long count, struct list_head *list)
{
	int allocated = 0;
	struct page *page;
	...
	for (int i = 0; i < count; ++i) {
		page = __rmqueue(zone, order);
		if (page == NULL)
			break;
		allocated++;
		list_add_tail(&page->lru, list);
	}
	...
	return allocated;
}

static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
	unsigned long flags;
	struct page *page = NULL;
	/*获取冷热标志*/
	int cold = !!(gfp_flags & __GFP_COLD);

	if (order == 0) {
		/*只需要一页时，从缓存在per-cpu中的冷热页中获取*/
		struct per_cpu_pages *pcp;
		/*禁用调度和中断后，只有当前CPU的当前路径能操作此缓存*/
		pcp = &zone->pageset[get_cpu()].pcp[cold];
		...
		/*过低，则从buddy中分配一批 (pcp->batch) 单个的页框，并插入pcp->list中*/
		if (pcp->count <= pcp->low)
			pcp->count += rmqueue_bulk(zone, 0, pcp->batch, &pcp->list);
		if (pcp->count) {
			/*如果有缓存，弹出一个用于本次分配*/
			page = list_first_entry(&pcp->list, struct page, lru);
			list_del(&page->lru);
			pcp->count--;
		}
		...
    } else {
		//非单页分配
		...
		page = __rmqueue(zone, order);
		...
	}

	/*其他初始化工作*/
	...
	return page;
}
```

上面核心就是缓存的单页数过低时，批量插入一批。

2. 释放到缓存

释放单页时，如果缓存的单页数超过一个阈值则清空，然后再缓存释放的页。是否为冷页，还是热页，同样由使用者指定。

```c
/*批量释放 在 list 链表中的 count 个 2^order 的连续页*/
static int free_pages_bulk(struct zone *zone, int count,
		struct list_head *list, unsigned int order)
{
	unsigned long flags;
	struct page *base, *page = NULL;
	int ret = 0;

	base = zone->zone_mem_map;
	...
	while (!list_empty(list) && count--) {
		/*从尾部开始释放*/
		page = list_last_entry(list, struct page, lru);
		list_del(&page->lru);
		/*释放到伙伴系统*/
		__free_pages_bulk(page, base, zone, order);
		ret++;
	}
	...
	return ret;
}
static void fastcall free_hot_cold_page(struct page *page, int cold)
{
	struct zone *zone = page_zone(page);
	struct per_cpu_pages *pcp;
	...

	pcp = &zone->pageset[get_cpu()].pcp[cold];/*归还到当前cpu的冷热页混存*/
	...
	if (pcp->count >= pcp->high)/*过多则归还给伙伴系统*/
		pcp->count -= free_pages_bulk(zone, pcp->batch, &pcp->list, 0);
	list_add(&page->lru, &pcp->list);/*归还到头部*/
	pcp->count++;
	...
}
```

最新的内核优化了入缓存的先后，先释放到缓存，再清空缓存（如果需要），因为先清空缓存，但此次释放却能填满缓存，这样缓存就不能得到及时的清空。

## 分配策略

页分配策略就是内核可以通过一些列手段来控制从某个具体的管理区 `zone` 或按某种优先级排列的一批 `zone` 来满足调用者的分配。

1. 获取页标志

首先调用这可以通过传入一组标志来控制分配的行为。这些常用标志如下：

|标志名|解释|
|:-:|:-|
|`__GFP_DMA`|请求分配 `DMA` 区域的内存|
|`__GFP_HIGHMEM`|请求分配 `HIGHMEM` 区域的内存 |
|`__GFP_WAIT`|运行分配请求被阻塞，直到有可用的内存能分配为止|
|`__GFP_HIGH`|高优先级分配，允许分配伙伴系统保留的页框|
|`__GFP_COLD`|分配缓存的冷页，仅单页分配请求时有效|
|`__GFP_COMP`|分配组合页，仅非单页分配请求时有效|
|`__GFP_ZERO`|分配成功后，清零所有页中的数据|

其中 `__GFP_COMP` 和 `__GFP_ZERO` 分配成功后的控制标志，`__GFP_COMP` 称为组合页，他可以通过这组页的首页就知道后续有多少张连续的页，并且从后续任何一张页中可以获取到首页，从而使页的一些操作变得简单，比如释放，因为连续页的释放必须知道页数，通过构建组合页，我们在释放时直接从首页中获取。构建代码片段如下：

```c
static struct page *
buffered_rmqueue(struct zone *zone, int order, int gfp_flags)
{
	...
	if (page != NULL) {
		/*检查flags，mapping字段，并设置 _count=0 字段*/
		prep_new_page(page, order);
		/*通过临时映射清零对应的页框数据*/
		if (gfp_flags & __GFP_ZERO)
			prep_zero_page(page, order, gfp_flags);
		/*复合页，则设置flags和private字段*/
		if (order && (gfp_flags & __GFP_COMP))
			prep_compound_page(page, order);
	}
	...
}

static void prep_new_page(struct page *page, int order)
{
	...
	/*移除遗留的不需要的标志*/
	page->flags &= ~(1 << PG_uptodate | 1 << PG_error |
			1 << PG_referenced | 1 << PG_arch_1 |
			1 << PG_checked | 1 << PG_mappedtodisk);
	page->private = 0;
	/*初始化页的引用计数，新分配的仅首页会设置引用计数，并且为1*/
	set_page_refs(page, order);
}

/*由于分配的页可能是高端内存的页，所以不能直接使用，需要映射到临时映射区才能寻址并清零*/
static inline void prep_zero_page(struct page *page, int order, int gfp_flags)
{
	int i;
	/*遍历连续页组中的所有页，并清零*/
	for(i = 0; i < (1 << order); i++) {
		void *kaddr = kmap_atomic(&page[i], KM_USER0);
		memset(kaddr, 0, PAGE_SIZE);
		kunmap_atomic(kaddr, KM_USER0);
	}
}

static void prep_compound_page(struct page *page, unsigned long order)
{
	int i;
	int nr_pages = 1 << order;

	page[1].mapping = NULL;
	/*第二页 index 存入 数量信息*/
	page[1].index = order;
	for (i = 0; i < nr_pages; i++) {
		struct page *p = page + i;

		/*每一页都有 PG_compound 标志*/
		SetPageCompound(p);
		/*每一页的 private 都存放首页描述符*/
		p->private = (unsigned long)page;
	}
}
```

组合页示意图：

```

	+------+---+---+---+
	|      |   |   |   |
	+^-^-^-+-+-+-+-+-+-+
	 | | |   |   |   |
	 | | +---+   |   |
	 | +---------+   |
     +---------------+
```

其他标志将在最终的分配页策略函数中介绍。

2. 后备管理区列表

内核分配页时，不论策略怎样，一定会从一个节点中的某个区分配，如果一个区分配没有可分配的页，甚至整个区没有可分配的页，那么必须有一种后备方案，从最近的节点的相应区请求分配页，如果仍没有则再继续向更远一点的节点请求，直到找到分配成功或遍历完所有的节点的所有区域，后备管理列表就是满足这样的分配策略的数据结构，数据结构片段如下：

```c
typedef struct pglist_data {
	...
	struct zonelist node_zonelists[GFP_ZONETYPES];/**<
			*分配区使用的链表优先顺序，
			*包含所有numa上的节点中的区域
			* @see build_zonelists()
			*/
	...
} pg_data_t;
```

内核通过将自身的管理区放置在 `后备列表` 的前面，其他节点的管理区放后面，就可以做到以 `struct zonelist node_zonelists[]` 作为请求分配页的入口。该数据结构整体是一个二维数组，由 `build_zonelists()` 初始化，代码片段展示省略，我们给出初始化后的分布示意图：

```

   node_zonelists[3]                 node_zonelists[x]->zones[NODES][4]

 		              |<------ NODE  ----->|<------ NODE' ----->|<----- NODE"  ----->|
 		+------+      +------+------+      +------+------+      +------+------+       
 		|NORMAL| ---> |NORMAL| DMA  |      |NORMAL| DMA  |      |NORMAL| DMA  |       
 		+------+      +------+------+------+------+------+------+------+------+       
 		| DMA  | ---> | DMA  |             | DMA  |             | DMA  |
 		+------+      +------+------+------+------+------+------+------+------+------+
 		| HIGH | ---> | HIGH |NORMAL| DMA  | HIGH |NORMAL| DMA  | HIGH |NORMAL| DMA  |
 		+------+      +------+------+------+------+------+------+------+------+------+
```

简单的说明上图，第二维数组存储的 `zone` 描述符，对于一个节点来说，`NODE`区域的 `zone` 数据组存储的 一定是自身的 `zone` 描述符，后续的 `NODE'` 和 `NODE"` 是其他节点的 `zone`，并根据 其他节点的与 `NODE` 在物理上距离排序的，这样就可以做到当最近节点的课分配页使用完毕，就去最近的节点请求分配。第一维是请求分配页各个类型的 `zone` 列表入口，注意0号下标是 `NORMAL` 内存区的列表，因为这个数组的分布是与 `__GFP_DMA`（值为1）和 `__GFP_HIGHMEM`(值为2) 对应的。因为请求 `HIGHMEM` 其他的类型的内存肯定可以满足，请求 `NORMAL` 类型的内存 `DMA` 类型的页可以满足，而请求 `DMA` 只能是 `DMA` 类型的页，因为请求者可能需要内存进行磁盘 `IO` 操作。所以当我们没有使用任何标志，则分配的是普通页，那么将优先从本地节点请求，包括本地的 `DMA` 区域，本地没有了才以同样的方式从其他节点请求，不能分配 `HIGHMEM`，所以整个链都没有相应的 `zone`（紧凑排列的，没有使用 `NULL` 来站位）；使用 `__GFP_DMA` 标志的内存肯定只能从 `DMA`，所以这个列表中只有 `DMA` 的描述符；`__GFP_HIGHMEM` 标志同样。

内核通过 `__alloc_pages()` 作为带有策略的页分配入口，实现片段如下：

```c
struct page * fastcall
__alloc_pages(unsigned int gfp_mask, unsigned int order,
		struct zonelist *zonelist)
{
	const int wait = gfp_mask & __GFP_WAIT;
	...

	/*
     * 如果当前上下文是实时进程且不再中断环境，或者调用者不想休眠
     * 那么可能会多动用一些保留的页
	 */
	can_try_harder = (unlikely(rt_task(p)) && !in_interrupt()) || !wait;

	zones = zonelist->zones;

    /*
     * 获取内存管理区域的类型
     */
	classzone_idx = zone_idx(zones[0]);

restart:
	{
        /* 这个循环是分配的核心，每次减低分配阈值并复用此段代码，
		 * 逐步的减少保留页数的要求来进行分配，我们使用一个伪码来表示，
		 * 以使整个分配策略流程清晰，详细的讲解在下面的片段中
		 */

		/*首次尝试分配，使用较小的水平位来分配*/
		{alloc_page_through_zone_with_watermark}(low_watermark)

	    /*到这里表示没有足够连续的页，则唤醒交换线程进行页框回收*/
	    for (i = 0; (zone = zones[i]) != NULL; i++)
	    	wakeup_kswapd(zone, order);

	    /*
	     * 来到这里，说明整个内存比较吃紧了，如果传入的标志允许动用保留的页，
		 * 则使用中等水平位再次获尝试获取页
	     */
		{alloc_page_through_zone_with_watermark}(min_watermark)

	    /* 
	     * 如果不在中断中，且当前进程正在试图回收页，基本上是因为回收页需要内存而产生的递归调用
	     */
	    if (((p->flags & PF_MEMALLOC) ||
                 unlikely(test_thread_flag(TIF_MEMDIE)))
            && !in_interrupt()) {
			/*则不判断水平位的分配满足需要的页*/
			{alloc_page_through_zone}()
			/*如果失败只有失败返回，交由上层处理，因为已经在回收页了*/
	    	goto nopage;
	    }

	    /* 到此整个内核都没有可分配的内存了，只能选择启动并等待回收页，
		 * 如果调用不愿意等待只能失败返回 */
	    if (!wait)
            goto nopage;
	}
rebalance:
	...	
	/*设置进程标志：正在因分配页而回收页框*/
	p->flags |= PF_MEMALLOC;
	...

	/*试图释放一些页 : 直接回收*/
	did_some_progress = try_to_free_pages(zones, gfp_mask, order);

	...

	if (likely(did_some_progress)) {
		/*
		 * 成功回收了一些页，使用较小的水平位再次尝试。
		 */
		{alloc_page_through_zone_with_watermark}(min_watermark)
	} else if ((gfp_mask & __GFP_FS) && !(gfp_mask & __GFP_NORETRY)) {
		/*如果没有回收到页，请求者分配页是用于文件系统的操作，比如回收页时的换出页到磁盘，
		 *且有不是不愿意重试标志，那么使用较高的水平位再次尝试*/
		{alloc_page_through_zone_with_watermark}(high_watermark)
		/*既然直接回收没有回收到任何页，且又需要页来写磁盘（被视为紧急的任务），
		 *只有选择杀死一些可以杀死的进程，释放他们使用的内存，然后重试*/
		out_of_memory(gfp_mask);
		goto restart;
	}

	/*
	 * 在分配大块内存时不要循环重试，除非调用者明确的通过给出
	 * __GFP_REPEAT 提出这样的要求。然后等待一些换出写请求完成。
	 */
	do_retry = 0;
	if (!(gfp_mask & __GFP_NORETRY)) {
		if ((order <= 3) || (gfp_mask & __GFP_REPEAT))
			do_retry = 1;
		if (gfp_mask & __GFP_NOFAIL)
			do_retry = 1;
	}
	if (do_retry) {
		/*休眠一段时间，继续尝试回收*/
		blk_congestion_wait(WRITE, HZ/50);
		goto rebalance;
	}

nopage:
	...
	return NULL;
got_pg:
	...
	return page;
}
```

内核在每个 `zone` 都设置了一组页数水平位属性，即不同数量的不动用的页数，然后根据请求的缓急来适当动用这些保留的页。比如比起硬件中断中的页请求，用户态的缺页异常是一个可以缓慢的请求，那么中断分配可以使用低水平位来请求，即可以动用更多的保留页，而缺页异常使用较高的水平位分配。这个算法由如下片段和函数完成，就是上面伪码 `{alloc_page_through_zone_with_watermark}(<#some_watermark>)` 的实现细节：

```c
struct page * fastcall
__alloc_pages(unsigned int gfp_mask, unsigned int order,
		struct zonelist *zonelist)
{
	...
	/*遍历当前给定的后备管理区列表，以不同的水平位值来请求内存*/
	for (i = 0; (zone = zones[i]) != NULL; i++) {
		if (!zone_watermark_ok(zone, order, zone->pages_low,
	       			classzone_idx, 0, 0))
	       	continue;
	   	/*从伙伴系统中分配满足要求的页*/
	    page = buffered_rmqueue(zone, order, gfp_mask);
	    if (page)
	    	goto got_pg;
    }
	...
}
```

核心函数是 `zone_watermark_ok()` 只有他返回真，表示该管理区有足够的内存可以用于分配，或运行动用什么水平的保留页，才允许从伙伴系统中分配内存，否则继续查看后备管理区。

```c
int zone_watermark_ok(struct zone *zone, int order, unsigned long mark,
		      int classzone_idx, int can_try_harder, int gfp_high)
{
	/* free_pages my go negative - that's OK */
	long min = mark, /**< 水平位值，越小则要求保留的页更少*/
		free_pages/**<假设分配后的剩余页数*/ = zone->free_pages - (1 << order) + 1;
	int k;

	/* 根据参数修正水平位，分配高端页时，要求保留内存少一些*/
	if (gfp_high)
		min -= min / 2;
	/* 根据参数修正水平位，，要求保留内存更少一些*/
	if (can_try_harder)
		min -= min / 4;

	/*加上固定保留页数，剩余页数不能小于水平位*/
	if (free_pages <= min + zone->lowmem_reserve[classzone_idx])
		return 0;
	
	/*要求 每种 2^k 连续页 剩余数要 满足 free_pages > (min/(2^k))*/
	for (k = 0; k < order; k++) {
		free_pages -= zone->free_area[k].nr_free << k;
		min >>= 1;
		if (free_pages <= min)
			return 0;
	}
	return 1;
}
```

几个水平位 `zone->pages_low` 、 `zone->pages_low` 和 `zone->pages_high` 由 `zone` 总页数决定，参见 `setup_per_zone_pages_min()`。

3. 内存策略

可以用一组属性来描述请求内存时更喜好从哪个节点分配内存，比如如果一个进程绑定了在一组CPU上运行，通过这样描述请求内存时，伙伴系统就优先从离这些CPU最近的节点上分配内存，或者进程通过虚地址映射，让不同的虚地址使用不同节点的内存来处理缺页。这个就是内存策略属性，每个进程描述符中都有这样的一个属性，描述如下：

```c
#define MPOL_DEFAULT	0 /*默认策略，使用当前所属的节点*/
#define MPOL_PREFERRED	1 /*喜好策略，使用固定节点*/
#define MPOL_BIND	2 /*绑定策略，使用固定管理区列表*/
#define MPOL_INTERLEAVE	3 /*交错策略，使用多个固定节点*/

struct mempolicy {
	atomic_t refcnt;
	short policy; 	/* 是 MPOL_XXX 的一个值，用于确定确定策略类型和联合中的哪个字段 */
	union {
		struct zonelist  *zonelist;	/* 直接从绑定的一后备列表分配内存 */
		short 		 preferred_node; /* 从喜好的某个节点分配内存*/
		DECLARE_BITMAP(nodes, MAX_NUMNODES); /* 从喜欢的一组节点分配内存 */
	} v;
};
```