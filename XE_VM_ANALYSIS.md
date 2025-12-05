# Xe Driver Virtual Memory Management Analysis

## Table of Contents
1. [Virtual Address Management](#1-virtual-address-management)
2. [Buffer Objects (Physical Memory)](#2-buffer-objects-physical-memory)
3. [Page Table Structure](#3-page-table-structure)
4. [Address Translation Flow](#4-address-translation-flow)
5. [Page Table Manipulations](#5-page-table-manipulations)
6. [Memory Mapping Operations](#6-memory-mapping-operations)

---

## 1. Virtual Address Management

### 1.1 Virtual Memory Area (VMA) Structure

**Evidence:** `drivers/gpu/drm/xe/xe_vm_types.h:89-154`

```c
struct xe_vma {
    /** @gpuva: Base GPUVA object */
    struct drm_gpuva gpuva;  // Contains virtual address range
    
    /** @tile_mask: Tile mask of where to create binding for this VMA */
    u8 tile_mask;
    
    /** @tile_present: Tile mask of binding are present for this VMA */
    u8 tile_present;
    
    /** @attr: The attributes of vma which determines the migration policy */
    struct xe_vma_mem_attr attr;
};
```

**Key Points:**
- VMA represents a contiguous virtual address range in GPU address space
- Uses `drm_gpuva` base structure from DRM GPUVA framework
- Tracks which GPU tiles have bindings for this VMA
- Stores memory attributes (PAT index, atomic access, migration policy)

### 1.2 Virtual Address Accessors

**Evidence:** `drivers/gpu/drm/xe/xe_vm.h:112-125`

```c
static inline u64 xe_vma_start(struct xe_vma *vma)
{
    return vma->gpuva.va.addr;  // Virtual address start
}

static inline u64 xe_vma_size(struct xe_vma *vma)
{
    return vma->gpuva.va.range;  // Size in bytes
}

static inline u64 xe_vma_end(struct xe_vma *vma)
{
    return xe_vma_start(vma) + xe_vma_size(vma);  // Virtual address end
}
```

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:2072-2079`

```c
static bool vma_matches(struct xe_vma *vma, u64 page_addr)
{
    if (page_addr > xe_vma_end(vma) - 1 ||
        page_addr + SZ_4K - 1 < xe_vma_start(vma))
        return false;
    
    return true;
}
```

**Key Points:**
- Virtual addresses are 64-bit values
- Address ranges are tracked as (start, size) pairs
- Used for VMA lookup by address (e.g., in page fault handler)

### 1.3 Virtual Memory (VM) Structure

**Evidence:** `drivers/gpu/drm/xe/xe_vm_types.h:168-343`

```c
struct xe_vm {
    /** @gpuvm: base GPUVM used to track VMAs */
    struct drm_gpuvm gpuvm;
    
    u64 size;  // Total VM size
    
    /** @pt_root: Page table root for each tile */
    struct xe_pt *pt_root[XE_MAX_TILES_PER_DEVICE];
    
    /** @usm: unified memory state */
    struct {
        /** @asid: address space ID, unique to each VM */
        u32 asid;
        /** @last_fault_vma: Last fault VMA, used for fast lookup */
        struct xe_vma *last_fault_vma;
    } usm;
};
```

**Key Points:**
- Each VM has its own address space identified by ASID
- Multiple page table roots (one per GPU tile)
- Tracks VMAs through `drm_gpuvm` tree structure
- Fast path for page fault lookups using `last_fault_vma`

### 1.4 VMA Lookup by Address

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:2087-2099`

```c
struct xe_vma *xe_vm_find_vma_by_addr(struct xe_vm *vm, u64 page_addr)
{
    struct xe_vma *vma = NULL;
    
    if (vm->usm.last_fault_vma) {   /* Fast lookup */
        if (vma_matches(vm->usm.last_fault_vma, page_addr))
            vma = vm->usm.last_fault_vma;
    }
    if (!vma)
        vma = xe_vm_find_overlapping_vma(vm, page_addr, SZ_4K);
    
    return vma;
}
```

**Key Points:**
- Optimized lookup using cached `last_fault_vma` for page faults
- Falls back to tree-based lookup for misses
- Used extensively in page fault handling

---

## 2. Buffer Objects (Physical Memory)

### 2.1 Buffer Object Structure

**Evidence:** `drivers/gpu/drm/xe/xe_bo.h:127-130`

```c
static inline struct xe_bo *ttm_to_xe_bo(const struct ttm_buffer_object *bo)
{
    return container_of(bo, struct xe_bo, ttm);
}
```

**Key Points:**
- Buffer objects wrap TTM (Translation Table Manager) buffer objects
- Represent physical memory allocations (VRAM, system memory, etc.)

### 2.2 Physical Address Extraction

**Evidence:** `drivers/gpu/drm/xe/xe_res_cursor.h:318-331`

```c
static inline u64 xe_res_dma(const struct xe_res_cursor *cur)
{
    if (cur->dma_addr)
        return cur->dma_start + cur->start;
    else if (cur->sgl)
        return sg_dma_address(cur->sgl) + cur->start;
    else
        return cur->start;
}
```

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:1287-1307`

```c
static u64 xelp_pde_encode_bo(struct xe_bo *bo, u64 bo_offset)
{
    u64 pde;
    
    pde = xe_bo_addr(bo, bo_offset, XE_PAGE_SIZE);  // Get physical address
    pde |= XE_PAGE_PRESENT | XE_PAGE_RW;
    pde |= pde_encode_pat_index(pde_pat_index(bo));
    
    return pde;
}

static u64 xelp_pte_encode_bo(struct xe_bo *bo, u64 bo_offset,
                              u16 pat_index, u32 pt_level)
{
    u64 pte;
    
    pte = xe_bo_addr(bo, bo_offset, XE_PAGE_SIZE);  // Physical address
    pte |= XE_PAGE_PRESENT | XE_PAGE_RW;
    pte |= pte_encode_pat_index(pat_index, pt_level);
    pte |= pte_encode_ps(pt_level);
    
    if (xe_bo_is_vram(bo) || xe_bo_is_stolen_devmem(bo))
        pte |= XE_PPGTT_PTE_DM;  // Domain mode for VRAM
    
    return pte;
}
```

**Key Points:**
- Physical addresses extracted from BOs using `xe_bo_addr()`
- Resource cursor (`xe_res_cursor`) tracks physical memory segments
- Supports VRAM, system memory, scatter-gather lists, and DMA arrays
- Physical addresses encoded into page table entries

### 2.3 Resource Cursor for Physical Memory Walking

**Evidence:** `drivers/gpu/drm/xe/xe_res_cursor.h:45-70`

```c
struct xe_res_cursor {
    /** @start: Start of cursor */
    u64 start;
    /** @size: Size of the current segment */
    u64 size;
    /** @remaining: Remaining bytes in cursor */
    u64 remaining;
    /** @dma_addr: Current element in a struct drm_pagemap_addr array */
    const struct drm_pagemap_addr *dma_addr;
    /** @sgl: Scatterlist for cursor */
    struct scatterlist *sgl;
    /** @dma_start: DMA start address for the current segment */
    u64 dma_start;
    /** @dma_seg_size: Size of the current DMA segment */
    u64 dma_seg_size;
};
```

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:765-777`

```c
if (!xe_vma_is_null(vma) && !range) {
    if (xe_vma_is_userptr(vma))
        xe_res_first_dma(to_userptr_vma(vma)->userptr.pages.dma_addr, 0,
                         xe_vma_size(vma), &curs);
    else if (xe_bo_is_vram(bo) || xe_bo_is_stolen(bo))
        xe_res_first(bo->ttm.resource, xe_vma_bo_offset(vma),
                     xe_vma_size(vma), &curs);
    else
        xe_res_first_sg(xe_bo_sg(bo), xe_vma_bo_offset(vma),
                        xe_vma_size(vma), &curs);
}
```

**Key Points:**
- Cursor abstracts different physical memory types (VRAM, system, SG lists)
- Used to walk physical memory segments during page table construction
- Advances through memory segments as page table entries are created

---

## 3. Page Table Structure

### 3.1 Page Table Node Structure

**Evidence:** `drivers/gpu/drm/xe/xe_pt_types.h:27-38`

```c
struct xe_pt {
    struct xe_ptw base;      // Base walker structure
    struct xe_bo *bo;        // Buffer object backing the page table
    unsigned int level;      // Level in the tree (0 = leaf)
    unsigned int num_live;   // Number of live entries
    bool rebind;
    bool is_compact;         // Whether using compact format
};
```

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:29-34`

```c
struct xe_pt_dir {
    struct xe_pt pt;
    /** @children: Array of page-table child nodes */
    struct xe_ptw *children[XE_PDES];
    /** @staging: Array of page-table staging nodes */
    struct xe_ptw *staging[XE_PDES];
};
```

**Key Points:**
- Multi-level page table tree (up to 5 levels)
- Each page table is a 4KB buffer object
- Level 0 = leaf PTEs, higher levels = page directories
- Staging structure for safe updates

### 3.2 Page Table Levels and Shifts

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:45-48`

```c
static const u64 xe_normal_pt_shifts[] = {12, 21, 30, 39, 48};
static const u64 xe_compact_pt_shifts[] = {16, 21, 30, 39, 48};

#define XE_PT_HIGHEST_LEVEL (ARRAY_SIZE(xe_normal_pt_shifts) - 1)
```

**Evidence:** `drivers/gpu/drm/xe/xe_pt.h:25`

```c
/* Largest huge pte is currently 1GiB. May become device dependent. */
#define MAX_HUGEPTE_LEVEL 2
```

**Key Points:**
- Normal mode: 4KB base pages (shift 12), supports 2MB (shift 21) and 1GB (shift 30)
- Compact mode: 64KB base pages (shift 16)
- Level 0 = 4KB/64KB pages, Level 1 = 2MB pages, Level 2 = 1GB pages
- Up to 48-bit virtual address space

### 3.3 Page Table Root in VM

**Evidence:** `drivers/gpu/drm/xe/xe_vm_types.h:206-207`

```c
struct xe_pt *pt_root[XE_MAX_TILES_PER_DEVICE];
struct xe_pt *scratch_pt[XE_MAX_TILES_PER_DEVICE][XE_VM_MAX_LEVEL];
```

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:1532-1548`

```c
for (id = 0; id < xe->info.tile_count; id++) {
    struct xe_tile *tile = xe_device_get_tile(xe, id);
    
    vm->pt_root[id] = xe_pt_create(vm, tile, xe->info.vm_max_level,
                                    exec);
    if (IS_ERR(vm->pt_root[id])) {
        err = PTR_ERR(vm->pt_root[id]);
        vm->pt_root[id] = NULL;
        xe_vm_pt_destroy(vm);
        goto err_unlock;
    }
}
```

**Key Points:**
- Each GPU tile has its own page table root
- Scratch page tables point to blank pages for invalid accesses
- Root created at VM creation time

---

## 4. Address Translation Flow

### 4.1 Page Table Entry Encoding

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:1310-1325`

```c
static u64 xelp_pte_encode_vma(u64 pte, struct xe_vma *vma,
                               u16 pat_index, u32 pt_level)
{
    pte |= XE_PAGE_PRESENT;
    
    if (likely(!xe_vma_read_only(vma)))
        pte |= XE_PAGE_RW;
    
    pte |= pte_encode_pat_index(pat_index, pt_level);
    pte |= pte_encode_ps(pt_level);  // Page size bits
    
    if (unlikely(xe_vma_is_null(vma)))
        pte |= XE_PTE_NULL;
    
    return pte;
}
```

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:544-548`

```c
pte = vm->pt_ops->pte_encode_vma(is_null ? 0 :
                                 xe_res_dma(curs) +  // Physical address
                                 xe_walk->dma_offset,
                                 xe_walk->vma,
                                 pat_index, level);
```

**Key Points:**
- PTE combines physical address with attributes (present, RW, PAT, page size)
- Physical address comes from resource cursor (`xe_res_dma()`)
- Attributes come from VMA (read-only, PAT index, etc.)

### 4.2 Page Table Walk During Binding

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:516-580`

```c
static int
xe_pt_stage_bind_entry(struct xe_ptw *parent, pgoff_t offset,
                      unsigned int level, u64 addr, u64 next,
                      struct xe_ptw **child,
                      enum page_walk_action *action,
                      struct xe_pt_walk *walk)
{
    // ... setup ...
    
    /* Is this a leaf entry ?*/
    if (level == 0 || xe_pt_hugepte_possible(addr, next, level, xe_walk)) {
        struct xe_res_cursor *curs = xe_walk->curs;
        bool is_null = xe_vma_is_null(xe_walk->vma);
        bool is_vram = is_null ? false : xe_res_is_vram(curs);
        
        // Encode PTE with physical address
        pte = vm->pt_ops->pte_encode_vma(
            is_null ? 0 : xe_res_dma(curs) + xe_walk->dma_offset,
            xe_walk->vma, pat_index, level);
        
        // Insert PTE into page table
        ret = xe_pt_insert_entry(xe_walk, xe_parent, offset, NULL, pte);
        
        // Advance physical memory cursor
        if (!is_null && !xe_walk->clear_pt)
            xe_res_next(curs, next - addr);
        xe_walk->va_curs_start = next;
        
        return ret;
    }
    
    // Descend to lower level or create new page table
    // ...
}
```

**Key Points:**
- Walks virtual address range (`addr` to `next`)
- For each virtual page, gets corresponding physical address from cursor
- Encodes PTE and inserts into page table
- Advances both virtual and physical cursors in lockstep

### 4.3 Complete Binding Flow

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:698-787`

```c
static int
xe_pt_stage_bind(struct xe_tile *tile, struct xe_vma *vma,
                 struct xe_svm_range *range,
                 struct xe_vm_pgtable_update *entries,
                 u32 *num_entries, bool clear_pt)
{
    struct xe_bo *bo = xe_vma_bo(vma);
    struct xe_res_cursor curs;
    struct xe_vm *vm = xe_vma_vm(vma);
    
    // Initialize physical memory cursor based on BO type
    if (!xe_vma_is_null(vma) && !range) {
        if (xe_vma_is_userptr(vma))
            xe_res_first_dma(..., &curs);
        else if (xe_bo_is_vram(bo) || xe_bo_is_stolen(bo))
            xe_res_first(bo->ttm.resource, ..., &curs);
        else
            xe_res_first_sg(xe_bo_sg(bo), ..., &curs);
    }
    
    // Walk page table tree for virtual address range
    ret = xe_pt_walk_range(&pt->base, pt->level,
                           range ? range->base.itree.start : xe_vma_start(vma),
                           range ? range->base.itree.last + 1 : xe_vma_end(vma),
                           &xe_walk.base);
    
    return ret;
}
```

**Key Points:**
- Initializes physical memory cursor based on BO type
- Walks page table tree for VMA's virtual address range
- Creates/updates PTEs mapping virtual to physical addresses
- Handles VRAM, system memory, userptr, and scatter-gather cases

---

## 5. Page Table Manipulations

### 5.1 Page Table Entry Insertion

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:386-432`

```c
static int
xe_pt_insert_entry(struct xe_pt_stage_bind_walk *xe_walk, struct xe_pt *parent,
                  pgoff_t offset, struct xe_pt *xe_child, u64 pte)
{
    struct xe_pt_update *upd = &xe_walk->wupd.updates[parent->level];
    
    // Check if this is a shared page table
    ret = xe_pt_new_shared(&xe_walk->wupd, parent, offset, true);
    
    if (likely(!upd->preexisting)) {
        /* Continue building a non-connected subtree. */
        struct iosys_map *map = &parent->bo->vmap;
        
        if (unlikely(xe_child)) {
            parent->base.children[offset] = &xe_child->base;
            parent->base.staging[offset] = &xe_child->base;
        }
        
        // Write PTE directly to page table BO
        xe_pt_write(xe_walk->vm->xe, map, offset, pte);
        parent->num_live++;
    } else {
        /* Shared pt. Stage update. */
        // Stage update for later GPU job
        entry->pt_entries[idx].pt = xe_child;
        entry->pt_entries[idx].pte = pte;
        entry->qwords++;
    }
    
    return 0;
}
```

**Key Points:**
- New page tables: Write PTEs directly via CPU
- Shared page tables: Stage updates for GPU job (to avoid races)
- Maintains both `children` and `staging` arrays

### 5.2 Page Table Update Operations

**Evidence:** `drivers/gpu/drm/xe/xe_pt_types.h:56-74`

```c
struct xe_vm_pgtable_update {
    /** @bo: page table bo to write to */
    struct xe_bo *pt_bo;
    
    /** @ofs: offset inside this PTE to begin writing to (in qwords) */
    u32 ofs;
    
    /** @qwords: number of PTE's to write */
    u32 qwords;
    
    /** @pt: opaque pointer useful for the caller */
    struct xe_pt *pt;
    
    /** @pt_entries: Newly added pagetable entries */
    struct xe_pt_entry *pt_entries;
    
    /** @flags: Target flags */
    u32 flags;
};
```

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:984-1000`

```c
static void
xe_vm_populate_pgtable(struct xe_migrate_pt_update *pt_update, struct xe_tile *tile,
                      struct iosys_map *map, void *data,
                      u32 qword_ofs, u32 num_qwords,
                      const struct xe_vm_pgtable_update *update)
{
    struct xe_pt_entry *ptes = update->pt_entries;
    u64 *ptr = data;
    u32 i;
    
    for (i = 0; i < num_qwords; i++) {
        if (map)
            xe_map_wr(tile_to_xe(tile), map, (qword_ofs + i) *
                     sizeof(u64), u64, ptes[i].pte);
        else
            ptr[i] = ptes[i].pte;
    }
}
```

**Key Points:**
- Batched updates: Multiple PTEs written together
- Can be done via CPU (map) or GPU (data buffer)
- Used for shared page table updates

### 5.3 Page Table Clearing (Unbind)

**Evidence:** `drivers/gpu/drm/xe/xe_pt.c:855-883`

```c
static int xe_pt_zap_ptes_entry(struct xe_ptw *parent, pgoff_t offset,
                                unsigned int level, u64 addr, u64 next,
                                struct xe_ptw **child,
                                enum page_walk_action *action,
                                struct xe_pt_walk *walk)
{
    struct xe_pt_zap_ptes_walk *xe_walk =
        container_of(walk, typeof(*xe_walk), base);
    struct xe_pt *xe_child = container_of(*child, typeof(*xe_child), base);
    pgoff_t end_offset;
    
    // Determine non-shared entry offsets
    if (xe_pt_nonshared_offsets(addr, next, --level, walk, action, &offset,
                                &end_offset)) {
        // Zero out PTEs
        xe_map_memset(tile_to_xe(xe_walk->tile), &xe_child->bo->vmap,
                     offset * sizeof(u64), 0,
                     (end_offset - offset) * sizeof(u64));
        xe_walk->needs_invalidate = true;
    }
    
    return 0;
}
```

**Key Points:**
- Zeros out PTEs to invalidate mappings
- Only clears non-shared entries (shared entries handled separately)
- Sets flag for TLB invalidation

---

## 6. Memory Mapping Operations

### 6.1 VMA Map Operation

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:2580-2601`

```c
case DRM_GPUVA_OP_MAP:
{
    struct xe_vma_mem_attr default_attr = {
        .preferred_loc = {
            .devmem_fd = DRM_XE_PREFERRED_LOC_DEFAULT_DEVICE,
            .migration_policy = DRM_XE_MIGRATE_ALL_PAGES,
        },
        .atomic_access = DRM_XE_ATOMIC_UNDEFINED,
        .default_pat_index = op->map.pat_index,
        .pat_index = op->map.pat_index,
    };
    
    vma = new_vma(vm, &op->base.map, &default_attr, flags);
    if (IS_ERR(vma))
        return PTR_ERR(vma);
    
    op->map.vma = vma;
    if (((op->map.immediate || !xe_vm_in_fault_mode(vm)) &&
         !(op->map.vma_flags & XE_VMA_SYSTEM_ALLOCATOR)) ||
        op->map.invalidate_on_bind)
        xe_vma_ops_incr_pt_update_ops(vops, op->tile_mask, 1);
    break;
}
```

**Key Points:**
- Creates new VMA for virtual address range
- Sets memory attributes (PAT, atomic access, migration policy)
- Schedules page table update if immediate bind or not in fault mode

### 6.2 VMA Remap Operation (Split)

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:2603-2691`

```c
case DRM_GPUVA_OP_REMAP:
{
    struct xe_vma *old = gpuva_to_vma(op->base.remap.unmap->va);
    u64 start = xe_vma_start(old), end = xe_vma_end(old);
    
    // Adjust start/end based on prev/next VMAs
    if (op->base.remap.prev)
        start = op->base.remap.prev->va.addr + op->base.remap.prev->va.range;
    if (op->base.remap.next)
        end = op->base.remap.next->va.addr;
    
    // Create prev VMA if needed
    if (op->base.remap.prev) {
        vma = new_vma(vm, op->base.remap.prev, &old->attr, flags);
        op->remap.prev = vma;
        // Check if rebind needed
        op->remap.skip_prev = skip || (!xe_vma_is_userptr(old) &&
                                      IS_ALIGNED(xe_vma_end(vma),
                                                 xe_vma_max_pte_size(old)));
    }
    
    // Create next VMA if needed
    if (op->base.remap.next) {
        vma = new_vma(vm, op->base.remap.next, &old->attr, flags);
        op->remap.next = vma;
        // Similar skip logic
    }
    
    xe_vma_ops_incr_pt_update_ops(vops, op->tile_mask, num_remap_ops);
    break;
}
```

**Key Points:**
- Splits existing VMA when unmapping middle portion
- Creates new VMAs for remaining portions
- Optimizes rebind by skipping if page size aligned

### 6.3 VMA Unmap Operation

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:2693-2703`

```c
case DRM_GPUVA_OP_UNMAP:
    vma = gpuva_to_vma(op->base.unmap.va);
    
    if (xe_vma_is_cpu_addr_mirror(vma) &&
        xe_svm_has_mapping(vm, xe_vma_start(vma), xe_vma_end(vma)))
        return -EBUSY;
    
    if (!xe_vma_is_cpu_addr_mirror(vma))
        xe_vma_ops_incr_pt_update_ops(vops, op->tile_mask, 1);
    break;
```

**Key Points:**
- Removes VMA from address space
- Schedules page table update to clear PTEs
- Handles SVM (Shared Virtual Memory) special cases

### 6.4 Page Table Update Execution

**Evidence:** `drivers/gpu/drm/xe/xe_vm.c:3098-3111`

```c
err = xe_pt_update_ops_prepare(tile, vops);
if (err)
    goto err_unlock;

// ... commit operations ...

if (vops->pt_update_ops[id].num_ops) {
    fence = xe_pt_update_ops_run(tile, vops);
    if (IS_ERR(fence)) {
        err = PTR_ERR(fence);
        goto err_unlock;
    }
    dma_fence_wait(fence, false);
    dma_fence_put(fence);
}
```

**Key Points:**
- Prepares page table updates (collects all changes)
- Runs updates via GPU job (for shared PTs) or CPU (for new PTs)
- Waits for completion before proceeding

---

## Summary: Complete Address Translation Flow

1. **Virtual Address Range**: User requests mapping at virtual address `va_start` with size `size`
   - Evidence: `xe_vma_start()`, `xe_vma_size()`, `xe_vma_end()`

2. **VMA Creation**: Driver creates `xe_vma` structure tracking virtual address range
   - Evidence: `new_vma()` in `xe_vm.c:2590`

3. **Physical Memory**: Buffer object (`xe_bo`) provides physical memory backing
   - Evidence: `xe_vma_bo()`, `xe_res_first()`, `xe_res_dma()`

4. **Page Table Walk**: Driver walks page table tree for virtual address range
   - Evidence: `xe_pt_walk_range()` in `xe_pt_walk.c:73`

5. **PTE Encoding**: For each virtual page, encodes PTE with physical address
   - Evidence: `xe_pt_stage_bind_entry()` in `xe_pt.c:516-580`

6. **Page Table Update**: Inserts PTEs into page table (CPU for new, GPU for shared)
   - Evidence: `xe_pt_insert_entry()`, `xe_pt_update_ops_run()`

7. **TLB Invalidation**: Invalidates TLB so GPU sees new mappings
   - Evidence: `xe_tlb_inval_job` (referenced in various places)

8. **Address Translation**: GPU uses page table to translate virtual → physical addresses
   - Hardware performs translation using page table root in VM context

---

## Key Data Structures Summary

| Structure | Purpose | Key Fields |
|-----------|---------|------------|
| `xe_vma` | Virtual Memory Area | `gpuva.va.addr`, `gpuva.va.range`, `tile_present` |
| `xe_vm` | Virtual Memory | `pt_root[]`, `asid`, `gpuvm` |
| `xe_pt` | Page Table Node | `bo`, `level`, `num_live` |
| `xe_bo` | Buffer Object | `ttm.resource`, physical memory backing |
| `xe_res_cursor` | Physical Memory Cursor | `dma_start`, `size`, `dma_addr` |
| `xe_vm_pgtable_update` | PT Update Batch | `pt_bo`, `ofs`, `qwords`, `pt_entries` |

---

## Evidence File Locations

- **VMA Types**: `drivers/gpu/drm/xe/xe_vm_types.h`
- **VMA Operations**: `drivers/gpu/drm/xe/xe_vm.h`, `drivers/gpu/drm/xe/xe_vm.c`
- **Page Tables**: `drivers/gpu/drm/xe/xe_pt_types.h`, `drivers/gpu/drm/xe/xe_pt.h`, `drivers/gpu/drm/xe/xe_pt.c`
- **Page Table Walking**: `drivers/gpu/drm/xe/xe_pt_walk.h`, `drivers/gpu/drm/xe/xe_pt_walk.c`
- **Physical Memory**: `drivers/gpu/drm/xe/xe_res_cursor.h`, `drivers/gpu/drm/xe/xe_bo.h`
- **PTE Encoding**: `drivers/gpu/drm/xe/xe_vm.c:1283-1347`
- **Binding**: `drivers/gpu/drm/xe/xe_pt.c:698-787`
