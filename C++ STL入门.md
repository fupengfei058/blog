### 0x00 何为STL

**STL(Standard Template Library)** 即**标准模板库**。它是一个具有工业强度，高效的C++程序库。它包含了诸多在计算机科学领域里所常用的基本数据结构和算法。这些数据结构可以与标准算法一起很好的工作，这为我们的软件开发提供了良好的支持。如果你还不理解它的重要性，那我换个说法。这就好比你去打架，你不会使用STL，那就手里的武器就相当于弹弓。敌人熟练使用STL，人家手里拿的是AK47，开着坦克。

从名字就可以知道，它的设计基石是模板技术。这样的设计带来了更好的重用机会。STL共有以下六大组件。

* 迭代器(iterator)

* 容器(container)

* 算法(algorithm)

* 仿函数(function object)

* 适配器(Adaptor)

* 空间配制器(allocator)

仿函数和空间配制器不是很常用，我们主要讨论一下迭代器，容器，算法和适配器。其中，我们以容器的用法为重点。

### 0x01 迭代器

迭代器在STL中起着粘合剂的作用，它将算法和容器联系起来，主要用来存取容器中的元素。几乎所有的算法都是通过迭代器存取元素进行工作的。每一个容器也都定义了其本身所专有的迭代器，用以存取容器中的元素。想象一下，你面前有一缸水(缸就好比容器)，你喝水需要要到瓢(咱是文明人，不带用双手直接捧着喝的)。这个瓢就相当于迭代器，你可以用它来打水喝，也可以用瓢来把水缸装满。
```c++
#include <iostream>
#include <vector>

using namespace std;

int main()
{
    vector<int> v;        // 定义一个vector容器

    v.push_back(1);        // 向容器中添加3个元素
    v.push_back(2);
    v.push_back(3);

    // 遍历向量的元素
    vector<int>::iterator b = v.begin();        // 指向容器的第一个元素
    vector<int>::iterator e = v.end();            // 指向容器尾元素的下一个位置

    // C++11新标准的写法, auto关键字为类型推断，由编译器自动完成
    // auto b = v.begin();
    // auto e = v.end();

    for (vector<int>::iterator iter = b; iter != e; ++iter)
    {
        cout << *iter << endl;
    }

    return 0;
}
```
迭代器的使用，上面给了一段简单的代码，我们来精析一下。

迭代器最常用到的就是**begin**和**end**成员。其中begin成员负责返回指向第一个元素。end成员则负责返回指向容器的“ **尾元素的下一个位置(one past the end)** ”。要特别注意end成员不是指向尾元素，而是指向尾元素的下一个位置！ end成员返回的迭代器也叫尾后迭代器(off-the-end iterator)，简称尾迭代器。

如果容器为空呢？那么begin和end返回的是同一个迭代器，都是**尾后迭代器**。

这里要注意一下for循环的循环条件。

* 初始化语句：vector<int>::iterator iter = b; 如果你的环境支撑C++11标准，那么强烈建议你写成auto iter = b; 即使用类型自动推断关键字auto。使用auto使程序更为简洁，也不会出错，由编译器自动推断。

* 条件语句 iter != e; 一般的for循环里我们会用itet < e 这样的形式，当然，在vector里改成这样也是可以的。但是，并非所有的容器都重载了 < 运算符，所有的容器都重载了== 和 != 运算符。所以我们应该习惯使用 == 和 != 运算符。

* 表达式语句 ++iter。 建议使用前置++而非后置++。 在迭代器中，前置++的效率高于后置++。实际上，除非逻辑需要，一般都使用前置++ 进行向前迭代。关于前置++和后置++的本质区别，看官可自行查看其它资料。

**标准容器迭代器的运算符:**

* *iter： 返回迭代器iter所指元素的引用

* iter->mem: 解引用iter并获取该元素的名为mem的成员，等价于(*item).mem

