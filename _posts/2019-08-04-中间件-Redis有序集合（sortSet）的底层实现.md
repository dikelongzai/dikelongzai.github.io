---
title: Redis有序集合（sortSet）的底层实现
categories:
  - Redis

  
tags:
  - Redis
abbrlink: 33760
date: 2019-08-04 01:29:56
---
**Redis中支持的数据结构比Memcached要多，如基本的字符串、哈希表、列表、集合、可排序集，在这些基本数据结构上也提供了针对该数据结构的各种操作**，这也是Redis之所以流行起来的一个重要原因,当然Redis能够流行起来的原因，远远不只这一个,如支持高并发的读写、数据的持久化、高效的内存管理及淘汰机制...
从Redis的git提交历史中，可以查到，2009/10/24在1.050版本，Redis开始支持可排序集,在该版本中，只提供了一条命令zadd,宏定义如下所示:
```
1{"zadd",zaddCommand,4,REDIS_CMD_BULK|REDIS_CMD_DENYOOM},
```
那么什么是可排序集呢? 从Redis 1.0开始就给我们提供了集合(Set)这种数据结构，集合就跟数学上的集合概念是一个道理【无序性，确定性，互异性】,集合里的元素无法保证元素的顺序，而业务上的需求，可能不止是一个集合，而且还要求能够快速地对集合元素进行排序，于是乎，Redis中提供了可排序集这么一种数据结构，似乎也是合情合理,无非就是在集合的基础上增加了排序功能,也许有人会问，Redis中不是有Sort命令嘛，下面的操作不也是同样可以达到对无序集的排序功能嘛，是的,是可以，但是在这里我们一直强调的是快速这两个字，而Sort命令的时间复杂度为O(N+M*Log(M)),可排序集获取一定范围内元素的时间复杂度为O(log(N) + M)
```
root@bjpengpeng-VirtualBox:/home/bjpengpeng/redis-3.0.1/src# ./redis-cli 
127.0.0.1:6379> sort set
1) "1"
2) "2"
3) "3"
4) "5"
127.0.0.1:6379> sort set desc
1) "5"
2) "3"
3) "2"
4) "1"
127.0.0.1:6379>
```
在了解可排序集是如何实现之前，需要了解一种数据结构跳表(Skip List),跳表与AVL、红黑树...等相比，数据结构简单，算法易懂，但查询的时间复杂度与平衡二叉树/红黑树相当,跳表的基本结构如下图所示
![图1：redissortset1](/public/image/redis-sortset1.png)
上图中整个跳表结构存放了4个元素5->10->20->30,图中的红色线表示查找元素30时，走的查找路线,从Head指针数组里最顶层的指针所指的20开始比较，与普通的链表查找相比，跳表的查询可以跳跃元素，上图中查询30，发现30比20大，则查找就是20开始，而普通链表的查询必须一个元素一个元素的比较，时间复杂度为O(n)
有了上图所示的跳表基本结构，再看看如何向跳表中插入元素，向跳表中插入元素，由于元素所在层级的随机性，平均起来也是O(logn)，说白了，就是查找元素应该插入在什么位置，然后就是普通的移动指针问题，再想想往有序单链表的插入操作吧，时间复杂度是不是也是O(n),下图所示是往跳表中插入元素28的过程,图中红色线表示查找插入位置的过程，绿色线表示进行指针的移动，将该元素插入
 ![图二：redissortset2](/public/image/redis-sortset2.png)

有了跳表的查找及插入那么就看看在跳表中如何删除元素吧，跳表中删除元素的个程，查找要删除的元素，找到后，进行指针的移动，过程如下图所示，删除元素30
 ![图三：redissortset3](/public/image/redis-sortset3.png)
