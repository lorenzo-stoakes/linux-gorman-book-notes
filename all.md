# Understanding the Linux Virtual Memory Manager

## Chapter 1: Introduction

Skipped.

## Chapter 2: Describing Physical Memory

* NUMA - Non-Uniform Memory Access. Memory arranged into banks, incurring a
  different cost for access depending on their distance from the processor.

* Each of these banks is called a 'node', represented by
  [struct pglist_data][pglist_data] _even if the arch is UMA_.

* The struct is always referenced by a typedef, `pg_data_t`.

* Every node is kept on a `NULL` terminated linked list, `pgdat_list`, and
  linked by `pg_data_t->node_next`.

* On UMA arches, only one `pg_data_t` structure called `contig_page_data` is
  used.

* Each node is divided into blocks called zones, which represent ranges of
  memory, described by [struct zone_struct][zone_struct], typedef-d to `zone_t`,
  one of `ZONE_DMA`, `ZONE_NORMAL` or `ZONE_HIGHMEM`.

* `ZONE_DMA` is kept within lower physical memory ranges that certain ISA
  devices need.

* `ZONE_NORMAL` memory is directly mapped into the upper region of the linear
  address space.

* `ZONE_HIGHMEM` is what's is left.

* In a 32-bit kernel the mappings are:

```
ZONE_DMA - First 16MiB of memory
ZONE_NORMAL - 16MiB - 896MiB
ZONE_HIGHMEM - 896 MiB - End
```

* Many kernel operations can _only_ take place in `ZONE_NORMAL`. So this is the
  most performance critical zone.

* Memory is divided into fixed-size chunks called _page frames_, represented by
  [struct page][page] (typedef'd to `mem_map_t`), and all of these are kept in a
  global `mem_map` array, usually stored at the beginning of `ZONE_NORMAL` or
  just after the area reserved for the loaded kernel image in low memory
  machines (`mem_map_t` is a convenient name for accessing elements of this
  array.)

* Because the amount of memory directly accessible by the kernel (`ZONE_NORMAL`)
  is limited in size, Linux has the concept of _high memory_.

### 2.1 Nodes

* Each node is described by a `pg_data_t`, which is a `typedef` for
[struct pglist_data][pglist_data]:

```c
typedef struct pglist_data {
	zone_t node_zones[MAX_NR_ZONES];
	zonelist_t node_zonelists[GFP_ZONEMASK+1];
	int nr_zones;
	struct page *node_mem_map;
	unsigned long *valid_addr_bitmap;
	struct bootmem_data *bdata;
	unsigned long node_start_paddr;
	unsigned long node_start_mapnr;
	unsigned long node_size;
	int node_id;
	struct pglist_data *node_next;
 } pg_data_t;
```

#### Fields

* `node_zones` - `ZONE_HIGHMEM`, `ZONE_NORMAL`, `ZONE_DMA`.

* `node_zonelists` - Order of zones allocations are preferred
  from. [build_zonelists()][build_zonelists] in `mm/page_alloc.c` sets up the
  order, called by [free_area_init_core()][free_area_init_core]. A failed
  allocation in `ZONE_HIGHMEM` may fall back to `ZONE_NORMAL` or back to
  `ZONE_DMA`.

* `nr_zones` - Number of zones in this node, between 1 and 3 (not all nodes will
  have all zones.)

* `node_mem_map` - First page of the [struct page][page] array that represents
  each physical frame in the node. Will be placed somewhere in the `mem_map` array.

* `valid_addr_bitmap` - Used by sparc/sparc64.

* `bdata` - Boot memory information only.

* `node_start_paddr` - Starting physical address of the node.

* `node_start_mapnr` - Page offset within global `mem_map`. Calculated in
  [free_area_init_core()][free_area_init_core] by determining number of pages
  between `mem_map` and local `mem_map` for this node called `lmem_map`.

* `node_size` - Total number of pages in this zone.

* `node_id` - Node ID (NID) of node, starting at 0.

* `node_next` - Pointer to next node, `NULL` terminated list.

* Nodes are maintained on a list called `pgdat_list`. Nodes are placed on the
  list as they are initialised by the [init_bootmem_core()][init_bootmem_core]
  function. They can be iterated over using [for_each_pgdat][for_each_pgdat], e.g.:

```c
pg_data_t *pgdat;

for_each_pgdat(pgdat)
	pr_debug("node %d: size=%d", pgdat->node_id, pgdat->node_size);
```

### 2.2 Zones

* Each zone is described by a [struct zone_struct][zone_struct] (typedef'd to `zone_t`):

```c
typedef struct zone_struct {
	/*
	 * Commonly accessed fields:
	 */
	spinlock_t              lock;
	unsigned long           free_pages;
	unsigned long           pages_min, pages_low, pages_high;
	int                     need_balance;

	/*
	 * free areas of different sizes
	 */
	free_area_t             free_area[MAX_ORDER];

	/*
	 * wait_table           -- the array holding the hash table
	 * wait_table_size      -- the size of the hash table array
	 * wait_table_shift     -- wait_table_size
	 *                              == BITS_PER_LONG (1 << wait_table_bits)
	 *
	 * The purpose of all these is to keep track of the people
	 * waiting for a page to become available and make them
	 * runnable again when possible. The trouble is that this
	 * consumes a lot of space, especially when so few things
	 * wait on pages at a given time. So instead of using
	 * per-page waitqueues, we use a waitqueue hash table.
	 *
	 * The bucket discipline is to sleep on the same queue when
	 * colliding and wake all in that wait queue when removing.
	 * When something wakes, it must check to be sure its page is
	 * truly available, a la thundering herd. The cost of a
	 * collision is great, but given the expected load of the
	 * table, they should be so rare as to be outweighed by the
	 * benefits from the saved space.
	 *
	 * __wait_on_page() and unlock_page() in mm/filemap.c, are the
	 * primary users of these fields, and in mm/page_alloc.c
	 * free_area_init_core() performs the initialization of them.
	 */
	wait_queue_head_t       * wait_table;
	unsigned long           wait_table_size;
	unsigned long           wait_table_shift;

	/*
	 * Discontig memory support fields.
	 */
	struct pglist_data      *zone_pgdat;
	struct page             *zone_mem_map;
	unsigned long           zone_start_paddr;
	unsigned long           zone_start_mapnr;

	/*
	 * rarely used fields:
	 */
	char                    *name;
	unsigned long           size;
 } zone_t;
```

#### Fields

* `lock` - Spinlock protects the zone from concurrent accesses.

* `free_pages` - Total number of free pages in the zone.

* `pages_min`, `pages_low`, `pages_high` - Watermarks - If `free_pages <
  pages_low`, `kswapd` is woken up and swaps pages out asynchronously. If the
  page consumption doesn't slow down fast enough from this, `kswapd` switches
  into a mode where pages are freed synchronously in order to return the system
  to health (see 2.2.1.)

* `need_balance` - This indicates to `kswapd` that it needs to balance the zone,
  i.e. `free_pages` has hit one of the watermarks.

* `free_area` - Free area bitmaps used by the buddy allocator.

* `wait_table` - Hash table of wait queues of processes waiting on a page to be
  freed. This is meaningful to [wait_on_page()][wait_on_page] and
  [unlock_page()][unlock_page]. A 'wait table' is used because, if processes all
  waited on a single queue, there'd be a big race between processes for pages
  which are locked on wake up (known as a 'thundering herd'.)

* `wait_table_size` - Number of queues in the hash table (power of 2.)

* `wait_table_shift` - Number of bits in a long - binary logarithm of `wait_table_size`.

* `zone_pgdat` - Points to the parent `pg_data_t`.

* `zone_mem_map` - First page in a global `mem_map` that this zone refers to.

* `zone_start_paddr` - Starting physical address of the zone.

* `zone_start_mapnr` - Page offset within global `mem_map`.

* `name` - String name of the zone - `"DMA"`, `"Normal"` or `"HighMem"`.

* `size` - Size of zone in pages.

#### 2.2.1 Zone Watermarks

* When system memory is low, the pageout daemon `kswapd` is woken up to free pages.

* If memory pressure is high `kswapd` will free memory synchronously - the
  _direct-reclaim_ path.

* Each zone has 3 watermarks - `pages_min`, `pages_low` and `pages_high`.

* `pages_min` is determined by [free_area_init_core()][free_area_init_core]
  during memory initialisation and is based on a ratio related to the size of
  the zone in pages, initially as `zone_size_in_pages/128`, its value varies
  from 20 to 255 pages (80KiB - 1MiB on x86.) When this is reached it's time to
  get serious - memory is _synchronously_ freed.

* `pages_low = 2*pages_min` by default. When this amount of free memory is
  reached, `kswapd` is woken up by the 'buddy allocator' in order to start
  freeing pages.

* `pages_high = 3*pages_min` by default. After `kswapd` has been woken to start
  freeing pages, the zone won't be considered to be 'balanced' until
  `pages_high` pages are free again.

#### 2.2.2 Calculating the Sizes of Zones

* The size of each zone is calculated during [setup_memory()][setup_memory].

* _PFN_ - Page Frame Number - is an offset in pages within the physical memory
  map.

* The PFN variables mentioned below are kept in [mm/bootmem.c][bootmem.c].

* `min_low_pfn` - the first PFN usable by the system - is located in the first
  page after the global variable `_end` (this variable represents the end of the
  loaded kernel image.)

* `max_pfn` - the last page frame in the system - is determined in a very
  architecture-specific fashion. In x86 the function
  [find_max_pfn()][find_max_pfn] reads through the whole [e820][e820] map (a
  table provided by BIOS describing what physical memory is available, reserved,
  or non-existent) in order to find the highest page frame.

* `max_low_pfn` is calculated on x86 with
  [find_max_low_pfn()][find_max_low_pfn], and marks the end of
  `ZONE_NORMAL`. This is the maximum page of physical memory directly accessible
  by the kernel, and is related to the kernel/username split in the linear
  address space determined by [PAGE_OFFSET][__PAGE_OFFSET]. In low memory
  machines `max_pfn = max_low_pfn`.

* Once we have these values we can determine the start and end of high memory
  (`highstart_pfn` and `highend_pfn`) [very simply][highstart_pfn_ASSIGN]:

```c
	highstart_pfn = highend_pfn = max_pfn;
	if (max_pfn > max_low_pfn) {
		highstart_pfn = max_low_pfn;
	}
```

* These values are used later to initialise the high memory pages for the
  physical page allocator (see section 5.6)

#### 2.2.3 Zone Wait Queue Table

* When I/O is being performed on a page such as during page-in or page-out, I/O
  is locked to avoid exposing inconsistent data.

* Processes that want to use a page undergoing I/O have to join a wait queue
  before it can be accessed by calling [wait_on_page()][wait_on_page].

* When the I/O is complete the page will be unlocked with
  [UnlockPage()][UnlockPage] (`#define`'d as [unlock_page()][unlock_page]) and
  any processes waiting on the queue will be woken up.

* If every page had a wait queue it would use a lot of memory, so instead the
  wait queue is stored within the relevant [zone_t][zone_struct].

* The process of sleeping on a locked page can be described as follows:

1. Process A wants to lock page.

2. The kernel calls [__wait_on_page()][__wait_on_page]...

3. ...which calls [page_waitqueue()][page_waitqueue] to get the page's wait
   queue...

4. ...which calls [page_zone()][page_zone] to obtain the page's zone's
   [zone_t][zone_struct] structure using the page's `flags` field shifted by
   `ZONE_SHIFT`...

5. ...[page_waitqueue()][page_waitqueue] will then hash the page address to read
   into the `zone`'s `wait_table` field and retrieve the appropriate
   [wait_queue_head_t][__wait_queue_head].

6. This is used by [add_wait_queue()][add_wait_queue] to add the process to the
wait queue, at which point it goes beddy byes!

* As described above, a hash table is used rather than simply keeping a single
  wait list. This is done because a single list could result in a serious
  [thundering herd][thundering herd] problem.

* In the event of a hash collision processes might still get woken up
  unnecessarily, but collisions aren't expected that often.

* The `wait_table` field is allocated during
  [free_area_init_core()][free_area_init_core], its size is calculated by
  [wait_table_size()][wait_table_size] and stored in the `wait_table_size`
  field, with a maximum size of 4,096 wait queues.

* For smaller tables, the size of the table is the minimum power of 2 required
  to store `NoPages / PAGES_PER_WAITQUEUE` (`NoPages` is the number of pages in
  the zone and `PAGES_PER_WAITQUEUE` is defined as 256.) This means the size of
  the table is `floor(log2(2 * NoPages/PAGE_PER_WAITQUEUE - 1))`.

* The filed `zone_t->wait_table_shift` is the number of bits a page address has
  to be shifted right to return an index within the table (using a hash table as
  described above.)

### 2.3 Zone Initialisation

* Zones are initialised after kernel page tables have been fully set up by
  [paging_init()][paging_init]. The idea is to determine what parameters to send
  to [free_area_init()][free_area_init] for UMA architectures (where the only
  parameter required is `zones_size`, or
  [free_area_init_node()][free_area_init_node] for NUMA.

* The parameters are as follows:

```c
void __init free_area_init_node(int nid, pg_data_t *pgdat, struct page *pmap,
	unsigned long *zones_size, unsigned long zone_start_paddr,
	unsigned long *zholes_size)
```
1. `nid` - The node id.

2. `pgdat_` - Node's `pg_data_t` being initialised, in UMA this will be
   [contig_page_data][contig_page_data].

3. `pmap` - This parameter is determined by
   [free_area_init_core()][free_area_init_core] to point to the beginning of the
   locally defined `lmem_map` array which is ignored in NUMA because it treats
   `mem_map` as a virtual array starting at [PAGE_OFFSET][__PAGE_OFFSET] in UMA,
   this pointer is the global `mem_map` variable. __TODO:__ Check this, seems a
   bit vague.

4. `zones_size` - An array containing the size of each zone in pages.

5. `zone_start_paddr` - Starting physical address for the first zone.

6. `zone_holes` - An array containing the total size of memory holes in the zones.

* [free_area_init_core()][free_area_init_core] is responsible for filling in
  each [zone_t][zone_struct] with the relevant information and allocation of the
  `mem_map` for the node. Information on which pages are free for the zones is
  not determined at this point. This information isn't known until the boot
  memory allocator is being retired (discussed in [chapter 5](./5.md).)

### 2.4 Initialising mem_map

* The `mem_map` (type `mem_map_t`, typedef'd to [struct page][page]) area is
  created during system startup in one two ways - on NUMA systems it is treated
  as a virtual array starting at [PAGE_OFFSET][__PAGE_OFFSET].
  [free_area_init_node()][free_area_init_node] is called for each active node in
  the system, which allocates the portion of this array for each node being
  initialised.

* On UMA systems, [free_area_init()][free_area_init] uses
  [contig_page_data][contig_page_data] as the node and the global `mem_map` as
  the local `mem_map` for this node.

* [free_area_init_core()][free_area_init_core] allocates a local `lmem_map` for
  the node being initialised. The memory for this array is allocated by the boot
  memory allocator via [alloc_bootmem_node()][alloc_bootmem_node] (which in turn
  calls [__alloc_bootmem_node][__alloc_bootmem_node]) - for UMA this newly
  allocated memory becomes the global `mem_map`, but for NUMA things are
  slightly different.

* In NUMA, architectures allocate memory for `lmem_map` within each node's own
  memory. The global `mem_map` is never explicitly allocated, but is set to
  [PAGE_OFFSET][__PAGE_OFFSET] which is treated as a virtual array.

* The address of the local map is stored in `pg_data_t->node_mem_map` which
  exists _somewhere_ in the virtual `mem_map`. For each zone that exists in the
  node, the address within the virtual `mem_map` is stored in
  `zone_t->zone_mem_map`. All the rest of the code then treats `mem_map` as a
  real array, because only valid regions within it will be used by nodes.

### 2.5 Pages

* Every physical page 'frame' in the system has an associated
  [struct page][page] used to keep track of its status:

```c
typedef struct page {
        struct list_head list;          /* ->mapping has some page lists. */
        struct address_space *mapping;  /* The inode (or ...) we belong to. */
        unsigned long index;            /* Our offset within mapping. */
        struct page *next_hash;         /* Next page sharing our hash bucket in
                                           the pagecache hash table. */
        atomic_t count;                 /* Usage count, see below. */
        unsigned long flags;            /* atomic flags, some possibly
                                           updated asynchronously */
        struct list_head lru;           /* Pageout list, eg. active_list;
                                           protected by pagemap_lru_lock !! */
        struct page **pprev_hash;       /* Complement to *next_hash. */
        struct buffer_head * buffers;   /* Buffer maps us to a disk block. */

        /*
         * On machines where all RAM is mapped into kernel address space,
         * we can simply calculate the virtual address. On machines with
         * highmem some memory is mapped into kernel virtual memory
         * dynamically, so we need a place to store that address.
         * Note that this field could be 16 bits on x86 ... ;)
         *
         * Architectures with slow multiplication can define
         * WANT_PAGE_VIRTUAL in asm/page.h
         */
#if defined(CONFIG_HIGHMEM) || defined(WANT_PAGE_VIRTUAL)
        void *virtual;                  /* Kernel virtual address (NULL if
                                           not kmapped, ie. highmem) */
#endif /* CONFIG_HIGMEM || WANT_PAGE_VIRTUAL */
} mem_map_t;
```

#### Fields

* `list` - Pages might belong to many lists, and this field is used as the
  `list_head` field for those (kernel linked list work using an embedded field.)
  For example, pages in a mapping will be in one of `clean_pages`,
  `dirty_pages`, `locked_pages` kept by an [address_space][address_space]. In
  the slab allocator, the field is used to store pointers to the slab and cache
  structures managing the page once it's been allocated by the slab
  allocator. Additionally, it's used to link blocks of free pages together.

* `mapping` - When files or devices are memory mapped, their inode has an
  associated [address_space][address_space]. This field will point to this
  address space if the page belongs to the file. If the page is anonymous and
  `mapping` is set, the `address_space` is `swapper_space` which manages the
  swap address space.

* `index` - If the page is part of a file mapping, it is the offset within the
  file. If the page is part of the swap cache, then this will be the offset
  within the `address_space` for the swap address space (`swapper_space`.)
  Alternatively, if a block of pages is being freed for a particular process,
  the order (power of two number of pages being freed) of the block is stored
  here, set in [__free_pages_ok()][__free_pages_ok].

* `next_hash` - Pages that are part of a file mapping are hashed on the inode
  and offset. This field links pages together that share the same hash bucket.

* `count` - This is the reference count of the page - if it drops to zero, the
  page can be freed. If it is any greater, it is in use by one or more processes
  or the kernel (e.g. waiting for I/O.)

* `flags` - Describe the status of the page as declared in
  [linux/mm.h][mm.h]. The only really interesting flag is
  [SetPageUptodate()][SetPageUptodate] which calls an architecture-specific
  function, [arch_set_page_uptodate()][arch_set_page_uptodate] (this seems to
  only actually do something for the S390 and S390-X architectures.)

1. `PG_active` - This bit is set if a page is on the `active_list` LRU and
   cleared when it is removed. It indicates that the page is 'hot'. __Set__ -
   [SetPageActive()][SetPageActive] __Test__ - [PageActive()][PageActive]
   __Clear__ - [ClearPageActive()][ClearPageActive].

2. `PG_arch_1` - An architecture-specific page state bit. The generic code
   guarantees that this bit is cleared for a page when it is first entered into
   the page cache. This allows an architecture to defer the flushing of the
   D-cache (see section 3.9) until the page is mapped by a process. __Set__ -
   None __Test__ - None __Clear__ - None

3. `PG_checked` - Used by ext2. __Set__ - [SetPageChecked()][SetPageChecked]
   __Test__ - [PageChecked()][PageChecked] __Clear__ - None

4. `PG_dirty` - Does the page need to be flushed to disk? This bit ensures a
   dirty page is not freed before being written out. __Set__ -
   [SetPageDirty()][SetPageDirty] __Test__ - [PageDirty()][PageDirty]
   __Clear__ - [ClearPageDirty()][ClearPageDirty]

5. `PG_error` - Set if an error occurs during disk I/O. __Set__ -
   [SetPageError()][SetPageError] __Test__ - [PageError()][PageError]
   __Clear__ - [ClearPageError()][ClearPageError]

6. `PG_fs_1` - Reserved for a file system to use for its own purposes, e.g. NFS
   uses this to indicate if a page is in sync with the remote server. __Set__ -
   None __Test__ - None __Clear__ - None

7. `PG_highmem` - Pages in high memory cannot be mapped permanently by the
   kernel, so these pages are flagged with this bit during
   [mem_init()][mem_init]. __Set__ - None __Test__ -
   [PageHighMem()][PageHighMem] __Clear__ - None

8. `PG_launder` - Useful only for the page replacement policy. When the VM wants
   to swap out a page, it'll set the bit and call [writepage()][writepage]. When
   scanning, if it encounters a page with `PG_launder|PG_locked` set it will
   wait for the I/O to complete. __Set__ - [SetPageLaunder()][SetPageLaunder]
   __Test__ - [PageLaunder()][PageLaunder] __Clear__ -
   [ClearPageLaunder()][ClearPageLaunder]

9. `PG_locked` - Set when the page must be locked in memory for disk I/O. When
   the I/O starts, this bit is set, when it is complete it is cleared.
   __Set__ - [LockPage()][LockPage] __Test__ - [PageLocked()][PageLocked]
   __Clear__ - [UnlockPage()][UnlockPage]

10. `PG_lru` - If a page is either on the `active_list` or the `inactive_list`,
    this is set. __Set__ - [TestSetPageLRU()][TestSetPageLRU]
    __Test__ - [PageLRU()][PageLRU] __Clear__ -
    [TestClearPageLRU()][TestClearPageLRU]

11. `PG_referenced` - If a page is mapped and referenced through the
    mapping/index hash table this bit is set. It's used during page replacement
    for moving the page around the LRU lists. __Set__ -
    [SetPageReferenced()][SetPageReferenced] __Test__ -
    [PageReferenced()][PageReferenced] __Clear__ -
    [ClearPageReferenced()][ClearPageReferenced]

12. `PG_reserved` - Set for pages that can _never_ be swapped out. It is set by
    the boot memory allocator for pages allocated during system startup. Later,
    it's used to flag empty pages or ones that don't exist. __Set__ -
    [SetPageReserved()][SetPageReserved] __Test__ -
    [PageReserved()][PageReserved] __Clear__ -
    [ClearPageReserved()][ClearPageReserved]

13. `PG_slab` - Indicates the page is being used by the slab allocator. __Set__ -
    [PageSetSlab()][PageSetSlab] __Test__ - [PageSlab()][PageSlab]
    __Clear__ - [PageClearSlab()][PageClearSlab]

14. `PG_skip` - Defunct. Used to be used by some Sparc architectures to skip
    over parts of the address space but is no longer used. Completely removed
    in 2.6. __Set__ - None __Test__ - None __Clear__ - None

15. `PG_unused` - Does what it says on the tin. __Set__ - None __Test__ - None
    __Clear__ - None

16. `PG_uptodate` - When a page is read from disk without error, this bit will
    be set. __Set__ - [SetPageUptodate()][SetPageUptodate]
    __Test__ - [Page_Uptodate()][Page_Uptodate] __Clear__ -
    [ClearPageUptodate()][ClearPageUptodate]

* `lru` - For page replacement policy, pages that may be swapped out will exist
  on either the `active_list` or the `inactive_list` declared in
  [page_alloc.c][page_alloc.c]. This is the `struct list_head` field for these
  LRU lists (discussed in [chapter 10](./10.md).)

* `pprev_hash` - The complement to `next_hash`, making the list doubly-linked.

* `buffers` - If a page has buffers for a block device associated with it, this
  field is used to keep track of the [struct buffer_head][buffer_head]. An
  anonymous page mapped by a process may also have an associated `buffer_head`
  if it's backed by a swap file. This is necessary because the page has to be
  synced with backing storage in block-sized chunks defined by the underlying
  file system.

* `virtual` - Normally only pages from `ZONE_NORMAL` are directly mapped by the
  kernel. To address pages in `ZONE_HIGHMEM`, [kmap()][kmap] (which in turn
  calls [__kmap()][__kmap].) is used to map the page for the kernel (described
  further in [chapter 9](./9.md).) Only a fixed number of pages may be
  mapped. When a page is mapped, this field is its virtual address.

### 2.6 Mapping Pages to Zones

* As recently as 2.4.18, a [struct page][page] stored a reference to its zone in
  `page->zone`. This is wasteful as with thousands of pages, these pointers add
  up.

* In 2.4.22 the `zone` field is gone and `page->flags` is shifted by
[ZONE_SHIFT][ZONE_SHIFT] to determine the zone the pages belongs to.

* In order for this to be used to determine the zone, we start by declaring
  [zone_table][zone_table] (`EXPORT_SYMBOL` makes `zone_table` accessible to
  loadable modules.) This is treated like a multi-dimensional array of nodes and
  zones:

```c
zone_t *zone_table[MAX_NR_ZONES*MAX_NR_NODES];
EXPORT_SYMBOL(zone_table);
```

* [MAX_NR_ZONES][MAX_NR_ZONES] is the maximum number of zones that can be in a
  node (i.e. 3.) [MAX_NR_NODES][MAX_NR_NODES] is the maximum number of nodes
  that can exist.

* During [free_area_init_core()][free_area_init_core] all the pages in a node
  are initialised. First it sets the value for the table, where `nid` is the
  node ID, `j` is the zone index and `zone` is the `zone_t` struct:

```c
zone_table[nid * MAX_NR_ZONES + j] = zone;
```

* For each page, the function [set_page_zone()][set_page_zone] is called via:

```c
set_page_zone(page, nid * MAX_NR_ZONES + j);
```

* [set_page_zone()][set_page_zone] is defined as follows, which shows how this
  is used in conjunction with [ZONE_SHIFT][ZONE_SHIFT] to encode a page's zone:

```c
static inline void set_page_zone(struct page *page, unsigned long zone_num)
{
	page->flags &= ~(~0UL << ZONE_SHIFT);
	page->flags |= zone_num << ZONE_SHIFT;
}
```

### 2.7 High Memory

* Because memory in the `ZONE_NORMAL` zone is limited in size, the kernel
supports the concept of 'high memory'.

* Two thresholds of high memory exist on 32-bit x86 systems, one at 4GiB, and a
  second at 64GiB. 32-bit systems can only address 4GiB of RAM, but with
  [PAE][PAE] enabled 64 bits of memory can be addressed, though not all at once
  of course.

* Each page uses 44 bytes of memory in `ZONE_NORMAL`. This means 4GiB requires
  44MiB of kernel memory to describe it. At 16GiB, 176MiB is consumed, and once
  you factor in other structures, even smaller ones like Page Table Entries
  (PTEs), which require 16MiB in the worst case, it adds up. This makes 16GiB
  the maximum practical limit on a 32-bit system. At this point you should
  switch to 64-bit if you want access to more memory.


## Chapter 3: Page Table Management

### (LS) Overview

In linux 2.4.22 a 32-bit virtual address actually consists of 4 offsets,
indexing into the PGD, PMD, PTE and the address's actual physical page of
memory (the PGD, PMD and PTE are defined below.)

Each process has a known PGD address. Subsequently (noting these are all
_physical_ addresses):

```
pmd_table = pgd_table[pgd_offset] & PAGE_MASK
pte_table = pmd_table[pmd_offset] & PAGE_MASK
phys_page = pte_table[pte_offset] & PAGE_MASK
phys_addr = phys_page[page_offset]
```

__NOTE:__ This is pseudo-code.

__NOTE:__ `PAGE_MASK = ~(1<<12 - 1)` excludes the least significant bits since
pages are page-aligned.

This indirection exists in order that each process need only take up 1 physical
page of memory for its overall page directory - i.e. entries can be empty and
are filled as needed (the layout is 'sparse'.)

A virtual address therefore looks like:

```
<-                    BITS_PER_LONG                      ->
<- PGD bits -><- PMD bits -><- PTE bits -><- Offset bits ->
[ PGD Offset ][ PMD Offset ][ PTE Offset ][  Page Offset  ]
                                          <----- PAGE_SHIFT
                            <-------------------- PMD_SHIFT
              <-------------------------------- PGDIR_SHIFT
```

As discussed below, it's possible for offsets to be of 0 bits length, meaning
the directory in question is folded back onto the nearest higher directory.

In 32-bit non-[PAE][PAE] we have:

```
<-  10 bits -><-  0 bits  -><- 10 bits  -><-   12 bits   ->
[ PGD Offset ][ PMD Offset ][ PTE Offset ][  Page Offset  ]
```

In 32-bit [PAE][PAE] we have:

```
<-  2 bits  -><-  9 bits  -><-  9 bits  -><-   12 bits   ->
[ PGD Offset ][ PMD Offset ][ PTE Offset ][  Page Offset  ]
```

Importantly here all physical pages are stored as 64-bit values. A 32-bit intel
PAE configuration could physically address 36 bits (64GB) so only 4 bits of the
higher order byte are used for addressing (some of the other bits are used for
flags, see the [PAE wikipedia article][PAE] for more details.)

### (LS) Page Table Function Cheat Sheet

* `pgd_offset(mm, address)` - Given a process's [struct mm_struct][mm_struct]
  (e.g. for the current process this would be `current->mm`) and a _virtual_
  `address`, returns the _virtual address_ of the corresponding PGD
  _entry_ (i.e. a `pgd_t *`.)

* `pmd_offset(dir, address)` - Given `dir`, a _virtual_ address of type `pgd_t
  *`, i.e. a pointer to the _entry_ which contains the PMD table's _physical_
  address, and a _virtual_ `address`, returns the virtual address of the
  corresponding PMD entry of type `pmd_t *`.

* `pte_offset(dir, address)` - Given `dir`, a _virtual_ address of type `pmd_t
  *`, i.e. a pointer to the _entry_ which contains the PTE table's _physical_
  address, and a _virtual_ `address`, returns the virtual address of the
  corresponding PTE entry of type `pte_t *`.

* `__[pgd,pmd,pte]_offset(address)` - Given a _virtual_ `address`, returns the
  index of the `[pgd,pmd,pte]_t` entry for this address within the PGD, PMD,
  PTE.

* `pgd_index(address)` - Given the _virtual_ `address`, [pgd_index()][pgd_index]
  returns the index of the [pgd_t][pgd_t] entry for this address within the
  PGD. Equivalent to `__pgd_offset(address)`.

* `[pgd,pmd]_page(entry)` - Given `entry`, a page table entry containing a
  _physical_ address of type `[pgd,pmd]_t`, returns the _virtual_ address of the
  page referenced by this entry. So [pgd_page()][pgd_page] returns `pmd_t
  *pmd_table` and [pmd_page()][pmd_page] returns `pte_t *pte_table`.

* `pte_page(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, [pte_page()][pte_page] returns the [struct
  page][page] entry (also typedef'd to `mem_map_t`) for the referenced page.

* `[pgd,pmd,pte]_val(entry)` - Given `entry`, a page table entry containing a
  _physical_ address of type `[pgd,pmd,pte]_t`, returns the absolute value of
  the physical address (i.e. `unsigned long` or `unsigned long long`.) Note that
  when using [PAE][PAE] this will be a 64-bit value.

* `[pgd,pmd,pte]_none(entry)` - Given `entry`, a page table entry containing a
  _physical_ address of type `[pgd,pmd,pte]_t`, returns true if it does not
  exist, e.g. [pmd_none()][pmd_none].

* `[pgd,pmd,pte]_present(entry)` - Given `entry`, a page table entry containing
  a _physical_ address of type `[pgd,pmd,pte]_t`, returns true if it has the
  present bit set e.g. [pmd_present()][pmd_present].

* `[pgd,pmd,pte]_clear(entry)` - Given `entry`, a page table entry containing a
  _physical_ address of type `[pgd,pmd,pte]_t`, clears the entry e.g.
  [pmd_clear()][pmd_clear].

* `[pgd,pmd]_bad(entry)` - Given `entry`, a page table entry containing a
  _physical_ address of type `[pgd,pmd]_t`, returns true if the entry is not in
  a state where the contents can be safely modified, e.g. [pmd_bad()][pmd_bad].

* `pte_dirty(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, returns true if the PTE has the dirty bit set,
  e.g. [pte_dirty()][pte_dirty]. __Important:__ [pte_present()][pte_present]
  __must__ return true for this to be useable.

* `pte_mkdirty(entry)`, `pte_mkclean(entry)` - Given `entry`, a page table entry
  containing a _physical_ address of type `pte_t`, sets or clears the dirty bit
  respectively - [pte_mkdirty()][pte_mkdirty], [pte_mkclean()][pte_mkclean].

* `pte_young(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, returns true if the PTE has the accessed flag set (on
  x86), e.g. [pte_young()][pte_young]. __Important:__
  [pte_present()][pte_present] __must__ return true for this to be useable.

* `pte_mkyoung(entry)`, `pte_mkold(entry)` - Given `entry`, a page table entry
  containing a _physical_ address of type `pte_t`, sets or clears the dirty bit
  respectively - [pte_mkyoung()][pte_mkyoung], [pte_mkold()][pte_mkold].

* `pte_read(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, returns true if the PTE is readable, which is
  implemented by the user flag on x86, e.g. [pte_read()][pte_read].

* `pte_mkread(entry)`, `pte_rdprotect(entry)` - Given `entry`, a page table
  entry containing a _physical_ address of type `pte_t`, sets or clears the user
  flag (on x86) respectively - [pte_mkread()][pte_mkread],
  [pte_rdprotect()][pte_rdprotect].

* `pte_write(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, returns true if the PTE is writeable,
  e.g. [pte_write()][pte_write].

* `pte_mkwrite(entry)`, `pte_wrprotect(entry)` - Given `entry`, a page table
  entry containing a _physical_ address of type `pte_t`, sets or clears the R/W
  flag respectively - [pte_mkwrite()][pte_mkwrite],
  [pte_wrprotect()][pte_wrprotect].

* `pte_exec(entry)` - Given `entry`, a page table entry containing a _physical_
  address of type `pte_t`, returns true if the PTE is executable. Note that on
  x86 at the time of 2.4.22 there was no [NX bit][nx-bit], in fact it was only
  added circa pentium 4, and only in 32-bit mode if [PAE][PAE] is enabled, as it
  is set at the 63rd (most significant) bit. In i386, it's equivalent to
  [pte_read()][pte_read] and checks for the user bit.

* `pte_mkexec(entry)`, `pte_exprotect(entry)` - Given `entry`, a page table
  entry containing a _physical_ address of the type `pte_t`, sets or clears the
  exec bit (or rather user bit in i386) respectively -
  [pte_mkexec()][pte_mkexec], [pte_exprotect()][pte_exprotect].

* `mk_pte(page, pgprot)` - Given `page`, a [struct page][page] and `pgprot`, a
  set of protection bits of type [pgprot_t][pgprot_t], returns a `pte_t` ready
  to be inserted into the page table. This macro in turn calls
  [__mk_pte][__mk_pte/3lvl] which uses a page number (i.e. the address shifted
  to the right by [PAGE_SHIFT][PAGE_SHIFT] bits) to determine how to populate
  the `pte_t` - [mk_pte()][mk_pte].

* `mk_pte_phys(physpage, pgprot)` - Given `physpage`, a physical address and
  `pgprot`, a set of protection bits of type `pgprot_t`, returns a `pte_t` ready
  to be inserted into the page table, similar to `mk_pte()` -
  [mk_pte_phys()][mk_pte_phys].

* `set_pte(ptep, pte)` - Given `ptep`, a pointer to a `pte_t` in the page table,
  and a PTE entry of type `pte_t`, this function sets it in place. This is a
  function rather than simply `*ptep = ptep;` because in some architectures
  (including PAE x86) additional work needs to be done. This is where the PTE
  returned by [mk_pte()][mk_pte] or [mk_pte_phys()][mk_pte_phys] is assigned to
  the actual page table - [set_pte()][set_pte/2lvl] for 2-level (non-PAE) x86,
  [set_pte()][set_pte/3lvl] for 3-level ([PAE][PAE]) x86.

* `pte_clear(xp)` - Given `xp`, a pointer to a `pte_t` in the page table,
  clears the entry altogether - [pte_clear()][pte_clear].

* `ptep_get_and_clear(ptep)` - Given `ptep`, a pointer to a `pte_t` in the page
  table, returns the existing PTE entry and clears it - this is useful when
  modification needs to be made to either the PTE protection or the
  [struct page][page] itself - [pte_get_and_clear()][pte_get_and_clear/2lvl] for
  2-level x86 and [pte_get_and_clear()][pte_get_and_clear/3lvl] for 3-level
  ([PAE][PAE]) x86.

* `pgd_alloc(mm)` - Given a process's `mm`, a [struct mm_struct][mm_struct],
  allocate a physical page for a PGD table, zero
  [USER_PTRS_PER_PGD][USER_PTRS_PER_PGD] entries and copy over entries from
  [swapper_pg_dir][swapper_pg_dir] to populate the rest of the entries (in x86
  at least) - [pgd_alloc()][pgd_alloc]. Typically this calls
  [get_pgd_fast()][get_pgd_fast] in turn, resorting to
  [get_pgd_slow()][get_pgd_slow] if necessary - [pgd_alloc()][pgd_alloc]. Note
  this allocates a PGD table _page_.

* `pmd_alloc(mm, pgd, address)` - Given a process's `mm`, a
  [struct mm_struct][mm_struct], `pgd` a pointer to an entry in a PGD table
  which is to contain the PMD, and `address`, a virtual address, simply returns
  [pmd_offset()][pmd_offset/3lvl] on i386 - i.e. since all PMDs are either
  folded into the PGD (2-level), or are allocated along with PGD, the function
  simply returns a the virtual address of the associated `pmd_t` -
  [pmd_alloc()][pmd_alloc]. Note this allocates a PMD table _page_.

* `pte_alloc(mm, pmd, address)` - Given a process's `mm`, a
  [struct mm_struct][mm_struct], `pmd` a pointer to an entry in a PMD table
  which is to contain the PTE, and `address`, a virtual address, allocates a new
  PTE table if the PMD entry is empty via
  [pte_alloc_one_fast()][pte_alloc_one_fast] if possible (which uses
  [pte_quicklist][pte_quicklist] in a way very similar to
  [get_pgd_fast()][get_pgd_fast], otherwise it resorts to
  [pte_alloc_one()][pte_alloc_one] to allocate the slow way -
  [pte_alloc()][pte_alloc]. Note this allocates a PTE table _page_.

* `pgd_free(pgd)` - Given `pgd` a pointer to a PGD table, frees the associated
  page. In the PAE case, PMD entries are also freed. In i386 is just an alias
  for [free_pgd_slow()][free_pgd_slow] - [pgd_free()][pgd_free].

* `pmd_free(pmd)` - In (2.4.22 :) i386 this is a no-op. In the 2-level page
  table case PMDs don't exist, in the 3-level case (PAE), PMDs are cleared via
  `pgd_free()` - [pmd_free()][pmd_free].

* `pte_free(pte)` - Given `pte`, a pointer to a PTE table, frees the associated
  page. In i386 this is just an alias for [pte_free_fast()][pte_free_fast],
  which simply adds the PTE entry back to the cache's free list -
  [pte_free()][pte_free].

### Introduction

* Linux has a unique means of abstracting the architecture-specific details of
  physical pages. It maintains a concept of a three-level page table (note: this
  is relevant to 2.4.22 of course, in later kernels it is [4 levels][4level].)
  The 3 levels are:

1. Page Global Directory (PGD)

2. Page Middle Directory (PMD)

3. Page Table Entry (PTE)

* This three-level page table structure is implemented _even if_ the underlying
  architecture does not support it.

* Though this is conceptually easy to understand, it means the distinction
  between different types of pages is really blurry, and page types are
  identified by their flags or what lists they reside in rather than the objects
  they belong to.

* Architectures that manage their [Memory Management Unit (MMU)][mmu]
  differently are expected to emulate the three-level page tables. E.g. on x86
  without [PAE][PAE], only two page table levels are available, so the PMD is
  defined to be of size 1 (2^0) and 'folds back' directly onto the PGD, and this
  is optimised out at compile time.

* For architectures that do not manage their cache or
  [Translation lookaside buffer (TLB)][tlb] automatically,
  architecture-dependent hooks have to be left in the code for when the TLB and
  CPU caches need to be altered and flushed, even if they are null operations on
  some architectures like the x86 (discussed further in section 3.8.)

* Virtual memory is divided into separate directories so we can have a 'sparse'
  set of data structures for each process. Each PGD takes up a page of memory
  (4KiB on i386), rather than the 1MiB it would take to map the whole 4GiB
  address space if it were only a single list.

### 3.1 Describing the Page Directory

* Each process has a pointer [mm_struct][mm_struct]`->pgd` to its own PGD, which
  is a physical page frame containing an array of type [pgd_t][pgd_t], an
  architecture specific type defined in `asm/page.h`:

```c
struct mm_struct {
    struct vm_area_struct * mmap;           /* list of VMAs */
    rb_root_t mm_rb;
    struct vm_area_struct * mmap_cache;     /* last find_vma result */
    pgd_t * pgd;
    atomic_t mm_users;                      /* How many users with user space? */
    atomic_t mm_count;                      /* How many references to "struct mm_struct" (users count as 1) */
    int map_count;                          /* number of VMAs */
    struct rw_semaphore mmap_sem;
    spinlock_t page_table_lock;             /* Protects task page tables and mm->rss */

    struct list_head mmlist;                /* List of all active mm's.  These are globally strung
                                             * together off init_mm.mmlist, and are protected
                                             * by mmlist_lock
                                             */

    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
    unsigned long rss, total_vm, locked_vm;
    unsigned long def_flags;
    unsigned long cpu_vm_mask;
    unsigned long swap_address;

    unsigned dumpable:1;

    /* Architecture-specific MM context */
    mm_context_t context;
};
```

* Page tables are loaded differently depending on architecture. On x86, process
  page table is loaded by copying `mm_struct->pgd` into the `cr3` register,
  which has the side effect of flushing the [TLB][tlb].

* In fact, on x86, [__flush_tlb()][__flush_tlb] is implemented by copying `cr3`
  into a temporary register, then copying that result back in again.

* Each _active_ entry in the PGD table points to a page frame (i.e. physical
  page of memory) containing an array of PMD entries of type [pmd_t][pmd_t],
  which in term point to page frames containing PTEs of type [pte_t][pte_t],
  which finally point to page frames containing the actual user data.

* The following code defines how we can determine a physical address manually
  (of course in the majority of cases access is performed transparently by the
  MMU):

```c
/* Shared between 2-level and 3-level i386: */

#define pgd_offset(mm, address) ((mm)->pgd+pgd_index(address))
#define pgd_index(address) ((address >> PGDIR_SHIFT) & (PTRS_PER_PGD-1))
#define pgd_page(pgd) \
	((unsigned long) __va(pgd_val(pgd) & PAGE_MASK))
#define __va(x) ((void *)((unsigned long)(x)+PAGE_OFFSET))
/*
 * Note that pgd_t is a struct containing just the pgd pointer, used to take
 * advantage of C type checking.
 */
#define pgd_val(x) ((x).pgd)

#define PAGE_SHIFT   12 /* 4 KiB. */
#define PAGE_SIZE    (1UL << PAGE_SHIFT)
#define PAGE_MASK    (~(PAGE_SIZE-1))

#define __pte_offset(address) \
		((address >> PAGE_SHIFT) & (PTRS_PER_PTE - 1))
#define pte_offset(dir, address) ((pte_t *) pmd_page(*(dir)) + \
			__pte_offset(address))

#define pmd_page(pmd) \
	((unsigned long) __va(pmd_val(pmd) & PAGE_MASK))

/* In 2-level (non-PAE) configuration: */

#define PGDIR_SHIFT  22
#define PTRS_PER_PGD 1024

#define PMD_SHIFT    22
#define PTRS_PER_PMD 1

#define PTRS_PER_PTE 1024

static inline pmd_t * pmd_offset(pgd_t * dir, unsigned long address)
{
	return (pmd_t *) dir;
}

/* In 3-level (PAE) configuration: */

#define PGDIR_SHIFT  30
#define PTRS_PER_PGD 4

#define PMD_SHIFT    21
#define PTRS_PER_PMD 512

#define PTRS_PER_PTE 512

#define pmd_offset(dir, address) ((pmd_t *) pgd_page(*(dir)) + \
		__pmd_offset(address))
#define __pmd_offset(address) \
		(((address) >> PMD_SHIFT) & (PTRS_PER_PMD-1))
```

* [pgd_offset()][pgd_offset] Determines the _physical address_ of the PGD entry
  for a given virtual address given that address and the process's
  [mm_struct][mm_struct], via [pgd_index()][pgd_index]. We shift the address
  right by [PGDIR_SHIFT][PGDIR_SHIFT/2lvl] bits, before limiting to the number
  of possible pointers, [PTRS_PER_PGD][PTRS_PER_PGD/2lvl]. This number will be a
  power of 2, so the number subtracted by 1 is a mask of all possible indexes
  into the PGD, e.g. `1000` allows all permutations of `[0000,0111]`. Finally we
  add the known physical address of the pgd from `mm->pgd`.

#### 2-level i386

* The structure is a PGD table with 1024 entries -> PTE table with 1024
  entries. The PMD is just an alias for the PGD.

* In a 2-level i386 configuration, [PGDIR_SHIFT][PGDIR_SHIFT/2lvl] is 22 and
  [PTRS_PER_PGD][PTRS_PER_PGD/2lvl] is 1024 so we use the upper 10 bits to
  offset into 1024 entries. [pmd_offset][pmd_offset/2lvl] simply returns a
  (physical) pointer to the PGD entry, so when we translate to PMD, we are just
  looking at the same PGD entry again. The compiler will remove this unnecessary
  intermediate step, but we retain the same abstraction in generic code. Very
  clever!

* Note that the input `dir` value is a _virtual_ address of type `pgd_t *`,
  whose value is a _physical_ address for a PGD _entry_, so the returned `pmd_t
  *` is exactly the same - a virtual address which contains a physical address
  value.

* Now we have the physical address of our 'PMD' entry (i.e. the same PGD entry),
  we can use this to determine our PTE and finally get hold of the address of
  the physical page we want, exactly the same as we would with a 3-level system.

#### 3-level i386 (PAE)

* The structure is a PGD table with only 4 entries -> PMD table with 512 64-bit
  entries (to store more address bits) -> PTE table with 512 entries.

* In a 3-level i386 PAE configuration, [PGDIR_SHIFT][PGDIR_SHIFT/3lvl] is 30 and
  [PTRS_PER_PGD][PTRS_PER_PGD/3lvl] is 4 and we have an actually real PMD :) to
  get it, we use [pmd_offset()][pmd_offset/3lvl], which starts by calling
  [pgd_page()][pgd_page] to get the _virtual_ address of the physical PMD
  contained inside the PGD entry. It starts by calling [pgd_val()][pgd_val]:

* `pgd_val()` gets the physical address of the specified PGD _entry_ -
  [pgd_t][pgd_t] is actually a `struct` containing a physical page address to
  make use of C type checking, so it does this by simply accessing the member.

* Now we come back to `pgd_page()` which uses [PAGE_MASK][PAGE_MASK] (this masks
  out the lower 12 bits, i.e. 4KiB's worth) to get the physical page of the PMD,
  ignoring any flags contained in the PGD entry, then uses [__va][__va] to
  convert this physical _kernel_ address into a virtual _kernel_ address,
  i.e. simply offsetting by [__PAGE_OFFSET][__PAGE_OFFSET] (kernel addresses are
  mapped directly like this.)

* Finally now we have a virtual address for the start of the PMD entry table,
  `pmd_offset()` needs to get the offset of the specific `pmd_t` we're after in
  order to return a `pmd_t *` virtual address to it. It does this via
  [__pmd_offset][__pmd_offset] which does the same job as
  [pgd_index()][pgd_index], only using [PMD_SHIFT][PMD_SHIFT/3lvl] (21) and
  [PTRS_PER_PMD][PTRS_PER_PMD/3lvl] (512.) It is a 21-bit shift because we've
  used 2 bits for the PMD entry, leaving us with 9 bits to address 512 entries.

#### (LS) Looking up the PTE

* Similar to [pmd_offset()][pmd_offset/3lvl], [pte_offset()][pte_offset] grabs
  the contents of the PMD entry via [pmd_page()][pmd_page] (in a 2-level system
  this will just be the same as [pgd_page()][pgd_page]), using
  [pmd_val()][pmd_val] to grab the `pmd` field from the [pmd_t][pmd_t] and
  finally return a virtual `pte_t *` value. Finally, it returns the _physical_
  address of the PTE as a [pte_t *][pte_t].

* In a 3-level ([PAE][PAE]) system, the [pte_t][pte_t] is a 64-bit value, meaning
  [pte_val()][pte_val/3lvl] returns a 64-bit physical address.

#### (LS) Some additional functions/values

* [PAGE_SIZE][PAGE_SIZE] - Size of each page of memory.

* [PMD_SIZE][PMD_SIZE] - The size of values that are mapped by a PMD, which is
  derived from [PMD_SHIFT][PMD_SHIFT/3lvl] which determines the number of bits
  mapped by a PMD (`PMD_SIZE = 1 << PMD_SHIFT`.)

* [PMD_MASK][PMD_MASK] - The mask for the *upper bits* of a PMD address,
  i.e. PGD+PMD.

* [PGDIR_SIZE][PGDIR_SIZE], [PGDIR_MASK][PGDIR_MASK] - Similar to PMD
equivalents for the PGD.

* If a page needs to be aligned on a page boundary, [PAGE_ALIGN()][PAGE_ALIGN]
  adds `PAGE_SIZE - 1` to the address then simply bitwise ands
  [PAGE_MASK][PAGE_MASK] - since `PAGE_MASK` is a mask for the *high* bits of an
  address without the lower page offset bits, this means that addresses that are
  already page aligned will remain the same (since it's `PAGE_SIZE - 1`), and
  any other addresses will be moved to the next page.

* Somewhat discussed above, but [PTRS_PER_PGD][PTRS_PER_PGD/3lvl],
  [PTRS_PER_PMD][PTRS_PER_PMD/3lvl] and [PTRS_PER_PTE][PTRS_PER_PTE/3lvl] each
  describe the number of pointers provided in each table.

### 3.2 Describing a Page Table Entry

* Each entry in the page tables are described by [pgd_t][pgd_t], [pmd_t][pmd_t]
  and [pte_t][pte_t]. Though the values the entries contain are unsigned
  integers, they are defined as structs. This is firstly for type safety, to
  ensure these are not used incorrectly, and secondly to handle e.g. PAE which
  uses an additional 4 bits for addressing.

* As seen above, type casting from these structs to unsigned integers is done
  via [pgd_val()][pgd_val], [pmd_val()][pmd_val] and [pte_val()][pte_val/3lvl].

* Reversing the process, i.e. converting from an unsigned integer to the
  appropriate struct is done via [__pte()][__pte], [__pmd()][__pmd] and
  [__pgd()][__pgd].

* Each PTE entry has protection bits, for which [pgprot_t][pgprot_t] is used.

* Converting to and from `pgprot_t` is similar to the page table entries -
  [pgprot_val()][pgprot_val] and [__pgprot()][__pgprot].

* Where the protection bits are actually stored is architecture-dependent,
  however on 32-bit x86 non-PAE, [pte_t][pte_t] is just a 32-bit integer in a
  struct. Since each `pte_t` points to an address of a page frame, which is
  guaranteed to be page-aligned, we are left with [PAGE_SHIFT][PAGE_SHIFT] bits
  free for status bits.

* Some of the potential status values are:

1. [_PAGE_PRESENT][_PAGE_PRESENT] - The page is resident in memory and not
   swapped out.
2. [_PAGE_RW][_PAGE_RW] - Page can be written to.
3. [_PAGE_USER][_PAGE_USER] - Page is accessible from user space.
4. [_PAGE_ACCESSED][_PAGE_ACCESSED] - Page has been accessed.
5. [_PAGE_DIRTY][_PAGE_DIRTY] - Page has been written to.
6. [_PAGE_PROTNONE][_PAGE_PROTNONE] - The page is resident, but not accessible.

* The bit referenced by `_PAGE_PROTNONE` is, in pentium III processors and
  higher, referred to as the 'Page Attribute Table' (PAT) bit and used to
  indicate the size of the page that the PTE is referencing. Equivalently, this
  bit is referred to as the 'Page Size Extension' (PSE) bit.

* Linux doesn't use either PAT or PSE in userspace, so this bit is free for
  other uses. It's used to fulfil cases like an [mprotect()][mprotect]'d region
  of memory set to `PROT_NONE` where the memory needs to resident, but
  inaccessible to userland. This is achieved by clearing `_PAGE_PRESENT` and
  setting `_PAGE_PROTNONE`.

* Since `_PAGE_PRESENT` is clear, the hardware will invoke a fault if userland
  tries to access it, but the [pte_present()][pte_present] macro will detect
  that `PAGE_PROTNONE` is defined so the kernel will know to keep this page
  resident.

### 3.3 Using Page Table Entries

* (See the cheat sheet for details on useful functions.)

* There is a lot of page table walk code in the VM, and it's important to be
able to recognise it. For example, [follow_page()][follow_page] from
`mm/memory.c`:

```c
/*
 * Do a quick page-table lookup for a single page.
 */
static struct page * follow_page(struct mm_struct *mm, unsigned long address, int write)
{
	pgd_t *pgd;
	pmd_t *pmd;
	pte_t *ptep, pte;

	pgd = pgd_offset(mm, address);
	if (pgd_none(*pgd) || pgd_bad(*pgd))
		goto out;

	pmd = pmd_offset(pgd, address);
	if (pmd_none(*pmd) || pmd_bad(*pmd))
		goto out;

	ptep = pte_offset(pmd, address);
	if (!ptep)
		goto out;

	pte = *ptep;
	if (pte_present(pte)) {
		if (!write ||
		    (pte_write(pte) && pte_dirty(pte)))
			return pte_page(pte);
	}

out:
	return 0;
}
```

* We first grab a pointer to the PGD entry, check to make sure it isn't empty
  and isn't bad, use this to grab a pointer to the PMD entry, again checking for
  validity before finally grabbing a pointer to the PTE entry which we
  dereference and place a copy in local variable `pte`.

* After this we are done with our walking code and use [pte_page()][pte_page] to
  return the PTE's associated [struct page][page] if the present bit is set and
  if the `write` argument is truthy, checking whether the page is writeable and
  dirty before returning it.

### 3.4 Translating and Setting Page Table Entries

* (See the cheat sheet for details on useful functions.)

### 3.5 Allocating and Freeing Page Tables

* (See the cheat sheet for details on useful functions.)

* The allocation and freeing of page tables is a relatively expensive operation,
  both in terms of time and the fact that interrupts are disabled during page
  allocation.

* In addition, the allocation and deletion of page tables at any of the 3 levels
is very frequent, so it's important that the operation is as quick as possible.

* The pages used for page tables are therefore cached in a number of different
  lists known as _quicklists_.

* The implementation of these differ between architectures, e.g. not all cache
  PGDs because allocating and freeing them only happens during process
  creation/exit, and since these are expensive operations, the allocation of
  another page is negligible.

* Allocation of pages is achieved via [pgd_alloc()][pgd_alloc],
  [pmd_alloc()][pmd_alloc], and [pte_alloc()][pte_alloc].

* Freeing of pages is achieved via [pgd_free()][pgd_free],
  [pmd_free()][pmd_free], and [pte_free()][pte_free].

* Generally speaking, caching is implemented for each of the page tables using
  [pgd_quicklist][pgd_quicklist], [pmd_quicklist][pmd_quicklist], and
  [pte_quicklist][pte_quicklist].

* These lists are defined in an architecture-dependent way but one means this is
  achieved is via a LIFO list (i.e. a stack.) When a page is adding back to the
  cache (i.e. is freed), it becomes the head of the list and the cache size
  count is increment. A pointer to next page in the list is stored at the start
  of the page, e.g.:

```c
static inline void pte_free_fast(pte_t *pte)
{
        *(unsigned long *)pte = (unsigned long) pte_quicklist;
        pte_quicklist = (unsigned long *) pte;
        pgtable_cache_size++;
}
```

* The quick allocation function from `pgd_quicklist` is not necessarily the same
  for each architecture, however commonly, [get_pgd_fast()][get_pgd_fast] is
  used, which pops an entry off the free list if possible, otherwise allocates
  via [get_pgd_slow()][get_pgd_slow]:

```c
static inline pgd_t *get_pgd_fast(void)
{
        unsigned long *ret;

        if ((ret = pgd_quicklist) != NULL) {
                pgd_quicklist = (unsigned long *)(*ret);
                ret[0] = 0;
                pgtable_cache_size--;
        } else
                ret = (unsigned long *)get_pgd_slow();
        return (pgd_t *)ret;
}
```

* Here you can see the dereference of the next element in the list, followed by
clearing this from the page and decrementing the count.

* `pmd_alloc_one` and `pmd_alloc_one_fast` are not implemented in i386 as PMD
  handling is wrapped inside of the PGD.

* On the slow path, [get_pgd_slow()][get_pgd_slow] and
  [pte_alloc_one()][pte_alloc_one] are used to allocate these table pages when
  the cache doesn't have any free entries.

* Obviously over time these caches grow rather large. As the pages grow and
  shrink a counter is incremented/decremented, and there are high and low
  watermarks for this counter.

* [check_pgt_cache()][check_pgt_cache] is called from two different places to
  check these watermarks - the system idle task and after
  [clear_page_tables()][clear_page_tables] is run. If the high watermark is
  reached, pages are freed until the size returns to the low watermark.

### 3.6 Kernel Page Tables

* When the system first starts, paging is not enabled and needs to be
  initialised before turning paging on. This is handled differently by each
  architecture, but again we'll be focusing on i386.

* Page table initialisation is split into two phases - the bootstrap phase sets
  up the page tables up to 8MiB so the paging unit can be enabled, the second
  phase initialises the rest of the page tables.

#### 3.6.1 Bootstrapping

* The assembler function [startup_32()][startup_32] enables the paging unit.

* While all normal kernel code in `vmlinuz` is compiled with base address
  `PAGE_OFFSET` (== [__PAGE_OFFSET][__PAGE_OFFSET]) + 1MiB, the kernel is
  loaded beginning at the first megabyte of memory (0x00100000.)

* This is because the first megabyte of memory is (sometimes, potentially) used
  by some devices with the BIOS and is therefore skipped.

* The bootstrap code deals with this by simply subtracting `__PAGE_OFFSET` from
  any address until the paging unit is enabled.

* Before the paging unit is enabled, a page table mapping has to be established
  that translates 8MiB of physical memory to the virtual address `PAGE_OFFSET`.

* Initialisation begins at _compile_ time by statically defining
  [swapper_pg_dir][swapper_pg_dir] as a PGD and using a linker directive to
  place it at 0x101000 (i.e. `.org 0x1000` which specifies an absolute
  address.):

```as
.org 0x1000
ENTRY(swapper_pg_dir)
        .long 0x00102007
        .long 0x00103007
        .fill BOOT_USER_PGD_PTRS-2,4,0
        /* default: 766 entries */
        .long 0x00102007
        .long 0x00103007
        /* default: 254 entries */
        .fill BOOT_KERNEL_PGD_PTRS-2,4,0
```

* Note that the '7' (0b00000111) in the referenced addresses are flags to set
  the pages 'present' (i.e. in memory), read/write-able, and user-accessible.

* The two subsequent pages of memory (i.e. 0x102000 and 0x103000) are then
  statically defined as [pg0][pg0] and [pg1][pg1].

* If the processor supports the [Page Size Extension (PSE)][pse] feature, the
  pages referenced will be 4MiB in size rather than 4KiB (LS - I can't see how
  this is achieved in the code, confused :/)

* The first pointers to `pg0` and `pg1` in `swapper_pg_dir` are placed to cover
  the region 1-9MiB, the second pointers to them are placed at `PAGE_OFFSET +
  1MiB` - `PAGE_OFFSET + 9MiB`. This is done so that when paging is enabled,
  both physical and virtual addressing will work.

* Once the mapping has been established the paging unit is turned on by setting
  a bit in the `cr0` register and the code [jumps immediately][cr0-jump] to
  ensure `EIP` is set correctly.

* Once the bootstrap paging configuration is set, the rest of the kernel page
  tables are initialised ('finalised') via [paging_init()][paging_init] (and
  subsequently [pagetable_init()][pagetable_init].)

#### 3.6.2 Finalising

* [pagetable_init()][pagetable_init] initialises page tables necessary to access
  `ZONE_DMA` and `ZONE_NORMAL`.

* As discussed previously, high memory in `ZONE_HIGHMEM` cannot be directly
  referenced and temporary mappings are set up for it as needed.

* For each `pgd_t` used by the kernel, the boot memory allocator (more on this
  in chapter 5) is called to allocate a page for the PGD, and the [PSE][pse] bit
  is set if available to use 4MiB TLB entries instead of 4KiB.

* If the [PSE][pse] bit is not supported, a page for PTEs will be allocated for
  each `pmd_t`.

* If the [Page Global Enabled (PGE)][pge] flag is available to be set (this
  prevents automatic [TLB][tlb] invalidation for the page) this will be set for
  each page to make them globally visible to all processes.

* Next, [fixrange_init()][fixrange_init] is called to set up fixed address space
  mappings at the end of the virtual address space starting at
  [FIXADDR_START][FIXADDR_START].

* These mappings are used for special purposes such as the local
  [Advanced Programmable Interrupt Controller (APIC)][apic], and atomic
  kmappings between `FIX_KMAP_BEGIN` and `FIX_KMAP_END` required by
  [kmap_atomic()][kmap_atomic] (see the [highmem kernel doc][highmem-doc] for
  more on this.)

* Finally, [fixrange_init()][fixrange_init] is called again to initialise page
table entries required for normal high memory mapping with [kmap()][kmap].

* After [pagetable_init()][pagetable_init] returns, the page tables for kernel
  space are now fully initialised, so the static PGD
  ([swapper_pg_dir][swapper_pg_dir]) is loaded into the `cr3` register so the
  static table is used by the paging unit.

* The next task of [paging_init()][paging_init] is to call
  [kmap_init()][kmap_init] to initialise each of the PTEs with the `PAGE_KERNEL`
  protection flags.

* Finally [zone_sizes_init()][zone_sizes_init] is called to initialise zone
structures.

### 3.7 Mapping Addresses to a struct page

* Linux needs to be able to quickly map virtual addresses to physical addresses
  and for mapping [struct page][page] to their physical address.

* In order to achieve this by knowing where in both physical and virtual memory
  the global [mem_map][mem_map] array is - it contains pointers to all `struct
  page`s representing physical memory in the system.

* All architectures do this using a similar mechanism, however as always we'll
focus on i386.

#### 3.7.1 Mapping Physical to Virtual Kernel Addresses

* As discussed in 3.6, linux sets up a direct mapping from physical address 0 to
  virtual address `PAGE_OFFSET` at 3GiB on i386.

* This means that any virtual address can be translated to a physical address by
  simply subtracting `PAGE_OFFSET`, which is essentially what
  [virt_to_phys()][virt_to_phys] does, via the [__pa()][__pa] macro, and
  vice-versa with [phys_to_virt()][phys_to_virt] and [__va()][__va]:

```c
#define __pa(x)	                ((unsigned long)(x)-PAGE_OFFSET)
#define __va(x)                 ((void *)((unsigned long)(x)+PAGE_OFFSET))

/**
 *	virt_to_phys	-	map virtual addresses to physical
 *	@address: address to remap
 *
 *	The returned physical address is the physical (CPU) mapping for
 *	the memory address given. It is only valid to use this function on
 *	addresses directly mapped or allocated via kmalloc.
 *
 *	This function does not give bus mappings for DMA transfers. In
 *	almost all conceivable cases a device driver should not be using
 *	this function
 */

static inline unsigned long virt_to_phys(volatile void * address)
{
        return __pa(address);
}

/**
 *	phys_to_virt	-	map physical address to virtual
 *	@address: address to remap
 *
 *	The returned virtual address is a current CPU mapping for
 *	the memory address given. It is only valid to use this function on
 *	addresses that have a kernel mapping
 *
 *	This function does not handle bus mappings for DMA transfers. In
 *	almost all conceivable cases a device driver should not be using
 *	this function
 */

static inline void * phys_to_virt(unsigned long address)
{
        return __va(address);
}
```

#### 3.7.2 Mapping struct pages to Physical Addresses

* As we saw in 3.6.1, the kernel image is located at physical address 1MiB,
  translating to virtual address `PAGE_OFFSET + 0x00100000`, and a virtual
  region of 8MiB is reserved for the image (the maximum the 2 PGD entries allow
  for.)

* This seems to imply that the first available memory would be located at
  `0xc0800000` (given `PAGE_OFFSET` is `0xc0000000`, and we're using the first
  8MiB), however this is not the case.

* Linux tries to reserve the first 16MiB of memory for `ZONE_DMA` so the first
  virtual area used for kernel allocations is actually `0xc1000000`, and is
  where the global [mem_map][mem_map] is usually allocated.

* `ZONE_DMA` will still get used, but only when absolutely necessary.

* Physical addresses get translated into [struct page][page]'s by treated them
  as an index into the `mem_map` array.

* Shifting a physical address [PAGE_SHIFT][PAGE_SHIFT] bits to the right will
  convert them into a 'Page Frame Number' (PFN) from physical address 0, which
  is also an index within the `mem_map` array.

* This is precisely what the [virt_to_page()][virt_to_page] macro does:

```c
#define virt_to_page(kaddr)     (mem_map + (__pa(kaddr) >> PAGE_SHIFT))
```

### 3.8 Translation Lookaside Buffer (TLB)

* Initially, when a process neesd to map a virtual address to a physical one, it
has to traverse the full page directory structures to find the relevant PTE.

* In theory at least, this implies that each assembly instruction that
  references memory actually requires a number of memory references to conduct
  this traversal.

* This is a serious overhead, however the issues are mitigated somewhat by the
  fact that most processes exhibit a 'locality of reference', i.e. a lot of
  accesses tend to be for a small number of pages.

* This 'reference locality' can be taken advantage of by providing a
  ['Translation Lookaside Buffer' (TLB)][tlb], which is a small area of memory
  caching references from virtual to physical addresses.

* Linux assumes that most architectures will have a TLB, and exposes hooks where
  an architecture might need to provide a TLB-related operation, e.g. after a
  page fault has completed, the TLB may need to be updated for that virtual
  address mapping.

* Not all architectures need all of these hooks, but since some do they need to
  be present. The beauty of using pre-processor variables to determine the
  kernel configuration is that architectures which don't need these will have
  stub entries for these and have the code completely removed by the kernel, so
  there's no performance impact.

* There is a fairly large set of TLB API hooks available, described in the
[cachetlb documentation][cachetlb-doc].

* It is possible to just have a single TLB flush function, but since TLB
  flushes/refills are potentially _very_ expensive operations, unnecessary
  flushes need to be avoided if at all possible. For example, Linux will often
  avoid loading new page tables via 'Lazy TLB Flushing', which will be discussed
  further in section 4.3.

### 3.9 Level 1 CPU Cache Management

* Linux manages the CPU cache in a very similar fashion to the TLB. CPU caches,
  like TLB caches, take advantage of locality of reference - to avoid an
  expensive fetch from main memory, instead very small caches are used where possible.

* There are usually (at least) 2 levels of cache - level 1 (L1) and level 2
  (L2). L2 is larger but slower than L1, however linux (at 2.4.22) concerns
  itself with L1 only anyway.

* CPU caches are organised into 'line's. Each line is small, typically 32 bytes,
  and aligned to its boundary size, i.e. a 32-byte line will be aligned on a
  32-byte address.

* On Linux the size of the line is [L1_CACHE_BYTES][L1_CACHE_BYTES] and
  architecture-defined.

* How addresses map to cache lines varies between architectures, but generally
  can be categorised into direct, associative and set associative mappings.

* __Direct mapping__ is the simplest approach - each block of memory maps to
  only one possible cache line.

* __Associative mapping__ means that any block of memory can map to any cache
  line.

* Finally, __set associative mapping__ is a hybrid approach where any block of
  memory can map to any line, but only within a subset of available lines.

* Regardless of how the mappings are performed, addresses that are close
  together and aligned to the cache size are likely to use different lines.

* Given that this is the case, linux uses some simple tricks to improve cache
  'hits':

1. Frequently accessed structure fields are placed at the start of the structure
   to increase the chance that only 1 line is needed to access commonly used
   fields.

2. Unrelated items in a structure should be at least _cache-size_ bytes apart to
   avoid 'false sharing' between CPUs.

3. Objects in the general caches such as the `mm_struct` cache are aligned to
   the L1 CPU cache to avoid 'false sharing'.

* If the CPU references an address that is not in the cache, a 'cache miss'
  occurs and data is fetched from main memory. Roughly (in the 2.4.22 era) a
  reference to a cache can typically be performed in less than 10ns whereas a
  main memory access can take 100-200ns.

* Obviously the main objective is to have as many cache hits and as few cache
  misses as possible.

* Just as some architectures do not automatically manage their TLBs, some do not
  automatically manage their CPU caches, and as with TLBs, hooks are provided.

* The hooks are placed where virtual to physical mappings change, such as during
  a page table update.

* CPU cache flushes should always take place first as some CPUs require a
  virtual to physical mapping to exist when virtual memory is flushed - ordering
  matters.

* The functions that require careful ordering are:

1. [flush_cache_mm()][flush_cache_mm] - Change all page tables.

2. [flush_cache_range()][flush_cache_range] - Change page table range.

3. [flush_cache_page()][flush_cache_page] - Change single PTE.

* On i386 these are no-ops. There is a pleasing comment - 'Caches aren't
  brain-dead on the intel.' :)

* There are additional functions for dealing with cache coherency problems where
  an architecture selects cache lines based on virtual address meaning a single
  physical page may be mapped in several places. We are focusing on x86 here and
  since it's not 'brain-dead' these aren't needed either!

## Chapter 4: Process Address Space

* One of the major advantages of virtual memory is that each process has its own
  virtual address space, mapped to physical memory by the operating system.

* This chapter explores the address space as seen by a process, and how it is
  managed by linux.

* The kernel treats the userspace portion of the address space very differently
  from the kernel portion. For example, allocations for the kernel are satisfied
  immediately and are visible globally, no matter which process is current.

* An exception to this however is [vmalloc()][vmalloc] (and consequently
  [__vmalloc()][__vmalloc]), as it causes a minor page fault to occur to
  synchronise the process page tables with the reference page tables, however
  the page will still be allocated immediately upon request.

* For a process, space is simply reserved in the linear address space by
  pointing a page table entry to a read-only globally visible page filled with
  zeros.

* When the process tries to write to this table a page fault is triggered
  causing the kernel to allocate a new zeroed page and assign it to the PTE and
  mark it writeable. It's zeroed so it appears precisely the same as the global
  zero-filled page.

* The userspace portion of virtual memory is not trusted nor presumed
  constant. After each context switch, the userspace portion of the linear
  address space can change except when a 'Lazy TLB' switch is used (discussed in
  4.3.)

* As a result, the kernel has to be configured to catch all exceptions and
  address errors raised from userspace (discussed in 4.5.)

### 4.1 Linear Address Space

* From a user perspective, the address space is a flat, linear address
  space. The kernel's view is rather different - the address space is split
  between userspace which potentially changes on context switch and the kernel
  address space which remains constant.

* The split is determined by the value of [PAGE_OFFSET][PAGE_OFFSET] (==
  [__PAGE_OFFSET][__PAGE_OFFSET]) - 0xc0000000 on i386 - meaning that 3GiB is
  available for the process to use while the remaining 1GiB is always mapped by
  the kernel.

* Diagramatically the kernel address space looks as follows:

```
            0 -> |-----------------|                  ^                 ^
                 |     Process     |                  |                 |
                 |     Address     |                  |                 |
                 |      Space      |                  |                 |
                 /        .        /                  | TASK_SIZE       |
                 \        .        \                  |                 |
                 /        .        /                  |                 |
                 |                 |                  |                 |
  PAGE_OFFSET -> |-----------------|                  X                 |
                 |      Kernel     |                  | Physical        |
                 |      Image      |                  | Memory Map      |
                 |-----------------|                  |                 |
                 |   struct page   |                  | (Depends on #   | Linear Address
                 |  Map (mem_map)  |                  |  physical RAM)  | Space
                 |-----------------| ^                v                 | (2^BITS_PER_LONG bytes)
                 |    Pages Gap    | | VMALLOC_OFFSET                   |
VMALLOC_START -> |-----------------| v                ^                 |
                 |     vmalloc     |                  |                 |
                 |  Address Space  |                  |                 |
  VMALLOC_END -> |-----------------| ^                |                 |
                 |    Pages Gap    | | 2 * PAGE_SIZE  |                 |
   PKMAP_BASE -> |-----------------| X                |                 |
                 |      kmap       | | LAST_PKMAP *   | VMALLOC_RESERVE |
                 |  Address Space  | | PAGE_SIZE      | at minimum      |
FIXADDR_START -> |-----------------| X                |                 |
                 |  Fixed Virtual  | | __FIXADDR_SIZE |                 |
                 | Address Mapping | |                |                 |
  FIXADDR_TOP -> |-----------------| v                |                 |
                 |    Page Gap     |                  |                 |
                 |-----------------|                  v                 v
```

* To load the kernel image, 8MiB (the amount of memory addressed by two PGDs) is
  reserved at `PAGE_OFFSET`. The kernel image is placed in this reserved space
  during kernel page table initialisation as discussed in 3.6.1.

* Somewhere shortly after the image, the [mem_map][mem_map] for UMA
  architectures (as discussed in chapter 2) is stored. The location is usually at
  the 16MiB mark to avoid using `ZONE_DMA`, but not always.

* For NUMA architectures, portions of the virtual `mem_map` will be scattered
  throughout this region and where they are actually located is architecture
  dependent.

* The region between `PAGE_OFFSET` and `VMALLOC_START - VMALLOC_OFFSET`, is the
  'physical memory map' and the size of the region depends on the amount of
  available physical RAM.

* Between the physical memory map and the vmalloc address space there is a gap
  `VMALLOC_OFFSET` in size (8MiB on i386) used to guard against out-of-bound
  errors.

* As an example, an i386 system with 32MiB of RAM with have `VMALLOC_START`
  located at `PAGE_OFFSET + 0x02000000 + 0x00800000` (i.e. `PAGE_OFFSET` +
  32MiB + 8MiB.)

* In low-memory systems, the remaining amount of the virtual address space,
  minus a 2 page gap, is used by [vmalloc()][vmalloc] for representing
  non-contiguous memory allocations in a contiguous virtual address space.

* In high-memory systems, the `vmalloc` area extends as far as `PKMAP_BASE`
  minus the two-page gap, and two extra regions are introduced - kmap and fixed
  virtual address mappings.

* The `kmap` region, which begins at [PKMAP_BASE][PKMAP_BASE], is reserved for
  the mapping of high memory pages into low memory via [kmap()][kmap] (and
  subsequently [__kmap()][__kmap].) We'll go into this in more detail in chapter
  9.

* The fixed virtual address mapping region, which begins at
  [FIXADDR_START][FIXADDR_START] and ends at [FIXADDR_TOP][FIXADDR_TOP], is used
  by subsystems that need to know the virtual address of a mapping at compile
  time, e.g. [APIC][apic] mappings.

* On i386, `FIXADDR_TOP` is statically defined to be `0xffffe000`, which is one
  page prior to the end of the virtual address space. The size of this region is
  calculated at compile time via [__FIXADDR_SIZE][__FIXADDR_SIZE] and used to
  index back from `FIXADDR_TOP` to give the start of the region,
  `FIXADDR_START`.

* The region required for [vmalloc()][vmalloc], [kmap()][kmap] and the fixed
virtual address mappings is what limits the size of `ZONE_NORMAL`.

* As the running kernel requires these functions, a region of at least
  [VMALLOC_RESERVE][VMALLOC_RESERVE] (which is aliased to
  [__VMALLOC_RESERVE][__VMALLOC_RESERVE]) is reserved at the top of the address
  space.

* `VMALLOC_RESERVE` is architecture-dependent, but on i386 it's defined as
  128MiB. This explains why `ZONE_NORMAL` is generally considered to be only
  896MiB in size - it's the 1GiB of the upper portion of the linear address
  space minus the minimum 128MiB that is reserved for the vmalloc region.

#### 4.2 Managing the Address Space

* The address space that is usable by a process is managed by a high level
  [mm_struct][mm_struct].

* Each address space consists of a a number of page-aligned regions of memory
  that are in use.

* They never overlap, and represent a set of addresses which contain pages that
  are related to each other in protection and purpose.

* The regions are represented by a [struct vm_area_struct][vm_area_struct]:

```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
        struct mm_struct * vm_mm;       /* The address space we belong to. */
        unsigned long vm_start;         /* Our start address within vm_mm. */
        unsigned long vm_end;           /* The first byte after our end address
                                           within vm_mm. */

        /* linked list of VM areas per task, sorted by address */
        struct vm_area_struct *vm_next;

        pgprot_t vm_page_prot;          /* Access permissions of this VMA. */
        unsigned long vm_flags;         /* Flags, listed below. */

        rb_node_t vm_rb;

        /*
         * For areas with an address space and backing store,
         * one of the address_space->i_mmap{,shared} lists,
         * for shm areas, the list of attaches, otherwise unused.
         */
        struct vm_area_struct *vm_next_share;
        struct vm_area_struct **vm_pprev_share;

        /* Function pointers to deal with this struct. */
        struct vm_operations_struct * vm_ops;

        /* Information about our backing store: */
        unsigned long vm_pgoff;         /* Offset (within vm_file) in PAGE_SIZE
                                           units, *not* PAGE_CACHE_SIZE */
        struct file * vm_file;          /* File we map to (can be NULL). */
        unsigned long vm_raend;         /* XXX: put full readahead info here. */
        void * vm_private_data;         /* was vm_pte (shared mem) */
};
```

* A region might represent the process help for use with `malloc()`, a memory
  mapped file such as a shared library or some `mmap()`-ed memory.

* The pages for the region might be active and resident, paged out or even yet
  to be allocated.

* If a region is backed by a file its `vm_file` field will be set. By traversing
  `vm_file->f_dentry->d_inode->i_mapping`, the associated `address_space` for
  the region may be obtained. The `address_space` has all the
  filesystem-specific information needed to perform page-based operations on
  disk.

* The relationship between different address space-related structures
represented diagrammatically:

```
     |------------------|      |------------------|      |------------------|
- -> | struct mm_struct | ---> | struct mm_struct | ---> | struct mm_struct | - ->
     |------------------|      |------------------|      |------------------|
                                  mmap | ^
                                       | |
                 /---------------------/ \---------------------\
                 v                                             | vm_mm
     |-----------------------|       vm_next        |-----------------------| vm_next
     | struct vm_area_struct | -------------------> | struct vm_area_struct | - ->
     |-----------------------|                      |-----------------------|
                 ^    | vm_file                                 | vm_ops
                 |    |                                         |
                 |    \-----------------\                       |
                 |                      |                       |
                 |                      v                       |
                 |               |-------------|                |
                 |               | struct file |                |
                 |               |-------------|                |
                 |             f_dentry |                       v
                 |                      |       |-----------------------------|
                 |                      |       | struct vm_operations_struct |
                 |                      |       |-----------------------------|
                 |                      v
                 |              |---------------|
                 |              | struct dentry |
                 |              |---------------|
                 |                  ^      | d_inode
                 |                  |      |
                 |         i_dentry |      v
                 |              |--------------|
                 |              | struct inode |
                 |              |--------------|
                 |                       | i_mapping
                 |                       |
                 |                       v
                 |         |----------------------|
                 |         | struct address_space |
                 |         |----------------------|
                 |     i_mmap |     ^          | a_ops
                 \------------/     |          \-----------\
                                    |                      v
                                    |     |----------------------------------|
                                    |     |  struct address_space_operations |
                                    |     |----------------------------------|
                                    |
                            /-------X--------\- - -
                            |                |
                    mapping |                | mapping
                     |-------------|  |-------------|
                     | struct page |  | struct page |
                     |-------------|  |-------------|
```

* There are a number of system calls that affect address space and regions:

1. [fork()][fork] - Creates a new process with a new address space. All the
   pages are marked [Copy-On-Write (COW)][cow] and are shared between the two
   processes until a page fault occurs. Once a write-fault occurs a copy is made
   of the COW page for the faulting process. This is sometimes referred to as
   'breaking a COW page' or a 'COW break'.

2. [clone()][clone] - Similar to `fork()`, however allows context to be shared
   with its parent if the `CLONE_VM` flag is set - this is how linux implements
   threading.

3. [mmap()][mmap] - Creates a new region within the process linear address
   space.

4. [mremap()][mremap] - Remaps or resizes a region of memory. If the virtual
   address space is not available for mapping, the region may be moved, unless
   forbidden by the caller.

5. [munmap()][munmap] - Destroys part or all of a region. If the region being
   unmapped is in the middle of an existing region, the existing region is split
   into two separate regions.

6. [shmat()][shmat] - Attaches a shared memory segment to a process address
   space.

7. [shmdt()][shmdt] - Removes a shared memory segment from a process address
   space.

8. [execve()][execve] - Loads a new executable file and replaces the existing
   address space.

9. [exit()][exit] - Destroys an address space and all its regions.

### 4.3 Process Address Space Descriptor

* The process address space is described by [struct mm_struct][mm_struct],
  meaning that there is only one for each process and it is shared between
  userspace threads.

* Threads are in fact identified in the task list by finding all
  [struct task_struct][task_struct]s that have pointers to the same `mm_struct`.

* Each `task_struct` contains all the information the kernel needs about a
  process.

* A unique `mm_struct` is not needed for kernel threads because they will never
  page fault or access the userspace portion (LS - really? Surely
  `copy_to/from_user()`?)

* An exception to this is page faulting within the `vmalloc` space - the page
  fault handling code treats this as a special case and updates the current page
  table with information in the master page table from [init_mm][init_mm].

* Since an `mm_struct` is therefore not needed for kernel threads, the
  `task_struct->mm` field for kernel threads is always `NULL`.

* For some tasks, such as the boot idle task, the `mm_struct` is never set up,
  but, for kernel threads, a call to [daemonize()][daemonize] (LS - no longer a
  function in recent kernels) will call [exit_mm()][exit_mm] (an alias to
  [__exit_mm()][__exit_mm]) to decrement the usage counter.

* Because TLB flushes are extremely expensive (though read a message from Linus
  on [TLB fill performance][linus-tlb] for some interesting information on
  Intel's optimisations in this area), esp. for architectures such as PPC, a
  technique called 'lazy TLB' is employed.

* Lazy TLB avoids unnecessary TLB flushes by processes that do not access the
  userspace page tables because the kernel portion of the address space is
  always visible. The call to [switch_mm()][switch_mm] which ordinarily results
  in a TLB flush is avoided by borrowing the `mm_struct` used by the previous
  task and placing it in `task_struct->active_mm`. This technique has made large
  improvements to context switch times.

* When entering a lazy TLB, the function [enter_lazy_tlb()][enter_lazy_tlb] is
  called to ensure that a `mm_struct` is not shared between processors in SMP
  machines - it's a null operation on UP machines.

* During process exit [start_lazy_tlb()][start_lazy_tlb] is used birefly while
  the process is waiting to be reaped by the parent.

* Let's take a look at [struct mm_struct][mm_struct] again:

```c
struct mm_struct {
        struct vm_area_struct * mmap;           /* list of VMAs */
        rb_root_t mm_rb;
        struct vm_area_struct * mmap_cache;     /* last find_vma result */
        pgd_t * pgd;
        atomic_t mm_users;                      /* How many users with user space? */
        atomic_t mm_count;                      /* How many references to "struct mm_struct" (users count as 1) */
        int map_count;                          /* number of VMAs */
        struct rw_semaphore mmap_sem;
        spinlock_t page_table_lock;             /* Protects task page tables and mm->rss */

        struct list_head mmlist;                /* List of all active mm's.  These are globally strung
                                                 * together off init_mm.mmlist, and are protected
                                                 * by mmlist_lock
                                                 */

        unsigned long start_code, end_code, start_data, end_data;
        unsigned long start_brk, brk, start_stack;
        unsigned long arg_start, arg_end, env_start, env_end;
        unsigned long rss, total_vm, locked_vm;
        unsigned long def_flags;
        unsigned long cpu_vm_mask;
        unsigned long swap_address;

        unsigned dumpable:1;

        /* Architecture-specific MM context */
        mm_context_t context;
};
```

* Looking at each field:

1. `mmap` - The head of a linked list of all VMA regions in the address space.

2. `mm_rb` - The VMAs are arranged in a linked list and in a
   [red-black tree][rb-tree] for fast lookups - this is the head of the tree.

3. `mmap_cache` - The VMA found during the last call to [find_vma()][find_vma]
   is stored in this field, on the (usually fairly safe) assumption that the
   area will be used again soon.

4. `pgd` - The PGD for this process.

5. `mm_users` - Reference count of processes accessing the userspace portion of
   this `mm_struct` such as page tables and file mappings. Threads and the
   [swap_out()][swap_out] code, for example, will increment this count and make
   sure an `mm_struct` is not destroyed early. When it drops to 0,
   [exit_mmap()][exit_mmap] will delete all (userspace) mappings and tear down
   the page tables before decrementing `mm_count`.

6. `mm_count` - Reference count of the 'anonymous users' for the `mm_struct`,
   initialised at 1 for the real user. An anonymous user is one that does not
   necessarily care about the userspace portion and is just 'borrowing' the
   `mm_struct`, for example kernel threads that use lazy TLB switching. When
   this count drops to 0, the `mm_struct` can be safely destroyed. We need this
   reference count as well as `mm_users` because anonymous users need
   `mm_struct` to exist even if the userspace mappings get destroyed, and there
   is no point in delaying the teardown of the page tables.

7. `map_count` - Number of VMAs in use.

8. `mmap_sem` - A long-lived lock that protects the VMA list for readers and
   writers. Since users of this lock need it for a long time and may sleep, a
   spinlock is inappropriate. A reader of the list takes this semaphore with
   [down_read()][down_read], and a writer with [down_write()][down_write] before
   taking the `page_table_lock` spinlock when the VMA linked lists are being
   updated.

9. `page_table_lock` - Protects most fields on the `mm_struct`. As well as the
   page tables, it protects the Resident Set Size (RSS) count, `rss`, and the
   VMA from modification.

10. `mmlist` - [struct list_head][list_head] entry for `mm_struct`s
    [init_mm][init_mm]`.mmlist` which is protected by
    [mmlist_lock][mmlist_lock].

11. `start_code`, `end_code` - [Start, end) address for code section.

12. `start_data`, `end_data` - [Start, end) address for data section.

13. `start_brk`, `brk` - [Start, end) address of the heap.

14. `start_stack` - Surprisingly the address at the start of the stack region.

15. `arg_start`, `arg_end` - [Start, end) address for command-line arguments.

16. `env_start`, `env_end` - [Start, end] address for environment variables.

17. `rss` - Resident Set Size (RSS) is the number of resident pages for this
    process. The global zero page is not account for by RSS.

18. `total_vm` - The total memory space occupied by all VMA regions in this
    process.

19. `locked_vm` - The number of resident pages locked in memory.

20. `def_flags` - This has only one possible value if set - `VM_LOCKED` - and is
    used to determine all future mappings are locked by default.

21. `cpu_vm_mask` - A bitmask represents all possible CPUs in an SMP
    system. It's used by an [Inter-Processor Interrupt (IPI)][ipi] to determine
    if a processor should execute a particular function or not. This matters
    during TLB flush for each CPU.

22. `swap_address` - Used by the pageout daemon to record the last address that
    was swapped from when swapping out entire processes.

23. `dumpable` - Set via the [prctl()][prctl] userland function/system call -
    it's only meaningful when a process is being traced.

24. `context` - Architecture-specific MMU context.

* There are a number of functions which interact with `mm_struct`s:

1. [mm_init()][mm_init] - Initialises an `mm_struct` by setting starting values
   for each field, allocating a PGD, initialising spinlocks, etc.

2. [allocate_mm()][allocate_mm] - Allocates an `mm_struct` from the slab
   allocator (see chapter 8 for more on this.)

3. [mm_alloc()][mm_alloc] - Allocates an `mm_struct` using `allocate_mm()` and
   calls `mm_init()` to initialise it.

4. [exit_mmap()][exit_mmap] - Walks through an `mm_struct` and unmaps all VMAs
   associated with it.

5. [copy_mm()][copy_mm] - Makes an exact copy of the current tasks `mm_struct`
   needs for a new task. This is only used during `fork`.

6. [free_mm()][free_mm] - Returns the `mm_struct` to the slab allocator.

#### 4.3.1 Allocating a Descriptor

* Two functions are available to allocate an `mm_struct` -
  [allocate_mm()][allocate_mm] which simply a pre-processor macro which
  allocates an `mm_Struct` from the slab allocator, whereas
  [mm_alloc()][mm_alloc] calls `allocate_mm()` and initialises it via
  [mm_init()][mm_init].

#### 4.3.2 Initialising a Descriptor

* The first `mm_struct` in the system that is initialised is called
  [init_mm][init_mm]. All subsequent `mm_struct`s are copies of a parent
  `mm_struct`, meaning that `init_mm` has to be statically initialised at
  compile time.

* The static initialisation is performed by the macro [INIT_MM()][INIT_MM]:

```c
#define INIT_MM(name) \
{                                                       \
        mm_rb:          RB_ROOT,                        \
        pgd:            swapper_pg_dir,                 \
        mm_users:       ATOMIC_INIT(2),                 \
        mm_count:       ATOMIC_INIT(1),                 \
        mmap_sem:       __RWSEM_INITIALIZER(name.mmap_sem), \
        page_table_lock: SPIN_LOCK_UNLOCKED,            \
        mmlist:         LIST_HEAD_INIT(name.mmlist),    \
}
```

* After it is established, new `mm_struct`s are created using their parent
  `mm_struct` as a template via [copy_mm()][copy_mm] which uses
  [init_mm()][init_mm] to initialise process-specific fields.

#### 4.3.3 Destroying a Descriptor

* When a new user increments the `mm_struct`'s usage count (i.e. `mm_users`),
  this is done with a simple atomic increment
  e.g. `atomic_inc(&mm->mm_users)`.

* However, when this field is decremented, this is done via [mmput()][mmput] so
  that if this field reaches zero, the appropriate data can be properly teared
  down.

* When `mm_users` reaches 0, the mapped regions are destroyed via
  [exit_mmap()][exit_mmap], then the page tables are destroyed because there are
  no longer any users of the userspace portions. The `mm_count` is decremented
  via [mmdrop()][mmdrop], because all users of the page tables and VMAs are
  counted as a single `mm_struct` user.

* As discussed above, when `mm_count` reaches 0, the `mm_struct will be
  destroyed.

### 4.4 Memory Regions

* The full address space of a process is rarely used, only sparse regions are.

* As discussed previously, each region is represented by a
  [struct vm_area_struct][vm_area_struct] which never overlaps and represents a
  set of addresses with the same protection and purpose, for example a read-only
  shared library loaded into the address space, or the process heap.

* A full list of mapped regions that a process has may be viewed via
  `/proc/<PID>/maps`.

* A region might have a number of different structures associated with at as
  shown in the diagram from earlier in the chapter, with the `vm_area_struct`
  containing all the information associated with a region.

* If a region is backed by a file, the [struct file][file] is available via
  `vm_file` which subsequently has a pointer to the file's [struct inode][inode]
  which can be used to access [struct address_space][address_space]:

```c
struct address_space {
        struct list_head        clean_pages;    /* list of clean pages */
        struct list_head        dirty_pages;    /* list of dirty pages */
        struct list_head        locked_pages;   /* list of locked pages */
        unsigned long           nrpages;        /* number of total pages */
        struct address_space_operations *a_ops; /* methods */
        struct inode            *host;          /* owner: inode, block_device */
        struct vm_area_struct   *i_mmap;        /* list of private mappings */
        struct vm_area_struct   *i_mmap_shared; /* list of shared mappings */
        spinlock_t              i_shared_lock;  /* and spinlock protecting it */
        int                     gfp_mask;       /* how to allocate the pages */
};
```

* This structure contains all the private information about the file, including
  filesystem functions that perform filesystem-specific operations such as
  reading and writing pages to disk (i.e. `a_ops`.)

* Taking another look at [struct vm_area_struct][vm_area_struct]:

```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
        struct mm_struct * vm_mm;       /* The address space we belong to. */
        unsigned long vm_start;         /* Our start address within vm_mm. */
        unsigned long vm_end;           /* The first byte after our end address
                                           within vm_mm. */

        /* linked list of VM areas per task, sorted by address */
        struct vm_area_struct *vm_next;

        pgprot_t vm_page_prot;          /* Access permissions of this VMA. */
        unsigned long vm_flags;         /* Flags, listed below. */

        rb_node_t vm_rb;

        /*
         * For areas with an address space and backing store,
         * one of the address_space->i_mmap{,shared} lists,
         * for shm areas, the list of attaches, otherwise unused.
         */
        struct vm_area_struct *vm_next_share;
        struct vm_area_struct **vm_pprev_share;

        /* Function pointers to deal with this struct. */
        struct vm_operations_struct * vm_ops;

        /* Information about our backing store: */
        unsigned long vm_pgoff;         /* Offset (within vm_file) in PAGE_SIZE
                                           units, *not* PAGE_CACHE_SIZE */
        struct file * vm_file;          /* File we map to (can be NULL). */
        unsigned long vm_raend;         /* XXX: put full readahead info here. */
        void * vm_private_data;         /* was vm_pte (shared mem) */
};
```

* Looking at each field:

1. `vm_mm` - The [struct mm_struct][mm_struct] this VMA belongs to.

2. `vm_start`, `vm_end` - The [start, end) addresses for this region :].

3. `vm_next` - The next `vm_area_struct` in the region - all VMAs in an address
   space are linked together in an address-ordered singly linked list.
   Interestingly, this is one of a _very_ few cases where a singly-linked list
   is used in the kernel.

4. `vm_page_prot` - Protection flags that are set for each PTE in this VMA,
   described below.

5. `vm_flags` - Protections/properties of the VMA itself, described below.

6. `vm_rb` - As well as being stored in a linked list, all VMAs are stored on a
   red-black tree for fast lookups. This is important for page fault handling
   where finding the correct region quickly is important, especially for a large
   number of mapped regions.

7. `vm_next_share`, `vm_pprev_share` - Links together shared VMA regions based
   on file mappings (e.g. shared libraries.)

8. `vm_ops` - Of type [struct vm_operations_struct][vm_operations_struct]
   containing function pointers for `open()`, `close()` and `nopage()` - used
   for syncing information with the disk.

9. `vm_pgoff` - The page-aligned offset within the file that is memory-mapped.

10. `vm_file` - The [struct file][file] pointer to the file being mapped.

11. `vm_raend` - The end address of a 'read-ahead window'. When a fault occurs,
    a number of additional pages after the desired page will be paged in. This
    field determines how many additional pages are faulted in.

12. `vm_private_data` - Used by some drivers to store private information -
    unused by the memory manager.

* The available PTE protection flags (as used in `vm_page_prot`) are as follows
  (as discussed in 3.2):

1. [_PAGE_PRESENT][_PAGE_PRESENT] - The page is resident in memory and not
   swapped out.
2. [_PAGE_RW][_PAGE_RW] - Page can be written to.
3. [_PAGE_USER][_PAGE_USER] - Page is accessible from user space.
4. [_PAGE_ACCESSED][_PAGE_ACCESSED] - Page has been accessed.
5. [_PAGE_DIRTY][_PAGE_DIRTY] - Page has been written to.
6. [_PAGE_PROTNONE][_PAGE_PROTNONE] - The page is resident, but not accessible.

* The available VMA protection/property flags as set in `vm_flags` are as
  follows:

* Protection flags:

1. `VM_READ` - Pages may be read.
2. `VM_WRITE` - Pages may be written.
3. `VM_EXEC` - Pages may be executed.
4. `VM_SHARED` - Pages may be shared.
5. `VM_DONTCOPY` - VMA will not be copied on fork.
6. `VM_DONTEXPAND` - Prevents a region from being resized. Unused.

* `mmap`-related flags:

1. `VM_MAYREAD` - Allows the `VM_READ` flag to be set.
2. `VM_MAYWRITE` - Allows the `VM_WRITE` flag to be set.
3. `VM_MAYEXEC` - Allows the `VM_EXEC` flag to be set.
4. `VM_MAYSHARE` - Allows the `VM_SHARE` flag to be set.
5. `VM_GROWSDOWN` - Shared segment (probably stack) may grow _down_.
6. `VM_GROWSUP` - Shared segment (probably heap) may grow _up_.
7. `VM_SHM` - Pages are used by shared [SHM][shm] memory segment.
8. `VM_DENYWRITE` - Unused. What `MAP_DENYWRITE` for [mmap()][mmap] translates
   to (linux ignores it.)
9. `VM_EXECUTABLE` - Unused. What `MAP_EXECUTABLE` for [mmap()][mmap] translate
   to (linux ignores it.)
10. `VM_STACK_FLAGS` - Flags used by [setup_arg_pages()][setup_arg_pages] to set
    up the stack.

* Locking flags:

1. `VM_LOCKED` - If set, the pages will not be swapped out. Set by the userland
   function (hence system call) [mlock()][mlock].
2. `VM_IO` - Signals that the area is an `mmap`-ed region for I/O to a device -
   also prevents the region from being core dumped.
3. `VM_RESERVED` - Do not swap out this region, it is being used by device drivers.

* `madvise()` flags, set via the userland function (hence system call)
  [madvise()][madvise]:

1. `VM_SEQ_READ` - A hint that pages will be accessed sequentially.
2. `VM_RAND_READ` - A hint that read-ahead in the region is not useful.

* All the VMA regions are linked together in a linked list via `vm_next`. When
  searching for a free area, it's simple to traverse the list.

* However, when searching for a page in particular (for example during page
  faulting), using the red-black tree is more efficient is it provides average
  O(lg N) search time vs. the O(N) of a linear search. The tree is ordered such
  that lower address than the current node are on the left leaf, and higher
  addresses are on the right.

#### 4.4.1 Memory Region Operations

* There are three operations which a VMA may support - `open()`, `close()` and
  `nopage()`, provided via a [struct vm_operations_struct][vm_operations_struct]
  at `vma->vm_ops`:

```c
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs.
 */
struct vm_operations_struct {
        void (*open)(struct vm_area_struct * area);
        void (*close)(struct vm_area_struct * area);
        struct page * (*nopage)(struct vm_area_struct * area, unsigned long address, int unused);
};
```

* The `open()` and `close()` operations are called every time a region is
  created or deleted. Only a small number of devices care about these, in 2.4.22
  only one filesystem and the System V shared region functionality use them.

* The main callback of interest is the `nopage()` function. It is used during a
  page-fault by [do_no_page()][do_no_page]. It is responsible for locating the
  page in the page cache, or allocating a page and populating it with the
  required data before returning it.

* Most files that are mapped will use a generic `vm_operations_struct` called
  [generic_file_vm_ops][generic_file_vm_ops]:

```c
static struct vm_operations_struct generic_file_vm_ops = {
        nopage:         filemap_nopage,
};
```

* This provides a `nopage` entry of [filemap_nopage()][filemap_nopage] which
  either locates the page in the page cache, or reads the information from disk.

#### 4.4.2 File/Device-Backed Memory Regions

* In the event that a region is backed by a file, the `vm_file` field contains a
  [struct address_space][address_space] containing information of relevance to
  the filesystem such as the number of dirty pages that need to be flushed to
  disk.

* Let's review the `address_space` struct once again:

```c
struct address_space {
        struct list_head        clean_pages;    /* list of clean pages */
        struct list_head        dirty_pages;    /* list of dirty pages */
        struct list_head        locked_pages;   /* list of locked pages */
        unsigned long           nrpages;        /* number of total pages */
        struct address_space_operations *a_ops; /* methods */
        struct inode            *host;          /* owner: inode, block_device */
        struct vm_area_struct   *i_mmap;        /* list of private mappings */
        struct vm_area_struct   *i_mmap_shared; /* list of shared mappings */
        spinlock_t              i_shared_lock;  /* and spinlock protecting it */
        int                     gfp_mask;       /* how to allocate the pages */
};
```

* Looking at each field:

1. `clean_pages` - A list of clean pages that require no synchronisation with
   backing storage.
2. `dirty_pages` - A list of dirty pages that do require synchronisation with
   backing storage.
3. `locked_pages` - A list of pages that are locked in memory.
4. `nrpages` - Number of resident pages in use by the address space.
5. `a_ops` - A [struct address_space_operations][address_space_operations] for
   manipulating the filesystem. Each filesystem uses its own, though some may
   use generic functions.
6. `host` - The host inode the file belongs to.
7. `i_mmap` - A list of private mappings using this `address_space`.
8. `i_mmap_shared` - A list of VMAs that share mappings in this `address_space`.
9. `i_shared_lock` - A spinlock used to protect this structure.
10. `gfp_mask` - The mask to use when calling [__alloc_pages()][__alloc_pages]
    for new pages.

* Periodically, the memory manager will need to flush information to disk which
  is performed via
  [struct address_space_operations][address_space_operations]. Let's look at
  this data structure:

```c
struct address_space_operations {
        int (*writepage)(struct page *);
        int (*readpage)(struct file *, struct page *);
        int (*sync_page)(struct page *);
        /*
         * ext3 requires that a successful prepare_write() call be followed
         * by a commit_write() call - they must be balanced
         */
        int (*prepare_write)(struct file *, struct page *, unsigned, unsigned);
        int (*commit_write)(struct file *, struct page *, unsigned, unsigned);
        /* Unfortunately this kludge is needed for FIBMAP. Don't use it */
        int (*bmap)(struct address_space *, long);
        int (*flushpage) (struct page *, unsigned long);
        int (*releasepage) (struct page *, int);
#define KERNEL_HAS_O_DIRECT /* this is for modules out of the kernel */
        int (*direct_IO)(int, struct inode *, struct kiobuf *, unsigned long, int);
#define KERNEL_HAS_DIRECT_FILEIO /* Unfortunate kludge due to lack of foresight */
        int (*direct_fileIO)(int, struct file *, struct kiobuf *, unsigned long, int);
        void (*removepage)(struct page *); /* called when page gets removed from the inode */
};
```

* Looking at each field:

1. `writepage` - Writes a page to disk. The offset within the file to write is
   stored within [struct page][page]. It is up to the filesystem-specific code
   to find the block. See [block_write_full_page()][block_write_full_page] for
   more details.
2. `readpage` - Reads a page from disk. See
   [block_read_full_page()][block_read_full_page] for more details.
3. `sync_page` - Synchronises a dirty page with disk. See
   [block_sync_page()][block_sync_page] for more details.
4. `prepare_write` - This is called before data is copied from userspace into a
   page that will be written to disk. With a journaled filesystem, this ensures
   the filesystem log is up to date. With an unjournaled filesystem, it makes
   sure the pages are allocated. See
   [block_prepare_write()][block_prepare_write] (and subsequently
   [__block_prepare_write()][__block_prepare_write]) for more details.
5. `commit_write` - After the data has been copied from userspace, this function
   is called to commit the information to disk. See
   [block_commit_write()][block_commit_write] (and subsequently
   [__block_commit_write()][__block_commit_write]) for more details.
6. `bmap` - Maps a block so that raw I/O can be performed. It's mostly useful to
   filesystem-specific code, although it is also used when swapping out pages
   that are backed by a swap file instead of a swap partition.
7. `flushpage` - Makes sure there is no I/O pending on a page before releasing
   it. See [discard_bh_page()][discard_bh_page] for more details.
8. `releasepage` - Tries to flush all the buffers associated with a page before
   freeing the page itself. See [try_to_free_buffers()][try_to_free_buffers] for
   more details.
9. `direct_IO` - This function is used when performing direct I/O to an
   inode. The `#define` here is used so external modules can determine whether
   the function is available at compile time as it was only introduced in
   2.4.21.
10. `direct_fileIO` - Used to perform direct I/O with a
    [struct file][file]. Again the `#define#` is present for the same reason as
    above.
11. `removepage` - An optional callback that is used when a page is removed from
    the page cache in
    [remove_page_from_inode_queue()][remove_page_from_inode_queue].

#### 4.4.3 Creating a Memory Region

* The system call [mmap()][mmap] is provided for creating new memory regions
  within a process. For i386, the function calls [sys_mmap2()][sys_mmap2] which
  calls [do_mmap2()][do_mmap2] directly, with the same parameters.

* `do_mmap2()` is responsible for determining the parameters needed by
  [do_mmap_pgoff()][do_mmap_pgoff] which is the principal function for creating
  new areas for _all_ architectures.

* `do_mmap2()` first clears the `MAP_DENYWRITE` and `MAP_EXECUTABLE` bits from
  the `flags` parameter because they are ignored by linux.

* If a file is being mapped, `do_mmap2()` will look up its [struct file][file]
  based on the file descriptor passed a parameter, then acquire the
  `mm_struct->mmap_sem` semaphore before calling
  [do_mmap_pgoff()][do_mmap_pgoff].

* `do_mmap_pgoff()` starts by doing some sanity checks:

1. Ensuring that the appropriate filesystem or device functions are available if
  a file or device is being mapped.

2. Ensuring that the size of the mapping is page-aligned and doesn't attempt to
  create a mapping in the kernel portion of the address space.

3. Ensuring that the size of mapping does not overflow the range of `pgoff`.

4. Ensuring that the process does not have too many mapped regions already.

* The rest of the function is rather large and involved, but can
  broadly-speaking be described by the following steps:

1. Sanity check the parameters.

2. Find a free linear address space large enough for the memory mapping. If a
   filesystem or device-specific `get_unmapped_area()` function is provided in
   the file's `f_op` field (of type [struct file_operations][file_operations])
   it is used, otherwise [arch_get_unmapped_area()][arch_get_unmapped_area] is
   called.

3. Calculate the VM flags and check them against the file access permissions.

4. If an old area exists where the mapping is to take place, fix it up so it is
   suitable for for the new mapping (i.e.

5. Allocate a [struct vm_area_struct][vm_area_struct] from the slab allocator and
   fill in its entries.

6. Link in the new VMA.

7. Call the filesystem or device-specific `mmap` function.

8. Update statistics and exit.

#### (LS) Memory Region API

__NOTE:__ This is present in the book but not relevant to the any of the
sections below which reference these functions, so for clarity I'm putting this
into a separate section.

* [find_vma()][find_vma] - Finds the VMA that _covers_ a given
  address. Importantly, if the region does not exist, it returns the VMA
  _closest_ to the requested address.

* [find_vma_prev()][find_vma_prev] - The same as `find_vma()` except it also
  provides the VMA pointing to the returned VMA. It is rarely used as typically
  red-black tree nodes will be required as well so `find_vma_prepare()` is used
  instead ([sys_mprotect()][sys_mprotect] is a notable exception.)

* [find_vma_prepare()][find_vma_prepare] - The same as `find_vma_prev()` except
  it will also provide red-black tree nodes needed to perform an insertion into
  the tree.

* [find_vma_intersection()][find_vma_intersection] - Returns the VMA that
  intersects the specified address _range_. This is useful for checking whether
  a linear address region is in use by _any_ VMA.

* [vma_merge()][vma_merge] - Attempts to expand the specified VMA to cover a new
  address range. If the VMA can't be expanded forwards, then the next VMA is
  checked to see if it might be expanded backwards to cover the address range
  instead. Regions may be merged only if there is no file/device mapping, and
  permissions match.

* [get_unmapped_area()][get_unmapped_area] - Returns the address of a free
  region of memory large enough to cover the requested size of memory. Used
  principally when a new VMA is to be created.

* [insert_vm_struct()][insert_vm_struct] - Inserts a new VMA into a linear
  address space.

#### 4.4.4 Finding a Mapped Memory Region

* A common operation is to determine the VMA that a particular address belongs
  to, such as during operations like page faulting. This is handled via
  [find_vma()][find_vma].

* `find_vma()` first checks the `mmap_cache` field which caches the result of
  the last call to `find_vma()` as it is likely the same region will be needed a
  few times in succession.

* If `mmap_cache` doesn't match, the red-black tree stored in the `mm_rb` field
  is traversed.

* If the desired address is not contained within any VMA, the function will
  return the VMA _closest_ to the requested address. This is somewhat unexpected
  and confusing, so it's important callers double-check to ensure the returned
  VMA contains the desired address (or take the appropriate action if not.)

* [find_vma_prev()][find_vma_prev] is the same is `find_vma()` but also returns
  a pointer to the VMA prior to the located one (they are linked together using
  a singly-linked-list so this makes sense.) It is very similar to
  [find_vma_prepare()][find_vma_prepare] which additionally provides red-black
  tree nodes needed for inserting into the tree.

* These previous-VMA functions are rarely used but a notable use case is where
  two VMAs are being compared to determine if they may be merged. An additional
  use case is where a memory region is being removed and the singly-linked-list
  needs updating.

* The last function of note for searching VMAs is
  [find_vma_intersection()][find_vma_intersection] which is used to find a VMA
  that overlaps a given address range. The most notable use case for this is
  during a call to [sys_brk()][sys_brk] when a region is growing upwards - it's
  important to ensure the growing region will not overlap an old region.

#### 4.4.5 Finding a Free Memory Region

* When a new area is to be memory mapped, a free region has to be found that is
  large enough to contain the new mapping. The function responsible for finding
  a free area is [get_unmapped_area()][get_unmapped_area].

* If a device is being mapped, such as a video card, the associated
  `get_unmapped_area()` function is called (via the
  [struct file_operations][file_operations] `f_op` field.) This is because
  devices or files may have additional requirements for mapping above and beyond
  the generic code, e.g. alignment requirements.

* If there are no special requirements, the architecture-specific function
  [arch_get_unmapped_area()][arch_get_unmapped_area] is called. There is a
  generic version of this available for architectures which don't need to do
  anything unusual here (i386 included.)

#### 4.4.6 Inserting a Memory Region

* (In theory) the principal function for inserting a new memory region is
  [insert_vm_struct()][insert_vm_struct], which subsequently calls
  [__vma_link()][__vma_link] to link in the new VMA.

* However, this function is rarely called directly because it does not increase
  the `map_count` field, rather [__insert_vm_struct()][__insert_vm_struct] is
  used which performs the same tasks but also increments this field. LS: the
  code is literally duplicated apart from `mm->map_count++` :)

* Two varieties of linking functions are provided - [vma_link()][vma_link] and
  [__vma_link()][__vma_link].

* `vma_link()` is intended for use when no locks are held. It'll acquire all the
  necessary locks, including locking the file if the VMA is a file mapping
  before calling `__vma_link()` which assumes the appropriate locks are held.

* Many functions do not use the [insert_vm_struct()][insert_vm_struct] functions
  and instead prefer to call [find_vma_prepare()][find_vma_prepare] themselves
  followed by a later [vma_link()][vma_link] to avoid having to traverse the
  tree multiple times.

* The linking in [__vma_link()][__vma_link] consists of three stages that are
  contained in 3 separate functions:

1. [__vma_link_list()][__vma_link_list] inserts the VMA into the linear,
   singly-linked list. If it is the first mapping in the address space, it will
   become the red-black tree root node.

2. The red-black node is then linked into the tree via
   [__vma_link_rb()][__vma_link_rb].

3. Finally, the file share mappings are fixed up via
   [__vma_link_file()][__vma_link_file] which inserts the VMA into the linked
   list of VMAs using the `vm_pprev_share` and `vm_next_share` fields.

#### 4.4.7 Merging Contiguous Regions

* Prior to 2.4.22 linux used to have a `merge_segments()` function that merged
  adjacent regions of memory if the file and permissions matched. However, this
  ended up being overly expensive (esp. in [sys_mprotect()][sys_mprotect]) so
  was removed.

* The equivalent function in 2.4.22 is [vma_merge()][vma_merge] and is only used
  in two places.

* The first is [do_mmap_pgoff()][do_mmap_pgoff] (via [sys_mmap2()][sys_mmap2]),
  which calls it if an anonymous region is being mapped, as these are frequently
  mergeable.

* The second is during [do_brk()][do_brk], which is expanding one region into a
  newly allocated one where the two regions should be merged. Rather than
  merging two regions, `vma_merge()` checks if an existing region can be
  expanded to satisfy the new allocation, negating the need to create a new
  region (as discussed previously, a region can be expanded if there are no file
  or device mappings and the permissions of the two areas are the same.)

* Regions are merged elsewhere, although no function is explicitly called to
  perform the merging, for example in [sys_mprotect()][sys_mprotect] during the
  fixup of areas where two regions will be merged if permissions match after a
  permissions change, or during a call to [move_vma()][move_vma] when it is
  likely that similar regions will be located beside each other.

#### 4.4.8 Remapping and Moving a Memory Region

* [mremap()][mremap] is a userland function hence system call that allows an
  existing memory mapping to be grown or shrunk. It is implemented by
  [sys_mremap()][sys_mremap] which may move a memory region if it is growing or
  would overlap another region, and `MREMAP_FIXED` is not specified in the flags.

* If a region is to be moved, [do_mremap()][do_mremap] first calls
  [get_unmapped_area()][get_unmapped_area] to find a region large enough to
  contain the new resized mapping and then calls [move_vma()][move_vma] to move
  the old VMA to the new location.

* `move_vma()` does the following:

1. It first checks if the new location can be merged with the VMAs adjacent to
  the new location. If they cannot be merged a VMA is allocated, literally one
  PTE at a time.

2. [move_page_tables()][move_page_tables] is called, which copies all the page
   table entries from the old mapping to the new one. Though there might be
   better ways of moving the page tables, doing it this way makes error recovery
   trivial as backtracking is relatively straightforward.

3. The contents of the pages are not copied, rather
   [zap_page_range()][zap_page_range] is called to swap out or remove all the
   pages from the old mapping, and the normal page fault handing code will swap
   the pages back in from backing storage/files or will call the device-specific
   `do_nopage()` function.

#### 4.4.9 Locking a Memory Region

* Linux can lock pages from an address range into memory using the userland
  function (hence system call) [mlock()][mlock], which is implemented by
  [sys_mlock()][sys_mlock]. Them being 'locked' means that that they will not be
  swapped out to disk.

* At a high level `mlock` is simple - it creates a VMA for the address range to
  be locked, sets the `VM_LOCKED` flag on it and forces all the pages to be
  present via [make_pages_present()][make_pages_present].

* [mlockall()][mlockall] is another userland function/system call that does the
  same thing as `mlock()` only for every VMA in the calling process. It is
  implemented via [sys_mlockall()][sys_mlockall].

* Both `mlock` and `mlockall` use [do_mlock()][do_mlock] to do the heavy
  lifting - finding the affected VMAs and deciding which function is needed to
  fix up the regions.

* There are some limitations as to what memory can be locked:

1. The address range must be page-aligned, because VMAs are page-aligned. This
   can be addressed simply by rounding the range up to the nearest page-aligned
   range.

2. The process limit `RLIMIT_MLOCK` may not be exceeded. This means that each
   process may only lock half of physical memory at a time. This seems a bit
   silly since processes can fork and continue to lock further pages, however
   you need root permissions to do it so if a system admin is being this stupid
   it's their funeral :)

#### 4.4.10 Unlocking the Region

* The userland functions (hence system calls) [munlock()][munlock] and
  [munlockall()][munlockall] provide the corollary for the locking functions and
  are implemented by [sys_munlock()][sys_munlock] and
  [sys_munlockall()][sys_munlockall] respectively, which both rely on
  [do_mmap()][do_mmap] to fix up memory regions.

* These functions are a lot simpler than the locking ones as they do not have to
make as many checks as them.

#### 4.4.11 Fixing Up Regions After Locking

* When locking or unlocking, VMAs will be affected in one of four ways, each of
  which must be fixed up by [mlock_fixup()][mlock_fixup]:

1. The locking affects the whole of the VMA -
   [mlock_fixup_all()][mlock_fixup_all] is called to fix it up.

2. The locking affects the start of the VMA - this means that a new VMA will
   have to be allocated to map the new area. This is handled via
   [mlock_fixup_start()][mlock_fixup_start].

3. The locking affects the end of the VMA - this is handled via the aptly named
   [mlock_fixup_end()][mlock_fixup_end].

4. The locking affects the middle of the VMA - unsurprisingly, this is handled
   via [mlock_fixup_middle()][mlock_fixup_middle]. This case requires two new
   VMAs to be allocated.

* Interestingly, VMAs created as a result of locking are _never_ merged, even
  when unlocked. This is because it is presumed that processes that lock regions
  will need to lock the same regions over + over again so it's not worth
  constantly merging and splitting them.

#### 4.4.12 Deleting a Memory Region

* The function responsible for deleting memory regions or parts of memory
  regions is [do_munmap()][do_munmap].

* The process for this is relatively simple compared to the other memory
  region-related operations and is divided into 3 parts:

1. Fix up the red-black tree for the region that is about to be unmapped.

2. Release pages and PTEs related to the region to be unmapped.

3. Fix up the regions if a hole has been created.

* To ensure that the red-black tree is ordered correctly, all VMAs that are
  affected by the unmap are placed on a linked list local variable called `free`
  then deleted from the red-black tree via [rb_erase()][rb_erase].

* The regions, if they still exist, will be added with their new addresses later
  during the fixup.

* Next, the linked-list VMAs on `free` are walked to ensure it isn't a _partial_
  unmapping, if it is then this needs to be handled carefully.

* Regardless of whether the region is being partially or fully unmapped,
  [remove_shared_vm_struct()][remove_shared_vm_struct] is called to remove share
  file mappings. If it is partial, this will be recreated during fixup.

* Next, [zap_page_range()][zap_page_range] is called to remove all pages
  associated with the region about to be unmapped.

* [unmap_fixup()][unmap_fixup] is then called to handle partial unmappings.

* Finally, [free_pgtables()][free_pgtables] is called to try to free up all page
  table entries associated with the unmapped region. This isn't an exhaustive
  process - only full PGD directories and their entries are unmapped. This is
  because a finer-grained freeing of page table entries would be too expensive
  for data structures that are both small and likely to be used again.

#### 4.4.13 Deleting All Memory Regions

* When a process exits it's necessary to unmap all VMAs associated with a
  [struct mm_struct][mm_struct]. This is achieved via [exit_mmap()][exit_mmap].

* `exit_mmap()` simply flushes the CPU cache before walking through the linked
  list of VMAs, unmapping each of them in turn and freeing up the associated
  pages before flushing the TLB and deleting the page table entries.

### 4.5 Exception Handling

* A very important part of a VM system is how kernel address space exceptions
  (those that aren't bugs) are caught.

* There are two situations where a bad reference may occur:

1. A process sends an invalid pointer to the kernel via a system call which the
   kernel must be able to safely trap because the only check made initially is
   that the address is below `PAGE_OFFSET`.

2. The kernel uses [copy_from_user()][copy_from_user] or
   [copy_to_user()][copy_to_user] to read/write data from userspace.

* At compile time the linker creates an exception table in the
  [__ex_table][__ex_table] section of the kernel code segment, which starts at
  [__start___ex_table][__start___ex_table] and ends (exclusive bound) at
  [__stop___ex_table][__stop___ex_table].

* Each entry in the exception table is of type
  [struct exception_table_entry][exception_table_entry] which contains a pair of
  addresses - `insn` and `fixup` - execution point and fixup routine respectively.

* When an exception occurs that the page fault handler can't manage, it calls
  [search_exception_table()][search_exception_table] to see if a fixup routine
  has been provided for an error at the faulting instruction. If module support
  is enabled, each module's exception table will also be searched.

* If the address of hte current exception is found in the table, the
  corresponding location of the fixup code is returned and executed. Section 4.7
  will go into more detail as to how this is used to trap bad reads/writes to
  userspace.

### 4.6 Page Faulting

* Pages in the process linear address space are not necessarily resident in
  memory.

* For one, allocations made on behalf of a process are not satisfied
  immediately, because the space is simply reserved within the
  [struct vm_area_struct][vm_area_struct]. Alternatively, a page may have been
  swapped out to backing storage, or a user may be attempting to write to a
  read-only page.

* Linux, like many operating system, has a __Demand Fetch__ policy for dealing
  with pages that are not resident - a page is only fetched from backing storage
  when the hardware raises a page fault exception, which the operating system
  traps and allocates a page.

* (At the time of 2.4.22) the characteristics of backing storage suggest
  prefetching would help, but linux is pretty primitive in this regard.

* When a page is paged in from swap space, a number of pages up to
  `2^page_cluster` ([page_cluster][page_cluster] is a global var set in
  [swap_setup()][swap_setup]) are read in by
  [swapin_readahead()][swapin_readahead] and placed in the swap cache.

* Unfortunately, there is only a chance that pages likely to be used soon would
  be adjacent in the swap area which makes it a poor pre-paging policy, one that
  adapts to program behaviour would work better (LS - I am sure this is done a
  lot better in the modern kernel!)

* There are two types of page fault - major and minor. Major page faults occur
  when data has to be read from disk, which is an expensive operation. Those
  that do not require this are considered minor.

* Major and minor page faults are tracked in the `maj_flt` and `min_flt` fields
  of [struct task_struct][task_struct].

* The page fault handler needs to be able to deal with the following types of
  page fault (written in the form severity - description - resolution):

1. Minor - Region valid but page not allocated - Allocate a page frame from the
   physical page allocator.

2. Minor - Region not valid but is beside an expandable region like the stack -
   Expand the region and allocate a page.

3. Minor - Page swapped out, but present in swap cache - Re-establish the page
   in the process page tables and drop a reference to the swap cache.

4. Major - Page swapped out to backing storage - Find where the page with
   information is stored in the PTE and read it from disk.

5. Minor - Page write when marked read-only - If the page is a COW page make a
   copy of it, mark it writable and assign it to the process, otherwise send a
   `SIGSEGV` signal.

6. Error - Region is invalid or process has no permissions to access - Send a
   `SIGSEGV` signal.

7. Minor - Fault occurred in the kernel portion of address space - If the fault
   occurred in the `vmalloc` area of the address space, the current process page
   tables are updated against the master page table held by
   [init_mm][init_mm]. This is the only _valid_ kernel page fault that may
   occur.

8. Error - Fault occurred in the userspace region while in kernel mode - this
   means kernel code did not copy from userspace properly and caused a page
   fault. This is a kernel bug that is taken quite seriously.

* Each architecture registers an architecture-specific for the handling of page
  faults. Though this function can be named arbitrarily, it's normally (and
  certain on i386) called [do_page_fault()][do_page_fault].

* This function is provided with a lot of information - the address of the
  fault, whether the page was not found or there was a protection error, whether
  it was a read/write fault, whether the fault occurred in user/kernel space and
  more.

* `do_page_fault()` is responsible for determining which type of fault has
  occurred and how it should be handled by the architecture-independent code:

```
                                        ----------------
                                        | Read address |
                                        |   of fault   |
                                        ----------------
                                                |
                                                v
                                   /------------------------\
                                  /  address > TASK_SIZE &&  \---\
                                  \     In Kernel Mode       /   | Yes
                                   \------------------------/    |
                                      | No                       v
                                      v                  ------------------
------------------  No /-------------------\ Yes         | vmalloc_fault: |
|      Find      |<---/  in_interrupt() ||  \----\       ------------------
| vm_area_struct |    \    no mm context    /    |               |
------------------     \-------------------/     |               |
     |                                           |               v
     v                                           |        ---------------
 /-------\ No       /--------------\             |        |   Fix up    |
/  Valid  \------->/    Can grow    \            |        | page tables |
\ region? /        \ nearby region? /            |        ---------------
 \-------/          \--------------/             |               |
  | Yes                Yes |  | No               |               |
  |        /---------------/  |                  |               v
  |        |                  |                  |          /---------\ Yes
  |        v                  |                  |         /    pte    \------\
  |   /---------\             v                  |         \ _present? /      |
  |  / expand    \ No   -------------            |          \---------/       |
  |  \ _stack()? /----->| bad_area: |            |               | No         |
  |   \---------/       -------------            |               |            |
  |      | Yes           ^    |                  |               |            |
  v      v               |    |                  |               |            |
--------------    /------/    v                  v               |            |
| good_area: |    |       /---------\ Yes ---------------        |            |
--------------    |      / In kernel \--->| no_context: |<-------/            |
       |          |      \   space?  /    ---------------                     |
       v          |       \---------/         |                               |
 /-----------\ No |            | No           |                               |
/ Permissions \---/            |              v                               |
\     OK?     /                |   /-----------------\ No  --------------     |
 \-----------/                 |  / Exception handler \--->| Kernel Bug |     |
       | Yes                   |  \      exists?      /    |   oops()   |     |
       V                       |   \-----------------/     --------------     |
-------------------            |              | Yes                           |
| handle_mm_fault |            v              v                               |
-------------------        -----------  ---------------------                 |
         |                 |  Send   |  |        Call       |                 |
         |                 | SIGSEGV |  | Exception Handler |                 |
         |                 -----------  ---------------------                 |
         v                                                                    |
-------------------                                                           |
| Fault completed |<----------------------------------------------------------/
-------------------
```

* [handle_mm_fault()][handle_mm_fault] is an architecture-independent top-level
  functino for faulting in a page from backing storage, performing
  [Copy-On-Write (COW)][cow] amongst other tasks. If it returns 1, the fault was
  minor, if it returns 2 the fault was major, 0 sends a `SIGBUS` error and any
  other value invokes the out-of-memory handler.

#### 4.6.1 Handing a Page Fault

* After the exception handler has decided the fault is a _valid_ page fault in a
  _valid_ memory region, the architecture-independent
  [handle_mm_fault()][handle_mm_fault] function takes over.

* `handle_mm_fault()` allocates the required page table entries if they don't
  already exist and calls [handle_pte_fault()][handle_pte_fault].

* `handle_pte_fault()` checks whether the PTE is present or not via
  [pte_present()][pte_present] and [pte_none()][pte_none/3lvl]. If no PTE has
  been allocated - `pte_none()` returns true - then [do_no_page()][do_no_page]
  which handles __Demand Allocation__.

* If the PTE is present, then a page has been swapped out to disk and
  [do_swap_page()][do_swap_page] handles __Demand Paging__.


* There is a rare exception where swapped out pages belonging to a virtual file
  are handled by [do_no_page()][do_no_page] - page faulting within a virtual
  file. This case is discussed in section 12.4.

* If the PTE is write-protected, [do_wp_page()][do_wp_page] is called, as the
  page is a [COW][cow] page - as discussed previously, this is a page that is
  shared between multiple processes (usually parent and child) until a write
  occurs, after while a private copy is made for the process performing the
  write.

* The kernel is able to recognise a COW page because the VMA for the region is
  marked writable even though the individual PTE is not.

* If the page is not COW, the page is simply marked dirty because it has been
  written to.

* Finally, a page can be read and be marked present and still encounter a
  fault - this happens for some architectures which do not have a 3-level page
  table. In this case, the PTE is simply established and marked young.

#### 4.6.2 Demand Allocation

* When a process accesses a page for the very first time, the page has to be
  allocated and (possibly) filled with data via [do_no_page()][do_no_page].

* Additionally, if the [struct vm_operations_struct][vm_operations_struct]
  associated with the parent VMA (`vm->vm_ops`) provides a `nopage()` function,
  it is called - this matters for example for memory-mapped devices such as
  video cards which need to allocate page and supply data on access, or perhaps
  for a mapped file which must retrieve its data from backing storage.

* Let's consider a couple of cases:

##### Handling Anonymous Pages

* If the `vm->vm_ops` field is not filled or a `nopage()` function is not
  supplied, the function [do_anonymous_page()][do_anonymous_page] is called to
  handle an anonymous access.

* There are two cases to consider:

1. First time read - this is an easy case to deal with, because no data
   exists. In this case the system-wide [empty_zero_page][empty_zero_page],
   which is just a page of zeros, is mapped for the PTE and it is marked
   write-protected. On i386 this page is zeroed out in [mem_init][mem_init].

2. First time write - [alloc_page()][alloc_page] is called to allocate a free
   page (discussed further in chapter 6), which is zero-filled by
   [clear_user_highpage()][clear_user_highpage]. Assuming the page was
   successfully allocated, the `rss` field in the [struct mm_struct][mm_struct]
   will be incremented and `flush_page_to_ram()` is called on some brain-dead
   architectures that aren't i386 :) The page is then inserted on the LRU lists
   so it may be reclaimed later by the page reclaiming code. Finally, the page
   table entries for the process are updated for the new mapping.

##### Handling File/Device-Backed Pages

* If a VMA is backed by a file or device, a `nopage()` function will be provided
  within the VMA's `vm_ops` [struct vm_operations_struct][vm_operations_struct].

* In the case of a file-backed VMA, the function
  [filemap_nopage()][filemap_nopage] is often used for this - providing
  functionality to allocate a page and read a page-sized amount of data from
  disk.

* Pages backed by a virtual file, such as those provided by shmfs for example
  (later renamed to tmpfs), will use the function [shmem_nopage()][shmem_nopage]
  (see chapter 12 for more details on this.)

* Each device driver provides a different `nopage()` - we don't care about the
  internals of each, so long as it returns a `struct page` to use.

* Once the page is returned, checks are made to ensure a page was successfully
  allocated and appropriate errors were returned if not.

* A check is then performed to see if an early COW break should take place - if
  the fault is caused by a write to the page and the ¬VM_SHARED` flag is not set
  in the managing VMA then a break will occur.

* An 'early COW break' is a case of allocating a new page and copying the data
  across before reducing the reference count to the page returned by the
  `nopage()` function.

* In either case, a check is then made using [pte_none()][pte_none/3lvl] to
  ensure that a PTE is not already in the apge table that is about to be
  used. This is because, in SMP, two faults could occur for the same page at
  close to the same time and because the spinlocks are not held for the full
  duration of the fault, the check has to be made at the last instant. If there
  has been no race, the PTE is assigned, stats are updated and the brain-dead
  architecture hooks for cache coherency are called.

#### 4.6.3 Demand Paging

* When a page is swapped out to backing storage, the function
  [do_swap_page()][do_swap_page] is responsible for reading the page back in
  (with the exception of virtual files, which are covered in chapter 12.)

* The information needed to find the page in the swap is stored within the PTE
  itself (LS - but surely it's swapped out?! Maybe there's a shared page it
  references which it indexes in to? Confused). Because pages may be shared
  between multiple processes, they cannot always be swapped out immediately -
  instead they are placed within the 'swap cache' in this situation.

* A shared page cannot be swapped out immediately because (in 2.4.22) there is
  no way of mapping a [struct page][page] to all the PTEs of each process it is
  shared between - searching the page tables of all processes is simply too
  expensive. In more recent kernels (well at least in early 2.6, the
  implementation may have changed since) there is 'Reverse Mapping (RMAP)' which
  provides means of determining this information.

* Given there is a swap cache, it's possible that when a fault occurs the page
  still exists there. In this case, the reference count to the page is simply
  increased and it is placed within the process page tables again, and registers
  as a minor page fault.

* If the page exists only on disk, [swapin_readahead()][swapin_readahead] is
  called which reads the requested page _and_ a number of pages after it. The
  number of pages read is determined by [page_cluster][page_cluster] (on i386
  this is set to 3, or 2 if there is less than 16MiB of RAM on the system.)

* The number of pages read is `2^page_cluster`, unless a bad or empty swap entry
  is encountered. Read-ahead is performed on the assumption that seeking is the
  most expensive operation time-wise, so after it is complete succeeding pages
  should also be read in.

#### 4.6.4 COW Pages

* Once upon a time a fork would mean the entire parent address space would have
  to be duplicated - this is an extremely expensive operation because it could
  result a significant percentage of the process would have to be swapped in
  from backing storage.

* [Copy-on-Write (COW)][cow] is a solution to this issue - during a fork the
  PTEs of both the parent and the child process are made read-only so that when
  a write occurs, there will be a page fault.

* As discussed previously, linux recognises a COW page because even though the
  PTE is write-protected, the controlling VMA shows the region is writable.

* The function [do_wp_page()][do_wp_page] handles this case by making a copy of
  the page and assigning it to the writing process. If necessary, a new swap
  slot will be reserved for the page.

* Using the COW method, only the page table entries have to be copied during a
  fork, rather more efficient overall!

## 4.7 Copying to/from Userspace

* It is not safe to access memory in the process address space directly, because
  there is no way to quickly check if the page addressed is resident or not.

* Linux relies on the MMU to raise exceptions when the address is invalid and
  use the Page Fault Exception handler to catch the exception and clean up.

* When we are in kernel mode we can't do this in the usual way as we don't have
  anybody to clean up our mess for us, as previously discussed kernel page
  faults only make sense when interacting with the `vmalloc` region.

* In the i386 case, checks are performed by [access_ok()][access_ok] before
  [__copy_user()][__copy_user] or [__copy_user_zeroing()][__copy_user_zeroing]
  which use assembly to provide fixup code for cases where the address really is
  fucked - this will be found by
  [search_exception_table()][search_exception_table] on page fault and trapped,
  preventing the general kernel page fault code from running and a potential
  oops, and instead running fixup code which zeros the remaining buffer space.

* Let's take a look at the functions for safely copying data to/from userland:

1. [copy_from_user()][copy_from_user] -Copies bytes from the user address space
   to the kernel address space.
2. [copy_to_user()][copy_to_user] - Copies bytes from the kernel address space
   to the user address space.
3. [copy_user_page()][copy_user_page] - Copies data to an anonymous or COW page
   in userspace. Brain-dead architectures need to be careful about cache
   aliasing. LS - Doesn't seem to do any checks?!
4. [clear_user_page()][clear_user_page] - Similar to `copy_user_page()`, only it
   zeros a page instead. LS - doesn't seem to do any checks?!
5. [get_user()][get_user] - Copies an integer value from the user address space
   to the kernel address space.
6. [put_user()][put_user] - Copies an integer value from the kernel address
   space to the user address space.
7. [strncpy_from_user()][strncpy_from_user] - Copies a null-terminated string of
   maximum size as specified from the user address space to the kernel address
   space.
8. [strlen_user()][strlen_user] - Determines the length (with specified upper
   bound) of the userspace string including the trailing NULL byte.
9. [access_ok()][access_ok] - Returns non-zero if the specified userspace block
   of memory is valid, or 0 if not.

* Generally these functions are implemented as macros and all behave relatively
  similarly.

* Considering [copy_from_user()][copy_from_user] on i386 - if the size of the
  copy is known at compile time, it uses
  [__constant_copy_from_user()][__constant_copy_from_user], otherwise it uses
  [__generic_copy_from_user()][__generic_copy_from_user].

* In the constant copy case, the code eventually calls
  [__constant_copy_user()][__constant_copy_user] which optimises based on the
  bytes of the 'stride' of the data being read.

* In the generic copy case, it eventually calls
  [__copy_user_zeroing()][__copy_user_zeroing]. As discussed above, this sets an
  exception table for [search_exception_table()][search_exception_table] to find
  which allows fixup code to zero the remains of the buffer.

* Doing things this way allows the kernel to safely access userspace without
  performing overly expensive checks first.

## Chapter 5: Boot Memory Allocator

* It's just not practical to statically initialise all the core kernel memory
  structures at compile-time, because there are too many permutations of
  hardware configurations out there.

* To set up even the most basic structures requires memory - even the physical
  page allocator needs to allocate memory to initialise itself. But how?

* To get around this issue, a specialised allocator called the 'boot memory
  allocator' is used.

* It uses the most basic allocator algorithm - [first fit][first-fit]. This
  means the first available space that can store the requested amount of memory
  is used.

* Additionally the allocator uses a bitmap to represent memory rather than
  linked lists of free blocks. If a bit is 1, the page is allocated, if it is 0,
  it is unallocated.

* To satisfy allocations of sizes smaller than a page, the allocator records the
  Page Frame Number (PFN) - the index of the page, and an offset within that
  page, so subsequent small allocations can be merged together and stored on the
  same page.

* The reason this allocator is not just used as the allocator overall is that,
  despite it not suffering too badly from fragmentation, memory often has to be
  linearly searched to satisfy an allocation. Since bitmaps are being examined,
  the search is expensive and this is made worse by the fact that this algorithm
  tends to leave many small free blocks at the beginning of physical memory
  which are scanned for large allocations, which is very wasteful.

* There are two distinct APIs for the allocator - one for UMA architectures, and
  another for NUMA architectures. The principle difference is that the NUMA API
  requires specification of the node affected by the operation.

* Let's examine the API:

1. [init_bootmem()][init_bootmem] - Initialises the memory between 0 and the
   specified PFN `page` - the beginning of usable memory is at the specified PFN
   `start`. [init_bootmem_node()][init_bootmem_node] is the NUMA equivalent.

2. [reserve_bootmem()][reserve_bootmem] - Marks the pages in the specified range
   reserved. Requests to partially reserve a page will result in the full page
   being reserved. [reserve_bootmem_node()][reserve_bootmem_node] is the NUMA
   equivalent.

3. [free_bootmem()][free_bootmem] - Marks the pages in the specified range
   free. [free_bootmem_node()][free_bootmem_node] is the NUMA equivalent.

4. [alloc_bootmem()][alloc_bootmem] - Allocates specified `size` bytes from
   `ZONE_NORMAL`. The allocation will be aligned to the L1 hardware cache for
   efficiency. [alloc_bootmem_node()][alloc_bootmem_node] is the NUMA
   equivalent. [alloc_bootmem_pages_node()][alloc_bootmem_pages_node] is the
   NUMA equivalent.

5. [alloc_bootmem_low()][alloc_bootmem_low] - Allocates specified `size` bytes
   from `ZONE_DMA`. As with `alloc_bootmem()`, the allocation will be
   L1-aligned. [alloc_bootmem_low_pages_node()][alloc_bootmem_low_pages_node] is
   the NUMA equivalent.

6. [alloc_bootmem_pages()][alloc_bootmem_pages] - Allocates specified `size`
   bytes from `ZONE_NORMAL`, aligned on page size so that full pages will be
   returned to the caller.

7. [alloc_bootmem_low_pages()][alloc_bootmem_low_pages] - Same as
   `alloc_bootmem_pages()` only from `ZONE_DMA`.

8. [bootmem_bootmap_pages()][bootmem_bootmap_pages] - Calculates the number of
   pages required to store a bitmap representing the allocation state of the
   specified `pages` number of pages.

9. [free_all_bootmem()][free_all_bootmem] - Used at the end of the useful life
   of the boot allocator - cycles through all pages in the bitmap. For each that
   is free its flags are cleared and the page is freed to the physical page
   allocator (see chapter 6) so the runtime allocator can set up its free
   lists. [free_all_bootmem_node()][free_all_bootmem_node] is the NUMA
   equivalent.

### 5.1 Representing the Boot Map

* [struct bootmem_data][bootmem_data], typedef-d to `bootmem_data_t`, exists for
  each NUMA node of memory in the system. It contains information needed for the
  boot memory allocator to allocate memory for a node - the bitmap representing
  allocated pages and where the memory is located:

```c
/*
 * node_bootmem_map is a map pointer - the bits represent all physical
 * memory pages (including holes) on the node.
 */
typedef struct bootmem_data {
        unsigned long node_boot_start;
        unsigned long node_low_pfn;
        void *node_bootmem_map;
        unsigned long last_offset;
        unsigned long last_pos;
} bootmem_data_t;
```

* Looking at each field:

1. `node_boot_start` - The starting _physical_ address of the represented block.
2. `node_low_pfn` - The end physical address of the block (LS - likely exclusive
   bound), in other words the end of the `ZONE_NORMAL` this node represents.
3. `node_bootmem_map` - The _virtual_ address of the bitmap representing
   allocated or free pages with each bit.
4. `last_offset` - The offset within the page of the end of the last
   allocation. If 0, the page is full.
5. `last_pos` - The PFN of the page used with the last allocation. Combined with
   `last_offset` a test can be made to see if allocations can be merged into the
   last page rather than needing to allocate a whole new page.

### 5.2 Initialising the Boot Memory Allocator

* Each architecture is required to supply a [setup_arch()][setup_arch] function
  which, among other tasks, is responsible for determining the necessary
  parameters to initialise the boot memory allocator.

* Each architecture additionally has its own function to get the necessary
  parameters. On i386, this is achieved via [setup_memory()][setup_memory] (as
  discussed in 2.2.2.)

* `setup_memory()` calculates the following parameters:

1. `min_low_pfn` - The lowest PFN available in the system.

2. `max_low_pfn` - The highest PFN that may be addressed by low memory
   (`ZONE_NORMAL`.)

3. `highstart_pfn` - This is the PFN of the beginning of high memory
   (`ZONE_HIGHMEM`.)

4. `highend_pfn` - This is the last PFN in high memory.

5. `max_pfn` - This is the last PFN available to the system.

### 5.3 Initialising bootmem_data

* After the limits of usable physical memory are discovered by
  [setup_memory()][setup_memory], one of two boot memory initialisation
  functions is selected and provided with the start and end PFN for the node to
  be initialised.

* [init_bootmem()][init_bootmem], which initialises
  [contig_page_data][contig_page_data], is used by UMA architectures, whereas
  [init_bootmem_node()][init_bootmem_node] is used by NUMA systems to initialise
  a specified node. Both functions are trivial and rely on
  [init_bootmem_core()][init_bootmem_core] to do the real work.

* `init_bootmem_core()` does the following:

1. Inserts the node's [pg_data_t][pg_data_t] into [pgdat_list][pgdat_list] - at
   the end of this function this node is ready for use and needs to be
   established.

2. Records the starting and end address this node in its associated
   [bootmem_data_t][bootmem_data] structure.

3.  Allocates the bitmap representing page allocations. The size (in bytes,
    hence the division by 8) of the bitmap required is calculated as `mapsize =
    (end_pfn - start_pfn + 7)/8`.

4. The bitmap is stored at the _physical_ address pointed to by
   `bootmem_data_t->node_boot_start`, and the _virtual_ address to this map is
   placed in `bootmem_data_t->node_bootmem_map`.

5. Initialises the entire bitmap to 1, effectively marking all pages allocated,
   so in the next step it can determine available memory and mark them
   unallocated hence available.

6. On i386, [register_bootmem_low_pages()][register_bootmem_low_pages] is used
   to read through the [e820 map][e820], calling [free_bootmem()][free_bootmem]
   for each usable page to set the bit 0 (marking it available for allocation),
   before calling [reserve_bootmem()][reserve_bootmem] to reserve the pages
   needed by the actual bitmap itself.

### 5.4 Allocating Memory

* The [reserve_bootmem()][reserve_bootmem] function may be used to reserve pages
  for use by the caller, but is very cumbersome to use for general allocations.

* As discussed previously, there are a number of functions for allocating memory
  in both UMA and NUMA systems - [alloc_bootmem()][alloc_bootmem],
  [alloc_bootmem_low()][alloc_bootmem_low],
  [alloc_bootmem_pages()][alloc_bootmem_pages] and
  [alloc_bootmem_low_pages()][alloc_bootmem_low_pages] for UMA, and
  [alloc_bootmem_node()][alloc_bootmem_node],
  [alloc_bootmem_pages_node()][alloc_bootmem_pages_node], and
  [alloc_bootmem_low_pages_node()][alloc_bootmem_low_pages_node] for NUMA
  systems.

* All of these call [__alloc_bootmem()][__alloc_bootmem] or
  [__alloc_bootmem_node()][__alloc_bootmem_node] with different parameters:

1. (NUMA-only) `pgdat` - The node to allocate from. In the UMA case it is
   assumed to be [contig_page_data][contig_page_data].

2. `size` - The size in bytes of the requested allocation.

3. `align` - The number of bytes the request should be aligned to. For small
   allocations, they are aligned to `SMP_CACHE_BYTES` which on i386 will align
   to the L1 hardware cache.

4. `goal` - The preferred starting physical address to begin allocating
   from. The `_low` functions will start from physical address `0`, whereas the
   others will begin from [MAX_DMA_ADDRESS][MAX_DMA_ADDRESS] - the maximum
   address DMA transfers may be made from on the architecture.

* The core function for all the allocation APIs is
  [__alloc_bootmem_core()][__alloc_bootmem_core]. This performs the following
  tasks:

1. Linearly scan memory starting from `goal` for a block of memory large enough
   to satisfy the allocation.

2. Decides whether or not the new allocation can be merged with the previous
   one. A merge will be performed if the page for the previous allocation
   (`bootmem_data->pos`) is adjacent to the page found for this allocation, the
   previous page has some free space in it (`bootmem_data->offset != 0`), and
   the alignment is less than `PAGE_SIZE`.

3. Updates `pos` and `offset` fields to store the last page used for allocating
   and how much of the last page was used. If the page was fully used, offset
   will be set to `0`.

### 5.5 Freeing Memory

* Freeing is performed via either [free_bootmem()][free_bootmem] or
  [free_bootmem_node()][free_bootmem_node] for UMA and NUMA systems
  respectively.

* Both functions ultimately call [free_bootmem_core()][free_bootmem_core],
  `free_bootmem()` passes [contig_page_data][contig_page_data]`.bdata` and
  `free_bootmem_node()` passes the appropriate `pgdat->bdata`.

* `free_bootmem_core()` is relatively simple compared to other allocator
  functions. For each _full_ page affected by the free, the corresponding bit in
  the bitmap is set to 0, with double-frees highlighted via [BUG()][BUG].

* An important restriction of the free functions is that they may only be used
  for freeing _full_ pages. The code rounds down bitmap indexes to an integer
  value, meaning that the page containing the remaining portion of memory is
  left marked as allocated and thus is considered reserved.

* This ultimately doesn't turn out to be a huge problem as the allocations
  persist for the lifetime of the system, however it's an important restriction
  be aware of during boot time.

### 5.6 retiring the Boot Memory Allocator

* Late in the bootstrapping process, the function [start_kernel()][start_kernel]
  is called, which knows it is safe to remove the boot allocator and its
  associated data structures.

* Each architecture is required to provide a [mem_init()][mem_init] function
  that is responsible for:

1.  destroying the boot memory allocator and its associated structures.

2. Calculating the dimensions of low and high memory and outputting a message to
   a user describing these.

3. Providing final initialisations of hardware if necessary.

* For i386, the pertinent function here is [free_pages_init()][free_pages_init].

* `free_pages_init()` first tells the boot memory allocator to retire itself by
  calling [free_all_bootmem()][free_all_bootmem] for UMA systems or
  [free_all_bootmem_node()][free_all_bootmem_node] for NUMA systems.

* Both call [free_all_bootmem_core()][free_all_bootmem_core].

* For all unallocated pages known to the allocator for the node it:

1. Clears the `PG_reserved` flag in its [struct page][page].

2. Sets the `page->count` field to 1.

3. Calls [__free_pages()][__free_pages] so the buddy allocator (discussed in
   chapter 6) call build its free liss.

* Next, it frees all pages used for the bitmap and gives them to the buddy
  allocator.

* At this stage, the buddy allocator has control of all the pages in low memory,
  leaving only the high memory pages left to deal with.

* After [free_all_bootmem()][free_all_bootmem] returns,
  [free_pages_init()][free_pages_init] counts the number of reserved pages for
  accounting purposes.

* After this, `free_pages_init()` calls
  [one_highpage_init()][one_highpage_init] for every page between
  [highstart_pfn][highstart_pfn] and [highend_pfn][highend_pfn].

* `one_highpage_init()` does the same as `free_all_bootmem_core()` did for the
  low memory - clears the `PG_reserved` flag, sets the count to 1 and calls
  `__free_pages()` to hand them over to the buddy allocator.

* At this point, the boot memory allocator is no longer required and the buddy
  allocator is the main physical page allocator for the system.

* An interesting point to note is that not only is the data used for
  bootstrapping removed, so is the _code_. All initialisation functions that are
  required only during system startup are marked [__init][__init], e.g.
  `unsigned long __init free_all_bootmem(void);`

* All (non-module) uses of `__init` cause the functions specified this way to
  be placed together in the `.init` section of the kernel binary (typically
  [vmlinux][vmlinux] is an ELF format binary), and (on i386)
  [free_initmem()][free_initmem] walks all the pages from
  [__init_begin][__init_begin] to [__init_end][__init_end], potentially freeing
  a significant amount of RAM.

* The following flowchart describes how the global [mem_map][mem_map] array is
  allocated and initialised, and how the pages are given to the main allocator:

```
                  -----------------------------------------------------
                  | Initialise mem_map struct page array with bootmem |
                  |   On retirement, give pages to buddy allocator    |
                  -----------------------------------------------------
                                            |
                                            |
                             /--------------X-----------------------------\
                             |                                            |
                             |                                            v
                             |                               ----------------------------
                             |                               | Retire bootmem allocator |
                             |                               |  with free_pages_init()  |
                             |                               ----------------------------
                             v                                            |
                 --------------------------                               |
                 | Allocate mem_map array |                               v
                 --------------------------                  ----------------------------
                             |                               | free_all_bootmem() which |
                             |                               |     ultimately calls     |
                             v                               | free_all_bootmem_core()  |
                 -------------------------                   ----------------------------
                 | free_area_init_core() |                                |
                 -------------------------                                |
                             |                                            v
                             |                             -------------------------------
              /--------------X--------------\              |  For all pages that are not |
              |                             |              |  allocated by bootmem, call |
              v                             |              | FreePageReserved to mark it |
   ------------------------                 |              | free and set page->count to |
   | alloc_bootmem_node() |                 |              |   1 with set_page_count()   |
   ------------------------                 |              -------------------------------
              |                             |                             |
              |                             |                             |
              v                             v                             v
------------------------------ --------------------------- ------------------------------
| alloc_bootmem_core() uses  | |  Call SetPageReserved() | | Call __free_page() as the  |
|     pg_data_t->bata to     | |   record what zone the  | |  struct page looks like a  |
|    allocate a block of     | |   page belongs to with  | | normally allocated page to |
| memory that is zero-filled | | set_page_zone() and set | |   the main allocator. The  |
|   and stores the mem_map   | |    page->count to 0     | | page will be automatically |
------------------------------ --------------------------- |   placed on the main free  |
                                                           |         page lists         |
                                                           ------------------------------
```

## Chapter 6: Physical Page Allocation

* Physical pages are managed using the 'binary buddy allocator', originally
  devised by Knowlton and further described by Knuth ([TAocP][taocp] Volume 1,
  Section 2.5, pg. 442) and has shown to be extremely fast compared to other
  allocators (see David G. Korn and Kiem-Phong Bo. In search of a better malloc
  In Proceedings of the Summer 1985 USENIX Conference, pages 489–506, Portland,
  OR, 1985.)

* This is an allocation scheme that combines a power-or-two allocator with free
  buffer coalescing.

* The concept behind a binary buddy allocator is quite simple:

1. Memory is broken up into large blocks of pages where each block is a power
   of 2 number of pages.

2. If a block of the desired size is not available a large block is broken up in
   half and the two blocks are considered 'buddies' to each other - one half is
   used for the allocation, the other is free.

3. The blocks are continuously halved as necessary until a block of the desired
   size is available.

4. When a block is freed the buddy is examined, and the two are coalesced if it
   is free.

### 6.1 Managing Free Blocks

* The allocator maintains blocks of free pages where each block is a
  power-of-two number of pages.

* The exponent of the power of 2 is referred to as its _order_.

* An array of [free_area_t][free_area_t] structs are maintained for each order
  each of which point to a linked list of blocks of pages that are free, e.g.:

```
order    free_area_t             Free page blocks
       zone->free_area
     |---------------|-|
|    |       0       | | -> [] -> [] -> ...       2^0 page-sized blocks
|    |---------------|-|
|    |       1       | |
|    |---------------|-|
|    |       2       | |
|    |---------------|-|
|    |       3       | |
|    |---------------|-|
|    |       4       | | -> [==] -> [==] -> ...   2^4 page-sized blocks
|    |---------------|-|
|    |       5       | |
|    |---------------|-|
|    |       6       | |
|    |---------------|-|
|    |       7       | |
|    |---------------|-|
|    |       8       | |
|    |---------------|-|
|    |       9       | | -> [==========] -> ...   2^(MAX_ORDER - 1) page-sized blocks
|    |---------------|-|
|    |   MAX_ORDER   | | (exclusive bound)
v    |---------------|-|
```

* [MAX_ORDER][MAX_ORDER] is currently set at 10 unless otherwise specified by
  `CONFIG_FORCE_MAX_ZONEORDER`. This eliminates the chance that a larger block
  will be split to satisfy a request where a smaller block would suffice.

* The page blocks are maintained on a linear linked list using
  [struct page][page]`->list`.

* Each zone has a [free_area_t][free_area_t] array field of `free_area[MAX_ORDER]`.

* Looking at `free_area_t` itself:

```c
typedef struct free_area_struct {
        struct list_head        free_list;
        unsigned long           *map;
} free_area_t;
```

* `free_list` is a linked list of free page blocks, `map` is a bitmap
  representing the state of a pair of buddies.

* Linux saves memory by using one bit instead of two to represent each pair of
  buddies - each time a buddy is allocated or freed, the bit representing the
  pair of buddies is toggled so that the bit is 0 if the pair of pages are both
  free or both full and 1 if only one buddy is in use.

* To toggle the correct bit, the macro [MARK_USED()][MARK_USED] is used:

```c
#define MARK_USED(index, order, area) \
        __change_bit((index) >> (1+(order)), (area)->map)
```

* `index` is the index of the page within the global [mem_map][mem_map] array.

* The correct bit to flip is determined by shifting `index` right by `1 +
  order`. LS - Need to understand this better.

* [__change_bit()][__change_bit] ultimately uses the [btc][btc] 'bit test and
  complement' op-code.

### 6.2 Allocating Pages

* Let's have a look at the Linux API for the allocation of page frames:

1. [alloc_page()][alloc_page] - Allocats a single page and returns a
   [struct page][page].
2. [alloc_pages()][alloc_pages] - Allocates `2^order` pages and returns a
   `struct page`.
3. [get_free_page()][get_free_page] - Allocates a single page, zeroes it and
   returns a virtual address (deprecated name, macro wrapper for
   [get_zeroed_page()][get_zeroed_page].)
4. [__get_free_page()][__get_free_page] - Allocates a single page and returns a
   virtual address.
5. [__get_free_pages()][__get_free_pages] - Allocates `2^order` pages and
   returns a virtual address.
6. [__get_dma_pages()][__get_dma_pages] - Allocates `2^order` pages from
   `ZONE_DMA` and returns a `struct page`.

* Each of these functions take an integer `gfp_mask` parameter which is a set of
  flags that determine how the allocator will behave (discussed further in 6.4.)

* They all ultimately use the core function [__alloc_pages()][__alloc_pages],
  the reason these are provided is so the correct node and zone will be
  used. Different users will require different zones - some device drivers will
  need `ZONE_DMA`, disk buffers will need `ZONE_NORMAL` and callers shouldn't
  have to be aware of which node is being used.

* Allocations are always for a specified order - 0 when only a single page is
  required.

* If a free block can't be found of the requested order, a higher order block is
  split into two buddies - one is allocated and the other is placed on the free
  list for the lower order.

* For example, a process requesting a 2^2 block when one is not available, nor
  2^3, but a 2^4 block is:

```
order    free_area_t             Free page blocks
       zone->free_area
     |---------------|-|
|    |       0       | |            Requesting Process
|    |---------------|-|                    ^
|    |       1       | |                    |
|    |---------------|-|                    |
|    |       2       | | <---------------\  |
|    |---------------|-|                 |  |
|    |       3       | | <----------\  [==|==]
|    |---------------|-|            |     ^
|    |       4       | | ---\       |     |
|    |---------------|-|    |    [=====|=====]
|    |       5       | |    |          ^
|    |---------------|-|    |          |
|    |       6       | |    \--->[===========]
|    |---------------|-|
|    |       7       | |
|    |---------------|-|
|    |       8       | |
|    |---------------|-|
|    |       9       | |
|    |---------------|-|
|    |   MAX_ORDER   | |
v    |---------------|-|
```

* When a block is later freed, the buddy will be checked. If both are free, they
  are merged to form a higher order block and placed on the higher free list
  where its buddy is checked and so on.

* If the buddy is not free, the freed block will be placed on the free list at
  the current order.

* During these operations interrupts have to be disabled to prevent an interrupt
  handler manipulating the lists while a process has them in an inconsistent
  state - this is achieved by using an interrupt safe spinlock.

* The next decision that needs to be made is which memory node or
  [pg_data_t][pg_data_t] to use. Linux uses a node-local allocation policy - it
  aims to use the memory bank associated with the CPU running the
  page-allocating process.

* The function `_alloc_pages()` differs depending on the system type -
  [UMA _alloc_pages()][_alloc_pages/uma] and
  [NUMA _alloc_pages()][_alloc_pages/numa].

* Regardless of what API is used, [__alloc_pages()][__alloc_pages] is the heart
  of the allocator (note this function is never called directly.), it:

1. Examines the selected zone and checks whether it is suitable to allocate from
   based on the number of available pages.

2. If the zone is not available, the allocator may fall back to other zones. The
   order of zones to fall back on is decided at boot time by
   [build_zonelists()][build_zonelists], though generally `ZONE_HIGHMEM` falls
   back to `ZONE_NORMAL` which in turn will fall back to `ZONE_DMA`.

3. If the number of free pages reaches the zone's `pages_low` watermark, it will
   wake `kswapd` to begin freeing up pages from zones, and if memory is
   extremely tight, it will do the work of `kswapd` itself.

4. After the zone has been decided on, the function [rmqueue()][rmqueue] is
   called to allocate the block of pages or split higher level blocks if one of
   the appropriate size is not available.

### 6.3 Free Pages

* The API for freeing of pages is a lot simpler and exists to help remember the
  order of the block to free - this is one disadvantage of a buddy allocator -
  the caller has to remember the size of hte original allocation.

* Let's take a look at the API:

1. [__free_pages()][__free_pages] - Given a [struct page][page], Frees `2^order`
   pages from the given page.
2. [__free_page()][__free_page] - Given a [struct page][page], frees it.
3. [free_page()][free_page] - Given a virtual address, frees a page from it.

* The principle function is [__free_pages_ok()][__free_pages_ok], which should
  not be called directly, rather [__free_pages()][__free_pages] performs simple
  checks first.

* When a buddy is freed linux tries to coalesce the buddies together
  immediately, if possible. This is not optimal, because in the worst-case
  scenario there will be many coalitions followed by the immediate splitting of
  the same blocks.

* To detect whether the buddies can be merged, linux checks the bit
  corresponding to the affected pair of buddies in `free_area->map`.

* Recall that we track buddy status using a single bit - 0 if both are set or
  free, 1 if one is set and the other not. Since we just freed a buddy, we know
  that a 0 implies both are free, so we can go ahead and merge.

* We can calculate the address of the buddy relatively easily - since the
  allocations are always in blocks of size `2^k`, the address of the block or at
  least its offset within the `zone_mem_map` will also be a power of `2^k` -
  this means that the binary representation of this address will always have at
  least `k` zeros to the right of the address.

* The buddy will have the `k`th bit flipped. Here `imask` provides a mask for just
  that bit:

```
mask = (~0 << k)
imask = 1+~mask = -mask
```

* After the buddy is merged, it is removed from the free list and the newly
  coalesced pair moves to the next higher order to see if it may also be merged
  with its buddy there.

* Quoting from Knuth ([TAocP][taocp] Volume 1, Section 2.5, pg. 442):

>The key fact underlying the practical usefulness of this method is that if we
>know the address of a block (the memory location of its first word), and if we
>also know the size of that block, we know the address of its buddy. For
>example, the buddy of the block of size 16 beginning in binary location
>101110010110000 is a block starting in binary location 101110010100000. To see
>why this must be true, we first observe that as the algorithm proceeds, the
>address of a block of size `2^k` is a multiple of `2^k`. In other words, the
>address in binary notation has at least `k` zeros at the right. This
>observation is easily justified by induction: If it is true for all blocks of
>size `2^(k+1)`, it is certainly true when such a block is halved.  Therefore a
>block of size, say, 32 has an address of the form xx. . . x00000 (where the x’s
>represent either 0 or 1); if it is split, the newly formed buddy blocks have
>the addresses xx. . . x00000 and xx. . . x10000.

* Considering an example of order 2 allocations - addresses will look like
  `0101100`, `0111100`, `1000000`, etc. - we are allocating `2^2` pages at a
  time. If we need to split an address, the first of the two buddies will be the
  existing address, the second half way through, i.e. `address + 2^1`, so
  `0111100` will split to `0111100` and `0111110` - and the condition holds
  true.

### 6.4 Get Free Page (GFP) Flags

* A concept that is maintained throughout the VM is the Get Free Page (GFP)
  flags. These determine how the allocator and `kswapd` will behave for the
  allocation and freeing of pages.

* For example, an interrupt handler is not allowed to sleep, so it will _not_
  set `__GFP_WAIT` which indicates the caller may sleep.

* There are 3 sets of GFP flags all [set in include/linux/mm.h][gfp-flags]. The
  first set are _zone modifiers_:

1. `__GFP_DMA` - Allocate from `ZONE_DMA` if possible.

2. `__GFP_HIGHMEM` - Allocate from `ZONE_HIGHMEM` if possible.

3. `GFP_DMA` - Alias for `__GFP_DMA`.

* There isn't a zone modifier for `ZONE_NORMAL`, as this is the default (the
  zone modifier flag is used as an offset within an array and `ZONE_NORMAL` is
  offset 0.)

* The second set are _action modifiers_ - they change the behaviour of the VM
  and what the calling process may do:

1. `__GFP_WAIT` - The caller is not high priority and can sleep or reschedule.

2. `__GFP_HIGH` - Used by a high priority or kernel process (LS - the book says
   it's unused, but I see it in the 4.5 kernel so I'm not going to mark it as
   unused here in case 2.4.22 uses it too.)

3. `__GFP_IO` - The caller can perform low-level I/O. The main effect in 2.4.22
   is to determine whether [try_to_free_buffers()][try_to_free_buffers] can
   flush buffers. It is also used by at least one journaled filesystem.

4. `__GFP_HIGHIO` - Determines that I/O can be performed on pages mapped into
   high memory. Only used in `try_to_free_buffers()`.

5. `__GFP_FS` - The caller can make calls to the filesystem layer - this is
   provided when the caller is filesystem-related (e.g. the buffer cache), and
   wants to avoid recursively calling itself.

* These low-level flags on their own are too primitive to be easily used as-is,
  and it is often difficult to know the appropriate combination of these to use,
  which brings us to the third set - high-level combinations of the low-level
  action modifiers. These are more digestible forms of the primitives:

```c
#define GFP_NOHIGHIO    (__GFP_HIGH | __GFP_WAIT | __GFP_IO)
#define GFP_NOIO        (__GFP_HIGH | __GFP_WAIT)
#define GFP_NOFS        (__GFP_HIGH | __GFP_WAIT | __GFP_IO | __GFP_HIGHIO)
#define GFP_ATOMIC      (__GFP_HIGH)
#define GFP_USER        (             __GFP_WAIT | __GFP_IO | __GFP_HIGHIO | __GFP_FS)
#define GFP_HIGHUSER    (             __GFP_WAIT | __GFP_IO | __GFP_HIGHIO | __GFP_FS | __GFP_HIGHMEM)
#define GFP_KERNEL      (__GFP_HIGH | __GFP_WAIT | __GFP_IO | __GFP_HIGHIO | __GFP_FS)
#define GFP_NFS         (__GFP_HIGH | __GFP_WAIT | __GFP_IO | __GFP_HIGHIO | __GFP_FS)
#define GFP_KSWAPD      (             __GFP_WAIT | __GFP_IO | __GFP_HIGHIO | __GFP_FS)
```

* Considering each of these:

1. `GFP_NOHIGHIO` - This is only used in one place -
   [alloc_bounce_page()][alloc_bounce_page] - during the creation of a bounce
   buffer for I/O in high memory.
2. `GFP_NOIO` - This is used by callers who are already performing I/O, for
   example when the loopback device is trying to get a page for a buffer head,
   it uses this flag to make sure it doesn't introduce more I/O. In fact, it
   seems like the flag was introduced specifically to avoid a deadlock in the
   loopback device :)
3. `GFP_NOFS` - This issued only by the buffer cache and filesystems to make
   sure sure they do not recursively call themselves by accident.
4. `GFP_ATOMIC` - Used whenever the caller cannot sleep and must be serviced if
   at all possible. Any interrupt handler that requires memory must use this
   flag to avoid sleeping or performing I/O. Many subsystems use this system
   during init, e.g. [buffer_init()][buffer_init] or [inode_init()][inode_init].
5. `GFP_USER` - Defunct - this is a flag of historic interest - in the 2.2.x
   series memory was assigned low, medium, or high priority - if memory was
   tight, a request with `GFP_USER` (low priority) would fail whereas others
   would keep trying. Now it has no impact and is treated no differently than
   `GFP_KERNEL`.
6. `GFP_HIGHUSER` - The allocator should allocate from `ZONE_HIGHMEM` if
   possible - used when the page is allocated on behalf of a user process.
7. `GFP_KERNEL` - This is the most liberal of the combined flags - the caller is
   free to do whatever they want.
8. `GFP_NFS` - Defunct - in the 2.0.x series this flag would determine what the
   reserved page size was - usually this was 20, if this flag was set it'd
   be 5. Now this flag is identical to `GFP_KERNEL`.
9. `GFP_KSWAPD` - Defunct - Effectively the same as `GFP_KERNEL`, modulo
   `__GFP_HIGH`.

### 6.5 Process Flags

* A process may also set flags in the [struct task_struct][task_struct] which
  affect allocator behaviour.

* Considering the flags that affect the VM:

1. `PF_MEMALLOC` - Set by OOM Killer - flags the process as a memory
   allocator. This is set by `kswapd` and is set for any process that is about
   to be killed by the OOM killer which is discussed in detail in chapter 13. It
   indicates to the buddy allocator to ignore zone watermarks and assign pages
   if at all possible.

2. `PF_MEMDIE` - Set by OOM Killer - effectively functions the same as
   `PF_MEMALLOC`.

3. `PF_FREE_PAGES` - This is set when the buddy allocator calls
   [try_to_free_pages()][try_to_free_pages] itself to indicate that free pages
   should be reserved for the calling process in
   [__free_pages_ok()][__free_pages_ok] instead of returning them to the free
   lists.

### 6.6 Avoiding Fragmentation

* Any allocator needs to be careful to avoid both internal and external
  fragmentation.

* External fragmentation is the ability to service a request because the
  available memory exists only in small blocks.

* Internal fragmentation is where space is wasted when a large block had to be
  assigned to service a small request.

* In Linux, external fragmentation is not a serious problem because large
  requests for contiguous pages are rare and usually [vmalloc()][vmalloc]
  suffices to serve the request (see chapter 7 for more details on this) - the
  list of free blocks ensures that large blocks do not have to be split
  unnecessarily.

* Internal fragmentation is the biggest failing of the binary buddy
  system. Although it is expected to be in the region of 28% (LS - what does
  this measure actually mean?), it has been shown it can be in the region of
  60%, in comparison to just 1% with the first-fit allocator, and additionally
  it's been shown that variations of the buddy system does not help the
  situation significantly.

* Linux addresses this by using a 'slab allocator' to carve pages into small
  blocks of memory for allocation. This is discussed in further detail in
  chapter 8.

* With the combination of the binary buddy allocator and the slab allocator, the
  kernel can keep the amount of wasted memory due to internal fragmentation to a
  minimum.

## Chapter 7: Non-contiguous Memory Allocation

* When dealing with large amounts of memory it is better if possible to use
  physically contiguous pages both in terms of caching and memory access latency.

* Unfortunately this is not always possible due to external fragmentation
  problems with the binary buddy allocator (LS - I thought internal
  fragmentation was more the problem for the binary buddy system? Also is this
  the only cause of non-contiguous pages?)

* Linux provides a means for non-contiguous physical memory to be used in
  contiguous virtual memory via [vmalloc()][vmalloc].

* When `vmalloc()` is used, an area is reserved in the virtual address space
  between [VMALLOC_START][VMALLOC_START] and [VMALLOC_END][VMALLOC_END] - the
  location of `VMALLOC_START` varies depending on the amount of available
  physical memory, but the region will be at least
  [VMALLOC_RESERVE][VMALLOC_RESERVE] (aliased to
  [__VMALLOC_RESERVE][__VMALLOC_RESERVE]) in size, which on i386 is 128MiB - the
  exact size of the region was discussed in more detail in section 4.1

* The page tables in this region are adjusted as needed to point to physical
  pages which are allocated with the normal physical page allocator - this means
  that allocation _has_ to be a multiple of the hardware page size.

* Because performing allocations this way require modifications to be made to
  the kernel page tables, there is a limitation on how much memory can be mapped
  with `vmalloc()` because only the virtual address space between
  `VMALLOC_START` and `VMALLOC_END` is available.

* Because of the limited nature of `vmalloc()`, it is used sparingly in the core
  kernel - in fact in 2.4.22 it is only used for storing swap map information
  (see chapter 11) and for loading kernel modules into memory.

### 7.1 Describing Virtual Memory Areas

* The vmalloc address space is managed with a 'resource map
  allocator'. [struct vm_struct][vm_struct] is used for storing the base, size
  pairs for this allocator and is defined as follows:

```c
struct vm_struct {
        unsigned long flags;
        void * addr;
        unsigned long size;
        struct vm_struct * next;
};
```

* In theory a fully-fledged VMA count have been used, but a VMA contains more
  information that doesn't apply to vmalloc areas and so doing that would be
  wasteful.

* Looking at each field:

1. `flags` - Either set to `VM_ALLOC` if used with [vmalloc()][vmalloc] or
   `VM_IOREMAP` when [ioremap()][ioremap] is used to map high memory into the
   kernel virtual address space.

2. `addr` - Starting address of the memory block.

3. `size` - Size in bytes.

4. `next` - Pointer to the next `vm_struct`, these are ordered by address and
   the list is protected by the [vmlist_lock][vmlist_lock] lock.

* Each area is separated by at least one page to protect against overruns:

```
------------------------------------------------------/\/\
|  vmalloc   | Page |  vmalloc   | Page |  vmalloc   |    |
| Allocation | Gap  | Allocation | Gap  | Allocation |    |
------------------------------------------------------/\/\
^                                                         ^
| VMALLOC_START                               VMALLOC_END |
```

* When the kernel wants to allocate a new area, the `vm_struct` list is searched
  linearly via [get_vm_area()][get_vm_area].

* Space for the struct is allocated with [kmalloc()][kmalloc].

* When the virtual area is used for remapping an area for I/O (known as
  'ioremapping'), `get_vm_area()` will be called directly to map the requested
  area.

### 7.2 Allocating a Non-contiguous Area

* Let's take a look at the vmalloc allocation API:

1. [vmalloc()][vmalloc] - Allocate a number of pages in vmalloc space that
   satisfy the required size.

2. [vmalloc_dma()][vmalloc_dma] - Allocate a number of pages in vmalloc space
   from `ZONE_DMA`.

3. [vmalloc_32()][vmalloc_32] - Allocates memory that is suitable for 32-bit
   addressing. This ensures that physical page frames are in `ZONE_NORMAL` which
   is required by 32-bit devices.

* Each of these functions call [get_vm_area()][get_vm_area] to find a region
  large enough to store the requested amount of memory, searching through a
  linear linked-list of `vm_struct`s and returning a new struct describing the
  allocated region.

* Next they allocate the necessary PGD entries with
  [vmalloc_area_pages()][vmalloc_area_pages] (and subsequently
  [__vmalloc_area_pages()][__vmalloc_area_pages]), PMD entries with
  [alloc_area_pmd()][alloc_area_pmd] and PTE entries with
  [alloc_area_pte()][alloc_area_pte] before finally allocating the page with
  [alloc_page()][alloc_page].

* The page table updated by [vmalloc()][vmalloc] is not the that of the current
  process, but rather the reference page table stored at
  [init_mm][init_mm]`->pgd`. As a result processes accessing the vmalloc area
  will cause a page fault exception (its page tables aren't pointing there) and
  special handling has to be performed. Diagrammatically:

```
-------------------                          -------------------
| Process A Calls |                          |  Process B page |
|    vmalloc()    |                          |   page faults   |
-------------------                          | do_page_fault() |
         |                                   -------------------     Process B Address
         v                                             |             Space managed by
-------------------   Reference Page Table             |            Process Page Tables
|  Reserve space  |   --------------------             |            -------------------
|   in Reference  |-\ |                  |             v            |                 |
|   Page Table    | | |                  |    -------------------   |                 |
------------------- | |                  |    |   Fault is in   |   |                 |
         |          | |                  |    |  vmalloc region |   |                 |
         |          | |                  |    -------------------   |                 |
         |          \>|------------------|             |            |                 |
         v            |                  |             |            |                 |
-------------------   |  Inserted Pages  |             |            |                 |
| After reserving |   |                  |             v            |                 |
| space, allocate |   |------------------|   --------------------   |-----------------|
|      pages      |   |                  |   |      Copy in     |   |                 |
-------------------   |   In Reference   |-->|  necessary page  |-->|   Copied Entry  |
   |                  |                  |   |    table entry   |   |                 |
   |                  |------------------|   |  from reference  |   |-----------------|
   |                  |                  |   --------------------   |                 |
   |                  |    Page Table    |                          |                 |
   |                  |                  |                          |                 |
   |     /----------->|------------------|                          |                 |
   |     |            |                  |                          |                 |
   |     |            |------------------|<--    VMALLOC_START_  -->|                 |
   |     |            |                  |                          |                 |
   |     |            |                  |                          |                 |
   |     |            |                  |                          |                 |
   v     |            |                  |                          |                 |
-------------------   |                  |                          |                 |
| Buddy Allocator |   |------------------|<--     PAGE_OFFSET    -->|-----------------|
|  alloc_page()   |   |                  |                          |                 |
-------------------   |                  |                          |                 |
   ^      ^      ^    |    Userspace     |                          |    Userspace    |
   |      |      |    |     Portion      |                          |     Portion     |
------ ------ ------  |                  |                          |                 |
|page| |page| |page|  |                  |                          |                 |
------ ------ ------  --------------------<-- init_mm->pgd          -------------------
      Physically            Virtually
 Non-contiguous Pages   Contiguous Pages
```
### 7.3 Freeing a Non-contiguous Area

* The function [vfree()][vfree] is responsible for freeing a virtual area. It
  linearly scans the list of [vm_struct][vm_struct]s looking for the appropriate
  region and then calls [vmfree_area_pages()][vmfree_area_pages] on the region
  to be freed.

* `vmfree_area_pages()` is the exact opposite of
  [vmalloc_area_pages()][vmalloc_area_pages] - it walks the page table and frees
  up PTEs and associated pages for the region rather than allocating them.

## Chapter 8: Slab Allocator

* The kernel's 'general purpose' allocator is a 'slab allocator'. The basic
  concept is to have caches of commonly used objects kept in an initialised
  state, available for use by the kernel.

* Without an object-based allocator, the kernel would spend a lot of the time
  allocating, initialising and freeing the same object over and over again - the
  slab allocator aims to cache freed objects so they can be reused.

* The slab allocator consists of a variable number of caches linked together on
  a doubly-linked circular list called a 'cache chain'.

* In the context of the slab allocator a cache is a manager for a number of
  objects of a particular type, e.g. a [struct mm_struct][mm_struct] cache, each
  of which are managed by a [struct kmem_cache_s][kmem_cache_s].

* Each cache maintains blocks of contiguous pages in memory called 'slabs' that
  are carved into small chunks for the objects that the cache manages:

```
-------------          ---------          -------------
| lastcache |--------->| cache |--------->| nextcache |
-------------          ---------          -------------
                        |  |  |
                        |  |  |
          /-------------/  |  \--------------\
          |                |                 |
          v                v                 v
   --------------  -----------------  --------------
   | slabs_full |  | slabs_partial |  | slabs_free |
   --------------  -----------------  --------------
          |                |                 |
          |                |                 |
          |                |                 |
          |                |                 |
          v                v                 v
      ---------        ---------         ---------
      | slabs |        | slabs |         | slabs |
      ---------        ---------         ---------
          |                |                 |
          |                |                 |
          |                |                 |
          |                |                 |
          v                v                 v
      ---------        ---------         ---------
      | pages |        | pages |         | pages |
      ---------        ---------         ---------
          |                |                 |
          |                |                 |
      /-------\        /-------\         /-------\
      |       |        |       |         |       |
      v       v        v       v         v       v
   ------- -------  ------- -------   ------- -------
   | obj | | obj |  | obj | | obj |   | obj | | obj |
   ------- -------  ------- -------   ------- -------
```

* The slab allocator has 3 principle aims:

1. The allocation of small blocks of memory to help eliminate internal
   fragmentation that would be otherwise caused by the buddy system.

2. The caching of commonly used objects so the system does not waste time
   allocating, initialising and destroying objects.

3. Better use of the hardware cache by aligning objects to the L1 or L2 caches.

* To help eliminate internal fragmentation normally caused by a binary buddy
  allocator, two sets of caches of small memory buffers ranging from `2^5` (32)
  bytes to `2^17` (131,072) bytes are maintained.

* One cache set is suitable for use with DMA devices, the other not, these are
  called 'size-N(DMA)' and 'size-N' respectively, where N is the size of the
  allocation.

* The function [kmalloc()][kmalloc] is provided for allocating to these (see
  8.4.1.) Additionally, the sizes of the caches are discussed in further detail
  in 8.4.

* The slab allocator also maintains caches of commonly used objects - for many
  structures used in the kernel, the time needed to initialise an object is
  comparable with, or exceeds, the cost of allocating space for it.

* When a new slab is created, a number of objects are packed into it and
  initialised using a constructor if available. When an object is freed, it is
  left in its initialised state so that object allocation will be quick.

* The final task of the slab allocator is to optimise the use of hardware
  caches. In order to do this, 'slab colouring' is performed - if there is space
  left over after objects are packed into a slab, the remaining space is used to
  colour the slab.

* Slab colouring is a scheme that attempts to ensure that objects in different
  slabs use different lines in the cache. By placing objects at different
  starting offsets within the slab, they will likely use different lines in the
  CPU cache which helps ensure that objects from the same slab cache will be
  unlikely to flush each other.

* With this scheme, space that would otherwise be wasted is used to fulfil a new
  function. Diagrammatically:

```
                             ------------------
                             |                |
                             |                |
                             |                |
                             |                |
                             |  Rest of page  |
 ----------------------      |  filled with   |
 |   kmem_getpages()  |      |  objects and   |
 ----------------------      |    padding     |
            |                |                |
            |                |                |  ----------------------
            |                |                |  | kmem_cache_alloc() |
            v                |                |  ----------------------
 ----------------------      |----------------|             |
 | __get_free_pages() |      | Colour padding |             |
 ----------------------      |----------------|             |
            |                |                |<------------/
            |                |  Initialised   |                 -----------------------
            |                |     Object     |                 | Object allocated to |
            v                |                |---------------->| requesting process  |
 ----------------------      |----------------| ^               -----------------------
 |   Allocates pages  |      | Colour padding | |
 |      of order      |      |----------------| |
 | cachep->gfp_order  |      |                | |   L1 CPU
 ----------------------      |  Initialised   | | Cache Size
            |                |     Object     | |
            |                |                | |
            \--------------->|----------------- v
```

* Linux does not attempt to colour page allocations based on the physical
  address, or to order where objects are placed. The details of cache colours is
  discussed further in 8.1.5.

* On an SMP system, a further step is taken to help cache utilisation where each
  cache has a small array of objects reserved for each CPU - this is discussed
  in more detail in 8.5.

* The slab allocator allows for slab debugging if `CONFIG_SLAB_DEBUG` is
  set. This adds 'red zoning' and 'objection poisoning'.

* Red zoning adds a market at either end of the object - if it is disturbed the
  allocator knows which object encountered a buffer overflwo, and reports it.

* Poisoning an object will fill it with a predefined bit pattern
  ([POISON_BYTE][POISON_BYTE] which is defined as `0x5a`) at slab creation and
  after free. On each allocation this pattern is checked - if it is changed, the
  allocator knows the object was used before it was allocated, and flags it.

* Taking a look at the slab allocator's API:

1. [kmem_cache_create()][kmem_cache_create] - Creates a new cache and adds it to
   the cache chain.

2. [kmem_cache_reap()][kmem_cache_reap] - Scans at most
   [REAP_SCANLEN][REAP_SCANLEN] caches and selects one for reaping all per-CPU
   objects and free slabs from. This is called when memory is tight.

3. [kmem_cache_shrink()][kmem_cache_shrink] - Deletes all per-CPU objects
   associated with a cache and deletes all slabs in the
   `kmem_cache_s->slabs_free` list. It returns the number of freed pages.

4. [kmem_cache_alloc()][kmem_cache_alloc] (and subsequently
   [__kmem_cache_alloc()][__kmem_cache_alloc]) - Allocates a single object from
   the cache and returns it to the caller.

5. [kmem_cache_free()][kmem_cache_free] - Frees an object and returns it to the cache.

6. [kmalloc()][kmalloc] - Allocates a block of memory from one of the specified
   `size`s cache.

7. [kfree()][kfree] - Frees a block of memory allocated with `kmalloc()`.

8. [kmem_cache_destroy()][kmem_cache_destroy] - Destroys all objects in all
   slabs and frees up all associated memory before removing the cache from the
   chain.

### 8.1 Caches

* One cache exists for each type of object that is to be cached. To see a list
of caches available on a running system `/proc/slabinfo` can be queried,
e.g. (from my local [2.4.22 VM system][linux-archaeology]:

```
morgoth:~# cat /proc/slabinfo
slabinfo - version: 1.1 (SMP)
kmem_cache            64     64    244    4    4    1 :  252  126
tcp_tw_bucket          0      0     96    0    0    1 :  252  126
tcp_bind_bucket        0      0     32    0    0    1 :  252  126
tcp_open_request       0      0     64    0    0    1 :  252  126
inet_peer_cache        0      0     64    0    0    1 :  252  126
ip_fib_hash          113    113     32    1    1    1 :  252  126
ip_dst_cache          48     48    160    2    2    1 :  252  126
arp_cache             30     30    128    1    1    1 :  252  126
blkdev_requests     3080   3080     96   77   77    1 :  252  126
nfs_write_data         0      0    352    0    0    1 :  124   62
nfs_read_data          0      0    352    0    0    1 :  124   62
nfs_page               0      0     96    0    0    1 :  252  126
journal_head         234    234     48    3    3    1 :  252  126
revoke_table         126    253     12    1    1    1 :  252  126
revoke_record          0      0     32    0    0    1 :  252  126
dnotify_cache          0      0     20    0    0    1 :  252  126
file_lock_cache       80     80     96    2    2    1 :  252  126
fasync_cache           0      0     16    0    0    1 :  252  126
uid_cache            226    226     32    2    2    1 :  252  126
skbuff_head_cache    384    384    160   16   16    1 :  252  126
sock                  54     54    832    6    6    2 :  124   62
sigqueue              58     58    132    2    2    1 :  252  126
kiobuf                 0      0     64    0    0    1 :  252  126
cdev_cache           118    118     64    2    2    1 :  252  126
bdev_cache            59     59     64    1    1    1 :  252  126
mnt_cache            118    118     64    2    2    1 :  252  126
inode_cache          413    413    512   59   59    1 :  124   62
dentry_cache         570    570    128   19   19    1 :  252  126
filp                 150    150    128    5    5    1 :  252  126
names_cache            4      4   4096    4    4    1 :   60   30
buffer_head         2360   2360     96   59   59    1 :  252  126
mm_struct             48     48    160    2    2    1 :  252  126
vm_area_struct       400    400     96   10   10    1 :  252  126
fs_cache             118    118     64    2    2    1 :  252  126
files_cache           36     36    416    4    4    1 :  124   62
signal_act            27     27   1312    9    9    1 :   60   30
size-131072(DMA)       0      0 131072    0    0   32 :    0    0
size-131072            0      0 131072    0    0   32 :    0    0
size-65536(DMA)        0      0  65536    0    0   16 :    0    0
size-65536             0      0  65536    0    0   16 :    0    0
size-32768(DMA)        0      0  32768    0    0    8 :    0    0
size-32768             0      0  32768    0    0    8 :    0    0
size-16384(DMA)        0      0  16384    0    0    4 :    0    0
size-16384             0      0  16384    0    0    4 :    0    0
size-8192(DMA)         0      0   8192    0    0    2 :    0    0
size-8192              4      4   8192    4    4    2 :    0    0
size-4096(DMA)         0      0   4096    0    0    1 :   60   30
size-4096            267    267   4096  267  267    1 :   60   30
size-2048(DMA)         0      0   2048    0    0    1 :   60   30
size-2048              8      8   2048    4    4    1 :   60   30
size-1024(DMA)         0      0   1024    0    0    1 :  124   62
size-1024            108    108   1024   27   27    1 :  124   62
size-512(DMA)          0      0    512    0    0    1 :  124   62
size-512              56     56    512    7    7    1 :  124   62
size-256(DMA)          0      0    256    0    0    1 :  252  126
size-256             105    105    256    7    7    1 :  252  126
size-128(DMA)          0      0    128    0    0    1 :  252  126
size-128             510    510    128   17   17    1 :  252  126
size-64(DMA)           0      0     64    0    0    1 :  252  126
size-64              177    177     64    3    3    1 :  252  126
size-32(DMA)           0      0     32    0    0    1 :  252  126
size-32              565    565     32    5    5    1 :  252  126
```

* Each of the column fields correspond to a field in
[struct kmem_cache_s][kmem_cache_s]. Looking at each one in order:

1. `cache-name` - A human-readable name e.g. 'tcp_bind_bucket'

2. `num-active-objs` - Number of objects that are in use.

3. `total-objs` - Number of available objects, including unused.

4. `obj-size` - The size of each object, typically small.

5. `num-active-slabs` - Number of slabs containing objects that are active.

6. `total-slabs` - Total number of slabs.

7. `num-pages-per-slab` - The pages required to create one slab, typically 1.

8. `limit` (SMP systems, after the colon) - The number of free objects the pool
   can have before half of it is given to the global free pool.

9. `batchcount` (SMP systems, after the colon) - The number of objects allocated
   for the processor in a block when no objects are free.

* Columns 8 and 9 are SMP-specific (shown after the colon in the
  `/proc/slabinfo` output) and refer to the per-CPU cache which is explored
  further in 8.5.

* To speed allocation and freeing of objects and slabs, they are arranged into 3
  lists - `slabs_full`, `slabs_partial` and `slabs_free`. Entries in
  `slabs_full` have all of their objects in use. Entries in `slabs_partial` have
  free objects available so is a prime candidate for object allocation and
  finally `slabs_free` entries have no allocated objects, so is a candidate for
  slab destruction.

#### 8.1.1 Cache Descriptor

* All information describing a cache is stored in a
  [struct kmem_cache_s][kmem_cache_s]. This is a rather large data structure:

```c
struct kmem_cache_s {
/* 1) each alloc & free */
        /* full, partial first, then free */
        struct list_head        slabs_full;
        struct list_head        slabs_partial;
        struct list_head        slabs_free;
        unsigned int            objsize;
        unsigned int            flags;  /* constant flags */
        unsigned int            num;    /* # of objs per slab */
        spinlock_t              spinlock;
#ifdef CONFIG_SMP
        unsigned int            batchcount;
#endif

/* 2) slab additions /removals */
        /* order of pgs per slab (2^n) */
        unsigned int            gfporder;

        /* force GFP flags, e.g. GFP_DMA */
        unsigned int            gfpflags;

        size_t                  colour;         /* cache colouring range */
        unsigned int            colour_off;     /* colour offset */
        unsigned int            colour_next;    /* cache colouring */
        kmem_cache_t            *slabp_cache;
        unsigned int            growing;
        unsigned int            dflags;         /* dynamic flags */

        /* constructor func */
        void (*ctor)(void *, kmem_cache_t *, unsigned long);

        /* de-constructor func */
        void (*dtor)(void *, kmem_cache_t *, unsigned long);

        unsigned long           failures;

/* 3) cache creation/removal */
        char                    name[CACHE_NAMELEN];
        struct list_head        next;
#ifdef CONFIG_SMP
/* 4) per-cpu data */
        cpucache_t              *cpudata[NR_CPUS];
#endif
#if STATS
        unsigned long           num_active;
        unsigned long           num_allocations;
        unsigned long           high_mark;
        unsigned long           grown;
        unsigned long           reaped;
        unsigned long           errors;
#ifdef CONFIG_SMP
        atomic_t                allochit;
        atomic_t                allocmiss;
        atomic_t                freehit;
        atomic_t                freemiss;
#endif
#endif
};
```

* Looking at each field in turn:

1. `slabs_[full, partial, free]` - These are the three lists where the slabs are
   stored as discussed above.

2. `objsize` - Size of each object packed into the slab.

3. `flags` - Flags that determine how parts of the allocator will behave when
   dealing with the cache. Discussed in 8.1.2

4. `num` - This is the number of objects contained in each slab.

5. `spinlock` - Protects the structure from concurrent accesses.

6. `batchcount` - Number of objects that will be allocated in batch for the
   per-cpu caches, as discussed above.

7. `gfporder` - This indicates the size of the slab in pages. Each slab consumes
   `2^gfporder` pages, because these are the allocation sizes that the buddy
   allocator provides.

8. `gfpflags` - The GFP flags used when calling the buddy allocator to allocate
   pages. See 6.4 for more details.

9. `colour` - The number of different offsets that can be used for cache
   colouring (cache colouring is discussed further in 8.1.5.)

10. `colour_off` - The byte alignment to keep slabs at. For example, slabs for
    the size-X caches are aligned on the L1 cache.

11. `colour_next` - The next colour line to use. Wraps back to 0 when it reaches
    `colour`.

12. `slabp_cache` - Cache for off-slab slab management objects (see 8.2.1.)

13. `growing` - Set to indicate if the cache is growing or not. If so, it is
    much less likely that this cache will be selected to reap free slabs under
    memory pressure.

14. `dflags` - Dynamic flags that change during cache lifetime (see 8.1.3.)

15. `ctor` - Callback that can be called to initialise each new object, defaults
    to NULL meaning no such function is called.

16. `dtor` - Similar to `ctor` but for object destruction.

17. `failures` - Unused - Set to 0 then ignored. We have no failures bitches!

18. `name` - Human-readable name for the cache.

19. `next` - [struct list_head][list_head] field for next cache on the cache
    line.

20. `cpudata` - Per-CPU data. Discussed further in 8.5.

21. `num_active` - (debug) - The current number of active objects

22. `num_allocations` - (debug) - Running total of the number of objects that
    have been allocated on this cache.

23. `high_mark` - (debug) - The highest value `num_active` has had, to date.

24. `grown` - (debug) - The number of times [kmem_cache_grow()][kmem_cache_grow]
    has been called.

25. `reaped` - (debug) - Number of times this cache has been reaped.

26. `errors` - (debug, unused)

27. `allochit` - (debug) - Total number of times an allocation has used the
    per-CPU cache.

28. `allocmiss` - (debug) - Total number of times an allocation has missed the
    per-CPU cache.

29. `freehit` - (debug) - Total number of times a freed object was placed on a
    per-CPU cache.

30. `freemiss` - (debug) - Total number of times an object was freed and placed
    in the global pool.

* The statistical fields (`num_active`, etc. behind the `#if STATS`) are only
  available if `CONFIG_SLAB_DEBUG` is enabled. `/proc/slabinfo` does not rely on
  these being present, rather it calculates the values when the proc entry is
  read by another process by examining every slab used by each cache.

#### 8.1.2 Cache Static Flags

* A number of flags are set at cache creation time that remain the same for the
  lifetime of the cache and affect how the slab is structured and how objects
  are stored within it.

* All of the flags are stored in a bitmask in the `flags` field of the cache
  descriptor.

* Looking at each of the internal flags [defined in mm/slab.c][slab-cflgs]:

1. `CFLGS_OFF_SLAB` - Indicates that the slab managers for this cache are kept
   off-slab. Discussed further in 8.2.1.

2. `CFLGS_OPTIMIZE` - Only set, but never used. We always optimise, baby!

* Looking at each of the flags that are set by the cache creator:

1. `SLAB_HWCACHE_ALIGN` - Align objects to the L1 CPU cache.

2. `SLAB_MUST_HWCACHE_ALIGN` - Force alignment to the L1 CPU cache even if it is
   very wasteful or slab debugging is enabled.

3. `SLAB_NO_REAP` - Never reap slabs in this cache.

4. `SLAB_CACHE_DMA` - Allocate slabs with memory from `ZONE_DMA`.

* Finally, let's consider some debug flags, which require `CONFIG_SLAB_DEBUG` to
  be set:

1. `SLAB_DEBUG_FREE` - Perform expensive checks on free.

2. `SLAB_DEBUG_INITIAL` - On free, call the constructor as a verifier to ensure
   the object is still initialised correctly.

3. `SLAB_RED_ZONE` - Places a marker at either end of objects to trap overflows.

4. `SLAB_POISON` - Poison objects with a known pattern for trapping changes made
   to objects not allocated or initialised (see above)

* To prevent callers from using the wrong flags, a [CREATE_MASK][CREATE_MASK] is
  provided that 'masks in' all the allowable flags. When a cache is being
  created, the requested flags are compared against the `CREATE_MASK` and
  reported as a bug if invalid flags are used.

#### 8.1.3 Cache Dynamic Flags

* The `dflags` field has only one flag - `DFLGS_GROWN` - but it is important. It
  is set during [kmem_cache_grow()][kmem_cache_grow] so that
  [kmem_cache_reap()][kmem_cache_reap] will be unlikely to choose the cache for
  reaping.

* When `kmem_cache_reap()` finds a cahce with `DFLGS_GROWN` set it skips it,
  removing the flag int he process.

#### 8.1.4 Cache Allocation Flags

* These flags correspond to the GFP page flag options for allocating pages for
  slabs. Some callers use `GFP_...` and `SLAB_...` flags interchangeably,
  however this is incorrect and only `SLAB_...` flags should be used:

1. `SLAB_ATOMIC`
2. `SLAB_DMA`
3. `SLAB_KERNEL`
4. `SLAB_NFS`
5. `SLAB_NOFS`
6. `SLAB_NOHIGHIO`
7. `SLAB_NOIO`
8. `SLAB_USER`

* A small number of flags may be passed to constructor and destructor functions:

1. `SLAB_CTOR_CONSTRUCTOR` - Set if the function is being called as a
   constructor for caches that use the same function for constructors and
   destructors.

2. `SLAB_CTOR_ATOMIC` - Indicates that the constructor may not sleep.

3. `SLAB_CTOR_VERIFY` - Indicates that the constructor should just verify that
   the object is initialised properly.

#### 8.1.5 Cache Colouring

* As discussed previously, the slab allocator attempts to better make use of the
  hardware cache by offsetting objects in different slabs by differing amounts
  depending on the amount of space left over in the slab.

* The offset is in units of `BYTES_PER_WORD` unless the `SLAB_HWCACHE_ALIGN`
  flag is set, in which case it is aligned to blocks of `L1_CACHE_BYTES` for
  alignment to the L1 hardware cache.

* During cache creation the number of objects that can fit on a slab (see 8.2.7)
  and how many bytes would be wasted are calculated. Based on wastage, 2 figures
  are calculated for the cache descriptor:

1. `colour` - The number of different offsets that can be used.

2. `colour_off` - The multiple to offset each object in the slab.

* With the objects offset, they will use different lines on the associative
  hardware cache. This makes it less likely for objects from slabs to cause
  flushes for another object.

* Consider an example - Let's say that `s_mem` (the address of the first object
  on the slab) is 0, alignment is to 32 bytes for the L1 hardware cache, and 100
  bytes are wasted on the slab.

* In this scenario, the first slab created will have its objects start at 0. The
  second will start at 32, the third at 64, the fourth at 96 and the fifth will
  start back at 0.

* This makes it unlikely objects from each of the slabs will hit the same
  hardware cache line on the CPU.

* In this example, `colour` would be 3 (there are 3 offsets - 32, 64, 96), and
`colour_off` would be 32.

#### 8.1.6 Cache Creation

* The function [kmem_cache_create()][kmem_cache_create] is responsible for
  creating new caches and adding them to the cache chain. The tasks that are
  performed to create a cache are:

1. Perform basic sanity checks to avoid bad usage.

2. Perform debugging checks if `CONFIG_SLAB_DEBUG` is set.

3. Allocate a [kmem_cache_t][kmem_cache_s] (typedef'd from a
   [struct kmem_cache_s][kmem_cache_s]) from the [cache_cache][cache_cache] slab
   cache.

4. Align the object size to the word size.

5. Calculate how many objects will fit on a slab.

6. Align the object to the hardware cache.

7. Calculate colour offsets.

8. Initialise remaining fields in the cache descriptor.

9. Add the new cache to the cache chain.

#### 8.1.7 Cache Reaping

* When a slab is freed, it is placed on the `slabs_free` list for future
  use. Caches do not automatically shrink themselves, so, when `kswapd` notices
  memory is tight, it calls [kmem_cache_reap()][kmem_cache_reap] to free some
  memory.

* `kmem_cache_reap` is responsible for for selecting a cache which will be
  required to shrink its memory usage.

* Note that cache reaping does not take into account what memory node or zone is
  under pressure. This means that with a NUMA or high memory machine it is
  possible the kernel will spend a lot of time freeing memory from regions that
  are under no memory pressure. This is not a problem for an architecure like
  i386 which has only 1 bank of memory.

* In the event that the system has a large number of caches, only
  [REAP_SCANLEN][REAP_SCANLEN] (10 by default) caches are examined in each call.

* The last cache to be scanned is stored in the variable
  [clock_searchp][clock_searchp] to avoid examining the same caches repeatedly.

* For each scanned cache, the reaper:

1. Checks flags for `SLAB_NO_REAP` and skip if set.

2. If the cache is growing, skip it.

3. If the cache has grown recently, or is currently growing, `DFLGS_GROWN` will
   be set. If this flag is set, the slab is skipped, but the flag is cleared so
   the cache will be a reap candidate the next time round.

4. Count the number of free slabs in the `slabs_free` field and calculate how
   many pages reaping this cache would free, storing in the local variable
   `pages`.

5. If the cache has constructors or large slabs, adjust `pages` to make it less
   likely for the cache to be selected.

6. If the number of pages that would be free exceeds
   [REAP_PERFECT][REAP_PERFECT] (defaults to 10), free half of the slabs in
   `slabs_free`.

7. Otherwise, scan the rest of the caches and select the one that would free the
   most pages if the function was to free half of the slabs in `slabs_free`.

#### 8.1.8 Cache Shrinking

* When a cache is selected to shrink itself, the steps it takes are simple and
  brutal:

1. Delete all objects in the per-CPU caches.

2. Delete all slabs from the `slabs_free` field unless the growing flag gets
   set.

* Linux is nothing, if not subtle :) Two varieties of shrink functions are
  provided with confusingly similar names -
  [kmem_cache_shrink()][kmem_cache_shrink] and
  [__kmem_cache_shrink()][__kmem_cache_shrink].

* `kmem_cache_shrink()` removes all slabs from the cache's `slabs_free` field
  and returns the number of pages freed as a result - it's the principle
  shrinking function exported for use by slab allocator users.

* `__kmem_cache_shrink()` frees all slabs from the cache's `slabs_free` field
  and then verifies that `slabs_partial` and `slabs_full` are empty. This is for
  internal use _only_, and used during cache destruction when it doesn't matter
  how many pages are freed, just that the cache is empty.

#### 8.1.9 Cache Destroying

* When a module is unloaded, it is responsible for destroying any cache via
  [kmem_cache_destroy()][kmem_cache_destroy]. It is vital that the cache is
  properly destroyed because two caches of the same human-readable name are not
  allowed to exist.

* Core kernel systems often don't bother to destroy its caches because they last
  for the entire life of the system.

* The steps taken to destroy a cache are:

1. Delete the cache from the cache chain.

2. Shrink the cache to delete all slabs.

3. Free any per-CPU caches via [kfree()][kfree].

4. Delete the cache descriptor from the [cache_cache][cache_cache].

### 8.2 Slabs

* [struct slab_s][slab_s] (typedef'd to `slab_t`) describes a slab. It is much
  simpler than the cache descriptor (though the way it's arranged is
  considerably more complex):

```c
/*
 * slab_t
 *
 * Manages the objs in a slab. Placed either at the beginning of mem allocated
 * for a slab, or allocated from an general cache.
 * Slabs are chained into three list: fully used, partial, fully free slabs.
 */
typedef struct slab_s {
        struct list_head        list;
        unsigned long           colouroff;
        void                    *s_mem;         /* including colour offset */
        unsigned int            inuse;          /* num of objs active in slab */
        kmem_bufctl_t           free;
} slab_t;
```

* Looking at each field:

1. `list` - The linked list the slab belongs to - it'll be one of
   `slab_[full|partial|free]` from the cache manager.

2. `colouroff` - The colour offset from the base address of the first object
   within the slab. The address of the first object is therefore `s_mem +
   colouroff`.

3. `s_mem` - The starting address of the first object within the slab.

4. `inuse` - The number of active objects in the slab.

5. `free` - The index of the next free object in the array of
   [kmem_bufctl_t][kmem_bufctl_t]'s (`kmem_bufctl_t` is typedef'd to `unsigned
   int`) that starts _after_ the end of `slab_t` (`slab_t` is always kept at the
   start of a page frame so there is space for more data afterwards.) Discussed
   further in 8.2.3.

* There is no obvious way to determine which cache a slab belongs to or
  vice-versa. It turns out that we can use the `list` field of the
  [struct page][page] (see 2.5 - the `list` field is a generic field that can be
  used for different purposes, which is quite surprising :)

* In order to set/get the page's cache/slab you can use the
  [SET_PAGE_CACHE()][SET_PAGE_CACHE], [SET_PAGE_SLAB()][SET_PAGE_SLAB],
  [GET_PAGE_CACHE()][GET_PAGE_CACHE], and [GET_PAGE_SLAB()][GET_PAGE_SLAB]
  macros which simply place caches in the `next` field of the list and slabs in
  the `prev` field:

```c
/* Macros for storing/retrieving the cachep and or slab from the
 * global 'mem_map'. These are used to find the slab an obj belongs to.
 * With kfree(), these are used to find the cache which an obj belongs to.
 */
#define SET_PAGE_CACHE(pg,x)  ((pg)->list.next = (struct list_head *)(x))
#define GET_PAGE_CACHE(pg)    ((kmem_cache_t *)(pg)->list.next)
#define SET_PAGE_SLAB(pg,x)   ((pg)->list.prev = (struct list_head *)(x))
#define GET_PAGE_SLAB(pg)     ((slab_t *)(pg)->list.prev)
```

* Diagrammatically (for a case where a slab is on the `slabs_partial` list):

```
      struct kmem_cache_s (=kmem_cache_t)
  /-->-----------------------------------
  |   |   struct list_head *slabs_full  |
  |   |---------------------------------|
  |   | struct list_head *slabs_partial |-------------\
  |   |---------------------------------|             |
  |   /                                 /             v
  |   \               ...               \      ---------------
  |   /                                 /      | Linked list | More slab_t's...
  |   -----------------------------------      |  of slab_t  |- - >
  |                                            ---------------
  |                                                   |
  |                                                   |
  |                                                   |
  |               struct page                         v
  |   next ------------------------- prev        ----------
  \--------| struct list_head list |------------>| slab_t |
           |-----------------------|             ----------
           /                       /
           \          ...          \
           /                       /
           |-----------------------|
           |   flags with PG_slab  |
           |        bit set        |
           |-----------------------|
           /                       /
           \          ...          \
           /                       /
           -------------------------
```

* The last issue is where the slab management data structure [slab_t][slab_s] is
  kept. They are either kept on-slab or off-slab depending on whether
  `CFLGS_OFF_SLAB` is set in the static flags.

* The size of the object determines whether `slab_t` is stored on or
  off-slab. In the above diagram it's possible for `slab_t` to be kept at the
  beginning of the page frame, though the diagram implies it is kept separate,
  this is not necessarily the case.

#### 8.2.1 Storing the Slab Descriptor

* If the objects are _larger_ than a threshold ([PAGE_SIZE>>3][cflgs-off-set] so
  512 bytes on i386), `CFLGS_OFF_SLAB` is set in the cache flags and the slab
  descriptor/manager/management data structure is kept off-slab in one of the
  'sizes caches' (see 8.4.)

* The selected cache will be large enough to contain the [slab_t][slab_s] and
  [kmem_cache_slabmgmt()][kmem_cache_slabmgmt] allocates from it as necessary.

* Keeping the `slab_t` off-slab limits the number of objects that can be stored
  on the slab because there is limited space for the
  [kmem_bufctl_t][kmem_bufctl_t]'s (see 8.2.3) - however that shouldn't matter
  because the objects are large so there should be fewer stored on a single
  slab.

* If the slab manager is kept on-slab it is reserved at the beginning of the
  slab. Enough space is kept to store the `slab_t` and the `kmem_bufctl_t`s.

* The `kmem_bufctl_t`s are an array of unsigned integers used for tracking the
  index of the next free object available for use, see 8.2.3 for more details.

* The actual objects in the slab are stored after the `kmem_bufctl_t` array.

* A slab with on-slab descriptor, diagrammatically (again, assuming the slab
  we're looking at lives in `slabs_partial`):

```
struct kmem_cache_s (=kmem_cache_t)
-----------------------------------
|   struct list_head *slabs_full  |
|---------------------------------|
| struct list_head *slabs_partial |-------------\
|---------------------------------|             |
/                                 /             v
\               ...               \      ---------------
/                                 /      | Linked list | More slab_t's...
-----------------------------------      |  of slab_t  | - - >
                                         ---------------
                                                |
                                                |
   /--------------------------------------------/
   |
   |
   v        Page frame
   ---------------------------
   | struct list_head list   |
   |-------------------------|
   | unsigned long colouroff |
   |-------------------------|
/--|       void *s_mem       |
|  |-------------------------|
|  |   unsigned int inuse    |
|  |-------------------------|
|  |    kmem_bufctl_t free   |
|  |-------------------------|
|  /                         /
|  \  kmem_bufctl_t array... \
|  /                         /
\->|-------------------------|
   |         Object          |
   |-------------------------|
   |         Object          |
   |-------------------------|
   |         Object          |
   |-------------------------|
   /                         /
   \       More objects...   \
   /                         /
   ---------------------------
```

* A slab with off-slab descriptor, diagrammatically (again, assuming the slab
  we're looking at lives in `slabs_partial`):

```
struct kmem_cache_s (=kmem_cache_t)
-----------------------------------
|   struct list_head *slabs_full  |
|---------------------------------|
| struct list_head *slabs_partial |-------------\
|---------------------------------|             |
/                                 /             v
\               ...               \      ---------------
/                                 /      | Linked list | More slab_t's...
-----------------------------------      |  of slab_t  | - - >
                                         ---------------
                                                |
                                                |
   /--------------------------------------------/
   |
   |
   v  Page frame for slab_t                                   Page frame for objects
   ---------------------------               /------------->---------------------------
   | struct list_head list   |               |              |         Object          |
   |-------------------------|               |              |-------------------------|
   | unsigned long colouroff |               |              |         Object          |
   |-------------------------|               |              |-------------------------|
   |       void *s_mem       |---------------/              |         Object          |
   |-------------------------|                              |-------------------------|
   |   unsigned int inuse    |                              /                         /
   |-------------------------|                              \       More objects...   \
   |    kmem_bufctl_t free   |                              /                         /
   |-------------------------|                              ---------------------------
   /                         /
   \  kmem_bufctl_t array... \
   /                         /
   ---------------------------
```

#### 8.2.2 Slab Creation

* So far we've seen how the cache is created, but not how the `slabs_full`,
  `slabs_partial` and `slabs_free` lists are grown, as they all start empty -
  i.e., how slabs are allocated.

* New slabs are allocated to a cache via [kmem_cache_grow()][kmem_cache_grow]
  and is referred to as 'cache growing'.

* Cache growing occurs when no objects are left in the `slabs_partial` list and
  when there are no slabs in `slabs_free`.

* `kmem_cache_grow()` does the following:

1. Performs basic sanity checks to guard against bad usage.

2. Calculates the colour offset for objects in the slab.

3. Allocates memory for the slab and acquires a slab descriptor.

4. Links the pages used for the slab to the slab and cache descriptors (see 8.2.)

5. Initialises objects in the slab.

6. Adds the slab to the cache.

#### 8.2.3 Tracking Free Objects

* The slab allocator needs to have a quick and simple means of tracking where
  free objects are located on the partially-filled slabs. This is achieved by
  using an array of unsigned integers of type [kmem_bufctl_t][kmem_bufctl_t]
  that is associated with each 'slab manager'.

* The number of objects in the array is the same as the number of objects on the
  slab.

* The array of `kmem_bufctl_t`s is kept _after_ the slab descriptor in the page
  frame in which it is kept. In order to make life easier there is a macro which
  gives access to the array:

```c
#define slab_bufctl(slabp) \
        ((kmem_bufctl_t *)(((slab_t*)slabp)+1))
```

* The `free` field of the slab management structure contains the index of the
  next free object in the slab. When objects are allocated or freed, this is
  updated based on information in the `kmem_bufctl_t` array.

#### 8.2.4 Initialising the kmem_bufctl_t Array

* When a cache is grown, all the objects and the [kmem_bufctl_t][kmem_bufctl_t]
  array on the slab are initialised.

* The array is filled with the index of each object begining with 1 and ending
  with the marker [BUFCTL_END][BUFCTL_END], e.g. for a slab with five objects:

```
------------------------------
| 1 | 2 | 3 | 4 | BUFCTL_END |
------------------------------
```

* The value 0 is stored in `slab_t->free` because the 0th object is the first
  free object to be used. The idea here is that for a given object `i`, the
  index of the next free object will be stored in `slab_bufctl(slabp)[i]`.

* Using the array this way makes it act like a LIFO (stack) for free objects.

#### 8.2.5 Finding the Next Free Object

* When allocating an object, [kmem_cache_alloc()][kmem_cache_alloc] updates the
  `kmem_bufctl_t` array by calling
  [kmem_cache_alloc_one_tail()][kmem_cache_alloc_one_tail].

* As discussed in 8.2.4, `slab_t->free` contains the index of the first free
  object and the index of the next free object is at
  `slab_bufctl(slabp)[slab_t->free]`. The code for navigating this (taken from
  `kmem_cache_alloc_one_tail()`) looks like:

```c
void *objp;

...

objp = slabp->s_mem + slabp->free*cachep->objsize;
slabp->free=slab_bufctl(slabp)[slabp->free];
```

* This is in effect 'popping' an index off the stack.

* Note that the `kmem_bufctl_t` array is not changed during allocations, just
  updated on allocation/free. LS - there is some really confusing stuff here
  about _unallocated_ elements being unreachable (after allocations?), skipping
  as I think there may be a mistake here.

#### 8.2.6 Updating kmem_bufctl_t

* The `kmem_bufctl_t` list is only updated when an object is freed via
  [kmem_cache_free_one()][kmem_cache_free_one]:

```c
unsigned int objnr = (objp-slabp->s_mem)/cachep->objsize;

slab_bufctl(slabp)[objnr] = slabp->free;
slabp->free = objnr;
```

* This is in effect 'pushing' an index back on to the stack.

#### 8.2.7 Calculating the Number of Objects on a Slab

* During cache creation [kmem_cache_estimate()][kmem_cache_estimate] is called
  to calculate how many objects may be stored on a single slab, taking into
  account whether the slab descriptor is to be stored on or off-slab and the
  size taken up by each `kmem_bufctl_t` needed to track whether an object is
  free or not.

* `kmem_cache_estimate()` returns the number objects that may be stored and how
  many bytes are wasted. The number of wasted bytes is important to know for
  cache colouring.

* The calculation is fairly straightforward:

```c
/* Cal the num objs, wastage, and bytes left over for a given slab size. */
static void kmem_cache_estimate (unsigned long gfporder, size_t size,
                 int flags, size_t *left_over, unsigned int *num)
{
        int i;
        size_t wastage = PAGE_SIZE<<gfporder;
        size_t extra = 0;
        size_t base = 0;

        if (!(flags & CFLGS_OFF_SLAB)) {
                base = sizeof(slab_t);
                extra = sizeof(kmem_bufctl_t);
        }
        i = 0;
        while (i*size + L1_CACHE_ALIGN(base+i*extra) <= wastage)
                i++;
        if (i > 0)
                i--;

        if (i > SLAB_LIMIT)
                i = SLAB_LIMIT;

        *num = i;
        wastage -= i*size;
        wastage -= L1_CACHE_ALIGN(base+i*extra);
        *left_over = wastage;
}
```

* Step-by-step:

1. Initialise `wastage` to the total size of the slab - `2^gfp_order`.

2. Subtract the amount of space needed to store the slab descriptor, if this
   is kept on-slab.

3. Count the number of objects that may be stored, taking into account the
   `kmem_bufctl_t` required for each object if kept on-slab. Keep incrementing
   `i` until the slab is filled.

4. Return the number of objects and bytes wasted.

#### 8.2.8 Slab Destroying

* When a cache is being shrunk or destroyed, the slabs will be deleted and if
  the objects have destructors these must be called.

* For example, [kmem_slab_destroy()][kmem_slab_destroy] performs the following
  steps:

1. Call the destructors for every object in the slab.

2. If debugging is enabled - check the red marking and poison pattern.

3. Free the pages the slab uses.

### 8.3 Objects

* Most of the really hard work of the slab allocator is handled by the cache and
  slab managers. Now we take a look at how objects themselves are managed.

#### 8.3.1 Initialising Objects in a Slab

* When a slab is created, all the objects in it are put in an initialised
  state. If a constructor is available, it is called for each object and it is
  expected that objects are left in an initialised state upon free.

* Conceptually, the initialisation is very simple - cycle through each object,
  call its constructor and initialise the `kmem_bufctl` for it. This performed
  by [kmem_cache_init_objs()][kmem_cache_init_objs].

#### 8.3.2 Object Allocation

* [kmem_cache_alloc()][kmem_cache_alloc] (and thus
  [__kmem_cache_alloc()][__kmem_cache_alloc]) is responsible for allocating one
  object to the specified cache:

```c
static inline void * __kmem_cache_alloc (kmem_cache_t *cachep, int flags)
{
        unsigned long save_flags;
        void* objp;

        kmem_cache_alloc_head(cachep, flags);
try_again:
        local_irq_save(save_flags);
#ifdef CONFIG_SMP
        {
                cpucache_t *cc = cc_data(cachep);

                if (cc) {
                        if (cc->avail) {
                                STATS_INC_ALLOCHIT(cachep);
                                objp = cc_entry(cc)[--cc->avail];
                        } else {
                                STATS_INC_ALLOCMISS(cachep);
                                objp = kmem_cache_alloc_batch(cachep,cc,flags);
                                if (!objp)
                                        goto alloc_new_slab_nolock;
                        }
                } else {
                        spin_lock(&cachep->spinlock);
                        objp = kmem_cache_alloc_one(cachep);
                        spin_unlock(&cachep->spinlock);
                }
        }
#else
        objp = kmem_cache_alloc_one(cachep);
#endif
        local_irq_restore(save_flags);
        return objp;
alloc_new_slab:
#ifdef CONFIG_SMP
        spin_unlock(&cachep->spinlock);
alloc_new_slab_nolock:
#endif
        local_irq_restore(save_flags);
        if (kmem_cache_grow(cachep, flags))
                /* Someone may have stolen our objs.  Doesn't matter, we'll
                 * just come back here again.
                 */
                goto try_again;
        return NULL;
}
```

* This function consists of 4 basic steps along with a 5th if the system is SMP:

1. Calls [kmem_cache_alloc_head()][kmem_cache_alloc_head] to perform some basic
   sanity checks (i.e. whether the allocation is allowable.)

2. (SMP) If there is an object available on the per-CPU cache, return it,
   otherwise allocate `cachep->batchcount` objects in bulk and put them on the
   per-CPU cache (see 8.5 for more details on per-CPU caches.)

3. Select which slab to allocate from - either `slabs_partial` or `slabs_free`.

4. If `slabs_free` is chosen and has no available slabs, grow the cache (see
   8.2.2.).

5. Allocate the object from the selected slab.

#### 8.3.3 Object Freeing

* [kmem_cache_free()][kmem_cache_free] frees objects and is relatively
  straightforward - differing only depending on whether the system is SMP or UP.

* If the system is SMP, the object is returned to the per-CPU cache.

* Regardless of whether the system is SMP or not, the destructor for an object
  will be called if one is available. It is responsible for returning the object
  to its initialised state.

### 8.4 Sizes Cache

* Linux keeps two sets of slab caches for small memory allocations for which the
  physical page allocator is unsuitable.

* One set is for use with DMA, and the other is suitable for normal use. The
  human-readable names for these caches are 'size-N cache' and 'size-N(DMA)
  cache' which are viewable from `/proc/slabinfo`.

* Each sized cache is stored in [struct cache_sizes][cache_sizes_t] (typedef'd to
  `cache_sizes_t`):

```c
/* Size description struct for general caches. */
typedef struct cache_sizes {
        size_t          cs_size;
        kmem_cache_t    *cs_cachep;
        kmem_cache_t    *cs_dmacachep;
} cache_sizes_t;
```

* Looking at each field:

1. `cs_size` - The size of the memory block.

2. `cs_cachep` - The cache of blocks for normal memory use.

3. `cs_dmacachep` - The cache of blocks for use with DMA.

* Because a limited number of these caches exist, a null-terminated static
  array, [cache_sizes][cache_sizes], is initialised at compile time, starting
  with 32 bytes on a 4KiB machine or 64 bytes for those systems with larger page
  sizes, and reaching `2^17` (131KiB):

```c
static cache_sizes_t cache_sizes[] = {
#if PAGE_SIZE == 4096
        {    32,        NULL, NULL},
#endif
        {    64,        NULL, NULL},
        {   128,        NULL, NULL},
        {   256,        NULL, NULL},
        {   512,        NULL, NULL},
        {  1024,        NULL, NULL},
        {  2048,        NULL, NULL},
        {  4096,        NULL, NULL},
        {  8192,        NULL, NULL},
        { 16384,        NULL, NULL},
        { 32768,        NULL, NULL},
        { 65536,        NULL, NULL},
        {131072,        NULL, NULL},
        {     0,        NULL, NULL}
};
```

* This array is initialised via
  [kmem_cache_sizes_init()][kmem_cache_sizes_init] at startup.

#### 8.4.1 kmalloc()

* Given the existence of this cache, the slab allocator is now able to offer a
  new function, [kmalloc()][kmalloc], for use when small memory buffers are
  required.

* When a request is received, the appropriate sizes cache is selected and an
  object is assigned from it.

* The hard work was done in object/cache/slab allocation, so `kmalloc()` itself
  is fairly simple.

#### 8.4.2 kfree()

* Since there's a `kmalloc()` there needs to be an equivalent free function,
  [kfree()][kfree].

* As with `kmalloc()` the hard work is done in object/cache/slab freeing, so the
  `kfree()` function itself is fairly simple too.

### 8.5 Per-CPU Object Cache

* One of the tasks of the slab allocator is to improve hardware cache use. One
  important aspect of doing this is to try to use data on the same CPU for as
  long as possible.

* Linux achieves this by trying to keep objects in the same CPU cache via a
  per-CPU object cache (known as a 'cpucache') for each CPU in the system.

* When allocating or freeing object on SMP systems, they are taken from/placed
  in the cpucache. When no objects are free on allocation (see 8.3.2), a batch
  of objects is placed into the pool.

* When the cpucache pool gets too large, half of them are removed and placed
  into the global cache. This ensures the hardware cache will be used as long as
  is possible on the same CPU.

* Another major benefit is that spinlocks do not have to be held when accessing
  the CPU pool because we are guaranteed that another CPU won't access the local
  data. This is important because without the caches, spinlocks would have to be
  acquired for every allocation and free which would be very expensive.

#### 8.5.1 Describing the Per-CPU Object Cache

* Each cache descriptor has a pointer to an array of cpucache's
  ([struct cpucache_s][cpucache_s], typedef'd to `cpucache_t`), described in the
  cache descriptor as `cpucache_t *cpudata[NR_CPUS];` (where [NR_CPUS][NR_CPUS]
  is the _maximum_ number of CPUs a system might have.)

* The `cpucache_t` structure is simple:

```c
typedef struct cpucache_s {
        unsigned int avail;
        unsigned int limit;
} cpucache_t;
```

* Looking at each field:

1. `avail` - The number of free objects available on this cpucache.

2. `limit` - The total number of free objects that can exist.

* A helper macro [cc_data()][cc_data] is available to retrieve the cpucache for
  a given cache and processor:

```c
#define cc_data(cachep) \
        ((cachep)->cpudata[smp_processor_id()])
```

* Given a cache descriptor `cachep` this returns the appropriate pointer from
  the cpucache array `cpudata` via [smp_processor_id()][smp_processor_id].

* Pointers on the cpucache are placed immediately after the `cpucache_t` struct,
  similar to how objects are stored after a slab descriptor in the on-slab case.

#### 8.5.2 Adding/Removing Objects From the Per-CPU Cache

* To prevent fragmentation, objects are always added or removed from the end of
the array.

* [cc_entry()][cc_entry] returns a pointer to the first object in the
  cpucache. It is similar in operation to [slab_bufctl()][slab_bufctl]:

```c
#define cc_entry(cpucache) \
        ((void **)(((cpucache_t*)(cpucache))+1))
```

* To use `cc_entry()` to add an object to the CPU cache (`cc`):

```c
void *objp;

...

cc_entry(cc)[cc->avail++] = objp;
```

* To use `cc_entry()` to remove an object from the CPU cache (`cc`):

```c
void *objp;

...

objp = cc_entry(cc)[--cc->avail];
```

#### 8.5.3 enabling Per-CPU Caches

* When a cache is created, its CPU cache has to be enabled and memory has to be
  allocated to it via [kmalloc()][kmalloc], this is performed by
  [enable_cpucache()][enable_cpucache].

* `enable_cpucache()` calls [kmem_tune_cpucache()][kmem_tune_cpucache] to
  allocate memory for each CPU cache.

* A CPU cache cannot exist until the various sizes caches have been enabled, so
  a global variable [g_cpucache_up][g_cpucache_up] is used to prevent CPU caches
  from being enabled prematurely.

* The function [enable_all_cpucaches()][enable_all_cpucaches] cycles through all
  caches in the cache chain and enables their cpucache and is called via
  [kmem_cpucache_init()][kmem_cpucache_init].

* After the CPU cache has been set up it can be accessed without locking because
  a CPU will never access the wrong cpucache, so it is guaranteed safe access to
  it.

#### 8.5.4 Updating Per-CPU Information

* When the per-cpu caches have been created or changed, each CPU is signalled by
  an [IPI][ipi]. It's not sufficientt o change all the values in the cache
  descriptor because that would lead to cache coherency issues and we'd be back
  having to use spinlocks.

* A [struct ccupdate_struct_s][ccupdate_struct_s] (typedef'd to
  `ccupdate_struct_t`) is populated with all the information that each CPU
  needs:

```c
typedef struct ccupdate_struct_s
{
        kmem_cache_t *cachep;
        cpucache_t *new[NR_CPUS];
} ccupdate_struct_t;
```

* Looking at each field:

1. `cachep` - The cache being updated.

2. `new` - An array of the cpucache descriptors for each CPU on the system.

* [smp_call_function_all_cpus()][smp_call_function_all_cpus] is used to get each
  CPU to call [do_ccupdate_local()][do_ccupdate_local] which swaps the
  information from `ccupdate_struct_t` with the information in the cache
  descriptor. After the information is swapped, the old data can be deleted.

#### 8.5.5 Draining a Per-CPU Cache

* When a cache is being shrunk the first step that needs to be performed is
  draining the cpucache's of any object that might have via
  [drain_cpu_caches()][drain_cpu_caches].

* This is done so that the slab allocator will have a clearer view of which
  slabs can be freed or not. It matters because if just one object in a slab is
  placed in a per-CPU cache then the whole slab is not able to be freed.

* If the system is tight on memory, saving a few milliseconds on allocations has
  a low priority.

### 8.6 Slab Allocator Initialisation

* When the slab allocator creates a new cache, it allocates the
  [kmem_cache_t][kmem_cache_s] from the [cache_cache][cache_cache] or
  a `kmem_cache` cache.

* This is chicken-and-egg so the `cache_cache` is statically initialised:

```c
/* internal cache of cache description objs */
static kmem_cache_t cache_cache = {
        slabs_full:     LIST_HEAD_INIT(cache_cache.slabs_full),
        slabs_partial:  LIST_HEAD_INIT(cache_cache.slabs_partial),
        slabs_free:     LIST_HEAD_INIT(cache_cache.slabs_free),
        objsize:        sizeof(kmem_cache_t),
        flags:          SLAB_NO_REAP,
        spinlock:       SPIN_LOCK_UNLOCKED,
        colour_off:     L1_CACHE_BYTES,
        name:           "kmem_cache",
};
```

* Looking at the assignment of each field:

1.  `slabs_[fill|partial|free]` - Initialises the 3 lists.

2. `objsize` - Set to the size of a cache descriptor.

3. `flags` - The creating and deleting of caches is extremely rare, so don't
   even consider it for reaping.

4. `spinlock` - Initialised unlocked.

5. `colour_off` - Aligned to the L1 cache.

6. `name` - A sane human-readable name.

* The code statically defines all the fields that can be determined at
  compile-time, the rest are initialised via
  [kmem_cache_init()][kmem_cache_init] which is called from
  [start_kernel()][start_kernel].

### 8.7 Interfacing with the Buddy Allocator

* The slab allocator uses [kmem_getpages()][kmem_getpages] and
  [kmem_freepages()][kmem_freepages] to interact with the physical page
  allocator to obtain the actual pages of memory it needs.

* `kmem_getpages()` and `kmem_freepages()` are essentially wrappers around the
  buddy allocator API so that slab flags will be taken into account for allocations.

* For allocations, the default flags are taken from `cachep->gfpflags` and the
  order is taken from `cachep->gfporder` (assuming `cachep` is the
  [kmem_cache_t][kmem_cache_s] in question)

* When freeing pages, [PageClearSlab()][PageClearSlab] will be called for every
  page being freed before calling [free_pages()][free_pages].

## Chapter 9: High Memory Management

* The kernel can only directly address memory for which it has set up a page
  table entry. In the common case, the user/kernel address space split of
  3GiB/1GiB implies that, at best, only 896MiB of memory may be directly
  accessed at any given time on a 32-bit machine (see 4.1)

* On 64-bit hardware this is not an issue as there are vast amounts of virtual
  address space, however 32-bit is another story.

* On 32-bit hardware Linux temporarily maps pages from high memory into the
  lower page tables (discussed further in 9.2.)

* In I/O operations, not all devices are able to address high memory or even all
  the memory available to the CPU (consider [PAE][pae].) Asking the device to
  write to memory will fail at best and fuck the kernel at worst. To work around
  this issue we can use a 'bounce buffer' which is discussed in 9.5.

### 9.1 Managing the PKMap Address Space

* The 'Persistent Kernel Map' (PKMap) address space is reserved at the top of
  the kernel page tables from [PKMAP_BASE][PKMAP_BASE] to
  [FIXADDR_START][FIXADDR_START].

* On i386, `PKMAP_BASE` is `0xfe000000`, and `FIXADDR_START` varies depending on
  configuration options, but is typically only a few pages from the end of the
  linear address space - this leaves us around 32MiB of page table space for
  mapping pages from high memory into usable space.

* For mapping pages, a single page set of PTEs is stored at the beginning of the
  PKMap area to allow 1,024 high pages to be mapped into low memory for short
  periods via [kmap()][kmap] and to be unmapped with [kunmap()][kunmap].

* The pool of 1,024 pages (4MiB) seems very small, however pages mapped by
  `kmap()` are only mapped for a _very_ short period of time.

* Comments in the code suggest that there was a plan to allocate contiguous page
  table entries to expand this area however this hasn't been performed in
  2.4.22, so a large portion of the PKMap is unused.

* The page table entry for use with `kmap()` is
  [pkmap_page_table][pkmap_page_table] and is located at
  [PKMAP_BASE][PKMAP_BASE] which is set up during system initialisation (on i386
  this takes place at the end of [pagetable_init()][pagetable_init].

* The pages for the PGD and PMD entries are allocated by the boot memory
  allocator to ensure they exist.

* The state of these page table entries is managed by
  [pkmap_count][pkmap_count], which has [LAST_PKMAP][LAST_PKMAP] entries in it -
  on i386 without PAE this is 1,024 and with PAE it is 512. More accurately,
  though the code doesn't make it clear, `LAST_PKMAP` is equivalent to
  [PTRS_PER_PTE][PTRS_PER_PTE/3lvl].

* Each element in `pkmap_count` is not exactly a reference count, but is very
  close - if the entry is 0, the page is free and has not been used since the
  last TLB flush. If it's 1, the slot is unused, but a page is still mapped
  there waiting for a TLB flush, and finally if it's any higher it has `n - 1`
  users, taking `n` to be the element value.

* Flushes are delayed until every slot has been used at least once because a
  global flush is required for all CPUs when the global page tables are modified
  and this is _extremely_ expensive.

### 9.2 Mapping High Memory Pages

* Let's take a look at the API for mapping pages from high memory:

1. [kmap()][kmap] - Takes a [struct page][page] from high memory and maps it
   into low memory, returning a virtual address of the mapping.

2. [kmap_nonblock()][kmap_nonblock] - Same as `kmap()` only it won't block if
   slots are not available and instead returns `NULL`. This isn't the same as
   `kmap_atomic()` which uses specially reserved slots.

3. [kmap_atomic()][kmap_atomic] - This uses slots maintained in the map for
   atomic use by interrupts (see 9.4 for more details), however their use is
   _highly_ discouraged and callers may not sleep or schedule. You _really_ have
   to require atomic high memory mapping to use this.

* The kmap pool is rather small so it's vital that callers call
  [kunmap()][kunmap] as quickly as possible, as the pressure on this small
  window grows incrementally worse as the size of high memory grows in
  comparison to low memory.

* `kmap()` itself is fairly simple:

1. Check to ensure an interrupt is not calling the function (as it may sleep),
   calling [out_of_line_bug()][out_of_line_bug] if so - we call this rather than
   [BUG()][BUG] because the latter would panic the system if called from an
   interrupt handler (LS - it seems `out_of_line_bug()` calls `BUG()` though??)

2. Check that the page is below [highmem_start_page][highmem_start_page],
   because pages below this level are already visible and do not need to be
   mapped.

3. Check whether the page is already in low memory, if so simply return the
   address. This step is important - it means users can use it unconditionally
   knowing that if it is already a low memory page the function can still be
   used safely.

4. Finally call [kmap_high()][kmap_high] to do the real work.

* `kmap_high()` (and subsequently [map_new_virtual()][map_new_virtual] does the
  following:

1. Check the `page->virtual` field, which is set if the page is already
   mapped. If it's `NULL`, [map_new_virtual()][map_new_virtual] provides a
   mapping for the page.

2. `map_new_virtual()` simply linearly scans [pkmap_count][pkmap_count],
   starting at [last_pkmap_nr][last_pkmap_nr] to avoid searching the same areas
   repeatedly between `kmap()`s.

3. When `last_pkmap_nr` wraps around to 0,
   [flush_all_zero_pkmaps()][flush_all_zero_pkmaps] sets all entries from 1 to 0
   before flushing the TLB.

4. If, after another scan, an entry is _still_ not found, the process sleeps on
   the [pkmap_map_wait][pkmap_map_wait] wait queue until it is woken up after
   the next [kunmap()][kunmap].

5. After a mapping has been created, the corresponding entry in the
   `pkmap_count` array is incremented, and the virtual address in low memory is
   returned.

### 9.3 Unmapping Pages

* Taking a look at the API for unmapping pages from high memory:

1. [kunmap()][kunmap] - Unmaps a [struct page][page] from low memory and frees
   up the page table entry that was mapping it.

2. [kunmap_atomic()][kunmap_atomic] - Unamps a page that was mapped atomically
   via [kmap_atomic()][kmap_atomic].

* `kunmap()`, similar to [kmap()][kmap], checks that the caller is not from
  interrupt context and that the memory is below
  [highmem_start_page][highmem_start_page], before calling
  [kunmap_high()][kunmap_high] to do the real work of unmapping the page.

* `kunmap_high()` decrements the corresponding element for the page in
  [pkmap_count][pkmap_count]. If it reaches 1 (no users but a TLB flush
  required), any process waiting on the [pkmap_map_wait][pkmap_map_wait] wait
  queue is woken up because a slot is now available.

* `kunmap_high()` does not unmap the page from the page tables because that
  would require an expensive TLB flush, this is delayed until
  [flush_all_zero_pkmaps()][flush_all_zero_pkmaps] is called.

### 9.4 Mapping High Memory Pages Atomically

* The use of [kmap_atomic()][kmap_atomic] is discouraged, however slots are
  reserved for each CPU for when they are necessary such as when 'bounce
  buffers' are used by devices from interrupt.

* There are a varying number of different requirements an architecture has for
  atomic high memory mapping, which are enumerated by [enum km_type][km_type]:

```c
enum km_type {
        KM_BOUNCE_READ,
        KM_SKB_SUNRPC_DATA,
        KM_SKB_DATA_SOFTIRQ,
        KM_USER0,
        KM_USER1,
        KM_BH_IRQ,
        KM_SOFTIRQ0,
        KM_SOFTIRQ1,
        KM_TYPE_NR
};
```

* As declared here there are `KM_TYPE_NR` total number of uses, which on i386 is
  8 (LS - book says 6?!)

* `KM_TYPE_NR` entries per processor are reserved at boot time for atomic
  mapping at the location [FIX_KMAP_BEGIN][FIX_KMAP_BEGIN] and ending at
  [FIX_KMAP_END][FIX_KMAP_END]. These are members of
  [enum fixed_addresses][fixed_addresses] and can be converted to virtual
  address via [__fix_to_virt()][__fix_to_virt].

* A user of an atomic kmap, as you might imagine, may not sleep or exit before
  calling [kunmap_atomic()][kunmap_atomic] because the next process on the
  processor may try to use the same entry and fail.

* [kmap_atomic()][kmap_atomic] has the simple task of mapping the requested page
  to the slot side aside in the page tables for the requested type of operation
  and processor.

* [kunmap_atomic()][kunmap_atomic] on the other hand has an interesting quirk -
  it will only clear the PTE via [pte_clear()][pte_clear] if debugging is
  enabled. This is because bothering to unmap atomic pages is considered
  unnecessary since the next call to `kmap_atomic()` will simply replace it and
  make TLB flushes unnecessary.

### 9.5 Bounce Buffers

* Bounce buffers are required for devices that cannot access the full range of
  memory available to the CPU.

* An obvious example is when the device does not have an address with the same
  number of bits as the CPU, e.g. a 32-bit device on a 64-bit architecture, or a
  32-bit device on a 32-bit PAE-enabled system.

* The basic concept is very simple - a bounce buffer resides in memory low
  enough for a device to copy from and write data to (i.e. to allow DMA to
  occur), once the I/O completes, the data is then copied by the kernel to the
  buffer page in high memory, rendering the bounce buffer as a type of bridge.

* There is significant overhead to this operation because, at the very least, it
  involves copying a full page, but this performance penalty is insignificant in
  comparison to swapping out pages in low memory.

#### 9.5.1 Disk Buffering

* Blocks of typically around 1KiB in size are packed into pages and managed via
  a [struct buffer_head][buffer_head] allocated by the slab allocator.

* Users of buffer heads have the option of registering a callback function which
  is stored in `buffer_head->b_end_io()` and called when I/O completes. Bounce
  buffers use this mechanism to copy data out of the bounce buffers via the
  registered callback [bounce_end_io_write()][bounce_end_io_write] (and
  subsequently [bounce_end_io()][bounce_end_io].)

* More details on buffer heads/the block layer/etc. are rather beyond the scope
  of the book.

#### 9.5.2 Creating Bounce Buffers

* Creating a bounce buffer is fairly straightforward and done via
  [create_bounce()][create_bounce] - it creates a new buffer using the specified
  buffer head as a template. Let's take a look at what it does in detail:

1. A page is allocated for the buffer itself via
  [alloc_bounce_page()][alloc_bounce_page], which is a wrapper around
  [alloc_page()][alloc_page] with one important addition - if the allocation is
  unsuccessful, there is an emergency pool of pages and buffer heads available
  for bounce buffers (see 9.6.)

2. The buffer head is allocated via [alloc_bounce_bh()][alloc_bounce_bh] which,
  similar to `alloc_bounce_page()`, calls the slab allocator for a
  [struct buffer_head][buffer_head] and uses the emergency pool if one cannot be
  allocated. Additionally, `bdflush` is woken up to start flushing dirty buffers
  out to disk so that buffers are more likely to be freed soon via
  [wakeup_bdflush()][wakeup_bdflush].

3. After the page and `buffer_head` have been allocated, data is copied from the
   template `buffer_head` into the new one. Because part of this operation may
   use [kmap_atomic()][kmap_atomic], bounce buffers are only created with the
   IRQ-safe [io_request_lock][io_request_lock] held.

4. The I/O completion callbacks are changed to be either
   [bounce_end_io_write()][bounce_end_io_write] or
   [bounce_end_io_read()][bounce_end_io_read] so data will be copied to/from
   high memory.

5. GFP flags are set such that no I/O operations involving high memory may be
   used specified by `SLAB_NOHIGHIO` to the slab allocator and `GFP_NOHIGHIO` to
   the buddy allocator. This matters because bounce buffers are used for I/O
   operations with high memory - if the allocator tries to perform high memory
   I/O, it will recurse and eventually cause a crash.

#### 9.5.3 Copying via Bounce Buffers

* Data is copied via the bounce buffer different depending on whether it's a
  read or a write buffer. If writing to the device, the buffer is populated with
  data from high memory during bounce buffer creation via
  [copy_from_high_bh()][copy_from_high_bh], and the callback function
  [bounce_end_io_write()][bounce_end_io_write] will complete the I/O later when
  the device is ready for the data.

* When reading to the device, no data transfer may take place until it is
  ready. When it is, the interrupt handler for the device calls the callback
  funciton [bounce_end_io_read()][bounce_end_io_read] which copies the data to
  high memory via [copy_to_high_bh_irq()][copy_to_high_bh_irq].

* In either case, the buffer head and page may be reclaimed via
  [bounce_end_io()][bounce_end_io] after the I/O has completed, and the I/O
  completion function for the template [struct buffer_head][buffer_head] is called.

* After the operation is complete, if the emergency pools are not full, the
  resources are added there, otherwise they are freed via their respective
  allocators.

### 9.6 Emergency Pools

* Two emergency pools of [struct buffer_head][buffer_head]'s and pages are
  maintained for bounce buffers alone. The reason for this is that if memory is
  too tight for allocations, failing to complete I/O requests is only going to
  compound issues, because buffers from high memory cannot be freed until low
  memory is available, leading to processes halting, preventing them from
  freeing up their own memory.

* The pools are initialised via [init_emergency_pool()][init_emergency_pool] to
  contain [POOL_SIZE][POOL_SIZE] entries (currently hard-coded to 32.) The pages
  are linked by the `page->list` field in the [emergency_pages][emergency_pages]
  list.

* Looking at how pages are acquired from emergency pools diagrammatically:

```
                          -----------------------                ---------------          ^
                          | alloc_bounce_page() |                | struct page |          |
                          -----------------------                ---------------          |
                                     |                                  ^                 |
                                     v                                  | page->list      |
                       ----------------------------              /- - - - - - -/          |
                       | alloc_page(GFP_NOHIGHIO) |              \ struct page \          |
                       ----------------------------              /- - - - - - -/          |
                                     |                                  ^                 |    Max.
                                     v                                  |                 | POOL_SIZE
                         /----------------------\                       | page->list      |  entries
---------------     yes /  alloc_page() returns  \               ---------------          |
| return page |<--------\          page?         /               | struct page |          |
---------------          \----------------------/                ---------------          |
                                     | no                               ^                 |
                                     v                                  | page->list      |
---------------   -----------------------------------   -------------------------------   |
| return page |<--| Acquire emergency_lock spinlock |   |         struct page         |   |
---------------   |  Take page from emergency pool  |<--| (emergency_pages list head) |   |
                  |           Return page           |   -------------------------------   V
                  -----------------------------------                   ^
                                                                        |
                                                      -------------------------------------
                                                      | init_emergency_pool() initialises |
                                                      | this list with POOL_SIZE entries  |
                                                      -------------------------------------
```

* Similarly, `buffer_head`s are linked via `buffer_head->inode_buffers` on the
  [emergency_bhs][emergency_bhs] list.

* Both `emergency_pages` and `emergency_bhs` are protected by the
  [emergency_lock][emergency_lock] spinlock, with emergency pages counted via
  [nr_emergency_pages][nr_emergency_pages], and emergency buffer heads counted
  via [nr_emergency_bhs][nr_emergency_bhs].

## Chapter 10: Page Frame Reclamation

* A running system will eventually use all available page frames for disk
  buffers, [struct dentry][dentry]s, [struct inode][inode]s, process pages,
  etc. etc.

* Linux needs to select old pages that can be freed and invalidated for new uses
  before physical memory becomes exhausted - this chapter focuses on how linux
  implements its 'page replacement policy' and how different types of pages are
  invalidated.

* The way linux selects pages is rather empirical and contains a mix of ideas
  moderated by user feedback and benchmarks.

* The page cache is a cache of all data that is read from disk to reduce the
  amount of disk I/O required. Strictly speaking this is not directly related to
  page frame reclamation, however the LRU lists and page cache are closely related.

* With the exception of the slab allocator, all pages in use by the system are
  stored on [LRU][lru] lists and linked together via [struct page][page]`->lru`
  so they can be easily scanned for replacement.

* The slab pages are not stored on the LRU lists as it's considerably harder to
  age a page based on the objects used by the slab.

* Process-mapped pages are not easily swappable because there is no easy way to
  map [struct page][page]s to PTEs except by searching every page table which is
  very expensive.

* If the page cache has a large number of process-mapped pages in it, process
  page tables will be walked and pages will be swapped out via
  [swap_out()][swap_out] until enough pages have been freed.

* Regardless, `swap_out()` will have trouble dealing with shared pages - if a
  page is shared a swap entry is allocated, the PTE is filled out with the
  necessary information to find the page in swap again and the reference count
  is decremented. Only when the count reaches 0 will it be freed - pages like
  this are considered to be in the 'swap cache'.

* Page replacement is performed via the the `kswapd` daemon.

### 10.1 Page Replacement Policy

* The page replacement policy is said to be an [LRU][lru]-based algorithm, but
  technically this isn't quite true because the lists are not strictly
  maintained in LRU order.

* The LRU in linux consists of two lists - [active_list][active_list] and
  [inactive_list][inactive_list]. The idea is for the `active_list` to contain
  the [working set][working-set] of all processes and the `inactive_list` to
  contain reclaim candidates.

* Since we store all reclaimable pages in two lists and pages belonging to any
  process may be reclaimed rather than just those belonging to a faulting
  process, the replacement policy is global.

* The lists resemble a simplified [LRU 2Q][lru-2q], in which two lists are
  maintained - `Am` and `A1`. `A1` is a FIFO queue and if a page is referenced
  while on that queue they are placed in `Am` which is a normal LRU-managed
  list.

* The 2Q approach is roughly analogous to using [lru_cache_add()][lru_cache_add]
  to place pages on the `inactive_list` (`A1` in the analogy) and using
  [mark_page_accessed()][mark_page_accessed] to move them to the `active_list`
  (`Am` in the analogy.)

* The 2Q algorithm specifies how the size of the two lists have to be tuned but
  linux takes a simpler approach by using [refill_inactive()][refill_inactive]
  to move pages from the bottom of the `active_list` to the `inactive_list` to
  keep `active_list` about two-thirds the size of the total page
  cache. Diagrammatically:

```
                                      -------------------                 -------------------
                                      | activate_page() |                 | lru_cache_add() |
                                      -------------------                 -------------------
                                               |                                   |
                                               |                                   |
                                               v                                   v
                                 -----------------------------     -------------------------------
                                 | add_page_to_active_list() |  /->| add_page_to_inactive_list() |
                                 -----------------------------     -------------------------------
                                               |                |                  |
                                               |                                   |
                                               v                |                  v
                                        ---------------                    -----------------
                                        |  list head  |<------\ |          |   list head   |
    ---------------------               |-------------|       |            |---------------|
    | refill_inactive() |<-\   Page     |             |       | |          |               |
    ---------------------  | removed by | active_list |       |            | inactive_list |
              |               refill_   |             |       | |          |               |
              |            | inactive() |-------------|       |            |---------------|
              v            \ - - - - - -|  list tail  |       | |          |   list tail   |
   -----------------------              ---------------       |            -----------------
   | Move nr_pages pages |                                    | |                  |
   | from active_list to |                                    |                    |
   |    inactive_list    |                                    | |                  v
   -----------------------                                    |             ----------------
              |                                  active_list  | |           | page reclaim |
/------------>|                                   "rotates"   |             ----------------
|             v                                               | |
|   /---------------------\ no       /-------------------\    |
|  /   nr_pages != 0 &&    \------->/ Clear reference bit \---/ |
|  \ active_list has pages /        \     Was it set?     / yes
|   \---------------------/          \-------------------/      |
|             | yes                            | no
|             |                                V                |
|             v                          --------------
|     ------------------                 | nr_pages-- |         |
|     | nr_pages moved |                 --------------
|     |  Job complete  |                       |                |
|     ------------------                       |
|                                              v                |
|                               -------------------------------   page moves to
|                               | del_page_from_active_list() | | inactive_list
\-------------------------------|     Set referenced bit      |-/
                                | add_page_to_inactive_list() |
                                -------------------------------
```

* The list described for 2Q presumes `Am` is an LRU list, but the linux
  equivalent (`active_list`) is more like a [clock algorithm][clock-algorithm]
  where the 'handspread' is the size of the active list.

* When pages reach the bottom of the `active_list` the 'referenced' flag is
  checked. If it's set, the page is moved back to the top of the list and the
  next page is set. If it is cleared, it is moved to the
  `inactive_list`. Regardless of what is done, it is cleared.

* The 'move-to-front' heuristic means that the lists behave in an LRU-_like_
  manner, but there are just too many differences between the linux replacement
  policy and LRU to consider it a 'stack' algorithm.

* Other differences are that the list priority is not order because that would
  require list updates with every reference and the lists are all but ignored
  when paging out from processes because this is decided base on the location of
  the pages in the virtual address space of the process rather than their
  location within the page lists.

* To summarise, the algorithm exhibits LRU-_like_ behaviour and has been shown
  by benchmarks to perform well in practice.

* There are two cases where the algorithm performs badly:

1. When candidates for reclamation are mostly anonymous pages - in this case
   linux will keep on examining a large number of pages before linearly scanning
   process page tables searching for the pages to reclaim. Thankfully this is
   rare.

2. When a single process has a large number of file-backed resident pages in the
   `inactive_list` that are being written to frequently. In this case processes
   and `kswapd` may go into a loop of constantly 'laundering' these pages and
   putting them at the top of the `inactive_list` without freeing anything - in
   this case few pages are moved from the `active_list` to the `inactive_list`.

### 10.2 Page Cache

* The page cache is a set of  data structures that contain pages that are backed
  by ordinary files, block devices, or swap. There are basically 4 types of page
  that exist in the cache:

1. Pages that were 'faulted in' as a result of reading a memory-mapped file.

2. Pages read from a block device or filesystem - these are special pages known
   as 'buffer pages'. The number of blocks that may fit there depends on the
   size of the block and the page size of the architecture.

3. Anonymous pages that live in the 'swap cache', a special part of the page
   cache, when slots are allocated in the backing storage for
   page-out. Discussed further in chapter 11.

4. Pages belonging to shared memory regions are treated similarly to anonymous
   pages - the only different is that shared pages are added to the swap cache
   and space is reserved in backing storage immediately after the first write to
   the page.

* The main reason for the existence of the page cache is to eliminate
  unnecessary disk reads. Pages read from disk are stored in a _page hash_
  table, which is hashed on the [struct address_space][address_space] and the
  offset. The hash is always searched before resorting to a disk read.

* Let's take a look at the page cache API:

1. [add_to_page_cache()][add_to_page_cache] - Adds a page to the LRU via
   [lru_cache_add()][lru_cache_add] in addition to adding it to the inode queue
   and page hash tables.

2. [add_to_page_cache_unique()][add_to_page_cache_unique] - Same as
   `add_to_page_cache()` except it checks to make sure the page isn't already
   there. Required when the caller doesn't hold the
   [pagecache_lock][pagecache_lock].

3. [remove_inode_page()][remove_inode_page] - Removes a page from the inode and
   hash queues via
   [remove_page_from_inode_queue()][remove_page_from_inode_queue] and
   [remove_page_from_hash_queue()][remove_page_from_hash_queue] which
   effectively removes the page from the page cache altogether.

4. [page_cache_alloc()][page_cache_alloc] - Wrapper around
   [alloc_pages()][alloc_pages] that uses the
   [struct address_space][address_space]'s `gfp_mask` field as the GFP mask.

5. [page_cache_get()][page_cache_get] - Increment [struct page][page]`->count`,
   i.e. the page's reference count. Wrapper around [get_page()][get_page].

6. [page_cache_read()][page_cache_read] - Adds a page corresponding to a
   specified `offset` and `file` to the page cache if not already there. If
   necessary, page will be read from disk via
   [struct address_space_operations][address_space_operations]

7. [page_cache_release()][page_cache_release] - Alias for
   [__free_page()][__free_page] - reference count is decremented and if it drops
   to 0, the page is freed.

#### 10.2.1 Page Cache Hash Table

* Pages in the page cache need to be located quickly. Pages are inserted into
  the [page_hash_table][page_hash_table] hash table and
  [struct page][page]`->next_hash` and `->pprev_hash` are used to handle
  collisions.

* The table is declared as follows in `mm/filemap.c`:

```c
atomic_t page_cache_size = ATOMIC_INIT(0);
unsigned int page_hash_bits;
struct page **page_hash_table;
```

* The table is allocated during system initialisation via
  [page_cache_init()][page_cache_init]:

```c
void __init page_cache_init(unsigned long mempages)
{
        unsigned long htable_size, order;

        htable_size = mempages;
        htable_size *= sizeof(struct page *);
        for(order = 0; (PAGE_SIZE << order) < htable_size; order++)
                ;

        do {
                unsigned long tmp = (PAGE_SIZE << order) / sizeof(struct page *);

                page_hash_bits = 0;
                while((tmp >>= 1UL) != 0UL)
                        page_hash_bits++;

                page_hash_table = (struct page **)
                        __get_free_pages(GFP_ATOMIC, order);
        } while(page_hash_table == NULL && --order > 0);

        printk("Page-cache hash table entries: %d (order: %ld, %ld bytes)\n",
               (1 << page_hash_bits), order, (PAGE_SIZE << order));
        if (!page_hash_table)
                panic("Failed to allocate page hash table\n");
        memset((void *)page_hash_table, 0, PAGE_HASH_SIZE * sizeof(struct page *));
}
```

* This takes `mempages`, the number of physical pages in the system, as a
  parameter and uses it to determine the hash table's size, `htable_size`:

```c
htable_size = mempages;
htable_size *= sizeof(struct page *);
```

* This is sufficient to hold pointers to every [struct page][page] in the
  system.

* The system then determines an `order` such that `PAGE_SIZE * 2^order <
  htable_size` (roughly equivalent to `order = floor(lg((htable_size * 2) -
  1))`):

```c
for(order = 0; (PAGE_SIZE << order) < htable_size; order++)
        ;
```

* The pages are allocated via [__get_free_pages()][__get_free_pages] if
  possible, trying lower orders if not and panicking if unable to allocate
  anything.

* Next the function determines `page_hash_bits`, the number of bits to use in
  the hashing function [_page_hashfn()][_page_hashfn]:

```c
unsigned long tmp = (PAGE_SIZE << order) / sizeof(struct page *);

page_hash_bits = 0;
while((tmp >>= 1UL) != 0UL)
        page_hash_bits++;
```

* This is equivalent to `page_hash_bits = lg(PAGE_SIZE*2^order/sizeof(struct
  page *))`, which renders the table a power-of-two hash table, negating the
  need to use a modulus (a common choice for hashing functions.)

* Finally, the page table is zeroed.

#### 10.2.2 inode Queue

* The 'inode queue' is part of the [struct address_space][address_space]
  introduced in 4.4.2.

* `struct address_space` contains 3 lists associated with the inode -
  `clean_pages`, `dirty_pages`, and `locked_pages` - dirty pages are ones that
  have been changed since the last sync to disk and locked pages are
  unsurprisingly ones that are locked :)

* These three lists in combination are considered to be the inode queue for a
  given mapping, and the multi-purpose [struct page][page]`->list` field is used
  to link pages on it.

* Pages are added to the inode queue via
  [add_page_to_inode_queue()][add_page_to_inode_queue] and removed via
  [remove_page_from_inode_queue()][remove_page_from_inode_queue].

#### 10.2.3 Adding Pages to the Page Cache

* Pages read from a file or block device are generally added to the page cache
  to avoid further disk I/O. Most filesystems uses the high-level
  [generic_file_read()][generic_file_read] as their `file_operations->read()`.

* `generic_file_read()` does the following:

1. Performs a few sanity checks then, if direct I/O, hands over to
  [generic_file_direct_IO()][generic_file_direct_IO], otherwise calls
  [do_generic_file_read()][do_generic_file_read] to do the heavy lifting (we
  don't consider direct I/O here.)

2. Searches the page cache by calling [__find_page_nolock()][__find_page_nolock]
   with the [pagecache_lock][pagecache_lock] held to see if the page already
   exists in it.

3. If the page does not already exist, a new page is allocated via
   [page_cache_alloc()][page_cache_alloc] and added to the page cache via
   [__add_to_page_cache()][__add_to_page_cache].

4. After a page frame is present in the page cache,
   [generic_file_readahead()][generic_file_readahead] is called which uses
   [page_cache_read()][page_cache_read] to read pages from disk via
   `mapping->a_ops->readpage()` where `mapping` is the
   [struct address_space][address_space] managing the file.

* Anonymous pages are added to the swap cache when they are unmapped from a
  process (see 11.4) and have no `struct address_space` acting as a mapping, or
  any offset within a file, which leaves nothing to hash them into the page
  cache with.

* These anonymous pages will still exist on the LRU lists, however. Once in the
  swap cache, the only real difference between anonymous pages and file-backed
  pages is that anonymous pages will use [swapper_space][swapper_space] as their
  `struct address_space`.

* Shared memory pages are added when:

1. [shmem_getpage()][shmem_getpage] is called when a page has be either fetched
   from swap or allocated because it is the first reference.

2. The swapout code calls [shmem_unuse()][shmem_unuse] when a swap area is being
   deactivated and a page backed by swap space is found that does not appear to
   belong to any process. In this case the inodes related to shared memory are
   exhaustively searched until the correct page is found.

* In both cases the page is added with [add_to_page_cache()][add_to_page_cache].

### 10.3 LRU Lists

* As discussed in 10.1, [active_list][active_list] and
  [inactive_list][inactive_list] are declared in `mm/page_alloc.c` and are
  protected by [pagemap_lru_lock][pagemap_lru_lock] (a wrapper around
  [pagemap_lru_lock_cacheline][pagemap_lru_lock_cacheline].) The active and
  inactive lists roughly represent 'hot' and 'cold' pages in the system,
  respectively.

* In other words, `active_list` contains all the working sets in the systems,
  and `inactive_list` contains reclaim candidates.

* Taking a look at the LRU list API:

1. [lru_cache_add()][lru_cache_add] - Adds a 'cold' page to the
   [inactive_list][inactive_list]. It will be moved to the
   [active_list][active_list] with a call to `mark_page_accessed()` if the page
   is known to be 'hot' such as when a page is faulted in.

2. [lru_cache_del()][lru_cache_del] - Removes a page from the LRU lists by
   calling one of either
   [del_page_from_active_list()][del_page_from_active_list] or
   [del_page_from_inactive_list()][del_page_from_inactive_list] as appropriate.

3. [mark_page_accessed()][mark_page_accessed] - Marks that the specified page
   has been accessed. If it was not recently referenced (i.e. in the
   `inactive_list` and the `PG_referenced` flag is not set), the referenced flag
   is set. If the page is referenced a second time, `activate_page()` is called
   which marks the page 'hot' and the reference count is cleared.

4. [activate_page()][activate_page] - Removes a page from the `inactive_list`
   and places it on the `active_list`. It's rarely called directly because the
   caller has to know the page is on the `inactive_list`. In most cases
   `mark_page_accessed()` should be used instead.

#### 10.3.1 Refilling inactive_list

* When caches are shrunk, pages are moved from the `active_list` to the
  `inactive_list` via [refill_inactive()][refill_inactive].

* `refill_inactive()` is parameterised by the number of pages to move
  (`nr_pages`) which is calculated in [shrink_caches()][shrink_caches] as:

```
pages = nr_pages * nr_active_pages / (2 * (nr_inactive_pages + 1))
```

* This keeps the `active_list` about two-thirds the size of the `inactive_list`
  and the number of pages to move is determined as a ratio scaled by the number
  of pages we want to swap out (`nr_pages`.)

* Pages are taken from the end of the `active_list` and if the `PG_referenced`
  flag is set it is cleared and the page is put at the top of the `active_list`
  because it has been recently used and is still 'hot' (sometimes referred to as
  'rotating the list'.)

* If the `PG_referenced` flag is not set when checked the page is moved to the
  `inactive_list` and the `PG_referenced` flag is set so it will be quickly
  promoted to the `active_list` if necessary.

#### 10.3.2 Reclaiming Pages From the LRU Lists

* [shrink_cache()][shrink_cache] is the part of the replacement algorithm that
  takes pages from `inactive_list` and determines how they should be swapped
  out.

* `shrink_cache()` is parameterised by `nr_pages` and `priority`, with
  `nr_pages` starting as [SWAP_CLUSTER_MAX][SWAP_CLUSTER_MAX] (set at 32) and
  `priority` starting as [DEF_PRIORITY][DEF_PRIORITY] (set at 6.)

* The local variables `max_scan` and `max_mapped` determine how much work the
  function will do and are impacted by the `priority` parameter.

* Each time [shrink_caches()][shrink_caches] is called and enough pages are not
  freed, the priority is decreased (i.e. made higher priority) until priority 1
  is reached.

* `max_scan` simply represents the maximum number of pages that will be scanned
  by the function and is set as `max_scan = nr_inactive_pages/priority`. This
  means that at the lowest priority of 6, at most one-sixth of the pages in the
  `inactive_list` will be scanned, and at the highest priority _all_ of them
  will be.

* `max_mapped` determines how many process pages are allowed to exist in the
  page cache before whole processes will be swapped out. It is calculated as the
  minimum or either `0.1 * max_scan` or `nr_pages * 2^(10 - priority)`.

* This means that at the lowest priority the maximum number of mapped pages
  allowed is either one-tenth of `max_scan` or 16 (`16 = 2^(10-6) = 2^4`) times
  the number of pages to swap out, whichever is smaller.

* At the highest priority the maximum number of mapped pages is either one-tenth
  of `max_scan` or 512 (`512 = 2^(10-1) = 2^9`) times the number of pages to
  swap out, whichever is smaller.

* From this point on [shrink_cache()][shrink_cache] is basically a larger
  for-loop that scans at most `max_scan` pages to free up `nr_pages` pages from
  the end of `inactive_list` or until `inactive_list` is empty.

* After each page released, `shrink_cache()` checks whether it should reschedule
  itself so the CPU is not monopolised.

* For each type of page found on the list, `shrink_cache()` behaves differently
  depending on the state of the page:

1. _Page is mapped by a process_ - Jump to the `page_mapped` label, decrement
   `max_mapped` unless it's reached 0 at which point linearly scan the page
   tables of processes and swap them out via [swap_out()][swap_out].

2. _Page is locked and the `PG_launder` bit is set_ - The page is locked for I/O
   so it could be skipped, however `PG_launder` implies this is the 2nd time the
   page has been found locked and so it's better to wait until the I/O completes
   and get rid of it. A reference to the page is taken via
   [page_cache_get()][page_cache_get] so the page won't be freed prematurely and
   [wait_on_page()][wait_on_page] is called which sleeps until the I/O is
   complete. Once it's completed, the reference count is decremented via
   [page_cache_release()][page_cache_release] and when this count reaches 0 the
   page will be reclaimed.

3. _Page is dirty, unmapped by all process, has no buffers and belongs to a
   device or file mapping_ - Since the page belongs to a file/device mapping it
   has a `page->mapping->a_ops->writepage()` function. `PG_dirty` is cleared and
   `PG_launder` is set because I/O is about to begin. We take a reference for
   the page via [page_cache_get()][page_cache_get] before calling its
   `writepage()` function to synchronise it with the backing file before
   dropping the reference via [page_cache_release()][page_cache_release]. Note
   that anonymous pages that are part of the swap cache will also get
   synchronised because they use [swapper_space][swapper_space] as their
   `page->mapping`. The page remains on the LRU - when it's found again it'll be
   freed if the I/O has completed, if not the kernel will wait for it to
   complete (see case 2.)

4. _Page has buffers associated with data on disk_ - A reference is taken to the
   page and an attempt is made to free it via
   [try_to_release_page()][try_to_release_page]. If this is successful and the
   page is anonymous (i.e. `page->mapping` is `NULL`) the page is removed from
   the LRU and `page_cache_release()` is called to decrement the usage
   count. There is one case where the anonymous page has associated buffers, and
   that's when it's backed by a swap file because the page needs to be written
   out in block-sized chunks. In other cases however where the page is backed by
   a file or device the reference is simply dropped and the page will be freed
   as usual when its reference count reaches 0.

5. _Page is anonymous and is mapped by more than one process_ - The LRU then
   page are unlocked before dropping into the same `page_mapped` label as case 1
   visited. Hence `max_mapped` is decremented and [swap_out()][swap_out] is
   called if and when `max_mapped` reaches 0.

6. _Page has no process referencing it_ - This is the fall-through case - If the
   page is in the swap cache, it is removed because it is now synchronised with
   the backing storage and has no process referencing it. If it was part of a
   file, it's removed from the inode queue, deleted from the page cache and
   freed.

### 10.4 Shrinking All Caches

* [shrink_caches()][shrink_caches] is a simple function which takes a few steps
  to free up some memory. It is invoked by
  [try_to_free_pages_zone()][try_to_free_pages_zone]:

```c
int try_to_free_pages_zone(zone_t *classzone, unsigned int gfp_mask)
{
        int priority = DEF_PRIORITY;
        int nr_pages = SWAP_CLUSTER_MAX;

        gfp_mask = pf_gfp_mask(gfp_mask);
        do {
                nr_pages = shrink_caches(classzone, priority, gfp_mask, nr_pages);
                if (nr_pages <= 0)
                        return 1;
        } while (--priority);

        /*
         * Hmm.. Cache shrink failed - time to kill something?
         * Mhwahahhaha! This is the part I really like. Giggle.
         */
        out_of_memory();
        return 0;
}
```

* `nr_pages` is initialised to [SWAP_CLUSTER_MAX][SWAP_CLUSTER_MAX] (32) as an
  upper bound. This restriction is in place so that if `kswapd` schedules a
  large number of pages to be written to disk it'll sleep occasionally to allow
  the I/O to take place.

* As pages are freed, `nr_pages` is decremented to keep count.

* The amount of work that will beperformed also depends on the `priority`, which
  is initialised to [DEF_PRIORITY][DEF_PRIORITY] here (6) and is decremented
  after each pass that does not free up enough pages up to a maximum of priority
  1.

* Taking a look at [shrink_caches()][shrink_caches]:

```c
static int shrink_caches(zone_t * classzone, int priority, unsigned int gfp_mask, int nr_pages)
{
        int chunk_size = nr_pages;
        unsigned long ratio;

        nr_pages -= kmem_cache_reap(gfp_mask);
        if (nr_pages <= 0)
                return 0;

        nr_pages = chunk_size;
        /* try to keep the active list 2/3 of the size of the cache */
        ratio = (unsigned long) nr_pages * nr_active_pages / ((nr_inactive_pages + 1) * 2);
        refill_inactive(ratio);

        nr_pages = shrink_cache(nr_pages, classzone, gfp_mask, priority);
        if (nr_pages <= 0)
                return 0;

        shrink_dcache_memory(priority, gfp_mask);
        shrink_icache_memory(priority, gfp_mask);
#ifdef CONFIG_QUOTA
        shrink_dqcache_memory(DEF_PRIORITY, gfp_mask);
#endif

        return nr_pages;
}
```

* This first calls [kmem_cache_reap()][kmem_cache_reap] (see 8.1.7) which
  selects a slab cache to shrink. If `nr_pages` pages are freed the function is
  done. Otherwise, it tries to free from other caches.

* If other caches are affected, [refill_inactive()][refill_inactive] will move
  pages from [active_list][active_list] to [inactive_list][inactive_list] before
  shrinking the page cache by reclaiming pages at the end of the `inactive_list`
  with [shrink_cache()][shrink_cache].

* Finally if more pages need to be freed it will shrink 3 special caches - the
  dcache, icache and dqcache via [shrink_dcache_memory()][shrink_dcache_memory],
  [shrink_icache_memory()][shrink_icache_memory] and
  [shrink_dqcache_memory()][shrink_dqcache_memory] respectively.

* The objects in these special caches are relatively small themselves, but a
  cascading effect allows a lot more pages to be freed in terms of buffer and
  disk caches.

### 10.5 Swapping Out Process Pages

* When `max_mapped` pages have been found in the page cache in
  [shrink_cache()][shrink_cache], [swap_out()][swap_out] is called to swap out
  process pages. Starting from the [struct mm_struct][mm_struct] pointed to by
  [swap_mm][swap_mm] and `mm->swap_address`, page tables are searched forward
  until `nr_pages` (local variable defaulting to
  [SWAP_CLUSTER_MAX][SWAP_CLUSTER_MAX]) have been freed.

* All process-mapped pages are examined regardless of where they are in the
  lists or when they were last referenced, but pages that are part of the
  [active_list][active_list] or have recently been referenced will be skipped
  out.

* The examination of 'hot' pages is costly, but insignificant compared to
  linearly searching all processes for the PTEs that reference a particular
  [struct page][page].

* After it's been decided to swap out pages from a process, an attempt will be
  made to swap out at least [SWAP_CLUSTER_MAX][SWAP_CLUSTER_MAX] pages,
  examining the list of [struct mm_struct][mm_struct]s once only to avoid
  constant looping when no pages are available.

* Additionally, writing out the pages in bulk increases the chances that pages
  close together in the process address space will be written out to adjacent
  slots on the disk.

* [swap_mm][swap_mm] is initialised to point to [init_mm][init_mm] and its
  `swap_address` field is set to 0 the first time it's used.

* A task has been fully searched when its
  [struct mm_struct][mm_struct]`->swap_address` is equal to
  [TASK_SIZE][TASK_SIZE].

* After a task has been selected to swap pages from, the reference count to the
  `mm_struct` is incremented so that it will not be freed early, and
  [swap_out_mm()][swap_out_mm] is called parameterised by the chosen
  `mm_struct`.

* `swap_out_mm()` walks each of the process's VMAs and calls
  [swap_out_vma()][swap_out_vma] for each. This is done to avoid having to walk
  the entire page table which will largely be sparse.

* `swap_out_vma()` calls [swap_out_pgd()][swap_out_pgd] which calls
  [swap_out_pmd()][swap_out_pmd] which calls
  [try_to_swap_out()][try_to_swap_out] in turn.

* `try_to_swap_out()` does the following:

1. Ensures the page is not part of [active_list][active_list], nor has been
   recently referenced or belongs to a zone that we aren't interested in.

2. Removes the page from the process page tables.

3. Examine the newly removed PTE to see whether it's dirty - if so, the
   [struct page][page] flags will be updated such that it will get synchronised
   with the backing storage.

4. If the page is already part of the swap cache the RSS is simply updated, and
   the reference to the page is dropped. Otherwise it is added to the swap cache
   (chapter 11 goes into more detail about how pages are added to the swap
   cached and synchronised with backing storage.)

### 10.6 Pageout Daemon (kswapd)

* During system startup, a kernel thread called `kswapd` is started via
  [kswapd_init()][kswapd_init] which continuously executes [kswapd()][kswapd],
  which is usually asleep like a cheeky kitteh cat.

* The `kswapd` daemon is responsible for reclaiming pages when memory is running
  low. Historically it used to be woken every 10 seconds, but now it is woken by
  the physical page allocator when the zone's free pages count reaches its
  `pages_low` field (see 2.2.1.)

* The `kswapd` daemon that performs most of the tasks needed to maintain the
  page cache correctly - shrinking slab caches and swapping out processes when
  necessary.

* To avoid being woken up a great deal under high memory pressure, `kswapd`
  keeps on freeing pages until the zone's `pages_high` watermark is reached.

* Under _extreme_ memory pressure, processes will do the work of `kswapd`
  synchronously by calling [balance_classzone()][balance_classzone] which calls
  [try_to_free_pages_zone()][try_to_free_pages_zone] that does `kswapd`'s work.

* When `kswapd` is woken up, it does the following:

1. Calls [kswapd_can_sleep()][kswapd_can_sleep] which checks all zones'
   `need_balance` fields - if any are set it cannot sleep.

2. If it can't sleep, it's removed from the [kswapd_wait][kswapd_wait] wait
   queue.

3. Calls [kswapd_balance()][kswapd_balance] to cycle through all zones and free
   pages via [try_to_free_pages_zone()][try_to_free_pages_zone] if
   `need_balance` is set, freeing until the `pages_high` watermark is reached.

4. The task queue for [tq_disk][tq_disk] is run so that queued pages will be
   written out.

5. Adds [kswapd][kswapd] back to the [kswapd_wait][kswapd_wait] wait queue and
   loops back to step 1.

## Chapter 11: Swap Management

* Just as linux uses free memory for buffering data from a disk, the reverse is
  true as well - eventually there's a need to free up private or anonymous pages
  used by a process. The pages are copied to backing storage, sometimes called
  the 'swap area'.

* Strictly speaking, linux doesn't swap because 'swapping' refers to copying an
  entire process address space to disk and 'paging' to copying out individual
  pages, however it's referred to as 'swapping' in discussion and documentation
  so we'll call it swapping regardless.

* There are 2 principal reasons for the existence of swap space:

1. It expands the amount of memory a process may use - virtual memory and swap
   space allows a large process to run even if the process is only partially
   resident. Because old pages may be swapped out, the amount of memory
   addressed may easily exceed RAM because demand paging will ensure the pages
   are reloaded if necessary.

2. Even if sufficient memory, swap is useful - a significant number of pages
   referenced by a process early on in its life may only be used for
   initialisation then never used again. It's better to swap out these pages and
   create more disk buffers than leave them resident and unused.

* Swap is slow. Very slow as disks are slow (relatively to memory) - if
  processes are _frequently_ addressing a large amount of memory no amount of
  swap or fast disks will make it run without a reasonable time, only more RAM
  can help.

* It's very important that the correct page be swapped out (as discussed in
  chapter 10), and also that related pages be stored close together in the swap
  space so they are likely to be swapped in at the same time while reading
  ahead.

### 11.1 Describing the Swap Area

* Each active swap area has a [struct swap_info_struct][swap_info_struct]
  describing the area:

```c
/*
 * The in-memory structure used to track swap areas.
 */
struct swap_info_struct {
        unsigned int flags;
        kdev_t swap_device;
        spinlock_t sdev_lock;
        struct dentry * swap_file;
        struct vfsmount *swap_vfsmnt;
        unsigned short * swap_map;
        unsigned int lowest_bit;
        unsigned int highest_bit;
        unsigned int cluster_next;
        unsigned int cluster_nr;
        int prio;                       /* swap priority */
        int pages;
        unsigned long max;
        int next;                       /* next entry on swap list */
};
```

* All the `swap_info_struct`s in the running system are stored in a statically
  declared array, [swap_info][swap_info] which holds
  [MAX_SWAPFILES][MAX_SWAPFILES] (defined as 32) - this means that at most 32
  swap areas can exist on a running system.

* Looking at each field:

1. `flags` - A bit field with 2 possible values - [SWP_USED][SWP_USED] (`0b01`)
   and [SWP_WRITEOK][SWP_WRITEOK] (`0b11`). `SWP_USED` implies the swap area is
   currently active, and `SWP_WRITEOK` when linux is ready to write to the
   area. `SWP_WRITEOK` contains `SWP_USED` as the former implies the latter must
   be the case.

2. `swap_device` - The device corresponding to the partition for this swap
   area. If the swap area is a file, this is set to `NULL`.

3. `sdev_lock` - - Spinlock protecting the struct, most pertinently
   `swap_map`. It's locked and unlocked via
   [swap_device_lock()][swap_device_lock] and
   [swap_device_unlock()][swap_device_unlock].

4. `swap_file` - The [struct dentry][dentry] for the actual special file that is
   mounted as a swap area, for example this file may exist in `/dev` in the case
   that a partition is mounted. This field is needed to identify the correct
   `swap_info_struct` when deactivating a swap area.

5. `swap_vfsmnt` - The [struct vfsmount][vfsmount] object corresponding to where
   the device or file for this swap area is located.

6. `swap_map` - A large array containing one entry for every swap entry or
   page-sized slot in the area. An 'entry' is a reference count of the number of
   users of the page slot, with the swap cache counting as one user and every
   PTE that has been paged out to the slot as the other users. If the entry is
   equal to [SWAP_MAP_MAX][SWAP_MAP_MAX], the slot is allocated permanently. If
   it's equal to [SWAP_MAP_BAD][SWAP_MAP_BAD], the slot will never be used.

7. `lowest_bit` - Lowest possible free slot available in the swap area and is
   used to start from when linearly scanning to reduce the search space - there
   are definitely no free slots below this mark.

8. `highest_bit` - Highest possible free slot available. Similar to
   `lowest_bit`, there are definitely no free slots above this mark.

9. `cluster_next` - Offset of the next cluster of blocks to use. The swap area
   tries to have pages allocated in cluster blocks to increase the chance
   related pages will be stored together.

10. `cluster_nr` - Number of pages left to allocate in this cluster.

11. `prio` - The 'priority' of the swap area - this determines how likely the
    area is to be used. By default the priorities are arranged in order of
    activation, but the sysadmin may also specify it using the `-p` flag of
    [swapon][swapon].

12. `pages` - Because some slots on the swap file may be unusable, this field
    stores the number of usable pages in the swap area. This differs from `max`
    in that swaps marked [SWAP_MAP_BAD][SWAP_MAP_BAD] are not counted.

13. `max` - Total number of slots in this swap area.

14. `next` - The index in the [swap_info][swap_info] array of the next swap area
    in the system.

* The areas are not only stored in an array, they are also kept in a
  [struct swap_list_t][swap_list_t] 'pseudolist'
  [swap_list][swap_list]. `swap_list_t` is a simple type:

```c
struct swap_list_t {
        int head;       /* head of priority-ordered swapfile list */
        int next;       /* swapfile to be used next */
};
```

* `head` is the index of the swap area of the highest priority swap area in use,
  and `next` is the index of the next swap area that should be used.

* This list enables areas to be looked up in order of priority when necessary
  but still remain easily looked up in the `swap_info` array.

* Each swap area is divided into a number of page-sized slots on disk (e.g. 4KiB
  each on i386.)

* The first slot is always reserved because it contains information about the
  swap area that must not be overwritten, including the first 1KiB which stores
  a disk label for the partition that can be retrieved via userspace tools.

* The remaining space in this initial slot is used for information about the
  swap area which is filled when the swap area is created with
  [mkswap][mkswap]. This is represented by the [union swap_header][swap_header]
  union:

```c
/*
 * Magic header for a swap area. The first part of the union is
 * what the swap magic looks like for the old (limited to 128MB)
 * swap area format, the second part of the union adds - in the
 * old reserved area - some extra information. Note that the first
 * kilobyte is reserved for boot loader or disk label stuff...
 *
 * Having the magic at the end of the PAGE_SIZE makes detecting swap
 * areas somewhat tricky on machines that support multiple page sizes.
 * For 2.5 we'll probably want to move the magic to just beyond the
 * bootbits...
 */
union swap_header {
        struct
        {
                char reserved[PAGE_SIZE - 10];
                char magic[10];                 /* SWAP-SPACE or SWAPSPACE2 */
        } magic;
        struct
        {
                char         bootbits[1024];    /* Space for disklabel etc. */
                unsigned int version;
                unsigned int last_page;
                unsigned int nr_badpages;
                unsigned int padding[125];
                unsigned int badpages[1];
        } info;
};
```

* Looking at each of the fields:

1. `reserved` - Dummy field used to position `magic` correctly at the end of the
   page.

2. `magic` - Used for identifying the magic string that identifies a swap
   header - this is in place to ensure that a partition that is not a swap area
   will never be used by mistake, and to determine what version of swap area is
   to be used - if the string is `"SWAP-SPACE"`, then it's version 1 of the swap
   file format. If it's `"SWAPSPACE2"`, then version 2 will be used.

3. `bootbits` - Reserved area containing information about the partition such as
   the disk label, retrievable via userspace tools.

4. `version` - Version of the swap area layout.

5. `last_page` - Last usable page in the area.

6. `nr_badpages` - Known number of bad pages that exist in the swap area.

7. `padding` - Disk sectors are usually 512 bytes in size. `version`,
   `last_page` and `nr_badpages` take up 12 bytes, so this field takes up the
   remaining 500 bytes to sector-align `badpages`.

8. `badpages` - The remainder of the page is used to store the indices of up to
   [MAX_SWAP_BADPAGES][MAX_SWAP_BADPAGES] number of bad page slots. These are
   filled in by the [mkswap][mkswap] userland tool if the `-c` switch is
   specified to check the area.

* `MAX_SWAP_BADPAGES` is a compile-time constant that varies if the struct
  changes, but is 637 entries in its current form determined by:

```
MAX_SWAP_BADPAGES = (PAGE_SIZE - <bootblock size = 1024> - <padding size = 512> -
                     <magic string size = 10>)/sizeof(long)
```

### 11.2 Mapping Page Table Entries to Swap Entries

* When a page is swapped out, linux uses the corresponding PTE to store enough
  information to locate the page on disk again. Rather than storing the physical
  address of the page, the PTE contains this information and has the appropriate
  flags set to indicate that it is not an address.

* Clearly, a PTE is not large enough to store precisely where on the disk the
  page is located, but it's more than enough to store an index into the
  [swap_info][swap_info] array and an offset within the `swap_map`.

* Each PTE, regardless of architecture, is large enough to store a
  [swp_entry_t][swp_entry_t]:

```c
/*
 * A swap entry has to fit into a "unsigned long", as
 * the entry is hidden in the "index" field of the
 * swapper address space.
 *
 * We have to move it here, since not every user of fs.h is including
 * mm.h, but mm.h is including fs.h via sched .h :-/
 */
typedef struct {
        unsigned long val;
} swp_entry_t;
```

* [pte_to_swp_entry()][pte_to_swp_entry] and
  [swp_entry_to_pte()][swp_entry_to_pte] are used to translate between PTEs and
  swap entries.

* As always we'll focus on i386, however all architectures has to be able to
  determine if a PTE is present or swapped out. In the `swp_entry_t` two bits
  are always kept free - on i386 bit 0 is reserved for the
  [_PAGE_PRESENT][_PAGE_PRESENT] flag and bit 7 is reserved for
  [_PAGE_PROTNONE][_PAGE_PROTNONE] (both discussed in 3.2.)

* Bits 1 to 6 are used for the 'type' which is the index within the
  [swap_info][swap_info] array and is returned via [SWP_TYPE()][SWP_TYPE].

* Bits 8 through 31 (base-0 so all remaining bits) are used to the the _offset_
  within the `swap_map` from the `swp_entry_t`. This means there are 24 bits
  available, limiting the swap area to 64GiB (`2^24 * 4096` bytes.)

* The offset is extracted via [SWP_OFFSET()][SWP_OFFSET]. To encode a type and
  offset into a [swp_entry_t][swp_entry_t], [SWP_ENTRY()][SWP_ENTRY] is
  available which does all needed bit-shifting.

* Looking at the relationship between the various macros diagrammatically:

```
       -------          ---------- --------
       | PTE |          | Offset | | Type |
       -------          ---------- --------
          |                 |         |
          |                 |         |
          v                 v         v
----------------------   ---------------
| pte_to_swp_entry() |   | SWP_ENTRY() |         Bits reserved for
----------------------   ---------------   _PAGE_PROTNONE  _PAGE_PRESENT
          |                     |                  |              |
          /---------------------/                  |              |
          | BITS_PER_LONG                         8|7            1|0
          |       ---------------------------------v--------------v-
          \------>|            Offset             | |    Type    | |
                  --------------------------------------------------
             swp_entry_t        |                           |
                                |                           |
                                v                           v
                        ----------------             --------------
                        | SWP_OFFSET() |             | SWP_TYPE() |
                        ----------------             --------------
                                |                           |
                                |                           |
                                v                           v
                           ----------                   --------
                           | Offset |                   | Type |
                           ----------                   --------
```

* The six bits for type should allow up to 64 (2^6) swap areas to exist in a
  32-bit architecture, however [MAX_SWAPFILES][MAX_SWAPFILES] is set at 32. This
  is due to the consumption of the `vmalloc` address space (see chapter 7 for
  more on vmalloc.)

* If a swap area is the maximum possible size, 32MiB is required for the
  `swap_map` (`2^24 * sizeof(short)`) - remembering that each page uses one
  `short` for the reference count. This means that if `MAX_SWAPFILES = 32` swaps
  exist, 1GiB of virtual malloc space is required, which is simply impossible
  given the user/kernel linear address space split.

* You'd think this would mean that supporting 64 swap areas is not worth the
  additional complexity, but this is not the case - if a system has multiple
  disks, it's worth having this complexity in order to distribute swap across
  all disks to allow for maximal parallelism and hence performance.

### 11.3 Allocating a Swap Slot

* As discussed previously, all page-sized slots are tracked by the
  [struct swap_info][swap_info]`->swap_map` `unsigned short` array each of which
  act as a reference count (number of 'users' of the slot, with the swap cache
  acting as the first 'user' - a shared page can have a lot of these of course.)

* If the entry is [SWAP_MAP_MAX][SWAP_MAP_MAX] the page is permanently reserved
  for that slot. This is unlikely but not impossible - it's designed to ensure
  the reference count does not overflow.

* If the entry is [SWAP_MAP_BAD][SWAP_MAP_BAD], the slot is unusable.

* Finding and allocating a swap entry is divided into two major tasks -
  the first is performed by [get_swap_page()][get_swap_page]:

1. Starting with [swap_list][swap_list]`->next`, search swap areas for a
   suitable slot.

2. Once a slot has been found, record what the next swap to be used will be and
   return the allocated entry.

* The second major task is is the searching itself, which is performed by
  [scan_swap_map()][scan_swap_map]. In principle it's very simple because it
  linearly scans the array for a free slot, however the implementation is little
  more involved than that:

```c
static inline int scan_swap_map(struct swap_info_struct *si)
{
        unsigned long offset;
        /*
         * We try to cluster swap pages by allocating them
         * sequentially in swap.  Once we've allocated
         * SWAPFILE_CLUSTER pages this way, however, we resort to
         * first-free allocation, starting a new cluster.  This
         * prevents us from scattering swap pages all over the entire
         * swap partition, so that we reduce overall disk seek times
         * between swap pages.  -- sct */
        if (si->cluster_nr) {
                while (si->cluster_next <= si->highest_bit) {
                        offset = si->cluster_next++;
                        if (si->swap_map[offset])
                                continue;
                        si->cluster_nr--;
                        goto got_page;
                }
        }
        si->cluster_nr = SWAPFILE_CLUSTER;

        /* try to find an empty (even not aligned) cluster. */
        offset = si->lowest_bit;
 check_next_cluster:
        if (offset+SWAPFILE_CLUSTER-1 <= si->highest_bit)
        {
                int nr;
                for (nr = offset; nr < offset+SWAPFILE_CLUSTER; nr++)
                        if (si->swap_map[nr])
                        {
                                offset = nr+1;
                                goto check_next_cluster;
                        }
                /* We found a completly empty cluster, so start
                 * using it.
                 */
                goto got_page;
        }
        /* No luck, so now go finegrined as usual. -Andrea */
        for (offset = si->lowest_bit; offset <= si->highest_bit ; offset++) {
                if (si->swap_map[offset])
                        continue;
                si->lowest_bit = offset+1;
        got_page:
                if (offset == si->lowest_bit)
                        si->lowest_bit++;
                if (offset == si->highest_bit)
                        si->highest_bit--;
                if (si->lowest_bit > si->highest_bit) {
                        si->lowest_bit = si->max;
                        si->highest_bit = 0;
                }
                si->swap_map[offset] = 1;
                nr_swap_pages--;
                si->cluster_next = offset+1;
                return offset;
        }
        si->lowest_bit = si->max;
        si->highest_bit = 0;
        return 0;
}
```

* Linux tries to organise pages into 'clusters' on disk of size
  [SWAPFILE_CLUSTER][SWAPFILE_CLUSTER]. It allocates `SWAPFILE_CLUSTER` pages
  sequentially in swap, keeps count of the number of sequentially allocated
  pages in [struct swap_info_struct][swap_info_struct]`->cluster_nr` and records
  the current offset in `swap_info_struct->cluster_next`.

* After a sequential block has been allocated, it searches for a block of free
  entries of size [SWAPFILE_CLUSTER][SWAPFILE_CLUSTER]. If a large enough block
  can be found, it will be used as another cluster-sized sequence.

* If no free clusters large enough can be found in the swap area, a simple
  first-free search is performed starting from `swap_info_struct->lowest_bit`.

* The aim is to have pages swapped out at the same time close together, on the
  premise that pages swapped out together are likely to be related. This makes
  sense because the page replacement algorithm will use swap space most when
  linearly scanning the process address space.

* Without scanning for large free blocks and using them, it is likely the
  scanning would degenerate to first-free searches and never improve, with it
  exiting processes are likely to free up large blocks of slots.

### 11.4 Swap Cache

* Pages that are shared between many processes cannot be easily swapped out
  because there's no quick way to map a [struct page][page] to every PTE that
  references it.

* If special care wasn't taken this fact could lead to the rare condition where
  a page that is present for one PTE and swapped out for another gets updated
  without being synced to disk, thereby losing the update.

* To address the problem, shared pages that have a reserved slot in backing
  storage are considered to be part of the 'swap cache'.

* The swap cache is simply a specialisation of the page cache with the main
  difference between pages in the swap cache and the page cache being that pages
  in the swap cache always use [swapper_space][swapper_space] as their
  [struct address_space][address_space] in `page->mapping`.

* Another difference is that pages are added to the swap cache with
  [add_to_swap_cache()][add_to_swap_cache] instead of
  [add_to_page_cache()][add_to_page_cache].

* Taking a look at the swap cache API:

1. [get_swap_page()][get_swap_page] - Allocates a slot in a `swap_map` by
   searching active swap areas. Covered in more detail in 11.3.

2. [add_to_swap_cache()][add_to_swap_cache] - Adds a page to the swap cache -
   first checking to see whether it already exists by calling `swap_duplicate()`
   and if not adding it to the swap cache via
   [add_to_page_cache_unique()][add_to_page_cache_unique].

3. [lookup_swap_cache()][lookup_swap_cache] - Searches the swap cache and
   returns the [struct page][page] corresponding to the specified swap entry. It
   works by searching the normal page cache based on
   [swapper_space][swapper_space] and the `swap_map` offset.

4. [swap_duplicate()][swap_duplicate] - Verifies a swap entry is valid, and if
   so, increments its swap map count.

5. [swap_free()][swap_free] - The complement to `swap_duplicate()`. Decrements
   the relevant counter in the `swap_map`. Once this count reaches zero, the
   slot is effectively free.

* Anonymous pages are not part of the swap cache until an attempt is made to
  swap them out.

* Taking a look at [swapper_space][swapper_space]:

```c
struct address_space swapper_space = {
        LIST_HEAD_INIT(swapper_space.clean_pages),
        LIST_HEAD_INIT(swapper_space.dirty_pages),
        LIST_HEAD_INIT(swapper_space.locked_pages),
        0,                              /* nrpages      */
        &swap_aops,
};
```

* A page is defined as being part of the swap cache when its
  [struct page][page]`->mapping` field has been set to `swapper_space`. This is
  determined by [PageSwapCache()][PageSwapCache].

* Linux uses the exact same code for keeping pages between swap and memory sync
  as it uses for keeping file-backed pages and memory in sync - they both share
  the page cache code, the differences being in the functions used.

* The address space for backing storage, `swapper_space`, uses
  [swap_aops][swap_aops] in its [struct address_space][address_space]`->a_ops`.

* The `page->index` field is then used to store the [swp_entry_t][swp_entry_t]
  structure instead of a file offset (its usual purpose.)

* The [struct address_space_operations][address_space_operations] struct
  `swap_aops` is defined as follows, using [swap_writepage()][swap_writepage]
  and [block_sync_page()][block_sync_page]:

```c
static struct address_space_operations swap_aops = {
        writepage: swap_writepage,
        sync_page: block_sync_page,
};
```

* When a page is being added to the swap cache, a slot is allocated with
  [get_swap_page()][get_swap_page], added to the page cache with
  [add_to_swap_cache()][add_to_swap_cache] and then marked dirty.

* When the page is next laundered, it will actually be written to backing
  storage on disk as the normal page cache would operate. Diagrammatically:

```
    Anonymous
   struct page
-----------------
/               /
\      ...      \
/               /
|---------------|
|  mapping=NULL |
|---------------|       ---------------------
/               /       | try_to_swap_out() |
\      ...      \------>| attempts to swap  |
/               /       |  pages out from   |
|---------------|       |   process space   |
|    count=5    |       ---------------------
|---------------|                 |
/               /                 |
\      ...      \                 v
/               /       ---------------------
-----------------       |  get_swap_page()  |
                        | allocates slot in |-----------------\
                        |  backing storage  |                 |
                        ---------------------                 |   -----
                                  |                           |   |   |
                                  |                           |   |---|
                                  v                           \-> | 1 |
                 -----------------------------------              |---|
                 |       add_to_swap_cache()       |              /   /
                 | adds the page to the page cache |              \   \
                 |    using swapper_space as a     |              /   /
                 |  struct address_space. A dirty  |              -----
                 |   page will now sync to swap    |            Swap map
                 -----------------------------------
                                  |                       /---------\
                                  |                       |         |
                                  v                       |         v
                              Anonymous                   |     ---------
                             struct page                  |     |   |   |
                          -----------------               |     ----|----
                          /               /               |         v
                          \      ...      \               |     ---------
                          /               /               |     |   |   |
                          |---------------|               |     ----|----
                          |   mapping =   |               |         v
                          | swapper_space |               |     ---------
                          |---------------|               |     |       |
                          /               /               |     ---------
                          \      ...      \---------------/    Page Cache
                          /               /                  (inactive_list)
                          |---------------|
                          |    count=4    |
                          |---------------|
                          /               /
                          \      ...      \
                          /               /
                          -----------------
```

* Subsequent swapping of the page from shared PTEs results in a call to
  [swap_duplicate()][swap_duplicate] which simply increments the reference to
  the slot in the `swap_map`.

* If the PTE is marked dirty by the hardware as the result of a write, the bit
  is cleared and its [struct page][page] is marked dirty via
  [set_page_dirty()][set_page_dirty] so the on-disk copy will be synced before
  the page is dropped. This ensures that until all references to a page have
  been dropped, a check will be made to ensure the data on disk matches the data
  in the page frame.

* When the reference count to the page reaches 0, the page is eligible to be
  dropped from the page cache and the swap map count will equal the count of the
  number of PTEs the on-disk slot belongs to so the slot won't be freed
  prematurely. It is laundered then finally dropped with the same LRU ageing and
  logic described in chapter 10.

* If, on the other hand, a page fault occurs for a page that is swapped out,
  [do_swap_page()][do_swap_page] checks to see if the page exists in the swap
  cache by calling [lookup_swap_cache()][lookup_swap_cache] - if it exists, the
  PTE is updated to point to the page frame, the page reference count is
  incremented and the swap slot is decremented with [swap_free()][swap_free].

### 11.5 Reading Pages From Backing Storage

* The principal function used for reading in pages is
  [read_swap_cache_async()][read_swap_cache_async] which is mainly called during
  page faulting:

```c
/*
 * Locate a page of swap in physical memory, reserving swap cache space
 * and reading the disk if it is not already cached.
 * A failure return means that either the page allocation failed or that
 * the swap entry is no longer in use.
 */
struct page * read_swap_cache_async(swp_entry_t entry)
{
        struct page *found_page, *new_page = NULL;
        int err;

        do {
                /*
                 * First check the swap cache.  Since this is normally
                 * called after lookup_swap_cache() failed, re-calling
                 * that would confuse statistics: use find_get_page()
                 * directly.
                 */
                found_page = find_get_page(&swapper_space, entry.val);
                if (found_page)
                        break;

                /*
                 * Get a new page to read into from swap.
                 */
                if (!new_page) {
                        new_page = alloc_page(GFP_HIGHUSER);
                        if (!new_page)
                                break;          /* Out of memory */
                }

                /*
                 * Associate the page with swap entry in the swap cache.
                 * May fail (-ENOENT) if swap entry has been freed since
                 * our caller observed it.  May fail (-EEXIST) if there
                 * is already a page associated with this entry in the
                 * swap cache: added by a racing read_swap_cache_async,
                 * or by try_to_swap_out (or shmem_writepage) re-using
                 * the just freed swap entry for an existing page.
                 */
                err = add_to_swap_cache(new_page, entry);
                if (!err) {
                        /*
                         * Initiate read into locked page and return.
                         */
                        rw_swap_page(READ, new_page);
                        return new_page;
                }
        } while (err != -ENOENT);

        if (new_page)
                page_cache_release(new_page);
        return found_page;
}
```

* This does the following:

1. The function starts by searching the swap cache via
   [find_get_page()][find_get_page]. It doesn't use the usual
   [lookup_swap_cache()][lookup_swap_cache] swap cache search function as that
   updates statistics on number of searches performed however we are likely to
   search multiple times so `find_get_page()` makes more sense here.

2. If the page isn't already in the swap cache, allocate one using
   [alloc_page()][alloc_page] and add it to swap cache using
   [add_to_swap_cache()][add_to_swap_cache]. If, however, a page _is_ found,
   skip to step 5.

3. If the page cannot be added to the swap cache, it'll be searched again in
   case another process put the data there in the meantime.

4. The data is read from backing storage via [rw_swap_page()][rw_swap_page]
   (discussed in 11.7), and returned to the user.

5. If the page was found in the cache and _didn't_ need
   allocating,[page_cache_release()][page_cache_release] is called to decrement
   the reference count accumulated via `find_get_page()`.

### 11.6 Writing Pages to Backing Storage

* When any page is being written to disk the
  [struct address_space][address_space]`->a_ops` field is used to determine the
  appropriate 'write-out' function.

* In the case of backing storage, the `address_space` is
  [swapper_space][swapper_space] and the swap operations are contained in
  [swap_aops][swap_aops] which uses [swap_writepage()][swap_writepage] for its
  write-out function.

* `swap_writepage()` behaves differently depending on whether the writing
  process is the last user of the swap cache page or not - it determines this
  via [remove_exclusive_swap_page()][remove_exclusive_swap_page] which simply
  checks the page count with [pagecache_lock][pagecache_lock] held - if no other
  process is mapping the page it is removed from the swap cache and freed.

* If `remove_exclusive_swap_page()` removed the page from the swap cache and
  freed it, [swap_writepage()][swap_writepage] unlocks the page because it is no
  longer in use.

* If the page still exists in the swap cache, [rw_swap_page()][rw_swap_page] is
  called which writes the data to the backing storage.

### 11.7 Reading/Writing Swap Area Blocks

* The top-level function for reading/writing to the swap area is
  [rw_swap_page()][rw_swap_page] which ensures that all operations are performed
  through the swap cache to prevent lost updates.

* `rw_swap_page()` invokes [rw_swap_page_base()][rw_swap_page_base] in turn that
  does the heavy lifting:

```c
/*
 * Reads or writes a swap page.
 * wait=1: start I/O and wait for completion. wait=0: start asynchronous I/O.
 *
 * Important prevention of race condition: the caller *must* atomically
 * create a unique swap cache entry for this swap page before calling
 * rw_swap_page, and must lock that page.  By ensuring that there is a
 * single page of memory reserved for the swap entry, the normal VM page
 * lock on that page also doubles as a lock on swap entries.  Having only
 * one lock to deal with per swap entry (rather than locking swap and memory
 * independently) also makes it easier to make certain swapping operations
 * atomic, which is particularly important when we are trying to ensure
 * that shared pages stay shared while being swapped.
 */

static int rw_swap_page_base(int rw, swp_entry_t entry, struct page *page)
{
        unsigned long offset;
        int zones[PAGE_SIZE/512];
        int zones_used;
        kdev_t dev = 0;
        int block_size;
        struct inode *swapf = 0;

        if (rw == READ) {
                ClearPageUptodate(page);
                kstat.pswpin++;
        } else
                kstat.pswpout++;

        get_swaphandle_info(entry, &offset, &dev, &swapf);
        if (dev) {
                zones[0] = offset;
                zones_used = 1;
                block_size = PAGE_SIZE;
        } else if (swapf) {
                int i, j;
                unsigned int block = offset
                        << (PAGE_SHIFT - swapf->i_sb->s_blocksize_bits);

                block_size = swapf->i_sb->s_blocksize;
                for (i=0, j=0; j< PAGE_SIZE ; i++, j += block_size)
                        if (!(zones[i] = bmap(swapf,block++))) {
                                printk("rw_swap_page: bad swap file\n");
                                return 0;
                        }
                zones_used = i;
                dev = swapf->i_dev;
        } else {
                return 0;
        }

         /* block_size == PAGE_SIZE/zones_used */
         brw_page(rw, page, dev, zones, block_size);
        return 1;
}
```

* Looking at how the function operates:

1. It checks if the operation is a read - if so, it clears the
   [struct page][page]`->uptodate` flag via
   [ClearPageUptodate()][ClearPageUptodate] because the page is clearly not up
   to date if I/O is required to fill it with data (the flag is set again if the
   page is successfully read from disk.)

2. The device for the swap partition of the inode for the swap file is acquired
   via [get_swaphandle_info()][get_swaphandle_info] - this information is needed
   by the block layer which will be performing the actual I/O.

3. If the swap area is a file [bmap()][bmap] is used to fill a local array with
   a list of all blocks in the filesystem that contain the page data. If the
   backing storage is a partition, only one page-sized block requires I/O and
   since no filesystem is involved `bmap()` is not required.

4. Either a swap partition or files can be used because `rw_swap_page_base()`
   uses [brw_page()][brw_page] to perform the actual disk I/O which can handle
   both cases generically.

5. All I/O that is performed is asynchronous so the function returns quickly -
   after the I/O is complete the block layer will unlock the page and any
   waiting process will wake up.

### 11.8 Activating a Swap Area

* Activating a swap area is conceptually quite simple - open the file, load the
  header information from disk, populate a
  [struct swap_info_struct][swap_info_struct], and add that to the swap list.

* [sys_swapon()][sys_swapon] is the function that activates the swap via a
  syscall - `long sys_swapon(const char * specialfile, int swap_flags)`. It
  takes the path to the special file (quite possibly a device) and some swap
  flags.

* Since this is 2.4.22 :) the 'Big Kernel Lock' (BKL) is held during the
  process, preventing any application from entering kernel space while the
  operation is being performed.

* The function operates as follows:

1. Find a free `swap_info_struct` in the [swap_info][swap_info] array and
   initialise it with default values.

2. Call [user_path_walk()][user_path_walk] (and subsesquently
   [__user_walk()][__user_walk]) to traverse the directory tree for the
   specified `specialfile` and populates a [struct nameidata][nameidata] with
   the available data on the file, such as the [struct dentry][dentry] data and
   [struct vfsmount][vfsmount] data on where the file is stored.

3. Populates the `swap_info_struct` fields relating to the dimensions of the
   swap area and how to find it. If the swap area is a partition, the block size
   will be aligned to the [PAGE_SIZE][PAGE_SIZE] before calculating the size. If
   it is a file, the information is taken directly from the
   [struct inode][inode].

4. Ensures the area is not already activated. If it isn't, it allocates a page
   from memory and reads the first page-sized slot from the swap area which
   contains information about the number of good slots and how to populate the
   swap map with bad entries (see 11.1 for more details.)

5. Allocate memory with [vmalloc()][vmalloc] for `swap_info_struct->swap_map`
   and initialise each entry with 0 for good slots and
   [SWAP_MAP_BAD][SWAP_MAP_BAD] otherwise (ideally the header will indicate v2
   as v1 was limited to swap areas of just under 128MiB for architectures with a
   4KiB page size like x86.)

6. After ensuring the header is valid fill in the remaining `swap_info_struct`
   fields like the maximum number of pages and the available good pages, then
   update the global statistics for [nr_swap_pages][nr_swap_pages] and
   [total_swap_pages][total_swap_pages].

7. The swap area is not fully active and initialised so insert it into the swap
   list in the correct position based on the priority of the newly activated
   area.

8. Release the BKL.

### 11.9 Deactivating a Swap Area

* In comparison to activating a swap area, deactivation is incredibly
  expensive. The main reason for this is that the swap can't simply be removed -
  every page that was swapped out has to now be swapped back in again.

* Just as there's no quick way to map a [struct page][page] to every PTE that
  references it, there is no quick way to map a swap entry to a PTE either - it
  requires all process page tables be traversed to find PTEs that reference the
  swap area that is being deactivated and swap them all in.

* Of course all this means swap deactivation will fail if the physical memory is
  not available.

* The function concerned with deactivating an area is
  [sys_swapoff()][sys_swapoff]. The function is mostly focused on updating the
  [struct swap_info_struct][swap_info_struct], leaving the heavy lifting of
  paging in each paged-out page via [try_to_unuse()][try_to_unuse], which is
  _extremely_ expensive.

* For each slot used in the `swap_info_struct`'s `swap_map` field, the page
  tables have to be traversed search for it, and in the worst case all page
  tables belonging to all [struct mm_struct][mm_struct]s have to be traversed.

* Broadly speaking `sys_swapoff()` performs the following:

1. Call [user_path_walk()][user_path_walk] to retrieve information about the
   special file to be deactivated, then take the BKL.

2. Remove the `swap_info_struct` from the swap list and update the global
   statistics on the number of swap pages
   available([nr_swap_pages][nr_swap_pages]) and the total number of swap
   entries ([total_swap_pages][total_swap_pages].) Once complete, the BKL can be
   released.

3. Call [try_to_unuse()][try_to_unuse] which will page in all pages from the
   swap area to be deactivated. The function loops through the swap map using
   [find_next_to_unuse()][find_next_to_unuse] to locate the next used swap slot
   (see below for more details on what `try_to_unuse()` does.)

4. If there was not enough available memory to page in all the entries, the swap
   area is reinserted back into the running system because it cannot simply be
   dropped. If, on the other hand, the process succeeded, the `swap_info_struct`
   is placed into an uninitialised state and the `swap_map` memory is freed with
   [vfree()][vfree].

* [try_to_unuse()][try_to_unuse] does the following:

1. Call [read_swap_cache_async()][read_swap_cache_async] to allocate a page for
   the slot saved on disk. Ideally, it exists in the swap cache already, but the
   page allocator will be called if it isn't.

2. Wait on the page to be fully paged in and lock it. Once looked, call
   [unuse_process()][unuse_process] for every process that has a PTE referencing
   the page. This function traverses the page table searching for the relevant
   PTE then updates it to point to the correct [struct page][page].If the page
   is a shared memory page with no remaining reference,
   [shmem_unuse()][shmem_unuse] is called instead.

3. Free all slots that were permanently mapped. It's believed that slots will
   never become permanently reserved, so a risk is taken here.

4. Delete the page from the swap cache to prevent
   [try_to_swap_out()][try_to_swap_out] from referencing a page in the event it
   still somehow has a reference in swap from a swap map.

## Chapter 12: Shared Memory Virtual Filesystem

* Sharing a region of memory backed by a file or device is simply a case of
  calling [mmap()][mmap] with the `MAP_SHARED` flag.

* Regardless, there are 2 important cases where an _anonymous_ region needs to
  be shared between processes:

1. When an `mmap()` with `MAP_SHARED` is used without file backing - these
   regions will be shared between a parent and a child process after a
   [fork()][fork] is executed.

2. A region explicitly sets up an anonymous mapping via (userland calls)
   [shmget()][shmget] and attaches to the virtual address space with
   [shmat()][shmat].

* When pages within a VMA are backed by a file on disk, the interface used is
  fairly straightforward. To read a page during a page fault, the required
  `nopage()` function is found in the
  [struct vm_area_struct][vm_area_struct]`->vm_ops` field and the required
  `writepage()` function is found in the
  [struct address_space_operations][address_space_operations] using
  `inode->i_mapping->a_ops` or alternatively `page->mapping->a_ops`.

* When normal file operations are taking place such as `mmap()`, [read()][read]
  and [write()][write], the [struct file_operations][file_operations] with the
  appropriate functions is found using `inode->i_fop`. The diagram shown in 4.2
  goes into more detail.

* The interface is nice and clean and is conceptually easy to understand, but it
  does not help _anonymous_ pages as there is no file backing there.

* To keep this nice interface, linux creates an artificial file backing for
  anonymous pages using a RAM-based filesystem where each VMA is backed by a
  file in this filesystem.

* Every inode in the artificial filesystem is placed on a linked list,
  [shmem_inodes][shmem_inodes], so they can be easily located.

* Overall this approach allows the same file-based interface to be used without
  treating anonymous pages as a special case.

* The filesystem comes in two variations in 2.4.22 - `shm` and `tmpfs`. They
  share core functionality and mainly different in what they're used for - `shm`
  is used by the kernel for creating file backings for anonymous pages and for
  backing regions created by [shmget()][shmget], the filesystem is mounted by
  [kern_mount()][kern_mount] so it's mounted internally and isn't visible to
  users.

* `tmpfs` is a temporary file system that is often mounted at `/tmp` publicly as
  as fast RAM-based temp fs. A secondary use for tmpfs is to mount `/tmp/shm`
  which allows [mmap()][mmap]-ing of files there in order to share information
  as a form of [IPC][ipc]. Regardless of the type of use, a sysadmin has to
  explicitly mount the tmpfs.

### 12.1 Initialising the Virtual Filesystem

* The virtual filesystem is initialised by [init_tmpfs()][init_tmpfs] either
  during system start or when the module is being loaded. This does the
  following:

1. Mounts `tmpfs` and `shm`, with `shm` mounted internally via
   [kern_mount()][kern_mount].

2. Calculates the maximum number of blocks and inodes that can exist in the
   filesystem.

3. As part of the registration, the function
   [shmem_read_super()][shmem_read_super] is used as a callback to populate a
   [struct super_block][super_block] with more information about the
   filesystems, such as making the block size equal to the page size.

* Every inode created in the filesystem will have a
  [struct shmem_inode_info][shmem_inode_info] associated with it containing
  private information specific to the filesystem.

* [SHMEM_I()][SHMEM_I] takes an inode as a parameter and returns a pointer to a
  `struct shmem_inode_info`, bridging the gap between the two. Taking a look at
  the latter:

```c
struct shmem_inode_info {
        spinlock_t              lock;
        unsigned long           next_index;
        swp_entry_t             i_direct[SHMEM_NR_DIRECT]; /* for the first blocks */
        void                    **i_indirect; /* indirect blocks */
        unsigned long           swapped;    /* data pages assigned to swap */
        unsigned long           flags;
        struct list_head        list;
        struct inode            *inode;
};
```

* Considered each field:

1. `lock` - Spinlock protecting the inode information from concurrent accesses.

2. `next_index` - Index of the last page being used in the file. This will be
   different from `inode->i_size` when a file is being truncated.

3. `i_direct` - direct block containing the first
   [SHMEM_NR_DIRECT][SHMEM_NR_DIRECT] swap vectors in use by the file (see
   12.4.1.)

4. `i_indirect` - Pointer to the first indirect block (see 12.4.1.)

5. `swapped` - Count of the number of pages belonging to the file that are
   currently swapped out.

6. `flags` - Currently only used to determine if the file belongs to shared
   region set by [shmget()][shmget]. It is set by specifying `SHM_LOCK` when
   calling `shmget()` and unlocked via `SHM_UNLOCK`.

7. `list` - List of all inodes used by the filesystem.

8. `inode` - Pointer to the parent inode.

### 12.2 Using shmem Functions

* Different structs contain pointers for `shmem`-specific functions. Regardless,
  in all cases `tmpfs` and `shm` share the same structs.

* For faulting in pages and writing them to backing storage, two structs - a
  [struct address_space_operations][address_space_operations],
  [shmem_aops][shmem_aops] and a
  [struct vm_operations_struct][vm_operations_struct],
  [shmem_vm_ops][shmem_vm_ops].

* Taking a look at the address space operations struct `shmem_aops`:

```c
static struct address_space_operations shmem_aops = {
        removepage:     shmem_removepage,
        writepage:      shmem_writepage,
#ifdef CONFIG_TMPFS
        readpage:       shmem_readpage,
        prepare_write:  shmem_prepare_write,
        commit_write:   shmem_commit_write,
#endif
};
```

* The most important function referenced here is
  [shmem_writepage()][shmem_writepage], which is called when a page is moved
  from the page cache to the swap cache.

* [shmem_removepage()][shmem_removepage] is called when apage is removed from
  the page cache so that the block can be reclaimed.

* [shmem_readpage()][shmem_readpage] is not used by `tmpfs`, but is provided so
  the [sendfile()][sendfile] system call can be used with tmpfs files.

* Similarly, [shmem_prepare_write()][shmem_prepare_write] and
  [shmem_commit_write()][shmem_commit_write] are also unused, but provided so
  that `tmpfs` can be used with with the loopback device.

* Taking a look at [struct shmem_vm_ops][shmem_vm_ops]:

```c
static struct vm_operations_struct shmem_vm_ops = {
        nopage:         shmem_nopage,
};
```

* `shmem_vm_ops` is used by anonymous VMAs as their
  [struct vm_operations_struct][vm_operations_struct] so that
  [shmem_nopage()][shmem_nopage] is called when a new page is faulted in.

* To perform operations on files and inodes a
  [struct file_operations][file_operations] and a
  [struct inode_operations][inode_operations] are required. The former is
  provided as [shmem_file_operations][shmem_file_operations]:

```c
static struct file_operations shmem_file_operations = {
        mmap:           shmem_mmap,
#ifdef CONFIG_TMPFS
        read:           shmem_file_read,
        write:          shmem_file_write,
        fsync:          shmem_sync_file,
#endif
};
```

* Three sets of `struct inode_operations` are provided -
  [shmem_inode_operations][shmem_inode_operations],
  [shmem_dir_inode_operations][shmem_dir_inode_operations], and a related pair -
  [shmem_symlink_inline_operations][shmem_symlink_inline_operations] [shmem_symlink_inode_operations][shmem_symlink_inode_operations] for handling
  file inodes, directory inodes and symlink inodes respectively.

* The two file operations supported are `truncate()` and `setattr()`:

```c
static struct inode_operations shmem_inode_operations = {
        truncate:        shmem_truncate,
        setattr:        shmem_notify_change,
};
```

* [shmem_truncate()][shmem_truncate] is used to truncate a file. since
  [shmem_notify_change()][shmem_notify_change] is called when file attributes
  change it's possible (amongst other things) for a file to be grown with
  `truncate()` and to use the global zero page as the data page.

* The directory [struct inode_operations][inode_operations] provides a number of
  functions:

```c
static struct inode_operations shmem_dir_inode_operations = {
#ifdef CONFIG_TMPFS
        create:         shmem_create,
        lookup:         shmem_lookup,
        link:           shmem_link,
        unlink:         shmem_unlink,
        symlink:        shmem_symlink,
        mkdir:          shmem_mkdir,
        rmdir:          shmem_rmdir,
        mknod:          shmem_mknod,
        rename:         shmem_rename,
#endif
};
```

* Finally we have the pair of symlink `struct inode_operations`s:

```c
static struct inode_operations shmem_symlink_inline_operations = {
        readlink:       shmem_readlink_inline,
        follow_link:    shmem_follow_link_inline,
};

static struct inode_operations shmem_symlink_inode_operations = {
        truncate:       shmem_truncate,
        readlink:       shmem_readlink,
        follow_link:    shmem_follow_link,
};
```

* The difference between the two `readlink()` and `follow_link()` functions is
  related to where the information is stored. A symlink inode does not require
  the private inode information stored in a
  [struct shmem_inode_info][shmem_inode_info].

* If the length of the symbolic link name is smaller than a `struct
  shmem_inode_info`, the space in the inode is used to store the name and
  [shmem_symlink_inline_operations][shmem_symlink_inline_operations] becomes the
  relevant [struct inode_operations][inode_operations] struct.

* Otherwise, a page is allocated via [shmem_getpage()][shmem_getpage], the
  symbolic link is copied to it and
  [shmem_symlink_inode_operations][shmem_symlink_inode_operations] is used.

* The inode symlink struct contains a `truncate()` function so the page will be
  reclaimed when the file is deleted.

* These structs ensure that the shmem equivalent of inode-related operations
  will be used when regions are backed by virtual files. The majority of the VM
  sees no difference between pages backed by a real file and those backed by
  virtual files.

### 12.3 Creating files in tmpfs

* Because `tmpfs` is mounted as a proper filesystem that is visible to the user
  it must support directory inode operations, as provided for by
  [shmem_dir_inode_operations][shmem_dir_inode_operations], discussed in the
  previous section.

* The implementation of most of the functions used by
  `shmem_dir_inode_operations` are rather small but they are all very
  interconnected.

* The majority of the inode fields provided by the VFS are filled by
  [shmem_get_inode()][shmem_get_inode].

* When creating a new file [shmem_create()][shmem_create] is used - this is a
  small wrapper around [shmem_mknod()][shmem_mknod] which is called with the
  `S_IFREG` flag set so a regular file will be created.

* Again `shmem_mknod()` is really just a wrapper around `shmem_get_inode()`
  which creates an inode and fills in the [struct inode][inode] fields as
  required.

* The 3 most interesting fields that are populated are
  `inode->i_mapping->a_ops`, `inode->i_op` and `inode->i_fop` fields.

* After the inode has been created, `shmem_mknod()` updates the directory
  inode's `size` and `mtime` statistics before creating the new inode.

* Files are created differently in `shm` despite the fact the filesystems are
  basically the same - this is looked at in more detail in 12.7.

### 12.4 Page Faulting Within a Virtual File

* When a page fault occurs, [do_no_page()][do_no_page] will invoke
  [struct vm_area_struct][vm_area_struct]`->vm_ops->nopage` if it exists. In the
  case of the virtual file filesystem, this means the function
  [shmem_nopage()][shmem_nopage] will be called.

* The core of `shmem_nopage()` is [shmem_getpage()][shmem_getpage] which is
  responsible for either allocating a new page or finding it in swap.

* This overloading of fault types is unusual because
  [do_swap_page()][do_swap_page] is usually responsible for locating pages that
  have been moved to the swap cache or backing storage, using information
  encoded within the relevant PTE.

* In the case of a virtual file, the relevant PTEs are set to 0 when they are
  moved to the swap cache. The inode's private filesystem data stores direct and
  indirect block information which is used to locate the pages later.

* Other than this, the overall operation is very similar to standard page
  faulting.

#### 12.4.1 Locating Swapped Pages

* When a page has been swapped out, a [struct swp_entry_t][swp_entry_t] will
  contain the information needed to locate the page again - instead of using
  PTEs to find it, we store this in the inode's filesystem-specific private
  information (LS - there doesn't appear to be a private data field for an
  inode, perhaps this is stored elsewhere?)

shmem_swp_alloc


* When faulting, [shmem_swp_alloc()][shmem_swp_alloc] is used to locate the swap
  entry (LS - couldn't be certain, book refers to `shmem_alloc_entry()` which
  doesn't exist, this seem most likely to be the correct function.)

* Its basic task is to do some sanity checks (ensuring
  `shmem_inode_info->next_index` always points to the page index at the end of
  the virtual file) before calling [shmem_swp_entry()][shmem_swp_entry] to
  search for the swap vector within the inode information, and if we can't find
  it allocate a new one.

* The first [SHMEM_NR_DIRECT][SHMEM_NR_DIRECT] entries are stored in
  [struct inode][inode]`->i_direct`. This means that for i386, files smaller
  than 64KiB (`SHMEM_NR_DIRECT * PAGE_SIZE`) will not need to use indirect
  blocks. Larger files must use indirect blocks starting with
  `inode->i_indirect`.

* The initial indirect block (`inode->i_indirect`) is broken into two halves -
  the first containing pointers to 'doubly-indirect' blocks, and the second half
  containing pointers to 'triply-indirect' blocks.

* The doubly-indirect blocks are pages containing swap vectors
  ([struct swp_entry_t][swp_entry_t]), and the triply-indirect blocks contain
  pointers to pages, which in turn are filled with swap vectors.

* This relationship means that the maximum number of pages in a virtual file
  ([SHMEM_MAX_INDEX][SHMEM_MAX_INDEX]) is defined as follows:

```c
#define SHMEM_MAX_INDEX  (SHMEM_NR_DIRECT + (ENTRIES_PER_PAGEPAGE/2) * (ENTRIES_PER_PAGE+1))
```

* Looking at how indirect blocks are traversed, diagrammatically:

```
                                                     double-
                                                 indirect block
                                                      pages
                                            /--->--------------- /------>---------------
                                            |    | swp_entry_t | |       | swp_entry_t |
                                            |    |-- - -- - -- | |       |-- - -- - -- |
                                            |    | swp_entry_t | |       | swp_entry_t |
                                            |    |-- - -- - -- | |       |-- - -- - -- |
                                            |    | swp_entry_t | |       | swp_entry_t |
           ---------------------            |    |-- - -- - -- | |       |-- - -- - -- |
           | inode->i_indirect |            |    | swp_entry_t | |       | swp_entry_t |
           ---------------------            |    --------------- |       ---------------
                     |                      |                    |
                     |                      | /->--------------- | /---->---------------
                     v indirect block page  | |  | swp_entry_t | | |     | swp_entry_t |
                     ---------------------- | |  |-- - -- - -- | | |     |-- - -- - -- |
                   ^ |                   ---/ |  | swp_entry_t | | |     | swp_entry_t |
ENTRIES_PER_PAGE/2 | |-- - -- - -- - -- - |   |  |-- - -- - -- | | |     |-- - -- - -- |
                   v |                   -----/  | swp_entry_t | | |     | swp_entry_t |
                     |--------------------|      |-- - -- - -- | | |     |-- - -- - -- |
                   ^ |                   -----\  | swp_entry_t | | |     | swp_entry_t |
ENTRIES_PER_PAGE/2 | |-- - -- - -- - -- - |   |  --------------- | |     ---------------
                   v |                   ---\ |                  | |
                     ---------------------- | |      triple-     | |  /->---------------
                                            | |  indirect block  | |  |  | swp_entry_t |
                                            | |       pages      | |  |  |-- - -- - -- |
                                            | \->--------------- | |  |  | swp_entry_t |
                                            |    |            ---/ |  |  |-- - -- - -- |
                                            |    |-- - -- - -- |   |  |  | swp_entry_t |
                                            |    |            -----/  |  |-- - -- - -- |
                                            |    |-- - -- - -- |      |  | swp_entry_t |
                                            |    |            --------/  ---------------
                                            |    |-- - -- - -- |
                                            |    |            ---------->---------------
                                            |    ---------------         | swp_entry_t |
                                            |                            |-- - -- - -- |
                                            \--->---------------         | swp_entry_t |
                                                 |            --->       |-- - -- - -- |
                                                 |-- - -- - -- |         | swp_entry_t |
                                                 |            --->       |-- - -- - -- |
                                                 |-- - -- - -- |         | swp_entry_t |
                                                 |            --->       ---------------
                                                 |-- - -- - -- |
                                                 |            --->
                                                 ---------------
```

#### 12.4.2 Writing Pages to Swap

* The function [shmem_writepage()][shmem_writepage] is the function registered
  in the filesystem's
  [struct address_space_operations][address_space_operations] for writing pages
  to the swap. It's responsible for simply moving the page from the page cache
  to the swap cache, implemented as follows:

1. Record the current [struct page][page]`->mapping` and information about the
   inode.

2. Allocate a free slot in the backing storage with
   [get_swap_page()][get_swap_page].

3. Allocate a [struct swp_entry_t][swp_entry_t] via
   [shmem_swp_entry()][shmem_swp_entry].

4. Remove the page from the page cache.

5. Add the page to the swap cache. If it fails, free the swap slot add back the
   page cache, and try again.

### 12.5 File Operations in tmpfs

* Four operations are supported with virtual files - `mmap()`, `read()`,
  `write()` and `fsync()` as stored in
  [shmem_file_operations][shmem_file_operations] (discussed in 12.2.)

* There's nothing too unusual here:

1. `mmap()` is implemented by [shmem_mmap()][shmem_mmap], and it simply updates
  the VMA that is managing the mapped region.

2. `read()` is implemented by [shmem_file_read()][shmem_file_read] which
   performs the operation of copying bytes from the virtual file to a userspace
   buffer, faulting in pages as necessary.

3. `write()` is implemented by [shmem_file_write()][shmem_file_write] and is
   essentially the same deal as `read()`.

4. `fsync()` is implemented by [shmem_sync_file()][shmem_sync_file] but is
   essentially a NULL operation because it doesn't do anything and simply
   returns 0 for success - because the files only exist in RAM, they don't need
   to be synchronised with any disk.

### 12.6 Inode Operations in tmpfs

* The most complex operation that is supported for inodes is truncation and
  involves four distinct stages:

1. [shmem_truncate()][shmem_truncate] - truncates a partial page at the end of
   the file and continually calls
   [shmem_truncate_indirect()][shmem_truncate_indirect] until the file is
   truncated to the proper size. Each call to `shmem_truncate_indirect()` will
   only process one indirect block at each pass, which is why it may need to be
   called multiple times.

2. [shmem_truncate_indirect()][shmem_truncate_indirect] - Deals with both doubly
   and triply-indrect blocks. It finds the next indirect block that needs to be
   truncated to pass to stage 3, and will contain pointers to pages which will
   in turn contain swap vectors.

3. [shmem_truncate_direct()][shmem_truncate_direct] - Selects a range of swap
   vectors from a page that need to be truncated and passes that range to
   [shmem_free_swp()][shmem_free_swp].

4. [shmem_free_swp()][shmem_free_swp] - Frees entries via
   [free_swap_and_cache()][free_swap_and_cache] which frees both the swap entry
   and the page containing data.

* The linking and unlinking of files is very simple because most of the work is
  performed by the filesystem layer.

* To link a file, the directory inode size is incremented, and `ctime` and
  `mtime` of the affected inodes are updated and the number of links to the
  inode being linked is incremented. A reference to the new
  [struct dentry][dentry] is then created via [dget()][dget] and
  [d_instantiate()][d_instantiate].

* To unlink a file the same inode statistics are updated before decrementing the
  references to the `struct dentry` only using [dput()][dput] (and subsequently
  [iput()][iput]) which will clear up the inode when its reference count hits
  zero.

* Creating a directory will use [shmem_mkdir()][shmem_mkdir] which in turn uses
  [shmem_mknod()][shmem_mknod] with the `S_IFDIR` flag set, before incrementing
  the parent directory [struct inode][inode]'s `i_nlink` counter.

* The function [shmem_rmdir()][shmem_rmdir] will delete a directory, first
  making sure it's empty via [shmem_empty()][shmem_empty]. If it is, the
  function then decrements the parent directory `struct inode->i_nlink` count
  and calls [shmem_unlink()][shmem_unlink] to remove it.

### 12.7 Setting Up Shared Regions

* A shared region is backed by a file created in `shm`.

* There are two cases where a new file will be created - during the setup of a
  shared region via (userland) [shmget()][shmget] or when an anonymous region is
  set up via (userland) [mmap()][mmap] with the `MAP_SHARED` flag specified.

* Both approaches ultimately use [shmem_file_setup()][shmem_file_setup] to
  create a file, which simply creates a new [struct dentry][dentry] and
  [struct inode][inode], fills in the relevant details and instantiates them.

* Because the file system is internal, the names of the files created do _not_
  have to be unique as the files are always located by inode, not name. As a
  result, [shmem_zero_setup()][shmem_zero_setup] always creates a file
  `"dev/zero"` which is how it shows up in `/proc/<pid>/maps`.

* Files created by `shmget()` are named `SYSV<NN>` where `<NN>` is the key that
  is passed as a parameter to `shmget()`.

### 12.8 System V IPC

* To avoid spending too long on this, we focus only on [shmget()][shmget] and
  [shmat()][shmat] and how they are affected by the VM.

* `shmget()` is implemented via [sys_shmget()][sys_shmget]. It performs sanity
  checks, sets up the IPC-related data structures and a new segment via
  [newseg()][newseg] which creates the file in `shmfs` with
  [shmem_file_setup()][shmem_file_setup] as previously discussed.

* `shmat()` is implemented via [sys_shmat()][sys_shmat]. It does sanity checks,
  acquires the appropriate descriptor and hands over to [do_mmap()][do_mmap] to
  map the shared region into the process address space.

* `sys_shmat()` is responsible for ensuring that VMAs don't overlap if the
  caller erroneously specifies to do so.

* The [struct shmid_kernel][shmid_kernel]`->shm_nattch` counter is maintained by
  [shm_vm_ops][shm_vm_ops], a
  [struct vm_operations_struct][vm_operations_struct] that registers `open()`
  and `close()` callbacks of [shm_open()][shm_open] and [shm_close()][shm_close]
  respectively.

* `shm_close()` is also responsible for destroying shared regions if the
  `SHM_DEST` flag is specified and the `shm_nattch` counter reaches zero.

## Chapter 13: Out of Memory Management

* The Out-of-Memory (OOM) manager is pretty straightforward as it has one simple
  task: check if there is enough memory to satisfy requests, if not verify the
  system is truly out of memory and if so, select a process to kill.

* The OOM killer is a controversial part of the VM and there has been much
  discussion about removing it but yet it remains.

### 13.1 Checking Available Memory

* For certain operations such as expanding the heap with [brk()][brk] or
  remapping an address space with [mremap()][mremap], the system will check if
  there is enough available memory to satisfy a request. Note that this is
  separate from the [out_of_memory()][out_of_memory] path that is covered in the
  next section, rather it is an attempt to _avoid_ the system being in a state
  of OOM if at all possible.

* When checking available memory, the number of requested pages is passed as a
  parameter to [vm_enough_memory()][vm_enough_memory]. Unless the sysadmin has
  specified that the system should overcommit memory, the amount of memory will
  be checked.

* To determine how many pages are potentially available, linux sums up the following:

1. __Total page cache__ - Easily reclaimed.

2. __Total free pages__ - They are already available.

3. __Total free swap pages__ - Because userspace processes may be paged out.

4. __Total pages managed by [swapper_space][swapper_space]__ - This
   double-counts free swap pages, but is somewhat mitigated by the fact that
   slots are sometimes reserved but not used.

5. __Total pages used by the [struct dentry][dentry] cache__ - Easily reclaimed.

6. __Total pages used by the [struct inode][inode] cache__ - Easily reclaimed.

* If the total number of pages added here is sufficient for the request,
  [vm_enough_memory()][vm_enough_memory] returns true otherwise it returns false
  and `-ENOMEM` is returned to userspace.

### 13.2 Determining OOM Status

* When the machine is low on memory, old page frames wil be reclaimed (see
  chapter 10) but, despite reclaiming pages, it may find it was unable to free
  enough to satisfy a request even when scanning at highest priority.

* If the system does fail to free page frames, [out_of_memory()][out_of_memory]
  is called to see if the system is _actually_ out of memory and if it is, kills
  a process:

```c
/**
 * out_of_memory - is the system out of memory?
 */
void out_of_memory(void)
{
        static unsigned long first, last, count, lastkill;
        unsigned long now, since;

        /*
         * Enough swap space left?  Not OOM.
         */
        if (nr_swap_pages > 0)
                return;

        now = jiffies;
        since = now - last;
        last = now;

        /*
         * If it's been a long time since last failure,
         * we're not oom.
         */
        last = now;
        if (since > 5*HZ)
                goto reset;

        /*
         * If we haven't tried for at least one second,
         * we're not really oom.
         */
        since = now - first;
        if (since < HZ)
                return;

        /*
         * If we have gotten only a few failures,
         * we're not really oom.
         */
        if (++count < 10)
                return;

        /*
         * If we just killed a process, wait a while
         * to give that task a chance to exit. This
         * avoids killing multiple processes needlessly.
         */
        since = now - lastkill;
        if (since < HZ*5)
                return;

        /*
         * Ok, really out of memory. Kill something.
         */
        lastkill = now;
        oom_kill();

reset:
        first = now;
        count = 0;
}
```

* The reason there are a series of checks here to see whether the system is out
  of memory is that the system may just be waiting for I/O to complete or pages
  to be swapped out to backing storage or some other similar condition - given
  this, we want to avoid killing a process as much as we can which is why there
  are checks in the first instance.

### 13.3 Selecting a Process

* [select_bad_process()][select_bad_process] determines the process to kill by
  stepping through each running task and calculating how suitable it is for
  killing via the function [badness()][badness], which determines this via:

```
badness_for_task = total_vm_for_task/(cpu_time_in_seconds^0.5 * cpu_time_in_minutes^0.25)
```

* The square roots are approximated by [int_sqrt()][int_sqrt].

* This has been chosen to prefer a process that is using a large amount of
  memory but is not that long-lived. Long-lived processes are unlikely to be the
  cause of memory shortage.

* If the process is a root process or has `CAP_SYS_ADMIN` capabilities, the
  points are divided by 4 since it is assumed that privileged processes are
  well-behaved.

* Similarly, if the process has `CAP_SYS_RAWIO` capabilities (access to raw
  device), the points are further divided by 4 because it's not a good idea to
  kill a process that has direct access to hardware.

### 13.4 Killing the Selected Process

* After a task is selected, the list is walked again and each process that
  shares the same [struct mm_struct][mm_struct] as the selected process
  (i.e. threads) is sent a signal.

* If the process has `CAP_SYS_RAWIO` capabilities, a `SIGTERM` signal is sent to
give the process a chance of exiting cleanly. Otherwise a `SIGKILL` is sent.

### 13.5 Is That It?

* Yep :)

[4level]:https://lwn.net/Articles/117749/
[BUFCTL_END]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L139
[BUG]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L91
[CREATE_MASK]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L109
[ClearPageActive]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L405
[ClearPageDirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L319
[ClearPageError]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L393
[ClearPageLaunder]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L327
[ClearPageReferenced]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L396
[ClearPageReserved]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L418
[ClearPageUptodate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L316
[DEF_PRIORITY]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L35
[FIXADDR_START]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L106
[FIXADDR_TOP]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L104
[FIX_KMAP_BEGIN]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L72
[FIX_KMAP_END]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L73
[GET_PAGE_CACHE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L326
[GET_PAGE_SLAB]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L328
[INIT_MM]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/sched.h#L238
[L1_CACHE_BYTES]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/cache.h#L11
[LAST_PKMAP]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L51
[LockPage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L321
[MARK_USED]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L174
[MAX_DMA_ADDRESS]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/dma.h#L76
[MAX_NR_NODES]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L219
[MAX_NR_ZONES]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L98
[MAX_ORDER]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L17
[MAX_SWAPFILES]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L11
[MAX_SWAP_BADPAGES]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L48
[NR_CPUS]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/threads.h#L12
[PAE]:https://en.wikipedia.org/wiki/Physical_Address_Extension
[PAGE_ALIGN]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L66
[PAGE_MASK]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L7
[PAGE_OFFSET]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L128
[PAGE_SHIFT]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L5
[PAGE_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L6
[PGDIR_MASK]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L139
[PGDIR_SHIFT/2lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L8
[PGDIR_SHIFT/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L14
[PGDIR_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L138
[PKMAP_BASE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L49
[PMD_MASK]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L137
[PMD_SHIFT/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L21
[PMD_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L136
[POISON_BYTE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L300
[POOL_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L197
[PTRS_PER_PGD/2lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L9
[PTRS_PER_PGD/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L15
[PTRS_PER_PMD/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L22
[PTRS_PER_PTE/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L27
[PageActive]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L403
[PageChecked]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L323
[PageClearSlab]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L400
[PageDirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L317
[PageError]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L391
[PageHighMem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L412
[PageLRU]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L407
[PageLaunder]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L325
[PageLocked]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L320
[PageReferenced]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L394
[PageReserved]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L401
[PageSetSlab]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L399
[PageSlab]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L398
[PageSwapCache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L531
[Page_Uptodate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L310
[REAP_PERFECT]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L104
[REAP_SCANLEN]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L103
[SET_PAGE_CACHE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L325
[SET_PAGE_SLAB]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L327
[SHMEM_I]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/shmem_fs.h?#L39
[SHMEM_MAX_INDEX]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L44
[SHMEM_NR_DIRECT]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/shmem_fs.h?#L6
[SWAPFILE_CLUSTER]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L34
[SWAP_CLUSTER_MAX]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L56
[SWAP_MAP_BAD]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L59
[SWAP_MAP_MAX]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L58
[SWP_ENTRY]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L354
[SWP_OFFSET]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L353
[SWP_TYPE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L352
[SWP_USED]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L53
[SWP_WRITEOK]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L54
[SetPageActive]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L404
[SetPageChecked]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L324
[SetPageDirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L318
[SetPageError]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L392
[SetPageLaunder]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L326
[SetPageReferenced]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L395
[SetPageReserved]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L417
[SetPageUptodate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L311
[TASK_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/processor.h#L262
[TestClearPageLRU]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L409
[TestSetPageLRU]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L408
[USER_PTRS_PER_PGD]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L141
[UnlockPage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L309
[VMALLOC_END]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L165
[VMALLOC_RESERVE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L129
[VMALLOC_START]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L161
[ZONE_SHIFT]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L334
[_PAGE_ACCESSED]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L192
[_PAGE_DIRTY]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L193
[_PAGE_PRESENT]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L187
[_PAGE_PROTNONE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L197
[_PAGE_RW]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L188
[_PAGE_USER]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L189
[__FIXADDR_SIZE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L105
[__PAGE_OFFSET]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L81
[__VMALLOC_RESERVE]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L87
[__add_to_page_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L653
[__alloc_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L326
[__alloc_bootmem_core]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L144
[__alloc_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L344
[__alloc_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L327
[__block_commit_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1676
[__block_prepare_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1571
[__change_bit]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/bitops.h#L90
[__constant_copy_from_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L675
[__constant_copy_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L442
[__copy_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L374
[__copy_user_zeroing]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L396
[__ex_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/entry.S#L126
[__exit_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/exit.c#L305
[__find_page_nolock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L443
[__fix_to_virt]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L108
[__flush_tlb]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L38
[__free_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L471
[__free_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L451
[__free_pages_ok]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L82
[__generic_copy_from_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/lib/usercopy.c#L54
[__get_dma_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L457
[__get_free_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L454
[__get_free_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L428
[__init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/init.h#L78
[__init_begin]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/vmlinux.lds#L42
[__init_end]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/vmlinux.lds#L53
[__insert_vm_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L1174
[__kmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L65
[__kmem_cache_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1338
[__kmem_cache_shrink]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L945
[__mk_pte/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L95
[__pa]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L132
[__pgd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L60
[__pgprot]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L61
[__pmd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L59
[__pmd_offset]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L336
[__pte]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L58
[__pte_offset]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L340
[__start___ex_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/vmlinux.lds#L23
[__stop___ex_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/vmlinux.lds#L25
[__user_walk]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/namei.c#L850
[__va]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L133
[__vma_link]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L329
[__vma_link_file]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L304
[__vma_link_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L282
[__vma_link_rb]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L297
[__vmalloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L261
[__vmalloc_area_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L155
[__wait_on_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L849
[__wait_queue_head]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/wait.h#L77
[_alloc_pages/numa]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/numa.c#L95
[_alloc_pages/uma]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L245
[_page_hashfn]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L62
[access_ok]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L80
[activate_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap.c#L47
[active_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L29
[add_page_to_inode_queue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L85
[add_to_page_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L667
[add_to_page_cache_unique]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L675
[add_to_swap_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L70
[add_wait_queue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L42
[address_space]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L406
[address_space_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L385
[alloc_area_pmd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L132
[alloc_area_pte]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L95
[alloc_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L326
[alloc_bootmem_low]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L40
[alloc_bootmem_low_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L44
[alloc_bootmem_low_pages_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L57
[alloc_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L53
[alloc_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L344
[alloc_bootmem_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L42
[alloc_bootmem_pages_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L55
[alloc_bounce_bh]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L369
[alloc_bounce_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L333
[alloc_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L449
[alloc_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L439
[allocate_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L227
[apic]:https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller
[arch_get_unmapped_area]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L615
[arch_set_page_uptodate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L305
[badness]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/oom_kill.c#L40
[balance_classzone]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L253
[block_commit_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1945
[block_prepare_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1933
[block_read_full_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1723
[block_sync_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L2892
[block_write_full_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L2047
[bmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/inode.c#L1115
[bootmem.c]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c
[bootmem_bootmap_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L32
[bootmem_data]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/bootmem.h#L25
[bounce_end_io]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L244
[bounce_end_io_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L324
[bounce_end_io_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L319
[brk]:http://man7.org/linux/man-pages/man2/brk.2.html
[brw_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L2397
[btc]:https://pdos.csail.mit.edu/6.828/2010/readings/i386/BTC.htm
[buffer_head]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L245
[buffer_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L2799
[build_zonelists]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L589
[cache_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L357
[cache_sizes]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L337
[cache_sizes_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L331
[cachetlb-doc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/Documentation/cachetlb.txt
[cc_data]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L180
[cc_entry]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L178
[ccupdate_struct_s]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L868
[cflgs-off-set]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L703
[check_pgt_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L136
[clear_page_tables]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L146
[clear_user_highpage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/highmem.h#L84
[clear_user_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L33
[clock-algorithm]:https://en.wikipedia.org/wiki/Page_replacement_algorithm#Clock
[clock_searchp]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L372
[clone]:http://man7.org/linux/man-pages/man2/clone.2.html
[contig_page_data]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/numa.c#L15
[copy_from_high_bh]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L215
[copy_from_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L732
[copy_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L315
[copy_to_high_bh_irq]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L228
[copy_to_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L751
[copy_user_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L34
[cow]:https://en.wikipedia.org/wiki/Copy-on-write
[cpucache_s]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L173
[cr0-jump]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L104
[create_bounce]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L405
[d_instantiate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/dcache.c#L651
[daemonize]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/sched.c#L1326
[del_page_from_active_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L193
[del_page_from_inactive_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L200
[dentry]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/dcache.h#L67
[dget]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/dcache.h#L244
[discard_bh_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1368
[do_anonymous_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1190
[do_brk]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L1033
[do_ccupdate_local]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L874
[do_generic_file_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1349
[do_mlock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L148
[do_mmap2]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/sys_i386.c#L43
[do_mmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L557
[do_mmap_pgoff]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L393
[do_mremap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mremap.c#L219
[do_munmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L924
[do_no_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1245
[do_page_fault]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/fault.c#L140
[do_swap_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1117
[do_wp_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L948
[down_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/rwsem.h#L43
[down_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/rwsem.h#L65
[dput]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/dcache.h#L268
[drain_cpu_caches]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L885
[e820]:https://en.wikipedia.org/wiki/E820
[emergency_bhs]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L208
[emergency_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L202
[emergency_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L205
[empty_zero_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L407
[enable_all_cpucaches]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1714
[enable_cpucache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1693
[enter_lazy_tlb]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/mmu_context.h#L17
[exception_table_entry]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L123
[execve]:http://man7.org/linux/man-pages/man2/execve.2.html
[exit]:http://man7.org/linux/man-pages/man2/exit.2.html
[exit_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/exit.c#L322
[exit_mmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L1127
[file]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L563
[file_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L858
[filemap_nopage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1994
[find_get_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L75
[find_max_low_pfn]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L895
[find_max_pfn]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L873
[find_next_to_unuse]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L475
[find_vma]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L661
[find_vma_intersection]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L673
[find_vma_prepare]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L246
[find_vma_prev]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L696
[first-fit]:http://www.memorymanagement.org/mmref/alloc.html#mmref-alloc-first-fit
[fixed_addresses]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/fixmap.h#L51
[fixrange_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L167
[flush_all_zero_pkmaps]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L42
[flush_cache_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L29
[flush_cache_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L31
[flush_cache_range]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L30
[follow_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L405
[for_each_pgdat]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L172
[fork]:http://man7.org/linux/man-pages/man2/fork.2.html
[free_all_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L321
[free_all_bootmem_core]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L245
[free_all_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L299
[free_area_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L838
[free_area_init_core]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L684
[free_area_init_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/numa.c#L61
[free_area_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L22
[free_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L316
[free_bootmem_core]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L103
[free_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L294
[free_initmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L580
[free_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L228
[free_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L472
[free_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L457
[free_pages_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L481
[free_pgd_slow]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L94
[free_pgtables]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L872
[free_swap_and_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L332
[g_cpucache_up]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L381
[generic_file_direct_IO]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1581
[generic_file_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1695
[generic_file_readahead]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1222
[generic_file_vm_ops]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L2243
[get_free_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L463
[get_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L196
[get_pgd_fast]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L74
[get_pgd_slow]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L32
[get_swap_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L99
[get_swaphandle_info]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L1197
[get_unmapped_area]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L644
[get_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L157
[get_vm_area]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L195
[get_zeroed_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L438
[gfp-flags]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L600
[handle_mm_fault]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1364
[handle_pte_fault]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1331
[highend_pfn]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L42
[highmem-doc]:https://www.kernel.org/doc/Documentation/vm/highmem.txt
[highmem_start_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L57
[highstart_pfn]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L42
[highstart_pfn_ASSIGN]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L1006
[inactive_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L28
[init_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L304
[init_bootmem_core]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L46
[init_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L284
[init_emergency_pool]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L282
[init_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/init_task.c#L12
[init_tmpfs]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1560
[inode]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L438
[inode_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/inode.c#L1126
[inode_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L879
[insert_vm_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L1187
[int_sqrt]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/oom_kill.c#L26
[io_request_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/drivers/block/ll_rw_blk.c#L66
[ioremap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/io.h#L122
[ipc]:https://en.wikipedia.org/wiki/Inter-process_communication
[ipi]:https://en.wikipedia.org/wiki/Inter-processor_interrupt
[iput]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/inode.c#L1027
[kern_mount]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/super.c#L866
[kfree]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1597
[km_type]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/kmap_types.h#L4
[kmalloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1555
[kmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L62
[kmap_atomic]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L83
[kmap_atomic]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/highmem.h#L72
[kmap_high]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L132
[kmap_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L81
[kmap_nonblock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L63
[kmem_bufctl_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L141
[kmem_cache_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1529
[kmem_cache_alloc_batch]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1305
[kmem_cache_alloc_head]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1231
[kmem_cache_alloc_one_tail]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1242
[kmem_cache_create]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L622
[kmem_cache_destroy]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L997
[kmem_cache_estimate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L388
[kmem_cache_free]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1576
[kmem_cache_free_one]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1414
[kmem_cache_grow]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1105
[kmem_cache_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L416
[kmem_cache_init_objs]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1058
[kmem_cache_reap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1738
[kmem_cache_s]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L190
[kmem_cache_shrink]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L966
[kmem_cache_sizes_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L436
[kmem_cache_slabmgmt]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1032
[kmem_cpucache_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L473
[kmem_freepages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L507
[kmem_getpages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L486
[kmem_slab_destroy]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L555
[kmem_tune_cpucache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L1639
[kswapd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L720
[kswapd_balance]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L667
[kswapd_can_sleep]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L695
[kswapd_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L767
[kswapd_wait]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L626
[kunmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L74
[kunmap_atomic]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/highmem.h#L109
[kunmap_high]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L157
[last_pkmap_nr]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L34
[linus-tlb]:http://lists-archives.com/linux-kernel/28329215-tlb-flush-multiple-pages-per-ipi-v5.html
[linux-archaeology]:https://github.com/lorenzo-stoakes/linux-archaeology
[list_head]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/list.h#L18
[lookup_swap_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L161
[lru-2q]:http://www.vldb.org/conf/1994/P439.PDF
[lru]:https://en.wikipedia.org/wiki/Cache_algorithms#LRU
[lru_cache_add]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap.c#L58
[lru_cache_del]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap.c#L90
[madvise]:http://man7.org/linux/man-pages/man2/madvise.2.html
[make_pages_present]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1460
[map_new_virtual]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L80
[mark_page_accessed]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L1332
[mem_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/alpha/mm/init.c#L360
[mem_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L507
[mem_map]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L73
[mk_pte]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L309
[mk_pte_phys]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L312
[mkswap]:http://man7.org/linux/man-pages/man8/mkswap.8.html
[mlock]:http://man7.org/linux/man-pages/man2/mlock.2.html
[mlock_fixup]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L117
[mlock_fixup_all]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L15
[mlock_fixup_end]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L49
[mlock_fixup_middle]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L75
[mlock_fixup_start]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L23
[mlockall]:http://man7.org/linux/man-pages/man2/mlockall.2.html
[mm.h]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h
[mm_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L248
[mm_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L230
[mm_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/sched.h#L206
[mmap]:http://man7.org/linux/man-pages/man2/mmap.2.html
[mmdrop]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/sched.h#L765
[mmlist_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L224
[mmput]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/fork.c#L276
[mmu]:https://en.wikipedia.org/wiki/Memory_management_unit
[move_page_tables]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mremap.c#L90
[move_vma]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mremap.c#L125
[mprotect]:http://man7.org/linux/man-pages/man2/mprotect.2.html
[mremap]:http://man7.org/linux/man-pages/man2/mremap.2.html
[munlock]:http://man7.org/linux/man-pages/man2/munlock.2.html
[munlockall]:http://man7.org/linux/man-pages/man2/munlockall.2.html
[munmap]:http://man7.org/linux/man-pages/man2/munmap.2.html
[nameidata]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L695
[newseg]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L178
[nr_emergency_bhs]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L207
[nr_emergency_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L204
[nr_swap_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L25
[nx-bit]:https://en.wikipedia.org/wiki/NX_bit
[one_highpage_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L450
[out_of_line_bug]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/kernel.h#L175
[out_of_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/oom_kill.c#L202
[pae]:https://en.wikipedia.org/wiki/Physical_Address_Extension
[page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L154
[page_alloc.c]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c
[page_cache_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L34
[page_cache_get]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L31
[page_cache_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L3318
[page_cache_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L702
[page_cache_release]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L32
[page_cluster]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap.c#L28
[page_hash_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L47
[page_waitqueue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L783
[page_zone]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L339
[pagecache_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L94
[pagemap_lru_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L162
[pagemap_lru_lock_cacheline]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L65
[pagetable_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L205
[paging_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L351
[pg0]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L395
[pg1]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L398
[pg_data_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L129
[pgd_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L144
[pgd_free]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L143
[pgd_index]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L327
[pgd_offset]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L331
[pgd_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L52
[pgd_quicklist]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L9
[pgd_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L42
[pgd_val]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L55
[pgdat_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L30
[pge]:https://en.wikipedia.org/wiki/Translation_lookaside_buffer#Address_space_switch
[pglist_data]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L129
[pgprot_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L52
[pgprot_val]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L56
[phys_to_virt]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/io.h#L81
[pkmap_count]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L33
[pkmap_map_wait]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L40
[pkmap_page_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/highmem.c#L38
[pmd_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L511
[pmd_bad]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L273
[pmd_clear]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L272
[pmd_free]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L156
[pmd_none]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L270
[pmd_offset/2lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L55
[pmd_offset/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L72
[pmd_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L323
[pmd_present]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L271
[pmd_quicklist]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L10
[pmd_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L41
[pmd_val]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L54
[prctl]:http://man7.org/linux/man-pages/man2/prctl.2.html
[pse]:https://en.wikipedia.org/wiki/Page_Size_Extension
[pte_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1431
[pte_alloc_one]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L107
[pte_alloc_one_fast]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L117
[pte_clear]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L268
[pte_dirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L284
[pte_exec]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L283
[pte_exprotect]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L289
[pte_free]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L142
[pte_free_fast]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L130
[pte_get_and_clear/2lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L59
[pte_get_and_clear/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L75
[pte_mkclean]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L290
[pte_mkdirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L295
[pte_mkexec]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L294
[pte_mkold]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L291
[pte_mkread]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L293
[pte_mkwrite]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L297
[pte_mkyoung]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L296
[pte_none/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L93
[pte_offset]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L342
[pte_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L92
[pte_present]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L267
[pte_quicklist]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgalloc.h#L11
[pte_rdprotect]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L288
[pte_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L282
[pte_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L40
[pte_to_swp_entry]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L355
[pte_val/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L43
[pte_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L286
[pte_wrprotect]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L292
[pte_young]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L285
[put_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L195
[rb-tree]:https://en.wikipedia.org/wiki/Red%E2%80%93black_tree
[rb_erase]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/lib/rbtree.c#L223
[read]:http://man7.org/linux/man-pages/man2/read.2.html
[read_swap_cache_async]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L184
[refill_inactive]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L533
[register_bootmem_low_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L954
[remove_exclusive_swap_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L287
[remove_inode_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L130
[remove_page_from_hash_queue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L107
[remove_page_from_inode_queue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L94
[remove_shared_vm_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L111
[reserve_bootmem]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L311
[reserve_bootmem_node]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/bootmem.c#L289
[rmqueue]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L199
[rw_swap_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_io.c#L85
[rw_swap_page_base]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_io.c#L36
[scan_swap_map]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L36
[search_exception_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/extable.c#L37
[select_bad_process]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/oom_kill.c#L121
[sendfile]:http://man7.org/linux/man-pages/man2/sendfile.2.html
[set_page_dirty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L152
[set_page_zone]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L344
[set_pte/2lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-2level.h#L42
[set_pte/3lvl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable-3level.h#L46
[setup_arch]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L1124
[setup_arg_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/exec.c#L325
[setup_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/setup.c#L991
[shm]:https://en.wikipedia.org/wiki/Shared_memory#Support_on_Unix-like_systems
[shm_close]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L139
[shm_open]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L110
[shm_vm_ops]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L172
[shmat]:http://man7.org/linux/man-pages/man2/shmat.2.html
[shmdt]:http://man7.org/linux/man-pages/man2/shmdt.2.html
[shmem_aops]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1500
[shmem_commit_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L912
[shmem_create]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1164
[shmem_dir_inode_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1524
[shmem_empty]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1201
[shmem_file_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1510
[shmem_file_read]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1088
[shmem_file_setup]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1607
[shmem_file_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L924
[shmem_free_swp]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L240
[shmem_get_inode]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L809
[shmem_getpage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L583
[shmem_inode_info]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/shmem_fs.h?#L20
[shmem_inode_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1519
[shmem_inodes]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L71
[shmem_mkdir]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1154
[shmem_mknod]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1139
[shmem_mmap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L796
[shmem_nopage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L763
[shmem_notify_change]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L375
[shmem_prepare_write]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L905
[shmem_read_super]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1452
[shmem_readpage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L896
[shmem_removepage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L83
[shmem_rmdir]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1232
[shmem_swp_alloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L179
[shmem_swp_entry]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L127
[shmem_symlink_inline_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1354
[shmem_symlink_inode_operations]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1359
[shmem_sync_file]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1446
[shmem_truncate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L351
[shmem_truncate_direct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L265
[shmem_truncate_indirect]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L309
[shmem_unlink]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1221
[shmem_unuse]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L213
[shmem_unuse]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L498
[shmem_vm_ops]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1547
[shmem_writepage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L522
[shmem_zero_setup]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/shmem.c#L1664
[shmget]:http://man7.org/linux/man-pages/man2/shmget.2.html
[shmid_kernel]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L29
[shrink_cache]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L338
[shrink_caches]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L560
[shrink_dcache_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/dcache.c#L550
[shrink_dqcache_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/dquot.c#L504
[shrink_icache_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/inode.c#L762
[slab-cflgs]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L249
[slab_bufctl]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L163
[slab_s]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L155
[smp_call_function_all_cpus]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/slab.c#L859
[smp_processor_id]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/smp.h#L84
[start_kernel]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/init/main.c#L352
[start_lazy_tlb]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/kernel/exit.c#L279
[startup_32]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L44
[strlen_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/uaccess.h#L781
[strncpy_from_user]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/lib/usercopy.c#L145
[super_block]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L740
[swap_aops]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L34
[swap_device_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L210
[swap_device_unlock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L211
[swap_duplicate]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L1161
[swap_free]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L214
[swap_header]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L25
[swap_info]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L32
[swap_info_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L64
[swap_list]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L30
[swap_list_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/swap.h#L153
[swap_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L251
[swap_out]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L269
[swap_out]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L296
[swap_out_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L256
[swap_out_pgd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L197
[swap_out_pmd]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L158
[swap_out_vma]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L227
[swap_setup]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap.c#L100
[swap_writepage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L24
[swapin_readahead]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L1093
[swapon]:http://man7.org/linux/man-pages/man8/swapon.8.html
[swapper_pg_dir]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/head.S#L380
[swapper_space]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c#L39
[swapper_space]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swap_state.c?v=linux-2.4.22#L39
[switch_mm]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/mmu_context.h#L28
[swp_entry_t]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/shmem_fs.h#L16
[swp_entry_to_pte]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/pgtable.h#L356
[sys_brk]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L147
[sys_mlock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L195
[sys_mlockall]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L266
[sys_mmap2]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/kernel/sys_i386.c#L68
[sys_mprotect]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mprotect.c#L267
[sys_mremap]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mremap.c#L347
[sys_munlock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L226
[sys_munlockall]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mlock.c#L266
[sys_shmat]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L568
[sys_shmget]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/ipc/shm.c#L229
[sys_swapoff]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L720
[sys_swapon]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L868
[taocp]:https://en.wikipedia.org/wiki/The_Art_of_Computer_Programming
[task_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/sched.h#L283
[thundering herd]:https://en.wikipedia.org/wiki/Thundering_herd_problem
[tlb]:https://en.wikipedia.org/wiki/Translation_lookaside_buffer
[total_swap_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L22
[tq_disk]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/drivers/block/ll_rw_blk.c#L52
[try_to_free_buffers]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L2680
[try_to_free_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L607
[try_to_free_pages_zone]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L587
[try_to_release_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.c#L1341
[try_to_swap_out]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmscan.c#L47
[try_to_unuse]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L513
[unlock_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/filemap.c#L874
[unmap_fixup]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L787
[unuse_process]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/swapfile.c#L454
[user_path_walk]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L1390
[vfree]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L237
[vfsmount]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mount.h#L19
[virt_to_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/page.h#L134
[virt_to_phys]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/asm-i386/io.h#L63
[vm_area_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L44
[vm_enough_memory]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L53
[vm_operations_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mm.h#L133
[vm_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/vmalloc.h#L15
[vma_link]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L337
[vma_merge]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/mmap.c#L350
[vmalloc]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/vmalloc.h#L37
[vmalloc_32]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/vmalloc.h#L55
[vmalloc_area_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L189
[vmalloc_dma]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/vmalloc.h#L46
[vmfree_area_pages]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/vmalloc.c#L80
[vmlinux]:https://en.wikipedia.org/wiki/Vmlinux
[vmlist_lock]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/vmalloc.h#L64
[wait_on_page]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/pagemap.h#L94
[wait_table_size]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L647
[wakeup_bdflush]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/fs/buffer.cL2857
[working-set]:https://en.wikipedia.org/wiki/Working_set
[write]:http://man7.org/linux/man-pages/man2/write.2.html
[writepage]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/fs.h#L386
[zap_page_range]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/memory.c#L360
[zone_sizes_init]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/arch/i386/mm/init.c#L323
[zone_struct]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/include/linux/mmzone.h#L37
[zone_table]:https://github.com/lorenzo-stoakes/linux-historical/blob/v2.4.22/mm/page_alloc.c#L38