* ++iter: 另iter指向容器的下一个元素

* --iter: 另iter指向元素的前一个元素
* iter1 == iter2：判断两个迭代器是否相等
* iter1 != iter2: 判断两个迭代器是否不相等
**迭代器类型：**

* iterator ：可读可写。

* const_iterator ： 可读不可写。使用迭代器带c的版本来返回，尤其是使用auto关键字的时候。

**迭代器的范围:**

**迭代器范围(iterator range)** 由一对迭代器表示，最常见的就是begin和end。begin和end所表示的范围恰好是容器的全部元素。这是一个左闭合区间(left-inclusive interval),其标准的数学表达式为:

**[begin,end)**

**其他迭代器:**

除了为每个容器定义迭代器外，标准库在头文件 iterator中还定义了额外几种迭代器，这些迭代器包括以下几种。

* 插入迭代器(insert iterator)： 这些迭代器被绑定到一个容器上，可用来向容器中插入元素。

* 流迭代器(stream iterator)： 这些迭代器被绑定到输入或输出流上，可用来遍历相关的IO流。

* 反向迭代器(reverse iterator) 这些迭代器和正常的迭代器移动方向相反。例如++操作是指向前一个元素。除了forward_list之外的标准库库容器都有反向迭代器。即迭代器的r版本。

* 移动迭代器(move iterator) 这些专用的迭代器不是拷贝其中的元素，而是移动它们。

**迭代器类别:**

算法所要求的迭代器可以分为5个迭代器类别(iterator category)。

* 输入迭代器 : 只读，不写。单遍扫描，只能递增。

* 输出迭代器 ： 只写，不读。单遍扫描，只能递增。

* 前向迭代器 ： 可读可写。多遍扫描，只能递增。

* 双向迭代器 ： 可读可写。多遍扫描，可递增递减。

* 随机访问迭代器 ： 可读可写。多遍扫描，支持全部迭代器运算。

下面的例子演示一下迭代器的运算，c版本的迭代器，r版本的迭代器。
```c++
#include <iostream>
#include <vector>

using namespace std;

int main()
{
    vector<int> v;        // 定义一个vector容器

    v.push_back(1);        // 向容器中添加5个元素
    v.push_back(2);
    v.push_back(3);
    v.push_back(4);
    v.push_back(5);

    // 使用c版本的迭代器
    auto b = v.cbegin();    // 带c版本的迭代器表示const_iterator类型的迭代器
    auto e = v.cend();        // 指向容器尾元素的下一个位置

    for (auto iter = b; iter != e; ++iter)
    {
        // *iter *= 2;            // 报错，试图给常量赋值！
    }

    // 反向输出容器中的元素，使用r版本的迭代器
    auto rb = v.rbegin();        // 实际指向尾元素
    auto re = v.rend();            // 指向第一个元素的前一个位置

    for (auto iter = rb; iter != re; ++iter)
    {
        cout << *iter << endl;
    }

    // 进行迭代器的运算，输出容器的中间元素
    auto mid = v.begin() + v.size() / 2;
    cout << "该容器的中间元素为:" << *mid << endl;

    return 0;
}
```
### 0x02 容器

容器的定义是：**特定类型对象的集合。**

在没有使用容器之前，我们可能会用数组解决一些问题。使用数组解决问题，那么我们必须要知道或者估算出大约要存储多少个对象，这样我们会创建能够容纳这些对象的内存空间大小。当我们要处理一些完全不知道要存储多少对象的问题时，数组显的力不从心。我们可以使用容器来解决这个问题。容器具有很高的可扩展性，我们不需要预先告诉它要存储多少对象，只要创建一个容器，并合理的调用它所提供的方法，所有的处理细节由容器自身完成。

> 新标准库的容器的性能几乎肯定与最精心优化过的同类数据结构一样好(通常会更好)。现代C++程序应该使用标准容器库，而不是更原始的数据结构，如内置数组。
**通用容器的分类**

