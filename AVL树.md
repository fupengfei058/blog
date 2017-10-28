AVL树是最先发明的自平衡二叉查找树。在AVL树中任何节点的两个儿子子树的高度最大差别为一，所以它也被称为高度平衡树。

查找、插入和删除在平均和最坏情况下都是O（log n）。增加和删除可能需要通过一次或多次树旋转来重新平衡这个树。

对二叉树的平衡调整过程，主要包含四种旋转操作：LL，LR，RR，RL 。

LR由当前节点左儿子的一次RR旋转和当前节点的一次LL旋转构成。同理， RR由当前节点右儿子的一次LL旋转和当前节点的一次RR旋转构成。

如图：

[!github](http://img.my.csdn.net/uploads/201212/14/1355457660_7501.png)

AVL树的 插入 和删除均要调整重新构成平衡树。

具体实现代码即注释如下：Head.h    fun.c    AvlTree.c

Head.h
```cpp
#ifndef HEAD_H_  
#define HEAD_H_  
  
#include <stdio.h>  
#include <stdlib.h>  
  
typedef int ElementType;  
typedef struct AvlNode              // AVL树的节点  
{  
    ElementType data;  
    struct AvlNode *left;           // 左孩子  
    struct AvlNode *right;          // 右孩子  
    int Height;  
}*Position,*AvlTree;  
  
AvlTree MakeEmpty(AvlTree T);  
Position Find(ElementType x,AvlTree T);  
Position FindMin(AvlTree T);  
Position FindMax(AvlTree T);  
AvlTree  Insert(ElementType x,AvlTree T);  
AvlTree  Delete(ElementType x,AvlTree T);  
ElementType Retrieve(Position P);  
void Display(AvlTree T);  
  
#endif /* HEAD_H_ */  
```
fun.c  
```cpp
#include"Head.h"  
/* 
 *   初始化AVL树 
 */  
AvlTree MakeEmpty(AvlTree T)  
{  
    if( T != NULL)  
    {  
        MakeEmpty(T->left);  
        MakeEmpty(T->right);  
        free(T);  
    }  
    return NULL;  
}  
  
/* 
 * 查找 可以像普通二叉查找树一样的进行，所以耗费O(log n)时间，因为AVL树总是保持平衡的。 
 * 不需要特殊的准备，树的结构不会由于查找而改变。（这是与伸展树查找相对立的，它会因为查找而变更树结构。） 
 */  
Position Find(ElementType x,AvlTree T)  
{  
    if(T == NULL)  
        return NULL;  
    if(x < T->data)  
        return Find(x,T->left);  
    else if(x > T->data)  
        return Find(x,T->right);  
    else  
        return  T;  
}  
/* 
 *   FindMax，FindMin 查找最大和最小值， 
 */  
Position FindMin(AvlTree T)  
{  
    if(T == NULL)  
        return NULL;  
    if( T->left == NULL)  
        return T;  
    else  
        return FindMin(T->left);  
}  
Position FindMax(AvlTree T)  
{  
    if(T != NULL)  
        while(T->right != NULL)  
            T=T->right;  
    return T;  
}  
/* 
 *  返回节点的高度 
 */  
static int Height(Position P)  
{  
    if(P == NULL)  
        return -1;  
    else  
        return P->Height;  
}  
static int Max(int h1,int h2)  
{  
    return h1 > h2 ?h1:h2;  
}  
/* 
 *  此函数用于k2只有一个左孩子的单旋转， 
 *  在K2和它的左孩子之间旋转， 
 *  更新高度，返回新的根节点 
 */  
static Position SingleRotateWithLeft(Position k2)     // LL旋转  
{  
    Position k1;  
    k1=k2->left;  
    k2->left=k1->right;  
    k1->right=k2;  
    // 因为比较的是左右孩子的高度，所以求父节点的高度要加1  
    k2->Height=Max(Height(k2->left),Height(k2->right)) + 1;  
    k1->Height=Max(Height(k1->left),Height(k2->right)) + 1;  
    return k1;  
}  
/* 
 *  此函数用于k1只有一个右孩子的单旋转， 
 *  在K1和它的右孩子之间旋转， 
 *  更新高度，返回新的根节点 
 */  
static Position SingleRotateWithRight(Position k1)  // RR旋转  
{  
    Position k2;  
    k2=k1->right;  
    k1->right=k2->left;  
    k2->left=k1;  
     /*结点的位置变了, 要更新结点的高度值*/  
    k1->Height=Max(Height(k1->left),Height(k1->right)) + 1;  
    k2->Height=Max(Height(k2->left),Height(k2->right)) + 1;  
    return k2;  
}  
/* 
 * 此函数用于当 如果 k3有一个左孩子，以及 
 * 它的左孩子又有右孩子，执行这个双旋转 
 * 更新高度，返回新的根节点 
 */  
static Position DoubleRotateLeft(Position k3)    // LR旋转  
{  
    //在 k3 的左孩子，执行右侧单旋转  
    k3->left=SingleRotateWithRight(k3->left);  
    // 再对 k3 进行 左侧单旋转  
    return SingleRotateWithLeft(k3);  
}  
/* 
 * 此函数用于当 如果 k4有一个右孩子，以及 
 * 它的右孩子又有左孩子，执行这个双旋转 
 * 更新高度，返回新的根节点 
 */  
static Position DoubleRotateRight(Position k4)    // RL旋转  
{  
    //在 k4 的右孩子，执行左侧单旋转  
    k4->right = SingleRotateWithLeft(k4->right);  
    // 再对 k4 进行 右侧单旋转  
    return SingleRotateWithRight(k4);  
}  
/* 
 *  向AVL树插入可以通过如同它是未平衡的二叉查找树一样把给定的值插入树中， 
 *  接着自底向上向根节点折回，于在插入期间成为不平衡的所有节点上进行旋转来完成。 
 *  因为折回到根节点的路途上最多有1.5乘log n个节点，而每次AVL旋转都耗费恒定的时间， 
 *  插入处理在整体上耗费O(log n) 时间。 
 */  
AvlTree  Insert(ElementType x,AvlTree T)  
{  
    //如果T不存在，则创建一个节点树  
    if(T == NULL)  
    {  
        T = (AvlTree)malloc(sizeof(struct AvlNode));  
        {  
            T->data = x;  
            T->Height = 0;  
            T->left = T->right = NULL;  
        }  
    }  
    // 如果要插入的元素小于当前元素  
    else if(x < T->data)  
    {  
        //递归插入  
        T->left=Insert(x,T->left);  
        //插入元素之后，若 T 的左子树比右子树高度 之差是 2，即不满足 AVL平衡特性，需要调整  
        if(Height(T->left) - Height(T->right) == 2)  
        {  
            //把x插入到了T->left的左侧，只需 左侧单旋转  
            if(x < T->left->data)  
                T = SingleRotateWithLeft(T);       // LL旋转  
            else  
                // x 插入到了T->left的右侧，需要左侧双旋转  
                T =  DoubleRotateLeft(T);          // LR旋转  
        }  
    }  
    // 如果要插入的元素大于当前元素  
    else if(x > T->data)  
    {  
        T->right=Insert(x,T->right);  
        if(Height(T->right) - Height(T->left) == 2)  
        {  
            if(x > T->right->data)  
                T = SingleRotateWithRight(T);     //RR 旋转  
            else  
                T =  DoubleRotateRight(T);        //RL旋转  
        }  
    }  
    T->Height=Max(Height(T->left),Height(T->right)) + 1;  
    return T;  
}  
/* 
 *  对单个节点进行的AVL调整 
 */  
AvlTree Rotate(AvlTree T)  
{  
  
    if(Height(T->left) - Height(T->right) == 2)  
    {  
        if(Height(T->left->left) >= Height(T->left->right))  
            T = SingleRotateWithLeft(T);  // LL旋转  
        else  
            T =  DoubleRotateLeft(T);     // LR旋转  
    }  
    if(Height(T->right) - Height(T->left) == 2)  
    {  
        if(Height(T->right->right) >= Height(T->right->left))  
            T = SingleRotateWithRight(T);  // RR旋转  
        else  
            T =  DoubleRotateRight(T);     // RL旋转  
    }  
    return T;  
}  
/* 
 * 首先定位要删除的节点，然后用该节点的右孩子的最左孩子替换该节点， 
 * 并重新调整以该节点为根的子树为AVL树，具体调整方法跟插入数据类似 
 * 删除处理在整体上耗费O(log n) 时间。 
 */  
AvlTree  Delete(ElementType x,AvlTree T)  
{  
    if(T == NULL)  
        return NULL;  
    if(T->data == x)           // 要删除的 x 等于当前节点元素  
    {  
        if(T->right == NULL )  // 若所要删除的节点 T 的右孩子为空,则直接删除  
        {  
            AvlTree tmp = T;  
            T = T->left;  
            free(tmp);  
        }  
        else                 /* 否则找到 T->right 的最左儿子代替 T */  
        {  
            AvlTree tmp = T->right;  
            while(tmp->left)  
                tmp=tmp->left;  
            T->data = tmp->data;  
            /* 对于替代后的T 即其字节点进行调整*/  
            T->right = Delete(T->data,T->right);  
            T->Height = Max(Height(T->left),Height(T->right))+1;  
        }  
        return T;  
    }  
    else if(x > T->data)                       // 要删除的 x 大于当前节点元素，在T的右子树中查找删除  
    {  
        T->right=Delete(x,T->right);  
    }  
    else                                       // 要删除的 x 小于当前节点元素，在T的左子树中查找删除  
    {  
        T->left=Delete(x,T->left);  
    }  
    /* 
     *   当删除元素后调整平衡 
     */  
    T->Height=Max(Height(T->left),Height(T->right)) + 1;  
    if(T->left != NULL)  
        T->left = Rotate(T->left);  
    if(T->right != NULL)  
        T->right = Rotate(T->right);  
    if(T)  
        T=Rotate(T);  
    return T;  
}  
/* 
 * 返回当前位置的元素 
 */  
ElementType Retrieve(Position P)  
{  
    return P->data;  
}  
/* 
 * 遍历输出 
 */  
void Display(AvlTree T)  
{  
    static int n=0;  
    if(NULL != T)  
    {  
        Display(T->left);  
        printf("[%d] ndata=%d \n",++n,T->data);  
        Display(T->right);  
    }  
} 
```
AvlTree.c
```cpp
#include"Head.h"  
#define N 15  
int main(void) {  
    AvlTree T=NULL;  
    int i;  
    int j = 0;  
    T = MakeEmpty( NULL );  
    for( i = 0; i < N; i++, j = ( j + 7 ) % 50 )  
    {  
        printf("j=%d \n",j);  
        T = Insert( j, T );  
    }  
    puts("插入 4 \n");  
    T = Insert( 4, T );  
    Display(T);  
   for( i = 0; i < N; i += 2 )  
   {  
       printf("delelte: %d \n",i);  
        T = Delete( i, T );  
   }  
   printf("detele:\n");  
   printf("height=%d \n",T->Height);  
   Display(T);  
  
    printf( "Min is %d, Max is %d\n", Retrieve( FindMin( T ) ),  
               Retrieve( FindMax( T ) ) );  
    return EXIT_SUCCESS;  
}
```
 
