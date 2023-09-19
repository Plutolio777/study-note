# 什么是redis基数树(Radix Tree)

基数树是一种有序字典树(前缀树)，它会根据key生成一个字典序列的树结构，让我们可以像查找字典一样快速定位key的所在位置。支持快速地定位、插入和删除操作。

## 基数树的基本结构

![Alt text](images/%E5%9F%BA%E6%95%B0%E6%A0%91.svg)

我们可以将一本英语字典看成一棵 radix tree，它所有的单词都是按照字典序进行排列，每个词汇都会附带一个解释，这个解释就是 key 对应的 value。有了这棵树，你就可以快速地检索单词，还可以查询以某个前缀开头的单词有哪些。

## redis中基数树的优化

rax 是一棵比较特殊的 radix tree，它在结构上不是标准的 radix tree。如果一个中间节点有多个子节点，那么路由键就只是一个字符。如果只有一个子节点，那么路由键就是一个字符串。后者就是所谓的「压缩」形式，多个字符压在一起的字符串。

比如我们先后插入 foot和footbar ，那么在 radix tree 中，它们会变成这样：

![Alt text](images/%E5%9F%BA%E6%95%B0%E6%A0%912.svg)

其中黄色节点则为压缩节点

## redis中源码中探究rax

rax的源码位置在 `redis/src/rax.h`

首先定义了RAX中最大的节点个数 `RAX_NODE_MAX_SIZE=(1<<29)-1`

~~~c++
typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? 节点是否存在key*/
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). 节点value是否为空*/
    uint32_t iscompr:1;   /* Node is compressed. 是否是压缩节点*/
    uint32_t size:29;     /* Number of children, or compressed string len. 如果为压缩节点则保存的是字符串的长度 如果不为压缩节点则保存的是子节点个数*/
    /* Data layout is as follows:
     *
     * If node is not compressed we have 'size' bytes, one for each children
     * character, and 'size' raxNode pointers, point to each child node.
     * Note how the character is not stored in the children but in the
     * edge of the parents:
     * 1. 如果节点不为压缩节点 则size存储子节点个数，那么会有size个字符以及size个子节点的指针
     * 2. 当前节点的字符不是存在于子节点而是父节点的边缘
     * [header iscompr=0][abc][a-ptr][b-ptr][c-ptr](value-ptr?)
     *
     * if node is compressed (iscompr bit is 1) the node has 1 children.
     * In that case the 'size' bytes of the string stored immediately at
     * the start of the data section, represent a sequence of successive
     * nodes linked one after the other, for which only the last one in
     * the sequence is actually represented as a node, and pointed to by
     * the current compressed node.
     * 1. 如果是压缩节点(代表只有一个子节点) size存储的是字符串的长度
     * 2. data的最后会有一个指向子节点的指针
     * [header iscompr=1][xyz][z-ptr](value-ptr?)
     *
     * Both compressed and not compressed nodes can represent a key
     * with associated data in the radix tree at any level (not just terminal
     * nodes).
     * 压缩节点和非压缩节点都可以表示关联数据的key
     * If the node has an associated key (iskey=1) and is not NULL
     * (isnull=0), then after the raxNode pointers pointing to the
     * children, an additional value pointer is present (as you can see
     * in the representation above as "value-ptr" field).
     * 如果节点有关联键 (iskey=1) 并且不为 NULL (isnull=0)，则在指向子节点的 raxNode 指针之后，会出现一个附加值指针（如您在上面的表示中看到的“value-ptr”字段）。
     */
    unsigned char data[];
} raxNode;
~~~

### 节点中的data是什么

data的存储主要有两种情况，一种是普通节点，一种是压缩节点

~~~c++
 /* Data layout is as follows: 数据按照如下格式排列
     * If node is not compressed we have 'size' bytes, one for each children
     * character, and 'size' raxNode pointers, point to each child node.
     * Note how the character is not stored in the children but in the
     * edge of the parents:
~~~

如果不为压缩节点的话 size存储的是子节点的个数，那么data实际上是一个拥有size字节大小并且不压缩的节点，每个字节代表一个子字符和size个节点指针，指向每个子节点。注意到字符保存在父节点的边缘而不是子节点。以上图的tb节点为例

![Alt text](images/%E5%9F%BA%E6%95%B0%E6%A0%913.svg)

如果节点是压缩的(iscompr值为1)，表示该节点只有一个孩子。在这种情况下size字节大小的字符串保存在数据段开始的地方，表示一些列前后连接的节点，只有序列最后一个节点实际代表一个节点，并且由当前节点指向
