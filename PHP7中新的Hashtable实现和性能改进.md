> 这篇文章是翻译自nikic的[最新大作](http://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html)，我从他的blog中学到了很多东西。这篇文章貌似是他半年多来发的第一篇文章，文章主要是讲PHP 7中的新的Hashtable的实现，Hashtable是PHP中非常核心的部分，数组就是基于此实现的，而数组在PHP中的使用是如此之频繁，所以一个好的Hashtable的实现必然会带来性能的极大提升，从文章来看，事实也确实如此。

大概三年前，我写一篇名为“[分析数组的内存使用量](https://nikic.github.io/2011/12/12/How-big-are-PHP-arrays-really-Hint-BIG.html)”（这篇文章也得相当不错，值得一读）的文章，那篇文章分析的是PHP 5中的数组使用内存的情况。作为我所参与的PHP 7的开发工作中的一部分，我专注于改进了一些小的数据结构的内存分配情况，为此重写了Zend Engine的大部分代码。在这篇文章中，我会大概说明一下新的hashtable的实现，以及为什么它会比之前的实现更高效。

我使用下面的代码来测试内存的使用情况：

```php
$startMemory = memory_get_usage();
$array = range(1, 100000);
echo memory_get_usage() - $startMemory, " bytes\n";
```
这段代码测试了创建一个含有100000个不同整数的数组所消耗的内存空间大小。

下面这个表格是上面的代码分别在PHP 5.6和PHP 7中的执行结果，包括32位和64位系统两种情况：
```

        | 32 bit      | 64 bit
----------------------------------------
PHP 5.6 | 7.37 MiB    | 13.97 MiB
----------------------------------------
PHP 7.0 | 3.00 MiB   | 4.00 MiB
```
我们可以说32系统中PHP 7中的数组所占的内存比PHP 5.6节省了2.5倍，64位系统则是3.5倍。这是一个相当不错的改进。

#### Hashtable简介
PHP中数组的本质是顺序字典，它可以表示一个包含键值对的顺序列表，键值对的映射就是使用Hashtable实现的。

[Hashtable](https://en.wikipedia.org/wiki/Hash_table)是非常常见的数据结构，它被设计出来解决计算机只能直接表示以连续的整数作为索引的数组的问题。使用Hashtable，程序员才能使用字符串或者其他的复合类型作为数组的键。

Hashtable的概念实际上非常简单：字符串的键先会被传递给一个hash函数（hashing function，中文也翻译为散列函数，本文统一使用hash函数）,然后这个函数会返回一个整数（我们把它叫做hash值），而这个整数就是“通常”的数组的索引。问题是对于两个不同的字符串，调用hash函数会得到同一个hash值，而现实情况是任意字符串都可以作为键，所以键会有无数个，而数组的大小必须是提前设定好的，因为hash值必须小于数组索引的最大值，所以可以生成的hash值必须是有限的。这样用有限的hash值表示无限的键，必然会导致冲突。我们把两个不同的键的hash值是一样的情况称为冲突，任何Hashtable算法都必须提供某种机制解决这种冲突。

有两种主要的处理冲突的方法。开放定址法，当冲突发生的时候，冲突的元素会被保存到一个不同的索引中；链接法，所有拥有相同的hash值的元素，它们都会被保存到一个链表中。PHP使用的就是第二种方法。

另外通常情况下，Hashtable并非是显式排序的：最终底层数组中保存的元素的顺序是跟hash函数相关的，并且这个顺序一般都是随机的。这个行为显然跟PHP数组的语义不符：PHP的数组的迭代顺序跟数组中元素的插入顺序完全一致。这也意味着，PHP中的Hashtable的实现必须有一种额外的机制记住数组中元素的插入顺序。

#### 老的Hashtable的实现方式
在此我只会大概介绍一下老的hashtable的实现方式，如果你想要更进一步的了解，可以参见[PHP内部机制这本书的hashtable](http://www.phpinternalsbook.com/hashtables/basic_structure.html)这一章。下面这张图高度概括了PHP5中的hashtable：

![github](https://github.com/fupengfei058/article-collection/raw/master/doc/g1.png)

“冲突处理（collision resolution）”链表中的元素被称为”buckets”。每一个bucket都是单独分配的。这个图片没有展示的是每个元素实际的值也是保存在这些buckets中的（图片中只展示了键）。值会存放zval结构，这些结构是分开单独分配内存的，它们的大小是16字节（32位）或者24个字节（64位）。

另外一点上面的图片没有展示出来的是，冲突处理链表实际上是一个双向链表（方便元素的删除）。在冲突处理链表的旁边有另外一个双向链表，它用于保存数组中元素的顺序。对于一个包含的键为”a”,”b”,”c”（而且这是它们的插入顺序）的数组，这个链表会是下面这个样子：

![github](https://github.com/fupengfei058/article-collection/raw/master/doc/g2.png)

所以为什么老的hashtable的结构比较低效，这里的低效从内存占用和性能上而言的？有以下原因：

* Bukets需要分开分配。内存分配总是低效的，而且每次还额外需要分配8/16个字节，这是内存分配的冗余。分开分配也意味着这些buckets会分布在内存空间的不同地址中，这又会降低缓存的效率（可以去了解下缓存的局部性原理，下面还会提到，这篇文章说的缓存基本都是指CPU缓存）。
* Zvals也需要分开分配。上面已经说明这种方式很低效，它也会产生一些额外的头开销冗余（header overhead）。另外这需要在每个bucket中保存一个指向zval结构的指针，由于老的实现过于考虑通用性，所以不止需要一个指针，而是两个指针。
* 双向链表中的每个bucket需要4个指针用于链表的连接，这会带来16/32个字节的开销，遍历这种链表也不利于缓存（cache-unfriendly）操作。

新的Hashtable的实现就是为了解决（至少是改善）这些问题。

#### 新的zval实现

在介绍新的Hashtable之前，我想先介绍一下新的zval结构，特别是它跟老的zval结构的差别。新的zval结构的定义如下：

```c
struct _zval_struct {
    zend_value value;
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar type,
                zend_uchar type_flags,
                zend_uchar const_flags,
                zend_uchar reserved
            )
        } v;
        uint32_t type_info;
    } u1;
    union {
        uint32_t var_flags;
        uint32_t next;       /* hash collision chain */
        uint32_t cache_slot; /* literal cache slot */
        uint32_t lineno;     /* line number (for ast nodes) */
    } u2;
};
```
你可以完全不用注意这个结构的定义中的ZEND_ENDIAN_LOHI_4这个宏，它仅仅只是用于表示拥有不同的字节序的机器中的可预测的内存布局情况。

zval结构有三个部分：第一部分是value。zend_value联合体有8个字节，它可以保存任何类型的值，包括整数、字符串、数组等。具体保存什么取决于zval的类型。

第二个部分是4字节的type_info，它包含变量的真正类型（类似于IS_STRING、IS_ARRAY），以及一系列的标志位，用于提供跟类型相关的信息。例如，如果zval保存的是一个对象，那么这些类型标志位会说明它是一个非常量（non-constant）、可引用计数（refcounted）、可垃圾回收（garbage-collectible）、不可复制（non-copying）的类型。

最后一部分占有4个字节，通常情况下不会被用到（它只是用于填充内存，如果不存在的话，编译器也会自动实现）（对于这一点完全不了解的同学，可以自行搜索内存对齐）。然而，在某些特殊情况下，这些空间也会被用于存放一些额外的信息。例如，AST（抽象语法树）的节点使用它来存放行号，VM（虚拟机）常量使用它来存放缓冲槽的索引，以及Hashtable使用它来保存冲突处理链上的下一个元素——这一部分才是我们要重点关注的。

新的zval的实现跟老的比较，最大的一点差别是：没有refcount字段。这是因为新的zval将不会被单独分配，它会被直接嵌入到任何需要存放它的地方（例如，一个hashtable bucket中）。

所以zvals将不再需要使用引用计数（refcounting），复杂数据类型例如字符串、数组、对象和资源（resources）仍需要使用。所以新的zval的设计将引用计数（包括跟垃圾回收相关的信息）从zval转移到了数组/对象/等中。这种方式有很多优点，在此列出几点：

* 保存简单的值（例如，boolean、integer或者float）的zval将不再需要额外分配内存。所以避免内存分配的头部冗余（allocation header overhead），以及减少不必要的内存分配和内存释放，可以提高缓存的局部性，从而提高性能。
* 保存简单的值的zval不需要保存refcount和GC的根缓冲区。
* 避免两次引用计数。例如，以前的对象即使用了zval的引用计数，又使用额外的对象的引用计数，对于支持按对象传递的语义而言，这是必须的。
* 现在所有的复杂的值都内嵌一个引用计数，它们可以不依赖于zval的机制而进行共享。特别是字符串现在也有可能共享。这对于hashtable的实现也很重要，因为这样就不用再拷贝非interned字符串的键了。
#### 新Hashtable的实现
我们已经了解了充足的预备知识了，现在终于可以开始介绍PHP 7中的新的Hashtable的实现了。我们先看一下bucket结构体：
```c
typedef struct _Bucket {
    zend_ulong        h;
    zend_string      *key;
    zval              val;
} Bucket;
```
bucket是Hashtable中的一个条目。它含有很多我们需要的东西：一个hash值——h，一个字符串键——key，以及一个zval值——val。整数键会保存在字段h中（整数键的hash值和整数键是一样的），在这种情况下key字段的值将是NULL。

你可以看到zval是直接嵌入到bucket结构体里面的，所以就没有必要单独为它分配内存，这样就不会因为内存分配而产生冗余信息，从而减少内存的浪费。

Hashtable的结构为：
```c
typedef struct _HashTable {
	uint32_t          nTableSize;
	uint32_t          nTableMask;
	uint32_t          nNumUsed;
	uint32_t          nNumOfElements;
	zend_long         nNextFreeElement;
	Bucket           *arData;
	uint32_t         *arHash;
	dtor_func_t       pDestructor;
	uint32_t          nInternalPointer;
	union {
		struct {
			ZEND_ENDIAN_LOHI_3(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				uint16_t      reserve)
		} v;
		uint32_t flags;
	} u;
} HashTable;
```
arData数组保存了所有的buckets（也就是数组的元素），这个数组被分配的内存大小为2的幂次方，它被保存在nTableSize这个字段（最小值为8）。数组中实际保存的元素的个数被保存在nNumOfElements这个字段。注意这个数组直接包含Bucket结构。老的Hashtable的实现是使用一个指针数组来保存分开分配的buckets，这意味着需要更多的分配和释放操作（alloc/frees），需要为冗余信息及额外的指针分配内存。
#### 元素的顺序
arData数组以插入的顺序保存元素。所以第一个数组元素会保存在arData[0]，第二元素在arData[1]等等。这跟元素对应的键没有任何关系，这只跟插入的顺序相关。

所以如果你在hashtable中保存了5个元素，arData[0]到arData[4]的槽（slot）会被用到，下一个空闲槽是arData[5]。这个数字会记录在nNumUsed中，你可能会问：为什么这两个值要分开保存，它跟nNumOfElements的值不是一样的么？

它们是一样的，不过仅仅只是在只执行插入操作的情况下。如果一个元素被从Hashtable中删除，我们肯定不想把arData数组中位于删除元素后面的元素都前移一位从而保持数组的连续性，相反我们只是将删除元素zval类型标记为IS_UNDEF。

以下面的代码为例：
```php
$array = [
	'foo' => 0,
	'bar' => 1,
	0     => 2,
	'xyz' => 3,
	2     => 4
];
unset($array[0]);
unset($array['xyz']);
```
Hashtable结构体以及arData数组的情况如下：
```
nTableSize = 8
nNumOfElements = 3
nNumUsed = 5

[0]: key="foo", val=int(0)
[1]: key="bar", val=int(1)
[2]: val=UNDEF
[3]: val=UNDEF
[4]: h=2, val=int(4)
[5]: NOT INITIALIZED
[6]: NOT INITIALIZED
[7]: NOT INITIALIZED
```
arData的前5个元素已被使用，但是第二个（key为0）和第三个（key为’xyz’）位置上的元素的值都被替换为IS_UNDEF了，因为它们都被unset了。现在这些元素的存在只是浪费内存。不过一旦nNumUsed的值到达nTableSize，PHP就会尝试调整arData数组，让它更紧凑，具体方式就是抛弃类型为UDENF的条目。只有当所有的buckets都包含一个正常类型的值的时候，arData才会重新分配双倍的内存来保存更多的元素。

新的Hashtable保持数组顺序的方式比起PHP 5.x中的双向链表有不少优势。其中一个很明显的优势就是新的方式中每个bucket只需要保存两个指针，这两个指针的开销为8/16字节。在新的方式下对一个数组的遍历变成下面这个样子：
```
uint32_t i;
for (i = 0; i < ht->nNumUsed; ++i) {
	Bucket *b = &ht->arData[i];
	if (Z_ISUNDEF(b->val)) continue;
	// do stuff with bucket
}
```
这相对于是对内存的线性扫描，比起遍历一个链表来说有更高的缓存效用（cache-efficient）（对链表的遍历是遍历一系列相对随机的内存地址，相对于线性的缓存地址遍历链表的缓存的局部性效用就不存在了）。

这种实现方式的一点不足就是arData很少会缩小（除非你进行显式操作）。所以如果你创建了一个有一百万个元素的数组，然后又把它们都删掉，这个数组还是依旧会占有很多内存。我们可能应该在arData的使用率降低的时候把它的大小减半。
#### Hashtable查找
到目前为止，我们只是探讨了怎么维持PHP的数组的顺序。实际上Hashtable的查找还是使用了arHash这个数组，这个数组由unit32_t类型的值组成。arHash数组跟arData的大小一样（都是nTableSize），并且都被分配了一段连续的内存区块。

hash函数（DJBX33A用于字符串的键，DJBX33A是PHP用到的一种hash算法）返回的hash值一般是32位或者64位的整型数，这个数有点大，不能直接作为hash数组的索引。首先通过求余操作将这个数调整到hashtable的大小的范围之内。求余是通过计算hash & (ht->nTableSize - 1)，而不是hash % ht->nTableSize，因为ht->nTableSize是2的幂，所以这两种计算的结果是一样的（注意第一种情况用的是与运算&），第二种计算方式需要使用开销更大的整数除法运算。ht->nTableSize -1 的值保存在ht->nTableMask中。

然后再在hash数组中查找索引为idx = ht->arHash[hash & ht->nTableMask]的元素。这个索引对应的元素是冲突处理链表的头元素。所以ht->arData[idx]是我们要检查的第一个元素。如果这个元素中保存的键跟我们要查找的键相同，那么查找就搞定了。

如果不的相同的话，则需要查找冲突处理链表的下一个元素，这个元素的索引保存在bucket->val.u2.next中，它保存在zval结构体中很少会用到的最后4个字节中。我们继续遍历这个链表（使用索引而不是指针）直到找到我们要找的bucket，或者是碰到INVALID_IDX——这意味着你要查找的key并不存在。

用代码表示整个查找过程就是下面这个样子：
```c
zend_ulong h = zend_string_hash_val(key);
uint32_t idx = ht->arHash[h & ht->nTableMask];
while ( idx != INVALID_IDX ) {
	Bucket *b = &ht->arData[idx];
	if (b->h == h && zend_string_equals(b->key,key)){
		return b;
	}
	idx = Z_NEXT(b->val);
}
return NULL;
```
我们现在使用的是单向链表，不再需要”prev”指针了。prev指针主要用于删除元素，因为当你删除某个元素的时候，需要调整“prev”元素的”next”指针（就是调整要删除的元素的前一个元素的next指针的指向，只有通过这个prev指针才能找到前一个元素）。不过如果删除发生在键上，通过遍历冲突处理链表，你实际上已经知道它的前一个元素了。

极少数情况下删除一个元素会导致遍历冲突处理链表来查找前一个元素（删除当前正在遍历的元素）。因为这并非一个重要的应用场景，所以我们更倾向于节省内存，而不是为了方便遍历而增加每个元素中的链表的指针数。
#### packed hashtables
PHP中的所有数组都使用了Hashtable。不过在通常情况下，对于连续的、整数索引的数组（真正的数组）而言，这些hash的东西没多大意义。这也是为什么PHP 7中引入了“packed hashtables”这个概念。

在packed hashtables中，arHash数组为NULL，查找只会直接在arData中进行。例如要查找key为5的元素，那么这个元素将是arData[5]，或者不存在，没必要再去遍历什么冲突处理链表了。

需要注意的是，即使是整数索引的数组，PHP也必须维持它的顺序。数组[0=>1,1=>2]和数组[1=>2,0=>1]并不是相同。packed hashtable只会作用于键递增的数组，这些数组的key之间可以有间隔，但必须总是递增的。所以如果一个元素以一个”错误“的顺序（例如逆序）插入到数组中，那么packed hashtable就不会被用到。

尽管packed hashtable会带来一些性能上的优化，但它也会存储一些没用的信息。例如我们可以通过bucket的内存地址来确定它的索引值，所以bucket->h是多余的。bucket->key的值永远都是NULL，这些都是浪费内存。

保留这些没用的值，是为了保证bucket总是一致的结构，而不用在乎是否用到了packing（可以翻译为压缩，但是个人觉得不翻译出来更好理解）技术。这意味着遍历可以总是使用同样的代码。不过也许将来我们可能会使用一个“fully packed（完全压缩）”后的结构体，如果可能的话只需要使用一个单一的zval数组就可以了。
#### 空hashtables
空hashtables在PHP 5.x和PHP 7中都会被特殊对待。如果你创建一个空数组[]，你很可能并不想插入任何元素到数组中，所以arData/arHash只会在插入第一个元素的时候才会被分配内存。

在大多数地方为了避免检测这种特殊情况，我们用了一个小把戏：当nTableSize被设置为一个建议值或者默认值8，nTableMask会被设置为0（通常它的值应该为nTableSize - 1），这意味着hash & ht->nTableMask的结果将总是为0。

所以在这种情况下arHash数组只需要含有一个元素（索引为0），它的值为INVALID_IDX（这个特殊数组叫做uninitialized_bucket，并且它是静态分配的）。这样在进行查找时，总是会返回INVALID_IDX，它表示要查找的key不存在（这也是查找一个空表所应得的结果）。
#### 内存占用量
这一节将讲解PHP 7中的Hashtable实现的最重要的几个方面。首先让我们总结一下为什么新的实现方式占用的内存更少。先申明下，这里我只会使用64位系统中的数字，而且只考虑每个元素的大小，而忽略Hashtable结构体。

在PHP 5.x中，一个元素需要占用144个字节（很恐怖）。在PHP 7中这个大小下降到36个字节，在packed情况下只需要32个字节。下面是内存变化的详细情况：

* Zvals不再单独分配，所以这节省了16个字节的内存分配冗余。
* Buckets也不用再单独分配，所以又节省了16个字节的内存分配冗余。
* 对于简单类型的值，zval本身就少了16个字节。
* 保持数组顺序不再需要16个字节用于维持双向链表的链接指针，而且这个顺序是隐式的。
* 冲突处理链表现在是单链表，这又节省了8个字节。更进一步来看，现在用的是一个索引列表，并且每个索引是嵌入到zval中的，所以这又节省了8个字节。
* zval是嵌入到bucket中的，没必要再保存一个指向它的指针。如果分析老版本的实现细节的话，我们实际上节省了2个指针，这就又是16个字节了。
* 键的长度不用再保存在bucket中，又是8个字节。不过，如果键是一个字符串，而不是一个整数的话，它的长度还是需要保存在zend_string结构体中。这种情况下，对内存的影响不可能精确估算，因为zend_string结构体是共享的，这意味着之前的hashtable需要拷贝字符串，如果这个字符串不是interned的话（这一条没怎么看清楚）。
* 包含冲突列表头部的数组现在是基于索引的，这样每个元素又节省了4个字节。对于packed数组，这个数组根本就不需要，又节省了4个字节。

首先申明一下，上面的总结只是为了说明PHP 7中的新的Hashtable实现在多个方面都比老版本的要好。首先，新Hashtable的实现使用了更多的内嵌结构体（相对于单独分配内存而言）。这会有什么不利的影响呢？

如果你看下文章开头的示例，你会发现在64位系统下，PHP 7中一个含有100000个元素的数组会占用4.00MB的内存。在这个示例中，我们处理的是packed数组，所以我们实际需要的内存使用量是32 * 100000 = 3.05MB。这个差别是因为所有分配的内存的大小都是2的幂次方，所以包含100000个元素数组，nTableSize的大小将是2^17=131072，所以最终分配的内存就是32 * 131072（4MB）。

当然老的Hashtable的实现中分配的内存的大小也必须是2的幂次方。不过它只会分配一个包含bucket指针的数组（每个指针占8个字节），其他的所有东西都是根据具体需求来分配。所以在PHP 7中，我们多分配了32*31072(0.95MB)的未使用内存，而在PHP 5.x中只浪费了8*31072(0.24MB)的内存。

另外一种需要考虑的情况是，如果数组中所保存元素的值并非都不同，这又会有什么不同？简单起见，我们考察下数组中所有元素的值都相同的情况。所以将第一个示例中的函数range替换为array_fill：
```php
$startMemory = memory_get_usage();
$array = array_fill(0, 100000, 42);
echo memory_get_usage() - $startMemory, " bytes\n";
```
最终运行的结果为：
```
        | 32 bit   | 64 bit
---------------------------------------
PHP 5.6 | 4.70 MiB | 9.39 MiB
---------------------------------------
PHP 7.0 | 3.00 MiB | 4.00 MiB
```
从这个结果可以看到PHP 7中内存的占用量没有变，每个元素都独自使用一个zval，这样也没有变的理由。不过PHP 5.x中的内存占用量就降低了不少，这是因为只有一个zval用于表示所有的值。尽管如此，PHP 7的内存占用量还是要由于PHP 5.x，虽然差别已经小了一些。

当键为字符串（可能是共享的或者是interned）或者为复杂值的时候，事情就会变得更复杂。当然不管怎样，在这些情况下，最终PHP 7所占用的内存都远小于PHP 5.x，也许上面所列出的数据在某些情况下有些乐观了。
#### 性能
我们已经谈论了太多关于内存占用量方面的东西，现在我们转移到下一个话题：性能。phpng项目的最终目的不是为了改善内存的使用，而是提供性能。减少内存占用量只是提供性能的一种方式，因为更少的内存占用必然导致更好的CPU缓存利用率，从而导致更好的性能。

当然对于新的实现之所以更快还有其他一些原因：首先分配内存的次数减少了。不管数组中元素的值是否共享，我们都为每个元素节省了两次内存分配，由于分配内存是一个需要一定开销的操作，所以这些节省是很有价值的。

另外一方面遍历数组对于缓存更友好了(cache-friendly)，现在是在一段线性的内存地址上进行遍历，而不是在一段内存地址随机的链表上遍历。

关于性能还有很多可讨论的话题，不过这篇文章的重点是内存的使用，限于篇幅关系，我就不再过多谈论这方面的细节了。
#### 结语
就Hashtable的实现而言，这毫无疑问是PHP 7所做出的一项重大改进。简单点看就是消除了很多没用的冗余。

那么现在问题来了：我们还可以再做点什么呢？我已经提到过一个想法：为递增的整数键的数组提供”fully packed(完全压缩）“的hash。这就意味着要使用一个普通的zval数组（而不需要使用buckets数组了），

也许还有一些其他的方向可以尝试下。例如将冲突处理方式从冲突链表法替换成开放定址法（例如使用Robin Hood probing），这可以同时带来内存（不需要冲突处理链表了）和性能（缓存的使用效用更好）的改进。不过遗憾的是开发定址法很难跟保持数组顺序的需求结合起来，所以也许这种方法在实际情况中并不可行。

另外一个想法是把bucket结构中的h字段和key字段整合起来。整数键只使用h字段，字符串键也会在key字段中保存hash值。不过这么做也许对性能造成一些不利的影响，因为从key字段中提取hash值会增加内存的开销。

我想说的最后一点是，PHP 7不仅仅只是改进了Hashtable的内部实现，而且还改变了跟它们相关的API。我之前了解过一些简单的操作是怎么被使用的，像zend_hash_find这种操作，特别是它们需要间接调用多少次（提示：3次）（这个地方不是很清楚，估计他想说的是要得到一个结果需要调用多个函数，而实际上一个函数就可以搞定的）。在PHP7中，你只需要写zend_hash_find(ht, key)，就可以得到一个zval的指针（zval*）。所以我发现在PHP 7中编写扩展也变得更舒坦了。

希望我的这篇文章能够为PHP 7中的Hashtable的内部机制提供一些有价值的观点。也许我会再写一篇介绍zval的文章。这篇文章已经涵盖了部分相关内容，不过关于这个话题还有很多可探讨的东西。

链接：http://gywbd.github.io/posts/2014/12/php7-new-hashtable-implementation.html
