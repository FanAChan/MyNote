
###### 字典
> 类似hashmap的形式，使用数组加链表的方式

```
struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx;  /* rehashing not in progress if rehashidx == -1 是否正在rehash标记 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} ;

struct dictht{
    dictEntry** table; //二维数组
    long size;         //一维数组的长度
    long used;         //元素数量
    unsigned long sizemask;//size-1，进行hash
    ...
}

struct dictEntry{
    void* key;
    void* val;
    dictEntry* next;
}

```

扩容
> 正常情况下，当元素个数与二维数组大小相同是即扩容，新数组是原数组的两倍大小。
> 当元素个数小于数组长度的10%时，缩容。

渐进式rehash

步骤（有点类似于copyonwrite）
1. 初始化第二个hashtable，
2. 分步将旧hashtable中的数据迁移到新的hashtable中，每次只迁移一部分，避免集中式的rehash的庞大计算量，对redis性能产生影响可用性
迁移方式有两种
    - 对当前数据结构进行操作时辅助迁移少量数据（操作辅助rehash）
    - 定时器迁移数据（定时辅助rehash）
    - 标记迁移完成，删除旧hashtable占用的内存

>redis的数据结构中同时拥有两个hashtable
通常情况下只使用一个hashtable,，延迟初始化
只有在进行rehash 及扩容或者缩容的时候才会同时使用两个hashtable


>每次迁移在数据结构中标记当前迁移位置，以及未迁移数量（即旧hashtable中的数据量），
对redis操作时，delete,find ,update可能会在两个hashtable中进行查找，而执行add时，会判断两个hashtable中是否存在，存在则返回失败，但
>只在新的hashtable中进行添加，保证旧hashtable中的数量只减不增。


```
if(当前容量超过了扩容阈值 and 当前无子进程进行aof文件重写或生成rdb文件时)
    扩容或缩容：（初始化第二个hashtable,但不进行实际的迁移（rehash）操作）

执行增删改查操作时进行辅助迁移
    if（当前数据进行rehash)
        进行一步rehash操作
        
 周期函数定时辅助迁移
     if（当前数据进行rehash and 允许定时辅助迁移)  
         花费1ms辅助rehash操作
```
**dict.dictRehash()**
```
对dict迁移n个桶
//rehash完成返回0，未完成返回1
int dictRehash(dict *d, int n) {
    //限制最大迁移n*10个空桶，避免函数等待时间过长
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    
    //rehash完成
    if (!dictIsRehashing(d)) return 0;

    //迁移n个桶
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        //rehash桶的下表不可能大于原数组的大小
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        //空桶
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        //本次需要迁移的桶，将桶内所有的元素都迁移到新的数组中
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        //从链表头开始进行遍历，将节点插入到新数组中对应桶的头节点
        while(de) {
            uint64_t h;
            
            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    //判断是否完成了rehash，如果rehash完成，则释放旧数组，并指向新数组
    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
**周期性辅助迁移**
```
int dictRehashMilliseconds(dict *d, int ms) {
    if (d->pauserehash > 0) return 0;

    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```
**获取元素下标**
```
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    //如果正在rehash，则在两个hashtable中进行查找
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        //当前未进行rehash，无需在新的hashtable中查找，直接退出，修改删除等操作都使用了这个方式
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```
hash攻击
利用hash函数可能存在偏向性的特点，特定模式下的输入导致分布极度不均匀，导致查找效率急剧下降。

