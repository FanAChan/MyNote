
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

所有类型的数据结构的最终引用
```
struct RedisObject{
    int4 type;        //4bit 对象类型
    int4 encoding;    //4bit 存储形式，相同的对象类型可以有不同的存储形式
    int24 lru;        //24bit lru信息
    int32 refcount;   //4byte 引用计数
    void *ptr;        //8byte 指针指向对象内容的具体存储位置
}
```

##### 实现概述
1. string  
    使用sds存储
    短字符串时编码方式为为embstr，对象头与字符串内容相邻存储，最长长度为44，
    长字符串时encoding为raw，对象头与字符串内容内存上不相邻
2. list  
    quicklist
3. hash  
    ziplist。一个元素对使用两个节点存储，key在前，value在后的存储方式
    dict
4. set  
    intset，数据按序存储
    dict
5. zset  
    ziplist。一个元素对使用两个节点存储，element在前，score在后的存储方式，按照score排序。
    skiplist + dict


