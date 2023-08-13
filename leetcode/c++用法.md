## cmp比较函数的写法

* cmp

```c++
bool cmp(int a, int b) {return a > b;}
```



* priority_queue

less<>大顶堆（默认），greater<>是小顶堆

自定义的写法：

```C++
//小顶堆
struct cmp{
    bool operator() (ListNode *&l1, ListNode *&l2){return l1->val > l2->val;}
};
priority_queue<ListNode *, vector<ListNode *>, cmp> pq;
```



## 字符串技巧

