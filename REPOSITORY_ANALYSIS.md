# Linux Kernel Repository Analysis - Technical Deep Dive

## Overview

This is the Linux kernel source repository (version 6.18-rc7 "Baby Opossum Posse"), one of the largest and most complex open-source projects. The repository contains architecture-independent and architecture-specific code for a Unix-like operating system kernel supporting 23+ processor architectures.

**Repository Statistics:**
- Current version: Linux 6.18-rc7
- Number of architectures supported: 23+ (x86, ARM, ARM64, MIPS, PowerPC, RISC-V, s390, etc.)
- Top-level directories: 28 major directories
- Driver categories: 143+ subdirectories
- Build system: GNU Make-based with Kconfig
- Programming languages: C (primary), Assembly, Rust (emerging support)

---

## Part 1: Memory Management Deep Dive

### Overview

Memory management is a core subsystem that handles physical page allocation, virtual memory, page caching, memory debugging, and memory reclaim. The kernel implements a sophisticated multi-layer allocation system from low-level page allocator up to high-level slab caches.

### 1. Buddy Allocator - Page-Level Allocation

**Purpose:** Manages physical memory allocation at the page level (4KB granularity on most systems).

**File Location:** `/mm/page_alloc.c` (7,628 lines) - Core implementation

**Key Data Structures:**

```c
// Free area for each order (2^order pages)
// From include/linux/mmzone.h:138-141
struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  // Per-migration-type lists
    unsigned long nr_free;                      // Free pages count
};

// Zone structure containing free areas
// From include/linux/mmzone.h:879-1060
struct zone {
    unsigned long _watermark[NR_WMARK];         // Min/low/high watermarks
    struct free_area free_area[NR_PAGE_ORDERS]; // Orders 0-10 (1 to 1024 pages)
    spinlock_t lock;                            // Zone lock for thread safety
    struct per_cpu_pages __percpu *per_cpu_pageset;  // Per-CPU caches
    unsigned long nr_free_pages;
    unsigned long nr_free_cma;
    struct list_head lru_list;                  // LRU lists for reclaim
};
```

**Buddy Allocator Algorithm:**

The buddy allocator maintains free lists organized by power-of-2 page orders:
- **Order 0:** 1 page (4KB)
- **Order 1:** 2 pages (8KB)
- **Order 2:** 4 pages (16KB)
- ...
- **Order 10:** 1024 pages (4MB)

When allocating pages:
1. Find the lowest order that satisfies the request
2. If free list is empty, split a higher-order page from buddy list
3. Mark new order as free and remove from next higher order
4. Return pages to allocator

When freeing pages:
1. Mark pages as free in target order
2. Check if buddy is also free (same order, adjacent address)
3. If buddy is free, merge both into higher order (buddy merging)
4. Recursively try to merge at higher orders

**Key Functions:**

| Function | Location | Purpose |
|----------|----------|---------|
| `__alloc_pages_noprof()` | page_alloc.c:5207 | Core allocation function |
| `__free_one_page()` | page_alloc.c:940 | Buddy merging logic |
| `expand()` | page_alloc.c:1688 | Split higher-order pages |
| `rmqueue_buddy()` | page_alloc.c:3152 | Allocate from buddy lists |
| `free_pcppages_bulk()` | page_alloc.c:1442 | Return pages to buddy |

**Per-CPU Caching:**

To reduce lock contention, the kernel maintains per-CPU page lists:

```c
// From page_alloc.c:159-161
struct per_cpu_pages {
    int count;                    // Pages in list
    int high;                     // Refill threshold
    int batch;                    // Batch size for operations
    struct list_head lists[NR_PCP_LISTS];  // Free lists per migrate type
};
```

- Fast allocation path doesn't need zone lock (atomic CPU-local operation)
- Periodically batch frees back to zone lists
- Reduces lock contention by ~70% on high-core systems

**Allocation Flags (GFP - Get Free Pages):**

From `include/linux/gfp.h`:
- `GFP_KERNEL` - Normal kernel allocation (can sleep, reclaim, swap)
- `GFP_ATOMIC` - From IRQ/softirq context (no sleep, limited reclaim)
- `GFP_HIGHUSER` - User memory in high memory zones
- `GFP_NOWAIT` - Don't wait, fail immediately
- `GFP_RECLAIM` - Allow memory reclaim
- `GFP_COMP` - Prepare compound pages (multiple pages as one)

---

### 2. Virtual Memory Management

**Purpose:** Maps logical memory addresses to physical pages, implements page faults, and manages page tables.

**File Location:** `/mm/memory.c` (7,326 lines) - VMM implementation

**Page Table Hierarchy:**

Modern CPUs use hierarchical page tables (4-level on x86-64):

| Level | Name | Size | Entries |
|-------|------|------|---------|
| 1 | PGD (Page Global Dir) | 4KB | 512 (64-bit) |
| 2 | PUD (Page Upper Dir) | 4KB | 512 |
| 3 | PMD (Page Middle Dir) | 4KB | 512 |
| 4 | PTE (Page Table Entry) | 4KB | 512 |

Each entry is 8 bytes (64-bit), so 512 entries per table = 4KB per table.

**Address Translation Example (x86-64):**
```
Virtual Address (48-bit): [PGD(9)|PUD(9)|PMD(9)|PTE(9)|Offset(12)]
                          47-39   38-30   29-21   20-12  11-0

Process:
1. Load PGD base from CR3 register
2. Index into PGD[bits 47-39] → get PUD table address
3. Index into PUD[bits 38-30] → get PMD table address
4. Index into PMD[bits 29-21] → get PTE table address
5. Index into PTE[bits 20-12] → get physical page address
6. Add offset (bits 11-0) to get final physical address
```

**Page Table Structures (from mm_types.h):**

```c
typedef struct { unsigned long pte; } pte_t;
typedef struct { unsigned long pmd; } pmd_t;
typedef struct { unsigned long pud; } pud_t;
typedef struct { unsigned long pgd; } pgd_t;

// Opaque types prevent accidental mixing
```

**Key Page Fault Handlers:**

| Function | Line | Handles |
|----------|------|---------|
| `__handle_mm_fault()` | 6245 | Main page fault dispatcher |
| `handle_pte_fault()` | 6151 | PTE-level fault decisions |
| `do_page_fault()` | arch-specific | CPU exception handler |
| `do_swap_page()` | 4582 | Swap page faults (bring from disk) |
| `wp_page_copy()` | 3658 | Copy-on-write page faults |
| `do_anonymous_page()` | 4324 | Allocate new anonymous page |

**Page Fault Flow:**

