---
layout: post
title: "How to estimate Nginx shared memory size"
description: "How to estimate Nginx shared memory size"
date: 2024-09-01
tags: [Nginx, Shared Memory]
---

# Why it is 8K entries in one megabyte

From http://nginx.org/en/docs/http/ngx_http_proxy_module.html, we know that one megabyte zone can store about 8 thousand keys.

But why it is 8 thousand? Let's dive into the code.

The ngx_http_file_cache_node_t struct is as below.
And the size of the struct is 120 bytes.

```C
typedef struct {
    ngx_rbtree_node_t                node;
    ngx_queue_t                      queue;

    u_char                           key[NGX_HTTP_CACHE_KEY_LEN
                                         - sizeof(ngx_rbtree_key_t)];

    unsigned                         count:20;
    unsigned                         uses:10;
    unsigned                         valid_msec:10;
    unsigned                         error:10;
    unsigned                         exists:1;
    unsigned                         updating:1;
    unsigned                         deleting:1;
    unsigned                         purged:1;
                                     /* 10 unused bits */

    ngx_file_uniq_t                  uniq;
    time_t                           expire;
    time_t                           valid_sec;
    size_t                           body_start;
    off_t                            fs_size;
    ngx_msec_t                       lock_time;
} ngx_http_file_cache_node_t;
```

How to get the size of the struct ? Let's use GDB.

```GDB
(gdb) ptype /o ngx_http_file_cache_node_t
type = struct {
/*      0      |      40 */    ngx_rbtree_node_t node;
/*     40      |      16 */    ngx_queue_t queue;
/*     56      |       8 */    u_char key[8];
/*     64: 0   |       4 */    unsigned int count : 20;
/*     66: 4   |       4 */    unsigned int uses : 10;
/* XXX  2-bit hole       */
/*     68: 0   |       4 */    unsigned int valid_msec : 10;
/*     69: 2   |       4 */    unsigned int error : 10;
/*     70: 4   |       4 */    unsigned int exists : 1;
/*     70: 5   |       4 */    unsigned int updating : 1;
/*     70: 6   |       4 */    unsigned int deleting : 1;
/*     70: 7   |       4 */    unsigned int purged : 1;
/* XXX  1-byte hole      */
/*     72      |       8 */    ngx_file_uniq_t uniq;
/*     80      |       8 */    time_t expire;
/*     88      |       8 */    time_t valid_sec;
/*     96      |       8 */    size_t body_start;
/*    104      |       8 */    off_t fs_size;
/*    112      |       8 */    ngx_msec_t lock_time;

                               /* total size (bytes):  120 */
                             }
```

From the last line, we can see the total size of the struct. Is is 120 bytes.
1M can be divided into 8738 120-byte blocks. 
However, the memory allocation algorithm for shared memory takes up some extra memory.

When allocating a 120-byte memory block, Nginx's slab allocator handles memory consumption as follows:

1. Nginx uses a slab allocator to manage shared memory. This allocator is designed for efficient allocation and deallocation of small memory blocks within fixed-size memory regions.

2. For a 120-byte memory block, the slab allocator will round it up to 128 bytes (the nearest power of 2).

3. The actual allocated memory size is 128 bytes, with 8 bytes of internal fragmentation due to memory alignment.

4. Each slab (typically one or more contiguous memory pages) has a small header for management purposes. This header usually ranges from 16 to 32 bytes in size.

5. The overhead of this slab header is per slab, not per individual memory block. A single slab typically contains multiple memory blocks.

In summary:

For an individual 120-byte memory block, the directly related extra memory consumption is 8 bytes (internal fragmentation).
Additionally, there's the overhead of the slab header (16-32 bytes), but this overhead is shared among all memory blocks in that slab.

So there are 1M / 128 = 2^20 / 2^7 = 2^13 = 8K entries.

# The overhead of nginx slab allocation

From `sp->min_shift = 3` we know that the smallest allocation size is 8 bytes.

We need to know some key structs before dive into the details.

## key structs

### struct ngx_slab_pool_t

```GDB
(gdb) ptype /o ngx_slab_pool_t
type = struct {
/*      0      |      16 */    ngx_shmtx_sh_t lock;
/*     16      |       8 */    size_t min_size;
/*     24      |       8 */    size_t min_shift;
/*     32      |       8 */    ngx_slab_page_t *pages;
/*     40      |       8 */    ngx_slab_page_t *last;
/*     48      |      24 */    ngx_slab_page_t free;
/*     72      |       8 */    ngx_slab_stat_t *stats;
/*     80      |       8 */    ngx_uint_t pfree;
/*     88      |       8 */    u_char *start;
/*     96      |       8 */    u_char *end;
/*    104      |      64 */    ngx_shmtx_t mutex;
/*    168      |       8 */    u_char *log_ctx;
/*    176      |       1 */    u_char zero;
/*    177: 0   |       4 */    unsigned int log_nomem : 1;
/* XXX  7-bit hole       */
/* XXX  6-byte hole      */
/*    184      |       8 */    void *data;
/*    192      |       8 */    void *addr;

                               /* total size (bytes):  200 */
                             }
```

