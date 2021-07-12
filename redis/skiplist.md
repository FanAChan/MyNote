 ###### 跳跃列表 skiplist、
 **数据结构**
 ```
typedef struct zskiplistNode {
    sds ele;//元素值
    double score;//分数
    struct zskiplistNode *backward;//当前节点的前一节点，用于从后向前进行遍历
    //层
    struct zskiplistLevel {
        //前进节点
        struct zskiplistNode *forward;
        //当前节点到下一节点的跨度，记录两点之间的距离，用于计算排位
        unsigned long span;
    } level[];
} zskiplistNode;
```

```
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;//头、尾节点
    unsigned long length;//跳表长度，及节点数量
    int level;//跳表中最大层数
} zskiplist;
```

**基础方法**
**创建节点，zslCreateNode**
```
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```

**创建一个新的skiplist，zslCreate**
```
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    //创建一个空节点作为头节，包含了最高层数，32层
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

**返回一个随机层数用于创建新节点，从1到32，层数越高，概率越小**
```
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    //获取一个0~0xFFFF的随机数,如果小于ZSKIPLIST_P * 0xFFFF则层数加1，晋升概率即为ZSKIPLIST_P，0.25
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
**插入一个新节点，zslInsert**
```
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    //update数组保存每一层中小于当前节点的最大节点，x为当前节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    //对应update到头节点的跨度/排名
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    //寻找插入位置
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        //保存排名
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        //如果下一节点的小于插入节点，则继续向前直到节点大于插入节点
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            //保存节点排名
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    //获取一个插入的层数
    level = zslRandomLevel();
    //层数大于当前skiplist最大层数，则修改最大层数
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    //创建新节点
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        //将新节点插入到搜索路径之后
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        //计算排名，rank[i] - rank[0]为新节点与上一节点之间的跨度，所以新节点的该层span为上一节点的跨度减去两节点之间的跨度
        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        //修改上一节点的跨度
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    //节点插入最高层之上的跨度+1，如两节点之后有三层，新节点只插入到第二层，则第三层的跨度也需要+1
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    //设置新节点的回溯节点
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```



使用skiplist原因  
1 内存占用
2 范围查找
3 实现难易程度