通用容器分为3类：顺序容器、关联容器、容器适配器。

**顺序容器**

顺序容器是一种元素之间有顺序的线性表，是一种线性结构的可序群集。这和我们数据结构课程上所讲的线性表是一样的。顺序容器中的每个元素位置是固定的，除非你使用了插入或者删除操作改变了这个位置。顺序容器不会根据元素的特点排序而是直接保存了元素操作时的逻辑顺序。比如我们一次性对一个顺序容器追加三个元素，这三个元素在容器中的相对位置和追加时的逻辑次序是一致的。

顺序容器都提供了快速顺序访问元素的能力。但是，他们在以下方面都有不同的性能折中：

* 向容器中添加或者向容器中删除元素的代价。(不是末端)

* 非顺序访问容器中元素的代价。

**顺序容器的类型：**

* vector : 可变大小数组，支持快速随机访问。在尾部之外的位置插入或者删除元素可能很慢。

* deque : 双端队列。支持快速随机访问。在头尾位置插入、删除速度很快。

* list : 双向链表。只支持双向顺序访问。在list中任何位置进行插入、删除操作速度都很快。

* forward_list : 单向链表。只支持单向顺序访问。在链表的任何位置进行插入、删除操作都很快。(C++11标准新加)

* array : 固定大小数组。支持快速随机访问。不能添加或者删除元素。(C++11标准新加)

* string : 与vector相似的容器，但专门用于保存字符。随机访问快，在尾部插入删除快。

如何选择呢？是不是又犯了选择困难症？ 我们一般对症下药，了解这些容器的特性，根据自己的编程需求选择适合的容器。vector、deque和list这三者我们可以优先考虑vector。vector容器适用于大量读写，而插入、删除比较少的操作。list容器适用于少量读写，大量插入，删除的情况。deque折中了vector和deque， 如果你需要随机存取又关心数据的插入和删除，那么可以选择deque。forward_list适用于符合它这种逻辑结构的情况，array一般用来代替原生的数组。string用于和字符串操作有关的一些情况，也是实际开发中应用最多的。

