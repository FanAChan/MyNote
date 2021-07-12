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
**删除节点，zslDeleteNode**
```
/* Internal function used by zslDelete, zslDeleteRangeByScore and
 * zslDeleteRangeByRank. */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    //修改搜索路径
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    //修改后一节点的前缀节点
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        //修改尾节点
        zsl->tail = x->backward;
    }
    //删除节点后可能导致跳表上层为空，需要修改层高
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```

**根据score和element移除节点，zslDelete**  
> 节点为空则直接释放空间，否则返回节点，可以继续使用
```
/* Delete an element with matching score/element from the skiplist.
 * The function returns 1 if the node was found and deleted, otherwise
 * 0 is returned.
 *
 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
 * it is not freed (but just unlinked) and *node is set to the node pointer,
 * so that it is possible for the caller to reuse the node (including the
 * referenced SDS string at node->ele). */
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    //构建搜索路径
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    //判断底层的下一节点score和element与查找的相同
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        //相同则删除该节点
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```
**修改节点score,zslUpdateScore**
> 如果节点的分数修改前后不影响节点的位置，则不进行移动；
> 如果节点的分数修改前后需要移动，则先删除节点，再重新插入节点，并复用element
```
/* Update the score of an element inside the sorted set skiplist.
 * Note that the element must exist and must match 'score'.
 * This function does not update the score in the hash table side, the
 * caller should take care of it.
 *
 * Note that this function attempts to just update the node, in case after
 * the score update, the node would be exactly at the same position.
 * Otherwise the skiplist is modified by removing and re-adding a new
 * element, which is more costly.
 *
 * The function returns the updated element skiplist node pointer. */
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    /* We need to seek to element to update to start: this is useful anyway,
     * we'll have to update or remove it. */
    x = zsl->header;
    //节点搜索路径
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    x = x->level[0].forward;
    //找到的节点与需要修改的节点是同一个
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* If the node, after the score update, would be still exactly
     * at the same position, we can just update the score without
     * actually removing and re-inserting the element in the skiplist. */
    //如果修改之后是否仍然在修改之前的位置，则只修改score不移动节点
    if ((x->backward == NULL || x->backward->score < newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score > newscore))
    {
        x->score = newscore;
        return x;
    }

    /* No way to reuse the old node: we need to remove and insert a new
     * one at a different place. */
     //否则，删除旧节点，插入新节点
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* We reused the old node x->ele SDS string, free the node now
     * since zslInsert created a new one. */
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}
```
**zslGetRank**
> 获取一个元素的排名/排序
```
/* Find the rank for an element by both score and key.
 * Returns 0 when the element cannot be found, rank otherwise.
 * Note that the rank is 1-based due to the span of zsl->header to the
 * first element. */
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) <= 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele,ele) == 0) {
            return rank;
        }
    }
    return 0;
}
```

****
> 跟姐姐排名获取元素
```
/* Finds an element by its rank. The rank argument needs to be 1-based. */
zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank)
        {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```

使用skiplist原因  
1 内存占用
2 范围查找
3 实现难易程度