```
1. Hardware MMU exception (no PTE)
     ↓
2. CPU calls page_fault_handler (architecture-specific)
     ↓
3. Calls __handle_mm_fault(vma, address, flags)
     ↓
4. Walk page table hierarchy, allocate missing levels
     ↓
5. Call handle_pte_fault() if PTE exists but fault
     ↓
6. Determine fault type:
   - If swap: do_swap_page() - load from disk
   - If CoW: wp_page_copy() - duplicate page
   - If anon: do_anonymous_page() - allocate new
   - If file: do_shared_fault() - read from file
     ↓
7. Update PTE with new page address
     ↓
8. Return to faulting instruction, retry
```

**Memory Area Management (mm/mmap.c):**

Applications' virtual memory divided into regions with different permissions:

```c
struct vm_area_struct {
    unsigned long vm_start;           // Region start address
    unsigned long vm_end;             // Region end address
    struct vm_area_struct *vm_next;   // Next region (vm_rbtree often used)
    struct mm_struct *vm_mm;          // Process memory struct
    const struct vm_operations_struct *vm_ops;  // Fault handlers
    unsigned long vm_flags;           // VM_READ, VM_WRITE, VM_EXEC, etc.
    unsigned long vm_pgoff;           // Offset into file/swap
    struct file *vm_file;             // If memory-mapped file
};
```

Regions can be:
- **Anonymous** - Heap/stack, not backed by file
- **File-backed** - Memory-mapped file regions
- **Shared** - Shared with other processes
- **Private (CoW)** - Copy-on-write, private to process

---

### 3. Memory Zones and Watermarks

**Purpose:** Partition physical memory into zones for DMA, normal, and high memory allocation needs.

**File Location:** `include/linux/mmzone.h` (lines 784-860)

**Zone Types (enum zone_type):**

```c
#ifdef CONFIG_ZONE_DMA
    ZONE_DMA,           // DMA-capable memory (< 16MB on x86)
#endif
#ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,         // 32-bit DMA zone (< 4GB on 64-bit)
#endif
    ZONE_NORMAL,        // General purpose memory
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,       // High memory (mapped on-demand on 32-bit)
#endif
    ZONE_MOVABLE,       // Memory hotplug friendly
```

**Zone Purposes:**

| Zone | Use Case | Physical Address | Characteristics |
|------|----------|------------------|-----------------|
| **ZONE_DMA** | ISA DMA devices | 0-16MB | Small, contiguous |
| **ZONE_DMA32** | PCI 32-bit devices | 16MB-4GB | Can allocate large contiguous blocks |
| **ZONE_NORMAL** | General kernel | > 4GB (varies) | Directly mapped, largest zone |
| **ZONE_HIGHMEM** | Beyond direct map | > 896MB (32-bit) | Requires temporary mapping |
| **ZONE_MOVABLE** | Page migration | Configurable | Can be hot-unplugged |

**Zone Watermarks:**

Watermarks prevent memory exhaustion by maintaining minimum free memory:

```c
// From include/linux/mmzone.h:882-884
unsigned long _watermark[NR_WMARK];

#define WMARK_MIN   0   // Minimum free pages (critical)
#define WMARK_LOW   1   // Low free pages (start reclaim)
#define WMARK_HIGH  2   // High free pages (stop reclaim)
```

**Watermark Levels (example values):**
- **min_free_kbytes = 67584 KB** (default on 16GB system)
- **WMARK_MIN = 16896 pages** (67.5 MB)
- **WMARK_LOW = 21120 pages** (84 MB)
- **WMARK_HIGH = 25344 pages** (101 MB)

**Watermark Logic (from page_alloc.c:3519):**

```c
bool __zone_watermark_ok(zone, order, mark, highest_zoneidx, alloc_flags)
{
    // Check if allocating 2^order pages would cross watermark
    if (free_pages - (1 << order) >= watermark)
        return true;  // Safe to allocate

    // May allow allocation from reserved pages or other zones
    // depending on alloc_flags
}
```

**Reclaim Triggering:**
- When free pages < WMARK_LOW, kswapd wakes up
- Runs in background to evict pages back to buddy allocator
- Stops when free pages > WMARK_HIGH
- Atomic allocations (GFP_ATOMIC) use reserved pages below WMARK_MIN

---

### 4. SLUB Allocator - Object-Level Allocation

**Purpose:** Allocate small kernel objects (kmalloc) efficiently with minimal overhead.

**File Location:** `/mm/slub.c` (10,084 lines) - Main SLUB implementation

**Why SLUB over raw pages?**

Raw page allocation wastes memory for small objects:
- Kernel needs 10 bytes? Allocate 4096 bytes (4KB page)
- 4086 bytes wasted per object!

Solution: Pack multiple objects per page into "slabs"

**SLUB Data Structures:**

```c
// From slub.c:418 - Per-CPU slab cache
struct kmem_cache_cpu {
    struct slab *slab;              // Currently used slab
    unsigned long partial;          // Offset/flags
    unsigned long tid;              // Transaction ID (CAS optimized)
};

// From slub.c:490 - Per-node slab cache
struct kmem_cache_node {
    spinlock_t list_lock;
    struct list_head partial;       // Slabs with free objects
    unsigned long nr_partial;
    // ...
};

// From slab.h:52-100 - Slab metadata
struct slab {
    memdesc_flags_t flags;
    struct kmem_cache *slab_cache;
    void *freelist;                 // First free object (linked list)
    unsigned inuse:16;              // Objects in use count
    unsigned objects:15;            // Total objects in slab
    unsigned frozen:1;              // Slab allocated to CPU
};
```

**Kmalloc Allocation Path:**

```c
// kmalloc() → kmem_cache_alloc() → slab_alloc()

ptr = kmalloc(size, GFP_KERNEL);

// Internally:
1. Find cache for requested size:
   - 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096 bytes

2. Try per-CPU slab (fast path, no lock):
   - Read CPU-local slab structure
   - If has free objects: return object from freelist

3. If per-CPU slab exhausted:
   - Take lock on kmem_cache_node
   - Get new slab from partial list
   - Unlock
   - Continue allocation
```

**Kmalloc Cache Types (from slab.h:636-662):**

```c
enum kmalloc_cache_type {
    KMALLOC_NORMAL,         // Normal slabs
    KMALLOC_CGROUP,         // Memory cgroup tracking
    KMALLOC_RANDOM_START,   // Randomized for security
    // ... total 12 types with various configs
};

// Creates dedicated caches:
kmalloc_caches[NR_KMALLOC_TYPES] contains caches for all sizes
```

**Slab Allocation Sizes:**

Object sizes are power-of-2 or aligned:
```
8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096 bytes
```

**Advantages of SLUB:**

- **Memory efficiency:** Multiple objects per page
- **CPU affinity:** Objects tend to allocate on same CPU (cache locality)
- **Partial reuse:** Slabs with available objects reused
- **Minimal overhead:** Per-slab metadata is minimal (~16 bytes)
- **Debugging:** Integration with KASAN, KFENCE, KMSAN

