# Nginx内存池

## 操作接口

在使用内存池前，我们先得创建内存池：
``` c
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
```

如，在 *http* 模块中每个 `ngx_http_request_t`  结构都有一个内存池，这个内存池是在创建该结构前创建的；内存池创建成功之后，会基于该内存池分配 `ngx_http_request_t`  结构的空间：
``` c
ngx_http_request_t *
ngx_http_create_request(ngx_connection_t *c)
{
    ngx_pool_t                 *pool;
    ngx_time_t                 *tp;
    ngx_http_request_t         *r;
    ngx_http_log_ctx_t         *ctx;
    ngx_http_connection_t      *hc;
    ngx_http_core_srv_conf_t   *cscf;
    ngx_http_core_loc_conf_t   *clcf;
    ngx_http_core_main_conf_t  *cmcf;

    c->requests++;

    hc = c->data;

    cscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_core_module);

    pool = ngx_create_pool(cscf->request_pool_size, c->log);
    if (pool == NULL) {
        return NULL;
    }

    r = ngx_pcalloc(pool, sizeof(ngx_http_request_t));
    if (r == NULL) {
        ngx_destroy_pool(pool);
        return NULL;
    }

    r->pool = pool;

    ...
```

使用完之后的内存池必须通过下面的函数回收：
``` c
void ngx_destroy_pool(ngx_pool_t *pool);
```
如果不想完全回收，也可通过下面的函数重置内存池：
``` c
void ngx_reset_pool(ngx_pool_t *pool);
```

如，上面讨论的 `ngx_http_request_t` 结构中的内存池会在 `ngx_http_free_request` 函数中销毁：
 ``` c
void
ngx_http_free_request(ngx_http_request_t *r, ngx_int_t rc)
{
    ...

    /*
     * Setting r->pool to NULL will increase probability to catch double close
     * of request since the request object is allocated from its own pool.
     */

    pool = r->pool;
    r->pool = NULL;

    ngx_destroy_pool(pool);
}
```

## 内存池的结构和分配算法
我们先来看下内存池相关的结构体：
``` c
struct ngx_pool_large_s {
    ngx_pool_large_t     *next;
    void                 *alloc;
};

typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;

struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};

typedef struct {
    ngx_fd_t              fd;
    u_char               *name;
    ngx_log_t            *log;
} ngx_pool_cleanup_file_t;
```

一个内存池有两个单链表：
  1. *large* 大内存单链表，用于记录分配的大块内存（大于 `ngx_pagesize - 1`）
  2. *small* 小内存单链表，用于分配小块内存（小于等于 `ngx_pagesize - 1`）

大块内存的分配一般直接调用 *malloc* 直接从系统获取，分配好的内存插入 *large* 单链表；小块内存则从 *small* 链表的空闲区域分配。

大块内存的分配比较简单，在实际应用场景，需要进行大块内存分配的也不多，所以，我们不继续讨论；我们接着来看下小块内存具体的分配算法。

*small* 小块内存单链表的节点是 `ngx_pool_t`。
`ngx_create_pool` 创建的 `ngx_pool_t`  是链表的入口；最开始，`ngx_pool_t` 在链表中只存在一份，而在小块内存的分配过程中，如果预留内存不足，就会分配新的 `ngx_pool_t` 结构。

`ngx_pool_t` 链表在内存中的布局示意图如下：

![链表结构](nginx/core/_images/nginx_mempool_link.png)

通过 `ngx_ceate_pool` 创建的 *pool* 以及 *pool* 成员 *current* 都指向头节点；*last* 表示可分配地址；*end* 指向一个 `ngx_pool_t` 的尾部；*end-last* 则是一个 `ngx_pool_t` 对象剩余的可分配内存。
*small* 小块内存的分配过程如下：
``` c
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;
    p = pool->current;
    do {
        m = p->d.last;
        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }
        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;
            return m;
        }
        p = p->d.next;
    } while (p);

    return ngx_palloc_block(pool, size);
}
```

如上，当 `ngx_pool_t` 链表中没有可用内存后，就会调用 `ngx_allo_block` 来继续进行分配，该函数会创建一个新的 `ngx_pool_t` 对象：
``` c
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;

    psize = (size_t) (pool->d.end - (u_char *) pool);
    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;
    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;

    for (p = pool->current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            pool->current = p->d.next;
        }
    }
    p->d.next = new;

    return m;
}
```
