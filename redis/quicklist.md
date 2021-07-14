
### quicklist
> 链表的附加空间相同太高，且每个节点的内存时单独分配，会加剧内存的碎片化，影响内存管理效率。
> quicklist实际是ziplist和linkedlist的混合体

```
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists *///所有ziplist中的数据项总和
    unsigned long len;          /* number of quicklistNodes *///quicklist节点数
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes *///ziplist大小设置
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off *///节点压缩深度
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

```
typedef struct quicklistNode {
    struct quicklistNode *prev;//上一个节点
    struct quicklistNode *next;//下一个节点
    unsigned char *zl;//存储数据的ziplist
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 是否被压缩了*/
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? 是否已经解压？ */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

**quicklistLZF**
> ziplist是一段连续的内存，用LZ4算法压缩后，包装一个quicklistLZF节点，
> 压缩后quicklistNode.zl字段指向的就是一个quicklistLZF节点，而不是ziplist
```
/* quicklistLZF is a 4+N byte struct holding 'sz' followed by 'compressed'.
 * 'sz' is byte length of 'compressed' field.
 * 'compressed' is LZF data with total (compressed) length 'sz'
 * NOTE: uncompressed length is stored in quicklistNode->sz.
 * When quicklistNode->zl is compressed, node->zl points to a quicklistLZF */
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
```