关于各容器的操作，实在是太多了，下面的示例程序列举一些比较常见的操作和用法。
```c++
#include <iostream>
#include <vector>
#include <string>
#include <deque>
#include <list>
#include <forward_list>
#include <array>

using namespace std;

int main()
{
    /*--------------------- vector容器的一些操作  ------------------*/
    vector<int> vect1;            // 定义一个vector容器
    vect1.push_back(1);            // push_back: 向容器的末尾添加元素
    vect1.push_back(2);
    vect1.push_back(3);
    vect1.pop_back();            // pop_back: 去除末尾的元素

    vect1.insert(vect1.begin() + 1, 8);    // 在某个位置插入一个元素,效率低，不适合大批操作
    vect1.at(0);                        // at:取某个位置的元素
    vect1.capacity();                    // capacity: 不分配新的内存空间的前提下它最多能保存多少元素。这个和下面的size 是有区别的！！
    vect1.size();                        // size: 已经保存的元素的数目
    vect1.empty();                        // empty：判断容器是否为空
    vect1.front();                        // front：取第一个元素
    vect1.back();                        // back：取最后一个元素
    vect1.erase(vect1.begin() + 1);        // erase：删除指定位置的元素
    vector<int> vect2;
    vect2.assign(vect1.begin(), vect1.end()); // 赋值操作
    /*------------------------------------------------------------*/

    // 其他容器操作都和vector差不多，以下列举一些其他容器特有的操作


    /*--------------------- string容器一些操作  --------------------*/
    string str1 = "Hello Ace";            // string的几种构造方法
    string str2("Hello World");        
    string str3(str1, 6);                // 从str1下标6开始构造， str3 -> Ace

    string str4 = str2.substr(0, 5);    // 求子串： str4 -> Hello
    string str5 = str2.substr(6);        // 求子串： str5 -> World
    string str6 = str2.substr(6, 11);    // 求子串： str6 -> World
    // string str7 = str2.substr(12);    // 抛异常： out_of_range

    string str8 = str2.replace(6, 5, "Game");    // 替换：str8 -> Hello Game 从位置6开始，删除5个字符，并替换成"Game"

    string str9 = str2.append(", Hello Beauty");// 追加字符串： str9 -> Hello World, Hello Beauty

    auto pos1 = str1.find("Ace");                // 查找字符串    : pos1 -> 6 ,返回第一次出现字符串的位置，如果没找着，则返回npos

    int res = str1.compare("Hello, Ace");        // 比较字符串： res -> -1, 根据str1是等于、大于还是小于参数指定的字符串， 返回0、整数或者负数

    string str10 = "Pi = 3.14159";
    double pi = stod(str10.substr(str10.find_first_of("+-.0123456789")));    // 数值转换： pi -> 3.14159
    /*------------------------------------------------------------*/


    /*--------------------- deque容器一些操作  --------------------*/
    deque<int> d1;
    d1.push_back(1);                            // 尾后压入元素
    d1.push_back(2);
    d1.push_back(3);
    d1.push_front(4);                            // 队头压入元素
    d1.push_front(5);
    d1.push_front(6);
    d1.pop_back();                                // 尾后弹出一个元素
    d1.pop_front();                                // 队头弹出一个元素

    d1.front();                                    // 取队头元素
    d1.back();                                    // 取队尾元素
    /*------------------------------------------------------------*/


    /*--------------------- list容器一些操作  --------------------*/
    list<int> l;
    l.push_back(1);                                // 尾后压入元素
    l.push_back(2);
    l.push_back(3);
    l.push_front(4);                            // 队头压入元素
    l.push_front(5);
    l.push_front(6);
    l.pop_back();                                // 尾后弹出一个元素
    l.pop_front();                                // 队头弹出一个元素
    l.front();                                    // 取队头元素
    l.back();                                    // 取队尾元素

    l.insert(l.begin(), 88);                    // 某个位置插入元素(性能好)
    l.remove(2);                                // 删除某个元素(和所给值相同的都删除)
    l.reverse();                                // 倒置所有元素
    l.erase(--l.end());                            // 删除某个位置的元素(性能好)
    /*------------------------------------------------------------*/


    /*--------------------- forward_list容器一些操作  --------------*/
    forward_list<int> fl = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    fl.push_front(0);                // 压入元素，该容器没有push_back方法
    auto prev = fl.before_begin();    // 表示fl的"首前元素"
    auto curr = fl.begin();            // 表示fl的第一个元素

    // 循环遍历
    while (curr != fl.end())        // 表示仍有元素要处理
    {
        if (*curr % 2)                // 若元素为奇数，则删除
        {
            curr = fl.erase_after(prev);    // 删除它并移动curr
        }
        else
        {
            prev = curr;            // 移动迭代器curr，指向下一个元素，prev指向curr之前的元素
            ++curr;
        }
    }

    // 操作后： fl = {0, 2, 4, 6, 8}
    /*------------------------------------------------------------*/


    /*--------------------- array容器一些操作  --------------------*/
    array<int, 5> myArray1 = { 1, 2, 3, 4, 5 };    // 定义一个一维数组
    array<array<int, 2>, 3> myArray2 = {1, 2, 3, 4, 5, 6};    // 定义一个二维数组
    array<int, 5> myArray3 = {6, 7, 8, 9, 10};
    array<int, 5> myArray4;                // 此数组并未初始化

    // array.resize();        // array 不能有改变容器大小的操作，它的效率比vector高
    myArray1.swap(myArray3);// 交换两个数组的的元素
    myArray4 = myArray1;    // 支持直接这样赋值，原生的数组不可以这样。它把值全部复制过去，而不是引用
    myArray1.assign(0);        // 把myArray1的元素全部置为0

    // 遍历数组元素
    for (int i = 0; i < myArray1.size(); ++i)
    {
        cout << myArray1[i] << endl;
    }
    /*------------------------------------------------------------*/

    return 0;
}
```
**关联容器**