**Lock Hierarchy (slub.c:54-79):**

1. `slab_mutex` - Global configuration changes
2. `node->list_lock` - Node-level synchronization
3. Per-CPU structures - No locks (thread-local)
4. `slab_lock()` - Per-slab (when CAS unavailable)

---

### 5. Page Cache - File I/O Layer

**Purpose:** Cache file data in RAM for fast access, coordinate filesystem and memory subsystems.

**File Location:** `/mm/filemap.c` (4,689 lines)

**Page Cache Architecture:**

```
File
  ↓
inode (filesystem inode structure)
  ↓
address_space (contains page cache)
  ↓
Radix tree/XArray (page index → page* mapping)
  ↓
Pages in RAM (4KB each)
```

**Core Address Space Operations:**

```c
struct address_space_operations {
    int (*writepage)(struct page *);         // Write single page to disk
    int (*readpage)(struct file *, struct page *);  // Read page from disk
    int (*writepages)(struct address_space *);      // Batch write
    int (*releasepage)(struct page *, gfp_t);  // Can page be freed?
    void (*invalidatepage)(struct page *, unsigned int, unsigned int);
    // ... more operations
};
```

**Page Cache Functions (from filemap.c):**

| Function | Line | Purpose |
|----------|------|---------|
| `add_to_page_cache()` | 951 | Insert page into cache |
| `read_mapping_page()` | 4122 | Read page from file |
| `filemap_write_and_wait()` | 685 | Write all dirty pages and wait |
| `filemap_fdatawait_range()` | 565 | Wait for page writeback |
| `filemap_get_page()` | 3485 | Lookup page in cache |

**Page Cache State Flags (from page-flags.h):**

```c
#define PG_locked       0   // Page locked (I/O in progress)
#define PG_error        1   // I/O error occurred
#define PG_referenced   2   // Recently accessed
#define PG_uptodate     3   // Page contents are valid
#define PG_dirty        4   // Page contains unsaved changes
#define PG_lru          5   // On LRU list for reclaim
#define PG_active       6   // Active in LRU (recently used)
#define PG_slab         7   // Page is slab
#define PG_owner_priv_1 8   // Page owner's private flag
#define PG_writeback   15   // Page being written to disk
#define PG_head        16   // Head of compound page
#define PG_mappedtodisk 17  // Page has disk mapping (ext3 journaling)
```

**Writeback System (page-writeback.c - 3,143 lines):**

Manages delayed writes to disk (not immediate):

| Component | Function |
|-----------|----------|
| **Dirty tracking** | Marked when page modified |
| **Rate limiting** | `balance_dirty_pages()` - Prevent too many dirty pages |
| **Writeback daemon** | Periodic background writeback |
| **pdflush/kworker** | Thread that writes dirty pages |
| **Sync operations** | `sync_inodes_sb()` - Force all writes |

---

### 6. Memory Debugging Tools

#### **KASAN (Kernel Address Sanitizer)**

**Purpose:** Detect out-of-bounds memory access and use-after-free bugs at runtime.

**File Location:** `/mm/kasan/` (17 files, ~220KB)

**How KASAN Works:**

1. **Shadow Memory:** For every 8 bytes of real memory, 1 byte of shadow
   - Shadow byte = 0: All 8 bytes accessible
   - Shadow byte = 1-7: Bytes 0-(value-1) accessible, rest not
   - Shadow byte = -1: All 8 bytes inaccessible

2. **Instrumentation:** Every memory access checked against shadow
   ```c
   // Original code
   *ptr = value;

   // KASAN instrumented
   if (shadow_byte_at(ptr) != 0) {
       report_error(ptr);
   }
   *ptr = value;
   ```

3. **Setup (init.c):**

```c
unsigned char kasan_early_shadow_page[PAGE_SIZE];  // 4KB shadow
p4d_t kasan_early_shadow_p4d[MAX_PTRS_PER_P4D];    // P4D table
pud_t kasan_early_shadow_pud[MAX_PTRS_PER_PUD];    // PUD table
pmd_t kasan_early_shadow_pmd[MAX_PTRS_PER_PMD];    // PMD table
```

**KASAN Features:**

| Feature | Detection |
|---------|-----------|
| **Buffer overflows** | Writing beyond allocated buffer |
| **Use-after-free** | Accessing freed memory |
| **Double-free** | Freeing already freed pointer |
| **Integer overflow** | (UBSAN integration) |
| **Memory leaks** | Unfreed allocations at shutdown |

**KASAN Modes:**

- **Generic** - Software shadow memory (slower, ~3x)
- **HW_TAGS** - Hardware tagging (Armv8.5 pointers, ~1.2x overhead)
- **SW_TAGS** - Software tags (moderate speed/precision tradeoff)

**Quarantine System (quarantine.c):**

Delay freed object reuse to detect use-after-free:
```c
// Free marking page inaccessible
mark_shadow_for_memory_region(ptr, size, KASAN_FREE_PAGE);

// Store pointer in quarantine list
quarantine_put(ptr);

// Only reuse after other allocations happen
```

---

#### **KFENCE (Kernel Fence)**

**Purpose:** Detect heap buffer overflows and use-after-free with zero false positives, minimal overhead.

**File Location:** `/mm/kfence/` (6 files, ~82KB)

**How KFENCE Works:**

1. **Physical Isolation:** Place each allocation on separate pages with guard pages:

```
Guard Page (inaccessible)
   ↓
[Allocated Object]
   ↓
Guard Page (inaccessible)
```

2. **Probabilistic Sampling:** Only sample every Nth allocation (configurable via `kfence.sample_interval`)
   - Default: 1 in 100 allocations
   - Can be 1 in 10,000 for lower overhead

3. **Stack Traces:** Record allocation and free stack traces for debugging

**KFENCE Pool Structure (core.c:116-117):**

```c
char *__kfence_pool __read_mostly;     // Allocated pool
atomic_t kfence_allocation_gate;       // Sampling counter

// Pool layout:
// ┌─────────────┐
// │  Metadata   │ (allocation/free info, stack traces)
// ├─────────────┤
// │Guard Page 1 │ (inaccessible)
// ├─────────────┤
// │  Object 1   │ (allocatable, padded to page)
// ├─────────────┤
// │Guard Page 2 │ (inaccessible)
// └─────────────┘
```

**Key APIs (core.c):**

| Function | Purpose |
|----------|---------|
| `kfence_init_pool()` | Initialize KFENCE at boot (line 595) |
| `kfence_alloc()` | Allocate from KFENCE pool |
| `kfence_free()` | Free to KFENCE pool |