The size of ngx_slab_pool_t is 200 bytes

### struct ngx_slab_page_t

```GDB
type = struct ngx_slab_page_s {
/*      0      |       8 */    uintptr_t slab;
/*      8      |       8 */    ngx_slab_page_t *next;
/*     16      |       8 */    uintptr_t prev;

                               /* total size (bytes):   24 */
                             }
```

The size of ngx_slab_page_t is 24.

### struct ngx_slab_stat_t

```
type = struct {
/*      0      |       8 */    ngx_uint_t total;
/*      8      |       8 */    ngx_uint_t used;
/*     16      |       8 */    ngx_uint_t reqs;
/*     24      |       8 */    ngx_uint_t fails;

                               /* total size (bytes):   32 */
                             }
```

The size of ngx_slab_stat_t is 32.

## Memory layout

Take 4K pages for example:
 - ngx_pagesize_shift = 12
 - pool->min_shift = 3
 - ngx_slab_exact_size = ngx_pagesize / (8 * sizeof(uintptr_t)) = 64
 - ngx_slab_exact_shift = 6
 - pages = (ngx_uint_t) (size / (ngx_pagesize + sizeof(ngx_slab_page_t)))

1M shared memory zone will have 2^20 / 2^12 = 2^8 = 256 pages

## The memory layout

The memory layout of the shared memory zone looks like this:

|       Content      |
|--------------------|
| ngx_slab_pool_t    |
| ngx_slab_page_t[0] |
| ngx_slab_page_t[1] |
| ngx_slab_page_t[2] |
| ngx_slab_page_t[3] |
| ngx_slab_page_t[4] |
| ngx_slab_page_t[5] |
| ngx_slab_page_t[6] |
| ngx_slab_page_t[7] |
| ngx_slab_page_t[8] |
| ngx_slab_stat_t[0] |
| ngx_slab_stat_t[1] |
| ngx_slab_stat_t[2] |
| ngx_slab_stat_t[3] |
| ngx_slab_stat_t[4] |
| ngx_slab_stat_t[5] |
| ngx_slab_stat_t[6] |
| ngx_slab_stat_t[7] |
| ngx_slab_stat_t[8] |
| ngx_slab_page_t[0] |
| ngx_slab_page_t[1] |
| ngx_slab_page_t[2] |
| ... |
| ngx_slab_page_t[pages -1] |
|  empty hole |
|  data of pages |  <-- pool->start
|  data of pages |
|  ...           |
|  data of pages |


When allocate small memory from pages, we need to know which block has used
and which block has free. The slab allocator uses the bitmap.
How to calcuate the bitmap size ? The formula is:

map = (ngx_pagesize >> shift) / (8 * sizeof(uintptr_t));

(ngx_pagesize >> shift):  Calcuate One page can split into how many small blocks
(8 * sizeof(uintptr_t)):  Calculate the bit in one uintptr_t type.

So the above formula calculate how many uintptr_t needed.

So for example, if we cut 4K bytes into 32 bytes, then
map = (2^12 >> 5) / 64 = (2^12 >> 5) >> 6 = 2^12 >> 11 = 1
This means that 1 extra byte is needed for the bitmap, but it is actually 32 bytes, so 24 bytes are extra.

If you cut 4K into 8-byte chunks, then
map = (4096 >> 3) / 64 = 2^12 >> 8 = 16
So, at this point, two 8s on either side of the 8K page are allocated for use as a bitmap by themselves, and the third 8 bytes are allocated to the business as available memory.

The overhead for a 1M memory is 9 * (32  + 24) + 256 * 24 = 6648 ~= 8K

So the overhead is 8 * 1024 / 1024 / 1024 = 0.0078

Split the page into size smaller than 64 bytes, there will have the bitmap overhead.
The biggest bitmap overhead is 32 / 1024 = 0.03.

Split the page into size equal or bigger than 64 bytes, there will no bitmap overhead.


## Lua shared dict

For the Lua shared-dict, the basic size is 32 + 40 = 72.
So the allcation overhead for the Lua sharedict is 0.007.

```
type = struct ngx_rbtree_node_s {
/*      0      |       8 */    ngx_rbtree_key_t key;
/*      8      |       8 */    ngx_rbtree_node_t *left;
/*     16      |       8 */    ngx_rbtree_node_t *right;
/*     24      |       8 */    ngx_rbtree_node_t *parent;
/*     32      |       1 */    u_char color;
/*     33      |       1 */    u_char data;
/* XXX  6-byte padding   */

                               /* total size (bytes):   40 */
                             }

(gdb) ptype /o ngx_http_lua_shdict_node_t
type = struct {
/*      0      |       1 */    u_char color;
/*      1      |       1 */    uint8_t value_type;
/*      2      |       2 */    u_short key_len;
/*      4      |       4 */    uint32_t value_len;
/*      8      |       8 */    uint64_t expires;
/*     16      |      16 */    ngx_queue_t queue;
/*     32      |       4 */    uint32_t user_flags;
/*     36      |       1 */    u_char data[1];
/* XXX  3-byte padding   */

                               /* total size (bytes):   40 */
                             }
```

