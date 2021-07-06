
#### 源码
![img](../pic/redis源码学习步骤.png)
##### 内部数据结构
```
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```
```
struct RedisObject{
    int4 type;        //4bit 对象类型
    int4 encoding;    //4bit 存储形式，相同的对象类型可以有不同的存储形式
    int24 lru;        //24bit lru信息
    int32 refcount;   //4byte 引用计数
    void *ptr;        //8byte 指针指向对象内容的具体存储位置
}
```
所有类型的数据结构的最终引用

###### 字符串 Simple Dynamic String


```
struct SDS{
    T capactity;       //数组容量
    T len;             //字符串长度
    byte flags;        //特殊标志位
    byte[] content;    //数组内容
}
```

短字符串为embstr，长字符串为raw。
> empstr RedisObject对象头与SDS对象头在内存中连续存储，一起分配内存。
>
> raw ReidsObject对象头与SDS对象头在内存上一般是不连续的，需要分配两次内存。
>
> 分配内存时按照2的n次幂，即2/4/8/16/32/64等的大小分配，对象头的大小为16bytes，字符串的最小大小为content.len + 1 + 1 + 1字节，即分配一个embstr的最小大小为19（16 + 3)字节。若字符串稍微大点，可以分配64字节。若超过了64字节，则认为是一个大字符串，使用raw形式存储。最大为64 - 19 - 1(NULL结束符) = 44字节 

扩容策略
    字符串长度小于1MB时，每次扩容加倍，大于1MB时，每次扩容1MB
    
###### 双向链表 list
``` C
typedef struct listNode {
     //指向前一个节点
    struct listNode *prev;
    //指向后一个节点
    struct listNode *next;
    //值
    void *value;
} listNode;

```
``` C
typedef struct list {
    //头结点
    listNode *head;
    //尾结点
    listNode *tail;
    //复制方法
    void *(*dup)(void *ptr);
    //删除
    void (*free)(void *ptr);
    //匹配
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

```
typedef struct listIter {

    listNode *next;

    int direction;
} listIter;
```


###### set
> 与hash一样，但是value都为NULL
 ```
robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}
使用dict存储数据，encoding为hashtable
```

 ##### 内存压缩结构
 > 在条件容许的情况下，会使用压缩数据结构替代内部数据结构，消耗内存会少得多，但因为编码和操作更复杂，占用的CPU时间也会多
 ###### 整数集 intset
 
  ##### 内存存储结构
 ###### 快速列表 quicklist
 ```
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

typedef struct quicklistIter {
    const quicklist *quicklist;
    quicklistNode *current;
    unsigned char *zi;
    long offset; /* offset in current ziplist */
    int direction;
} quicklistIter;

typedef struct quicklistEntry {
    const quicklist *quicklist;
    quicklistNode *node;
    unsigned char *zi;
    unsigned char *value;
    long long longval;
    unsigned int sz;
    int offset;
} quicklistEntry;
```