**Configuration:**
- `KFENCE_POOL_SIZE` - Total pool size (default 4 MB)
- `kfence.sample_interval` - Sampling rate (kernel parameter)
- `kfence.skip_covered_objects` - Skip objects in certain regions

**Advantages:**
- Zero false positives (physical guard pages)
- Minimal overhead even when enabled (samples only)
- Perfect for production systems

---

#### **KMSAN (Kernel Memory Sanitizer)**

**Purpose:** Detect uninitialized memory usage (reading uninitialized variables).

**File Location:** `/mm/kmsan/` (9 files, ~87KB)

**How KMSAN Works:**

1. **Shadow Memory:** Tracks initialization state
   - 1 bit per byte of real memory
   - 0 = initialized, 1 = uninitialized

2. **Compiler Instrumentation:** Every memory operation checked:
   - Load from uninitialized → error
   - Arithmetic with uninitialized → propagate uninitialized state
   - Store uninitialized → mark destination as uninitialized

3. **Origin Tracking:** Keep stack trace of where uninitialized value originated

**KMSAN Core Functions (core.c):**

| Function | Purpose |
|----------|---------|
| `kmsan_internal_poison_memory()` | Mark memory as uninitialized |
| `kmsan_internal_unpoison_memory()` | Mark memory as initialized |
| `kmsan_internal_set_shadow_origin()` | Set shadow + origin info |

**Compiler Instrumentation (instrumentation.c):**

```c
// Original code
int x;
if (x > 5) { ... }

// KMSAN instrumented
int x;
__kmsan_check_memory(&x, sizeof(x));  // Check if x is initialized
if (x > 5) { ... }
```

**Shadow Tracking:**

```c
// Per-CPU context (core.c:32-38)
DEFINE_PER_CPU(struct kmsan_ctx, kmsan_percpu_ctx);

struct kmsan_ctx {
    struct kmsan_origin_info origin;  // Where did uninitialized come from?
    // ...
};
```

**KMSAN Modes:**
- **Runtime** - Full instrumentation, full detection
- **Off** - Can be disabled per function with `__no_sanitize_memory`

---

### 7. Memory Reclaim and Page Eviction

**Purpose:** Free memory when low, prioritizing least-used pages.

**File Location:** `/mm/vmscan.c` (7,919 lines)

**Reclaim System Architecture:**

The kernel uses **kswapd** (kernel swap daemon) to reclaim pages in background:

```
Memory pressure detected
    ↓
Wake kswapd thread
    ↓
shrink_node() called
    ↓
Scan LRU lists (active/inactive)
    ↓
Try to reclaim pages:
  - Inactive file pages → writeback and discard
  - Swap pages → write to swap device and free
    ↓
Return freed pages to buddy allocator
    ↓
Sleep until needed again
```

**LRU Lists (Least Recently Used):**

Kernel maintains per-zone LRU lists to track page usage:

