LRU算法，Least Recently Used，最近最久未使用。常用于页面置换算法，是为虚拟页式存储管理服务的



在内存有限的情况下，扩展一部分外存作为虚拟内存。

虚拟页式存储管理，则是将进程所需空间划分为多个页面，内存中只存放当前所需页面，其余页面放入外存的管理方式

对于页面置换算法来讲，当发生缺页中断时，都是要从内存中找到一个不需要的块换出去（对应物理内存的释放），然后将需要页面从磁盘的交换区中换进来 （虚拟内存的分配）

对于系统的所有文件I/O请求，操作系统都是通过page cache机制实现的，对于操作系统而言，磁盘文件都是由一系列的数据块顺序组成，数据块的大小随系统不同而不同，x86 linux系统下是4KB（一个标准页面大小）

内核在处理文件I/O请求时，首先到page  cache中查找（page cache中的每一个数据块都设置了文件以及偏移信息），如果未命中，则启动磁盘I/O，将磁盘文件中的数据块加载到page cache中的一个空闲块，之后再copy到用户缓冲区中

# C++实现LRU算法

LRU算法底层用的是一种双向哈希链表的数据结构，一开始规定这个双向哈希链表中的节点最大值，每个节点由键值对构成，关键字就是id等的编号，方便使用哈希函数进程检索，值域存放相关信息

向该数据结构中插入节点，一旦节点个数超过上限的话，将双向链表头节点删除，若是新插入节点id已经在链表中存在，就将该节点从他的位置删除并插入到链表尾部

```cpp
class LRUCache
{
private:
    typedef list<int> LI;  // 双向链表
    typedef pair<int, LI::iterator> PII;
    typedef unordered_map<int, PII> HIPII;  // 双向哈希链表
    
    void touch(HIPII::iterator it)  // 将双向链表中的it节点从其原位置删除并头插到已用链表中，表示才刚使用
    {
        int key = it->first;
        used.erase(it->second.second);  // 删
        used.push_front(key);  // 插
        it->second.second = used.begin();  // 更新
    }
    
    HIPII cache;
    LI used;
    int _capacity;
    
public:
    LRUCache(int capacity)
        : _capacity(capacity)
    {}
    
    int get(int key)  // 从双向哈希链表中获取id为key的页
    {
        auto it = cache.find(key);
        if(it == cache.end())
        {
            return -1;
        }
        touch(it);
        return it->second.first;
    }
    
    void put(int key, int value)
    {
        auto it = cache.find(key);
        if(it != cache.end())
        {
            touch(it);
        }
        else
        {
            if(cache.size() == _capacity)  // 需要删除最久未使用的页
            {
                cache.erase(used.back());
                used.pop_back();
            }
            used.push_front(key);  // 将新使用的页更新
        }
        cache[key] = {value, used.begin()};
    }
};
```