关联容器(associative-container)和顺序容器有着根本的不同：关联容器中元素定义是按关键字来保存和访问的。与之相对，顺序容器中的元素是按他们在容器中的位置来顺序保存和访问的。虽然关联容器的很多行为和顺序容器相同，但其不同之处反映了关键字的作用。

关联容器支持高效的关键字查询和访问。标准库一共定义了8个关联容器，最主要的类型是map和set。8个容器中，每个容器：

* 是一个map或者是一个set。map保存关键字-值对；set只保存关键字。

* 要求关键字唯一或者不要求。

* 保持关键字有序或者不保证有序。

**关联容器类型：**

按关键字有序保存元素

* map : 关联数组；保存关键字-值对

* set : 关键字即值，即只保存关键字的容器

* multimap : 关键字可重复的map

* multiset ：关键字可重复的set

无序集合

* unordered_map ： 用哈希函数组织的map

* unordered_set : 用哈希函数组织的set

* unordered_multimap ： 哈希组织的map;关键字可以重复出现

* unordered_multiset : 哈希组织的set;关键字可以重复出现

从上面的容器名称可以看出：允许重复关键字的容器名字都包含multi；而使用哈希技术的容器名字都以unordered开头。

**pair类型**

使用关联容器，绕不开pair类型。它定义在标准库头文件utility中。一个pair保存两个数据成员。类似容器，pair是一个用来生成特定类型的模板。当创建pair时，我们必须提供两个类型名，pair的成员将具有对应的类型。与其他标准库类型不同，pair的数据成员是public的。两个成员分别命名为first和second。
```
pair<string, string> author{"Stanley", "C++ Prime"};    // 构造一个pair

make_pair(v1, v2);                                        // 返回一个用v1和v2初始化的pair。pair的类型从v1和v2的类型推断出来
```
**map的使用**

下面的程序是统计每个单词在输入中出现的次数：
```c++
#include <iostream>
#include <map>
#include <string>

using namespace std;

int main()
{
    // 统计每个单词在输入中出现的次数
    map<string, size_t> word_count;        // string到map的空map
    string word;
    while (cin >> word)
    {
        ++word_count[word];                // 提取word的计数器并将其加1
    }

    for (const auto &w : word_count)    // 遍历map的每个元素
    {
        cout << w.first << "出现的次数为: " << w.second << endl;
    }

    return 0;
}
```
**set的使用**

对上面那个统计单词的程序做一个扩展，忽略常见单词。比如 the and or then等。 我们使用set保存想要忽略的单词，只对不在集合中的单词进行统计。
```c++
#include <iostream>
#include <map>
#include <set>
#include <string>

using namespace std;

int main()
{
    // 统计每个单词在输入中出现的次数
    map<string, size_t> word_count;        // string到map的空map
    set<string> exclude = {"The", "But", "And", "Or", "An", "A", 
                            "the", "but", "and", "or", "an", "a"};
    string word;
    while (cin >> word)
    {
        // 只统计不在exclude中的单词。find调用返回一个迭代器，如果在集合中，返回的迭代器指向其该关键中。否则返回尾后迭代器
        if (exclude.find(word) == exclude.end())
        {
            ++word_count[word];                // 提取word的计数器并将其加1
        }
    }

    for (const auto &w : word_count)    // 遍历map的每个元素
    {
        cout << w.first << "出现的次数为: " << w.second << endl;
    }

    return 0;
}
```
### 0x03 容器适配器

除了顺序容器外，标准库还定义了三个顺序容器适配器:**stack、 queue**和**priority_queue**。 适配器(adaptor)是标准库的一个通用概念。容器、迭代器和函数都有适配器。