| List | Contains | Reclaim Priority |
|------|----------|-----------------|
| LRU_INACTIVE_ANON | Unused user memory | High (easy to swap) |
| LRU_ACTIVE_ANON | Used user memory | Lower (in-use, harder) |
| LRU_INACTIVE_FILE | Unused file cache | High (can discard) |
| LRU_ACTIVE_FILE | Used file cache | Lower (needed soon) |
| LRU_UNEVICTABLE | Locked/pinned memory | Never (can't reclaim) |

**Reclaim Functions (vmscan.c):**

| Function | Line | Purpose |
|----------|------|---------|
| `shrink_slab()` | 434 | Shrink kernel caches (dentries, inodes) |
| `shrink_node()` | - | Reclaim from single NUMA node |
| `shrink_inactive_list()` | 2002 | Walk inactive LRU, try to free pages |
| `kswapd()` | - | Background reclaim daemon |
| `kswapd_is_running()` | 471 | Check if kswapd active |

**Page Migration (migrate.c):**

Move pages between zones (for NUMA optimization, memory hotplug):

```c
migrate_pages(page_list, alloc_page_fn, free_page_fn, ...)
{
    for_each_page(page in list) {
        new_page = alloc_page_fn();  // Allocate on target zone/node
        copy_page_contents(page, new_page);
        update_page_table(old→new);  // Update PTE
        flush_tlb();
        free_old_page();
    }
}
```

**Swap System:**

When memory pressure high, pages evicted to swap device (usually disk partition):

1. Select page to evict (usually LRU_INACTIVE_ANON)
2. Write page contents to swap device
3. Update page table → swap entry (page number in swap device)
4. Free physical page
5. On access: page fault → `do_swap_page()` → read from swap → restore

---

## Part 2: Driver Architecture Deep Dive

### Overview

Linux uses a unified **device model** where devices and drivers are registered with the kernel, buses match them, and standardized probe/remove functions initialize hardware.

**Key Architectural Principle:** Separation between what hardware does (device) and how to control it (driver).

### 1. Unified Device Model

**Core Purpose:** Provide standard abstraction for any hardware device.

**File Locations:**
- `/drivers/base/core.c` (2000+ lines) - Device registration and lifecycle
- `/drivers/base/driver.c` - Driver model and binding logic
- `/drivers/base/bus.c` - Bus type management
- `/drivers/base/dd.c` - Device-driver matching and probe/remove
- `/include/linux/device.h` - Core structures

**Core Structures:**

```c
struct device {
    struct device_private *p;
    struct device_node *of_node;          // Device tree node
    struct fwnode_handle *fwnode;         // Firmware node
    struct device *parent;                // Parent device
    struct device_type *type;
    struct bus_type *bus;                 // Device's bus (pci, usb, etc.)
    struct device_driver *driver;         // Bound driver
    struct dev_pm_domain *pm_domain;      // Power domain
    struct dev_pm_info power;             // Power management state
    struct dma_map_ops *dma_ops;          // DMA operations
    u64 dma_mask;                         // DMA capability mask
    struct mutex mutex;                   // Mutual exclusion
    struct list_head links;               // Device links (dependencies)
    struct list_head devres_head;         // Managed resources
    // ... more fields
};

struct device_driver {
    const char *name;
    struct bus_type *bus;                 // Associated bus type
    struct module *owner;
    const char *mod_name;                 // Module name for liftetime

    // Lifecycle callbacks
    int (*probe)(struct device *dev);     // Initialize hardware
    void (*sync_state)(struct device *dev);  // After all probed
    int (*remove)(struct device *dev);    // Cleanup hardware
    void (*shutdown)(struct device *dev); // System shutdown

    // Power management
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    // Attributes and operations
    const struct attribute_group **groups;
    const struct attribute_group **dev_groups;
    struct dev_pm_ops *pm;
    void (*coredump)(struct device *dev);
};

struct bus_type {
    const char *name;
    const char *dev_name;

    // Matching algorithm
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);

    // PM and attributes
    const struct dev_pm_ops *pm;
    const struct attribute_group **bus_groups;
};
```

**Device Registration Flow:**

1. Hardware discovered (PCI enumeration, USB hotplug, device tree, etc.)
2. Device structure created
3. `device_register()` or `device_add()` called
4. Device added to bus's device list
5. For each driver on bus: `bus->match(device, driver)` called
6. If match succeeds: `driver_probe_device(device, driver)` called
7. Driver's `probe()` function called to initialize hardware
8. If probe succeeds: driver listed as device owner, device ready
9. If probe fails: try next driver, may defer probe

**Driver Registration:**

1. Driver registers with bus via `driver_register()`
2. Added to bus's driver list
3. For each device on bus: `bus->match(device, driver)` called
4. Proceed with probe for matches

**Deferred Probe (dd.c):**

Handles device dependencies - if device X needs device Y but Y not yet probed:
- Probe failed for device X? Add to deferred list
- After device Y probes successfully, re-attempt X

---

### 2. Platform Device Drivers

**Purpose:** Support "pseudo" devices that aren't on standard buses (legacy devices, device tree, ACPI).

**File Locations:**
- `/include/linux/platform_device.h`
- `/drivers/base/platform.c`

**Platform Device Structure:**

```c
struct platform_device {
    const char *name;                   // Device name (used for driver matching)
    int id;                             // Device ID for disambiguation
    bool id_auto;
    struct device dev;
    u64 platform_dma_mask;
    struct device_dma_parameters dma_parms;

    u32 num_resources;
    struct resource *resource;          // I/O memory, IRQ, etc.
    const struct platform_device_id *id_entry;
    const char *driver_override;
};

struct resource {
    resource_size_t start;              // Start address
    resource_size_t end;                // End address (inclusive)
    const char *name;
    unsigned long flags;                // IORESOURCE_MEM, _IRQ, etc.
    void *parent;
};
```

**Resource Types:**

```c
#define IORESOURCE_MEM          0x00000200  // Memory-mapped I/O
#define IORESOURCE_IO           0x00000100  // Port-mapped I/O
#define IORESOURCE_IRQ          0x00000400  // Interrupt line
#define IORESOURCE_DMA          0x00000800  // DMA channel
```

**Example Platform Driver Probe:**

```c
static int my_device_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_device *my_dev;
    struct resource *res;
    int irq, ret;

    // Allocate driver-specific data (auto-freed on device removal)
    my_dev = devm_kzalloc(dev, sizeof(*my_dev), GFP_KERNEL);
    if (!my_dev)
        return -ENOMEM;

    // Store reference for remove function
    platform_set_drvdata(pdev, my_dev);

    // Get memory resource (first resource)
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);

    // Map memory (auto-unmapped on device removal)
    my_dev->regs = devm_ioremap_resource(dev, res);
    if (IS_ERR(my_dev->regs))
        return PTR_ERR(my_dev->regs);

    // Get IRQ
    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    // Request IRQ (auto-freed on device removal)
    ret = devm_request_irq(dev, irq, my_device_irq, 0, "my_device", my_dev);
    if (ret)
        return ret;

    // Initialize hardware
    my_device_init(my_dev);

    return 0;  // Success! All managed resources freed on error return
}

static void my_device_remove(struct platform_device *pdev)
{
    struct my_device *my_dev = platform_get_drvdata(pdev);

    // Shutdown hardware
    my_device_shutdown(my_dev);

    // NO MANUAL CLEANUP NEEDED!
    // All devm_* resources automatically freed:
    // - devm_kzalloc() → automatically freed
    // - devm_ioremap_resource() → automatically unmapped
    // - devm_request_irq() → automatically freed
}
```

---

### 3. PCI Drivers

**Purpose:** Support devices on the PCI/PCIe bus.

**File Locations:**
- `/include/linux/pci.h` (3000+ lines of APIs)
- `/drivers/pci/` (150+ device drivers)
- `/drivers/pci/bus.c`, `/drivers/pci/devres.c`

**PCI Device Structure:**

```c
struct pci_device_id {
    u32 vendor, device;                 // Vendor/device ID from lspci
    u32 subvendor, subdevice;           // Optional subvendor filtering
    u32 class, class_mask;              // Class code filtering
    kernel_ulong_t driver_data;         // Driver-specific data
};

struct pci_driver {
    const char *name;
    const struct pci_device_id *id_table;  // Must be non-NULL!

    int (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);

    int (*suspend)(struct pci_dev *dev, pm_message_t state);
    int (*resume)(struct pci_dev *dev);
    void (*shutdown)(struct pci_dev *dev);

    int (*sriov_configure)(struct pci_dev *dev, int num_vfs);  // SR-IOV
    const struct pci_error_handlers *err_handler;  // PCIe errors
    struct device_driver driver;
};

struct pci_dev {
    struct device dev;                  // Device structure
    unsigned int devfn;                 // Device/function numbers
    unsigned short vendor, device;      // Hardware IDs
    unsigned short subsystem_vendor, subsystem_device;
    const struct pci_device_id *driver_override;

    struct resource resource[DEVICE_COUNT_RESOURCE];  // 6 BARs + ROM
    const struct pci_driver *driver;

    u32 dma_mask;
    u64 dma_mask_64bit;
    struct pci_bus *bus;
    u16 devfn;
    // ... many more fields
};
```

**PCI Probe Example:**

```c
static const struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(0x1234, 0x5678) },  // Vendor 0x1234, Device 0x5678
    { PCI_DEVICE(0x1234, 0x6789) },  // Another device
    { 0, }  // Terminator
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);

static int my_pci_probe(struct pci_dev *pdev,
                       const struct pci_device_id *id)
{
    struct my_pci_device *my_dev;
    int ret;

    my_dev = devm_kzalloc(&pdev->dev, sizeof(*my_dev), GFP_KERNEL);
    if (!my_dev)
        return -ENOMEM;

    pci_set_drvdata(pdev, my_dev);

    // Enable PCI device (turn on power, enable memory/IO)
    ret = pci_enable_device(pdev);
    if (ret)
        return ret;

    // Request all BARs (Base Address Registers)
    ret = pci_request_regions(pdev, "my_driver");
    if (ret)
        goto disable;

    // Map BAR 0 to kernel virtual address
    my_dev->mmio = pci_iomap(pdev, 0, pci_resource_len(pdev, 0));
    if (!my_dev->mmio) {
        ret = -ENOMEM;
        goto release;
    }

    // Enable DMA master (bus mastering)
    pci_set_master(pdev);

    // Set DMA mask for 32-bit or 64-bit DMA
    if (dma_set_mask(&pdev->dev, DMA_BIT_MASK(64)))
        dma_set_mask(&pdev->dev, DMA_BIT_MASK(32));

    // Request MSI interrupts (vs legacy INTx)
    ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_MSIX);
    if (ret < 0)
        goto unmap;

    ret = devm_request_irq(&pdev->dev, pci_irq_vector(pdev, 0),
                          my_pci_irq, 0, "my_driver", my_dev);
    if (ret)
        goto free_irq;

    // Initialize hardware
    my_pci_init(my_dev);

    return 0;

free_irq:
    pci_free_irq_vectors(pdev);
unmap:
    pci_iounmap(pdev, my_dev->mmio);
release:
    pci_release_regions(pdev);
disable:
    pci_disable_device(pdev);
    return ret;
}

static void my_pci_remove(struct pci_dev *pdev)
{
    struct my_pci_device *my_dev = pci_get_drvdata(pdev);

    // Disable hardware
    my_pci_shutdown(my_dev);

    // Cleanup PCI resources
    pci_free_irq_vectors(pdev);
    pci_iounmap(pdev, my_dev->mmio);
    pci_release_regions(pdev);
    pci_disable_device(pdev);
}

static struct pci_driver my_pci_driver = {
    .name = "my_driver",
    .id_table = my_pci_ids,
    .probe = my_pci_probe,
    .remove = my_pci_remove,
};

module_pci_driver(my_pci_driver);  // Auto-creates module init/exit
```

---

### 4. Interrupt Handling in Drivers

**File Locations:**
- `/include/linux/interrupt.h` - Public API
- `/kernel/irq/manage.c` - IRQ request/free logic
- `/kernel/irq/handle.c` - Interrupt flow

**IRQ Request Structure:**

```c
struct irqaction {
    irq_handler_t handler;              // Hard interrupt handler
    void *dev_id;                       // Context cookie
    void __percpu *percpu_dev_id;       // Per-CPU context
    struct irqaction *next;             // Shared IRQ chain

    irq_handler_t thread_fn;            // Threaded handler
    struct task_struct *thread;         // Handler thread

    unsigned int irq;
    unsigned int flags;                 // IRQF_* flags
    const char *name;                   // For /proc/interrupts
};
```

**IRQ Handler Flags:**

```c
#define IRQF_SHARED         0x00000080  // Allow sharing this IRQ
#define IRQF_TRIGGER_RISING 0x00000001  // Rising edge trigger
#define IRQF_TRIGGER_FALLING 0x00000002 // Falling edge trigger
#define IRQF_TRIGGER_HIGH   0x00000004  // Active high level
#define IRQF_TRIGGER_LOW    0x00000008  // Active low level
#define IRQF_ONESHOT        0x00002000  // Keep IRQ disabled until threaded handler finishes
#define IRQF_NO_SUSPEND     0x00004000  // Don't disable during suspend
#define IRQF_NO_THREAD      0x00010000  // Cannot be threaded
#define IRQF_PROBE_SHARED   0x00100000  // Expect probe-time collision
```

**Requesting Interrupts:**

```c
// Simple IRQ handler
static irqreturn_t my_device_irq(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status;

    // Check if interrupt is from this device
    status = readl(dev->regs + STATUS_REG);
    if (!(status & INTERRUPT_PENDING))
        return IRQ_NONE;  // Not our interrupt

    // Handle interrupt
    my_device_handle_interrupt(dev);

    // Clear interrupt
    writel(status & ~INTERRUPT_PENDING, dev->regs + STATUS_REG);

    return IRQ_HANDLED;  // Interrupt handled
}

// Request interrupt
ret = devm_request_irq(&pdev->dev, irq, my_device_irq,
                       IRQF_SHARED, "my_device", my_dev);
if (ret)
    return ret;
```

**Threaded Interrupt Handlers:**

For interrupt handlers that need to sleep (acquire locks, allocate memory):

```c
// Handler runs in hard IRQ context - must be fast
static irqreturn_t my_device_irq_hard(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    // Just check status, schedule thread
    if (!(readl(dev->regs + STATUS_REG) & INTERRUPT_PENDING))
        return IRQ_NONE;

    return IRQ_WAKE_THREAD;  // Wake threaded handler
}

// Handler runs in process context - can sleep, lock, etc.
static irqreturn_t my_device_irq_thread(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    // Can acquire locks
    mutex_lock(&dev->lock);

    // Can allocate memory
    struct data *d = kmalloc(sizeof(*d), GFP_KERNEL);

    // Can sleep
    msleep(10);

    // Handle interrupt
    my_device_handle_interrupt(dev);

    kfree(d);
    mutex_unlock(&dev->lock);

    return IRQ_HANDLED;
}

// Request both handlers
ret = devm_request_threaded_irq(&pdev->dev, irq,
                               my_device_irq_hard,
                               my_device_irq_thread,
                               IRQF_ONESHOT,  // Keep IRQ disabled until thread completes
                               "my_device",
                               my_dev);
```

**Bottom Half Processing - Tasklets vs Workqueues:**

For deferred work from interrupt:

```c
// ═══════════════════════════════════════════
// TASKLET (Software IRQ context, atomic)
// ═══════════════════════════════════════════

struct tasklet_struct work;

void work_callback(unsigned long data)
{
    struct my_device *dev = (void *)data;

    // Atomic context!
    // Can't sleep, acquire locks, allocate memory
    // But fast - runs soon after IRQ

    my_device_process_data(dev);
}

// In probe:
tasklet_init(&my_dev->work, work_callback, (unsigned long)my_dev);

// In interrupt handler:
tasklet_schedule(&my_dev->work);

// In remove:
tasklet_kill(&my_dev->work);

// ═══════════════════════════════════════════
// WORKQUEUE (Process context, can sleep)
// ═══════════════════════════════════════════

struct work_struct work;

void work_callback(struct work_struct *w)
{
    struct my_device *dev = container_of(w, typeof(*dev), work);

    // Process context!
    // Can sleep, acquire locks, allocate memory
    // But slower than tasklet - runs when worker thread scheduled

    my_device_process_data(dev);
}

// In probe:
INIT_WORK(&my_dev->work, work_callback);

// In interrupt handler:
schedule_work(&my_dev->work);

// In remove:
flush_work(&my_dev->work);  // Wait for pending work to complete
```

---

### 5. DMA in Drivers

**File Locations:**
- `/include/linux/dma-mapping.h` - Public API
- `/kernel/dma/mapping.c` - DMA API implementation
- `/kernel/dma/coherent.c` - Coherent DMA buffers
- `/kernel/dma/pool.c` - DMA pools

**DMA Allocation Types:**

```c
// ═════════════════════════════════════════════════
// COHERENT DMA (Cache-Coherent)
// ═════════════════════════════════════════════════
// Device and CPU see same data without explicit sync

dma_addr_t dma_handle;
void *vaddr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

// vaddr = CPU virtual address
// dma_handle = device bus address
// CPU writes: immediately visible to device
// Device writes: immediately visible to CPU

dma_free_coherent(dev, size, vaddr, dma_handle);

// Managed version (auto-freed on device removal):
void *vaddr = dmam_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

// ═════════════════════════════════════════════════
// STREAMING DMA (Requires Explicit Sync)
// ═════════════════════════════════════════════════
// Data may not be immediately visible without cache flush

void *kernel_vaddr = kmalloc(size, GFP_KERNEL);

// Map for device read
dma_addr_t dma_handle = dma_map_single(dev, kernel_vaddr, size, DMA_TO_DEVICE);

// Flush CPU cache → device sees latest data
// Device reads from dma_handle

// When device done writing:
dma_unmap_single(dev, dma_handle, size, DMA_TO_DEVICE);

// Read back modified data
dma_unmap_single(dev, dma_handle, size, DMA_FROM_DEVICE);
// Invalidate cache, read from memory

// ═════════════════════════════════════════════════
// SCATTER-GATHER DMA
// ═════════════════════════════════════════════════

struct scatterlist sg_table[10];
int nents;

// Build scatter-gather table
sg_init_table(sg_table, 10);
for (i = 0; i < nents; i++) {
    sg_set_buf(&sg_table[i], buffer[i], size[i]);
}

// Map all in one call
nents = dma_map_sg(dev, sg_table, nents, DMA_FROM_DEVICE);
if (!nents)
    return -ENOMEM;

// Iterate mapped entries
for_each_sg(sg_table, sg, nents, i) {
    dma_addr_t addr = sg_dma_address(sg);
    u32 len = sg_dma_len(sg);

    // Program device to read/write to addr/len
}

// Unmap when done
dma_unmap_sg(dev, sg_table, nents, DMA_FROM_DEVICE);

// ═════════════════════════════════════════════════
// DMA POOLS
// ═════════════════════════════════════════════════
// Pre-allocate contiguous DMA buffers (useful for small objects)

struct dma_pool *pool = dma_pool_create("ring", dev, size, align, boundary);

dma_addr_t handle;
void *vaddr = dma_pool_alloc(pool, GFP_KERNEL, &handle);

// Use vaddr/handle...

dma_pool_free(pool, vaddr, handle);
dma_pool_destroy(pool);
```

**DMA Attributes:**

```c
#define DMA_ATTR_WRITE_COMBINE     (1UL << 2)  // Write combining (faster writes)
#define DMA_ATTR_NO_KERNEL_MAPPING (1UL << 4)  // Only device access (save CPU VA space)
#define DMA_ATTR_SKIP_CPU_SYNC     (1UL << 5)  // Skip cache sync (if you know it's safe)
#define DMA_ATTR_FORCE_CONTIGUOUS  (1UL << 6)  // Must be physically contiguous
#define DMA_ATTR_WEAK_ORDERING     (1UL << 1)  // Weak ordering OK

// Example:
vaddr = dma_alloc_attrs(dev, size, &handle, GFP_KERNEL,
                       DMA_ATTR_WRITE_COMBINE | DMA_ATTR_FORCE_CONTIGUOUS);
```

---

### 6. Power Management in Drivers

**File Locations:**
- `/include/linux/pm.h` - PM structures
- `/drivers/base/power/main.c` - System PM (suspend/resume)
- `/drivers/base/power/runtime.c` - Runtime PM (dynamic sleep)

**Power State Callbacks:**

```c
struct dev_pm_ops {
    // System suspend/resume (entire system sleep)
    int (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int (*suspend)(struct device *dev);      // Suspend to RAM
    int (*resume)(struct device *dev);       // Wake from RAM
    int (*freeze)(struct device *dev);       // Suspend to disk prep
    int (*thaw)(struct device *dev);         // Resume after hibernation prep
    int (*poweroff)(struct device *dev);     // Turn off for hibernate
    int (*restore)(struct device *dev);      // Restore after hibernation

    // Runtime PM (dynamic device power control)
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

**System Suspend Flow:**

```
User requests suspend (echo mem > /sys/power/state)
    ↓
prepare() - Prevent new operations
    ↓
suspend() - Save device state, stop I/O
    ↓
Device powered down (CPU sleeps)
    ↓
Wake event (button, network, etc.)
    ↓
resume() - Restore device state, restart I/O
    ↓
complete() - Resume operations
```

**System Suspend Example:**

```c
#ifdef CONFIG_PM_SLEEP
static int my_device_suspend(struct device *dev)
{
    struct my_device *my_dev = dev_get_drvdata(dev);

    // Stop I/O
    disable_irq(my_dev->irq);
    my_device_stop_dma(my_dev);

    // Save device state
    my_dev->saved_state = readl(my_dev->regs + STATE_REG);

    return 0;
}

static int my_device_resume(struct device *dev)
{
    struct my_device *my_dev = dev_get_drvdata(dev);

    // Restore device state
    writel(my_dev->saved_state, my_dev->regs + STATE_REG);

    // Resume I/O
    my_device_start_dma(my_dev);
    enable_irq(my_dev->irq);

    return 0;
}

static const struct dev_pm_ops my_device_pm_ops = {
    .suspend = my_device_suspend,
    .resume = my_device_resume,
};
#else
#define my_device_pm_ops NULL
#endif

static struct platform_driver my_driver = {
    .probe = my_device_probe,
    .remove = my_device_remove,
    .driver = {
        .name = "my_device",
        .pm = &my_device_pm_ops,
    },
};
```

**Runtime PM (Dynamic Power Control):**

For devices that can be powered down when not in use:

```c
#ifdef CONFIG_PM
static int my_device_runtime_suspend(struct device *dev)
{
    struct my_device *my_dev = dev_get_drvdata(dev);

    // Device can go to sleep
    my_device_sleep(my_dev);
    clk_disable(my_dev->clock);

    return 0;
}

static int my_device_runtime_resume(struct device *dev)
{
    struct my_device *my_dev = dev_get_drvdata(dev);

    // Wake device
    clk_enable(my_dev->clock);
    my_device_wake(my_dev);

    return 0;
}

static const struct dev_pm_ops my_device_pm_ops = {
    .runtime_suspend = my_device_runtime_suspend,
    .runtime_resume = my_device_runtime_resume,
};
#endif

// In probe:
pm_runtime_enable(dev);
pm_runtime_set_autosuspend_delay(dev, 1000);  // Sleep after 1 second idle
pm_runtime_use_autosuspend(dev);

// Before using device:
pm_runtime_get_sync(dev);  // Wake up device
// Use device...
pm_runtime_put_autosuspend(dev);  // OK to sleep soon
```

---

### 7. Managed Resources (devres) Framework

**Purpose:** Automatic resource cleanup when device is removed.

**File Location:** `/drivers/base/devres.c`

**Key Concept:** Each resource allocation tracks a cleanup function that's auto-called on device removal.

**Common Managed Allocations:**

```c
// Memory allocation
void *devres_alloc(dr_release_t release, size_t size, gfp_t gfp);
void *devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp);
char *devm_kstrdup(struct device *dev, const char *s, gfp_t gfp);