有了上面的跳表基本结构图及原理，自已设计及实现跳表吧，这样当看到Redis里面的跳表结构时我们会更加熟悉，更容易理解些,
【下面是对Redis中的跳表数据结构及相关代码进行精减后形成的可运行代码】,首先定义跳表的基本数据结构如下所示
 ```
#include<stdio.h>#include<stdlib.h> 
#define ZSKIPLIST_MAXLEVEL 32
#define ZSKIPLIST_P 0.25
#include <math.h> //跳表节点
typedef struct zskiplistNode {    
int key;    
int value;    
struct zskiplistLevel {        
struct zskiplistNode *forward;    
} level[1];} 
zskiplistNode; //跳表
typedef struct zskiplist {    
struct zskiplistNode *header;    
int level;
} 
zskiplist;
```
在代码中我们定义了跳表结构中保存的数据为Key->Value这种形式的键值对，注意的是skiplistNode里面内含了一个结构体,代表的是层级，并且定义了跳表的最大层级为32级,下面的代码是创建空跳表，以及层级的获取方式
```
  //创建跳表的节点
  zskiplistNode *zslCreateNode(int level, int key, int value) 
  {    
  zskiplistNode *zn = (zskiplistNode *)
  malloc(sizeof(*zn)+level*sizeof(zn->level));    
  zn->key = key;    zn->value = value;    
  return zn;
  } 
  //初始化跳表
  zskiplist *zslCreate(void) { 
     int j;    
    zskiplist *zsl;    
    zsl = (zskiplist *) malloc(sizeof(*zsl));    
    zsl->level = 1;//将层级设置为1    
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,NULL,NULL);    
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {        
    zsl->header->level[j].forward = NULL;   
     }    
    return zsl;} 
    //向跳表中插入元素时，随机一个层级，表示插入在哪一层
    int zslRandomLevel(void) {    
    int level = 1;    
    while ((rand()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))        level += 1;    
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
    }
```
在这段代码中，使用了随机函数获取过元素所在的层级，下面就是重点，向跳表中插入元素，插入元素之前先查找插入的位置,代码如下所示，代码中注意

```
 //向跳表中插入元素
  zskiplistNode *zslInsert(zskiplist *zsl, int key, int value)
   {    
   zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;    
   int i, level;    
   x = zsl->header;    
   //在跳表中寻找合适的位置并插入元素    
   for (i = zsl->level-1; i >= 0; i--) {        
   while (x->level[i].forward &&            
          (x->level[i].forward->key < key ||               
           (x->level[i].forward->key == key &&
            x->level[i].forward->value < value))) { 
              x = x->level[i].forward;        
   }        
   update[i] = x;    
   }    
   //获取元素所在的随机层数    
   level = zslRandomLevel();    
   if (level > zsl->level) {        
   for (i = zsl->level; i < level; i++) {            
   update[i] = zsl->header;       
    }        
   zsl->level = level;    
   }    
   //为新创建的元素创建数据节点    
   x = zslCreateNode(level,key,value);    
   for (i = 0; i < level; i++) {        
   x->level[i].forward = update[i]->level[i].forward;        
   update[i]->level[i].forward = x;    
   }    
   return x;
   }
 ```
下面是代码中删除节点的操作，和插入节点类似
```
 //跳表中删除节点的操作
 void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) 
 {    int i;    
      for (i = 0; i < zsl->level; i++) {        
      if (update[i]->level[i].forward == x) {            
      update[i]->level[i].forward = x->level[i].forward;        
      }    
     }    
     //如果层数变了，相应的将层数进行减1操作    
     while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)        
     zsl->level--;
     } 
     //从跳表中删除元素
     int zslDelete(zskiplist *zsl, int key, int value) { 
        zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;   
      int i;    
     x = zsl->header;    
     //寻找待删除元素    for (i = zsl->level-1; i >= 0; i--) {        
     while (x->level[i].forward &&            
            (x->level[i].forward->key < key ||                
            (x->level[i].forward->key == key &&                
            x->level[i].forward->value < value))) {           
             x = x->level[i].forward;        
        }        
          update[i] = x;    
        }    
        x = x->level[0].forward;    
        if (x && key == x->key && x->value == value) {        
        zslDeleteNode(zsl, x, update);        
        //别忘了释放节点所占用的存储空间       
         free(x);        
        return 1;    
        } else {        
        //未找到相应的元素        
        return 0;    
        }    
        return 0;
        }
   ```