> 本质上，一个适配器是一种机制，能使某种事物的行为看起来像另一种事物一样。
**所有容器适配器都支持的操作和类型**

* size_type : 一种类型，足以保存当前类型的最大对象的大小

* value_type : 元素类型

* container_type : 实现适配器的底层容器类型

* A a : 创建一个名为a的空适配器

* A a(c) : 创建一个名为a的适配器，带有容器c的一个拷贝

* 关系运算符 : 每个适配器都支持所有关系运算符： ==、！=、<、<=、>、和>=。这些运算符返回底层容器的比较结果。

* a.empty() ： 若a包含任何元素，返回fasle，反正返回true

* a.size() : 返回a中的元素数目

* swap(a, b) : 或写作a.swap(b)、b.swap(a)。交换a和b的内容。a和b必须有相同的类型，包括底层容器类型也必须相同

**栈适配器(stack)的额外操作**

* s.pop() : 删除栈顶元素，但不返回该元素值。

* s.push(item) : 创建一个新元素压入栈顶

* s.emplace(args) ： 同push，其值由args构造

* s.top() ： 返回栈顶元素，但不将元素弹出栈

* queue和priority_queue的额外操作

* q.pop() : 返回queue的首元素或priority_queue的最高优先级的元素，但不删除此元素。

* q.front() : 返回首元素或者尾元素，但不删除此元素

* q.back() : 只使用于queue

* q.top() : 返回最高优先级元素，但不删除此元素

* q.push(item) : 在queue末尾或者priority_queue中恰当的位置创建一个元素，其值为item

* q.emplace(args) : 同push,其值由args构造

> 栈默认基于deque实现。queue默认基于deque实现。priority_queue默认基于vector实现。
stack和queue的使用方法比较简单，priority_queue在存储自己定义的数据结构时，必须重载 operator < 或者自己写仿函数。下面给个简单的例子：
```c++
#include <iostream>
#include <queue>

using namespace std;

struct Node
{
    int x;
    int y;
};

struct MyCmp
{
    // 自定义的比较函数
    bool operator ()(Node a, Node b)
    {
        if (a.x == b.x)
        {
            return a.y > b.y;
        }

        return a.x > b.x;
    }
};

int main()
{
    // priority_queue<Type, Container, Functional>
    // Type 为数据类型，Container 为保存数据的容器，Functional 为元素比较方式
    priority_queue<Node, vector<Node>, MyCmp>  myQueue;

    // 添加一些元素
    for (int i = 1; i <= 10; ++i)
    {
        Node node;
        node.x = i;
        node.y = i * i;
        myQueue.push(node);
    }

    // 遍历元素
    while (!myQueue.empty())
    {
        cout << myQueue.top().x << "," << myQueue.top().y << endl;
        myQueue.pop();            // 出队
    }

    return 0;
}
```
### 0x04 泛型算法

虽然容器提供了众多操作，但有些常见的操作，比如查找特定的元素，替换或者删除某个特定值，重新排序等，这些由一组泛型算法(generic algorithm)来实现。

大多数的算法都定义在头文件algorithm中，有些关于数值的泛型算法定义在numeric这个头文件中。

标准库提供了上百个算法，幸运地是，它们的算法结构基本上是一致的。这样我们就不用死记硬背了。

**算法的形参模式**

大多数的算法具有如下4种形式之一：

* alg(beg, end, other args);

* alg(beg, end, dest, other args);

* alg(beg, end, beg2, other args);

* alg(beg, end, beg2, end2, other args);

其中alg是算法的名字,beg和end表示算法所操作的输入范围。dest表示指定目的位置,beg2和end2表示接受第二个范围。

> 标准算法库对迭代器而不是容器进行操作。因此，算法不能直接添加或者删除元素(可以调用容器本身的操作来完成)。
find和sort是两个比较常见的泛型算法，我们以这两个为例子，来演示一下泛型算法的使用。