// I/O memory
void *devm_ioremap(struct device *dev, resource_size_t offset, size_t size);
void *devm_ioremap_resource(struct device *dev, struct resource *res);
void *devm_platform_ioremap_resource(struct platform_device *pdev, int index);

// Interrupts
int devm_request_irq(struct device *dev, unsigned int irq,
                    irq_handler_t handler, unsigned long flags,
                    const char *devname, void *dev_id);
int devm_request_threaded_irq(...);
void devm_free_irq(struct device *dev, unsigned int irq, void *dev_id);

// DMA
void *dmam_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t gfp);
void dmam_free_coherent(struct device *dev, size_t size,
                       void *vaddr, dma_addr_t dma_handle);

// Clocks
struct clk *devm_clk_get(struct device *dev, const char *id);
int devm_clk_prepare(struct device *dev, struct clk *clk);

// Regulators
struct regulator *devm_regulator_get(struct device *dev, const char *id);

// GPIO
int devm_gpio_request(struct device *dev, unsigned gpio, const char *label);
struct gpio_desc *devm_gpiod_get(struct device *dev, const char *con_id,
                                enum gpiod_flags flags);
```

**Managed Resource Cleanup Guarantee:**

```
On probe() failure or remove() call:
1. Walk list of managed resources in LIFO order (reverse allocation)
2. Call cleanup function for each resource
3. Free metadata

