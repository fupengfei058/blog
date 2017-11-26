在PHP开发过程中，我们都知道在循环的时候，foreach效率比for高，但是为什么foreach效率高呢？其实这是跟PHP变量的数据结构有关。
```c
typedef struct _zval_struct zval;    
    
struct _zval_struct {    
    /* Variable information */    
    zvalue_value value;     /* value */    
    zend_uint refcount__gc;    
    zend_uchar type;    /* active type */    
    zend_uchar is_ref__gc;    
};    
    
typedef union _zvalue_value {    
    long lval;  /* long value */    
    double dval;    /* double value */    
    struct {    
        char *val;    
        int len;    
    } str;    
    HashTable *ht;  /* hash table value */    
    zend_object_value obj;    
} zvalue_value;    
```
php数组是一个HashTable。HashTable的特点：

* 键(key)：用于操作数据的标示，例如PHP数组中的索引，或者字符串键等等。
* 槽(slot/bucket)：哈希表中用于保存数据的一个单元，也就是数据真正存放的容器。
* 哈希函数(hash function)：将key映射(map)到数据应该存放的slot所在位置的函数。
* 哈希冲突(hash collision)：哈希函数将两个不同的key映射到同一个索引的情况。

HashTable的数据结构如下：
```c
typedef struct _Bucket    
{    
    char *key;    
    void *value;    
    struct _Bucket *next;    
} Bucket;    
     
typedef struct _HashTable    
{    
    int size;    
    int elem_num;    
    Bucket** buckets;    
} HashTable;  
```
通过这段源码可以看出来，如果是foreach的话，可以直接通过_Bucket里的next获取到下一个值，而如果是for循环，$array['key']这样子获取数据，就会需要做一次hash才会知道bucket的位置，所以foreach比for循环效率更高一些。