最后，附上一个不优雅的测试样例
```
 //将链表中的元素打印出来
 void printZslList(zskiplist *zsl) {    
 zskiplistNode  *x;    
 x = zsl->header;    
 for (int i = zsl->level-1; i >= 0; i--) {        
 zskiplistNode *p = x->level[i].forward;        
 while (p) {            
 printf(" %d|%d ",p->key,p->value);            
 p = p->level[i].forward;        
 }        
 printf("\n");    
 }
 } 
 int main() {    
 zskiplist *list = zslCreate();    
 zslInsert(list,1,2);    
 zslInsert(list,4,5);    
 zslInsert(list,2,2);    
 zslInsert(list,7,2);    
 zslInsert(list,7,3);    
 zslInsert(list,7,3);    
 printZslList(list);    
 //zslDelete(list,7,2);    
 printZslList(list);
 }
 ```
有了上面的跳表理论基础，理解Redis中跳表的实现就不是那么难了
Redis中跳表的基本数据结构定义如下，与基本跳表数据结构相比，在Redis中实现的跳表其特点是不仅有前向指针，也存在后向指针，而且在前向指针的结构中存在span跨度字段，这个跨度字段的出现有助于快速计算元素在整个集合中的排名
```
//定义跳表的基本数据节点
typedef struct zskiplistNode {
    robj *obj; // zset value
    double score;// zset score
    struct zskiplistNode *backward;//后向指针
    struct zskiplistLevel {//前向指针
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

//有序集数据结构
typedef struct zset {
    dict *dict;//字典存放value,以value为key
    zskiplist *zsl;
} zset;
 ```
将如上数据结构转化成更形式化的图形表示，如下图所示
  ![图四：redissortset4](/public/image/redis-sortset4.png)

在上图中，可以看到header指针指向的是一个具有固定层级(32层)的表头节点,为什么定义成32,是因为定义成32层理论上对于2^32-1个元素的查询最优，而2^32=4294967296个元素，对于绝大多数的应用来说，已经足够了，所以就定义成了32层,到于为什么查询最优，你可以将其想像成一个32层的完全二叉排序树，算算这个树中节点的数量
Redis中有序集另一个值得注意的地方就是当Score相同的时候，是如何存储的，当集合中两个值的Score相同，这时在跳表中存储会比较这两个值，对这两个值按字典排序存储在跳表结构中
有了上述的数据结构相关的基础知识，来看看Redis对zskiplist/zskiplistNode的相关操作,源码如下所示(源码均出自t_zset.c)
创建跳表结构的源码
 ```
//#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;
    //分配内存
    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;//默认层级为1
    zsl->length = 0;//跳表长度设置为0
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        //因为没有任何元素，将表头节点的前向指针均设置为0
        zsl->header->level[j].forward = NULL;
        //将表头节点前向指针结构中的跨度字段均设为0
        zsl->header->level[j].span = 0;
    }
    //表头后向指针设置成0
    zsl->header->backward = NULL;
    //表尾节点设置成NULL
    zsl->tail = NULL;
    return zsl;
}
 ```
在上述代码中调用了zslCreateNode这个函数,函数的源码如下所示=
 ```
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->obj = obj;
    return zn;
}
 ```
执行完上述代码之后会创建如下图所示的跳表结构
 
  ![图五：redissortset5](/public/image/redis-sortset5.png)
创建了跳表的基本结构，下面就是插入操作了，Redis中源码如下所示
 ```
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x; //update[32]
    unsigned int rank[ZSKIPLIST_MAXLEVEL];//rank[32]
    int i, level;
    redisAssert(!isnan(score));
    x = zsl->header;
    //寻找元素插入的位置 
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
            (x->level[i].forward->score < score || //以下是得分相同的情况下，比较value的字典排序
                (x->level[i].forward->score == score &&compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    //产生随机层数
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        //记录最大层数
        zsl->level = level;
    }
    //产生跳表节点
    x = zslCreateNode(level,score,obj);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
        //更新跨度
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
    //此种情况只会出现在随机出来的层数小于最大层数时
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
 ```
上述源码中，有一个产生随机层数的函数，源代码如下所示:
 ```
int zslRandomLevel(void) {
    int level = 1;
    //#define ZSKIPLIST_P 0.25 
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    //#ZSKIPLIST_MAXLEVEL 32
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
 ```
图形化的形式描述如下图所示:
  ![图六：redissortset6](/public/image/redis-sortset6.png)
理解了插入操作，其他查询，删除，求范围操作基本上类似