**find的简单使用**

```c++
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int main()
{
    int val = 5;
    int arr[10] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
    vector<int> vec = { 11, 22, 33, 44, 55, 66, 77, 88, 99 };

    // 查找元素的范围是第2个元素到第8个元素，支持内置数组
    // 如果找到想要的元素，则返回结果指向它
    auto result = find(arr + 1, arr + 7, val);
    cout << *result << endl;    // 输出结果为 5,如果没找到返回7，想一下为什么

    int val2 = 100;
    // 没有找到这个值，返回vec.cend()
    auto res = find(vec.begin(), vec.end(), val2);
    if (res == vec.cend())
    {
        cout << "没找到元素!" << endl;
    }
    else
    {
        cout << *res << endl;
    }

    return 0;
}
```
简述一下find的执行步骤

1. 访问序列中的元素
2. 比较此元素与我们要查找的值
3. 如果此元素与我们要查找的值匹配，find返回标示此元素的值。
4. 否则,find前进到下一个元素，重复执行步骤2和3。
5. 如果到达序列尾,find停止。
6. 如果find到达序列末尾，它应该返回一个指出元素未找到的值。此值和步骤3中返回的值必须具有相同的类型。

**sort的简单使用**

参数形式为：sort(beg, end, cmp)

对于基本数据类型，第三个参数是可以省略的，有默认的实现。但对于自定义的数据类型，我们要提供第三个参数。第三个参数叫做谓词(predicate)。标准库有一元谓词(unary predicate)和二元谓词(binary predicate)之分，分别表示只接受1个参数和只接受2个参数。

下面实现一个小程序，有语文和数学两门课的成绩，按总分从大到小排序。如果总分相同，数学成绩高的排在前面。
```c++
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

using namespace std;

struct CoureSocre
{
    string name;    // 姓名
    int math;        // 数学成绩
    int chinese;    // 语文成绩
    int total;        // 总成绩

    CoureSocre(string _name, int _math, int _chinese)
    {
        name = _name;
        math = _math;
        chinese = _chinese;
        total = math + chinese;
    }
};

bool myCmp(CoureSocre c1, CoureSocre c2)
{
    // 如果总成绩相同
    if (c1.total == c2.total)
    {
        return c1.math >= c2.math;
    }

    return c1.total > c2.total;
}

int main()
{
    // 初始化5个学生的程序
    CoureSocre c1("Ace", 90, 95);
    CoureSocre c2("Shawna", 99, 100);
    CoureSocre c3("Kelly", 100, 99);
    CoureSocre c4("Jordan", 88, 90);
    CoureSocre c5("Kobe", 90, 88);

    // 加入容器
    vector<CoureSocre> vecScoreList = { c1, c2, c3, c4, c5 };

    // 调用sort算法进行排序
    sort(vecScoreList.begin(), vecScoreList.end(), myCmp);

    cout << "学生的成绩排名为:" << endl;
    for each (CoureSocre c in vecScoreList)        // 使用for each 算法进行遍历
    {
        cout << "姓名:" << c.name << "\t总成绩:" << c.total << "\t数学:" << c.math << "\t语文:" << c.chinese << endl;
    }

    return 0;
}
```
另外一个和sort相关的是stable_sort算法。这种稳定排序算法维持相等元素的原有顺序。

传递的谓词只能接受1个或者2个参数，如果我们想传入更多的参数怎么办呢，这就超出了算法对谓词的限制。这时候，我们就需要上lambda表达式了。具体细节以后会介绍。

### 0x05 结束语

C++ STL很好很强大，熟练使用它将使你如虎添翼。补充一个新手常踩的坑，在使用for循环时，不要在里面使用改变迭代器的操作，比如insert和erase，这些操作会使迭代器失效，从而引发意想不到的bug。


链接：http://www.jianshu.com/p/26d4d60233a4