Example sequence:
Probe:
  devm_kmalloc() - alloc A (cleanup: kfree)
  devm_request_irq() - alloc B (cleanup: free_irq)
  custom_init() - fails!
    ↓
Cleanup:
  free_irq(B)  - reverse order (last allocated, first freed)
  kfree(A)
```

**Custom Managed Resources:**

```c
struct my_resource {
    int data;
};

void my_resource_release(struct device *dev, void *res)
{
    struct my_resource *my_res = res;

    // Cleanup code
    my_resource_cleanup(my_res);
}

// In probe:
struct my_resource *res = devres_alloc(my_resource_release,
                                      sizeof(*res), GFP_KERNEL);
if (!res)
    return -ENOMEM;

res->data = initialize_my_resource();
devres_add(dev, res);  // Register cleanup on device removal
```

---

## Part 3: Key Files Summary

### Critical Files by Subsystem

**Memory Management:**

| File | Lines | Purpose |
|------|-------|---------|
| `mm/page_alloc.c` | 7,628 | Buddy allocator, page allocation |
| `mm/slub.c` | 10,084 | SLUB slab allocator for kmalloc |
| `mm/vmscan.c` | 7,919 | Page reclaim, LRU lists, kswapd |
| `mm/memory.c` | 7,326 | Page faults, virtual memory, page tables |
| `mm/filemap.c` | 4,689 | Page cache, file I/O |
| `mm/migrate.c` | 2,500+ | Page migration for hotplug/NUMA |
| `mm/kasan/` | ~220KB | Kernel AddressSanitizer |
| `mm/kfence/` | ~82KB | Kernel Fence allocator |
| `mm/kmsan/` | ~87KB | Kernel Memory Sanitizer |

**Device Model & Drivers:**

| File | Lines | Purpose |
|------|-------|---------|
| `drivers/base/core.c` | 2000+ | Device registration and lifecycle |
| `drivers/base/driver.c` | 1000+ | Driver binding and matching |
| `drivers/base/bus.c` | 1500+ | Bus type management |
| `drivers/base/dd.c` | 1500+ | Probe/remove sequencing, deferred probe |
| `drivers/base/devres.c` | 600+ | Managed resource framework |
| `drivers/base/platform.c` | 1000+ | Platform device framework |
| `drivers/base/power/main.c` | 1000+ | System suspend/resume |
| `drivers/base/power/runtime.c` | 1000+ | Runtime PM |

**Interrupt Handling:**

| File | Lines | Purpose |
|------|-------|---------|
| `kernel/irq/manage.c` | 2000+ | IRQ request/free, handler setup |
| `kernel/irq/handle.c` | 500+ | IRQ dispatcher |
| `kernel/irq/chip.c` | 1000+ | IRQ chip operations |
| `kernel/irq/irqdesc.c` | 1000+ | IRQ descriptor management |
| `kernel/irq/irqdomain.c` | 1500+ | IRQ domain mapping (device tree) |

**Module System:**

| File | Lines | Purpose |
|------|-------|---------|
| `kernel/module/main.c` | 2500+ | Module loading, lifecycle |
| `kernel/module/kallsyms.c` | 1000+ | Symbol resolution, .symtab |
| `kernel/module/sysfs.c` | 500+ | `/sys/module/*/` interface |
| `kernel/module/signing.c` | 500+ | Module signature verification |

### Critical Header Files

| Header | Purpose |
|--------|---------|
| `include/linux/device.h` | Core device/driver structures |
| `include/linux/platform_device.h` | Platform device API |
| `include/linux/pci.h` | PCI device/driver API |
| `include/linux/interrupt.h` | Interrupt request/handler API |
| `include/linux/dma-mapping.h` | DMA allocation/mapping API |
| `include/linux/gfp.h` | Page allocation flags |
| `include/linux/mm.h` | Memory management core API |
| `include/linux/mmzone.h` | Zone, page, NUMA structures |
| `include/linux/slab.h` | kmalloc/kfree API |
| `include/linux/workqueue.h` | Work queue API |
| `include/linux/module.h` | Module macros and structures |

---

## Conclusion

The Linux kernel's memory management and driver subsystems are highly sophisticated:

**Memory Management:**
- **Multi-level allocation:** Pages → slabs → kmalloc
- **Intelligent reclaim:** LRU-based eviction, kswapd background daemon
- **Comprehensive debugging:** KASAN (out-of-bounds), KFENCE (UAF), KMSAN (uninitialized)
- **Hardware abstraction:** Page tables, zones, NUMA support for diverse systems

**Driver Architecture:**
- **Unified device model:** All devices/drivers use standardized structures
- **Flexible bus support:** Platform, PCI, USB, I2C, SPI with common patterns
- **Automatic cleanup:** Managed resources (devres) prevent resource leaks
- **Deferred probing:** Solves device dependency ordering transparently

Together, these subsystems enable the kernel to efficiently manage hardware, memory, and software in a reliable, debuggable, and extensible manner across 23+ processor architectures and thousands of different devices.
