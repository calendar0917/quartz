## 数据结构
### FP-Node设计

需要有的属性
- name
- count
- parent
- children：用 map 存，方便查找后代有什么
- nodeLink：指向下一个同名节点，用于横向遍历

```c++
// 先定义数据结构
// 树节点结构
struct FPNode
{
    string name;                    // 物品名称
    int count;                      // 计数
    FPNode *parent;                 // 父节点
    map<string, FPNode *> children; // 子节点
    FPNode *nodeLink;               // 指向下一个同名节点 (用于横向链表)

    FPNode(string n, FPNode *p = nullptr) : name(n), count(1), parent(p), nodeLink(nullptr) {}

    ~FPNode()
    {
        for (auto &pair : children)
            delete pair.second;
    }
};

// 头指针表
struct HeaderItem
{
    int count;        // 全局支持度
    FPNode *headNode; // 链表头
    FPNode *tailNode; // 链表尾 (方便快速插入)

    HeaderItem() : count(0), headNode(nullptr), tailNode(nullptr) {}
};
```

### FP-Tree
```c++
// 定义 FPTree

class FPTree
{
public:
    FPNode *root;
    map<string, HeaderItem> headerTable;
    int minSupport; // 最小支持度计数

    FPTree(int minSup) : minSupport(minSup)
    {
        root = new FPNode("Root", nullptr);
    }

    ~FPTree() { delete root; }

    // 构建 FPtree
    void build(const vector<vector<string>> &dataset)
    {
        // 第一遍扫描: 统计频率
        map<string, int> frequency;
        for (const auto &trans : dataset)
        {
            for (const string &item : trans)
            {
                frequency[item]++;
            }
        }

        // 过滤并初始化头指针表
        for (auto &pair : frequency)
        {
            if (pair.second >= minSupport)
            {
                headerTable[pair.first].count = pair.second;
            }
        }

        // 第二遍扫描: 排序并插入树
        for (const auto &trans : dataset)
        {
            vector<string> filteredItems;
            for (const string &item : trans)
            {
                if (headerTable.count(item))
                {
                    filteredItems.push_back(item);
                }
            }

            // 按照频率降序排序 (如果频率相同，按字典序)
            sort(filteredItems.begin(), filteredItems.end(),
                 [&](const string &a, const string &b)
                 {
                     if (headerTable[a].count != headerTable[b].count)
                         return headerTable[a].count > headerTable[b].count;
                     return a < b;
                 });

            if (!filteredItems.empty())
            {
                insertTransaction(filteredItems, root);
            }
        }
    }

    // 工具函数，插入一条记录
    void insertTransaction(const vector<string> &items, FPNode *node)
    {
        if (items.empty())
            return;

        string firstItem = items[0];
        FPNode *nextNode = nullptr;

        // 检查孩子是否存在
        if (node->children.find(firstItem) != node->children.end())
        {
            nextNode = node->children[firstItem];
            nextNode->count++;
        }
        else
        {
            // 创建新节点
            nextNode = new FPNode(firstItem, node);
            node->children[firstItem] = nextNode;

            // 更新头指针表的链表 (side-links)
            HeaderItem &header = headerTable[firstItem];
            if (header.headNode == nullptr)
            {
                header.headNode = nextNode;
            }
            else
            {
                header.tailNode->nodeLink = nextNode;
            }
            header.tailNode = nextNode;
        }

        // 递归处理剩下的物品
        vector<string> remainingItems(items.begin() + 1, items.end());
        insertTransaction(remainingItems, nextNode);
    }
```
## 挖掘频繁模式

```c++
	// 挖掘频繁模式
    // prefix: 当前的前缀 (比如已经挖出了 {啤酒})
    // results: 存储最终结果
    void mine(vector<string> prefix, vector<pair<vector<string>, int>> &results)
    {
        // 1. 获取头指针表中所有项，并按频率升序排列 (从底向上挖)
        vector<pair<string, int>> sortedHeaders;
        for (auto &pair : headerTable)
        {
            sortedHeaders.push_back(make_pair(pair.first, pair.second.count));
        }
        sort(sortedHeaders.begin(), sortedHeaders.end(),
             [](const pair<string, int> &a, const pair<string, int> &b)
             {
                 return a.second < b.second;
             });

        // 2. 遍历每一个项 (作为后缀)
        for (const auto &pair : sortedHeaders)
        {
            string item = pair.first;
            int support = pair.second;

            // 生成新的频繁模式: prefix + item
            vector<string> newPattern = prefix;
            newPattern.push_back(item);
            results.push_back({newPattern, support});

            // 3. 构建条件模式基 (Conditional Pattern Base)
            vector<vector<string>> conditionalPatternBase;
            FPNode *curr = headerTable[item].headNode;

            while (curr != nullptr)
            {
                vector<string> path;
                FPNode *parent = curr->parent;
                // 向上回溯直到 Root
                while (parent != nullptr && parent->name != "Root")
                {
                    path.push_back(parent->name);
                    parent = parent->parent;
                }
                // 路径逆序才是从上到下，但在构建条件树时顺序不敏感，只要计数对就行
                // 关键: 这条路径出现的次数 = curr->count
                for (int i = 0; i < curr->count; i++)
                {
                    // 为了简化，这里笨办法重复添加路径。优化做法是带权重的FPTree。
                    // 但为了代码易读，模拟把路径重复加入。
                    // 注意: 标准FP-Growth这里是带计数的，为简化代码逻辑，
                    // 下面构建条件树时直接把路径 reverse 后传进去重新建树。
                    vector<string> pathCopy = path;
                    reverse(pathCopy.begin(), pathCopy.end());
                    conditionalPatternBase.push_back(pathCopy);
                }
                curr = curr->nodeLink;
            }

            // 4. 构建条件 FP-Tree (递归)
            if (!conditionalPatternBase.empty())
            {
                FPTree conditionalTree(minSupport);
                conditionalTree.build(conditionalPatternBase);
                if (!conditionalTree.headerTable.empty())
                {
                    conditionalTree.mine(newPattern, results);
                }
            }
        }
    }
```

## 测试

```c++
// 测试
int main()
{
    // 1. 准备数据 (课件的数据)
    vector<vector<string>> dataset = {
        {"牛奶", "鸡蛋", "面包", "薯片"},
        {"鸡蛋", "爆米花", "薯片", "啤酒"},
        {"鸡蛋", "面包", "薯片"},
        {"牛奶", "鸡蛋", "面包", "爆米花", "薯片", "啤酒"},
        {"牛奶", "面包", "啤酒"},
        {"鸡蛋", "面包", "啤酒"},
        {"牛奶", "面包", "薯片"},
        {"牛奶", "鸡蛋", "面包", "黄油", "薯片"},
        {"牛奶", "鸡蛋", "黄油", "薯片"}};

    int minSupport = 2; // 设定最小支持度 (例如 2 次)

    cout << "========== 正在构建 FP-Tree ==========" << endl;
    cout << "最小支持度计数: " << minSupport << endl;

    FPTree tree(minSupport);
    tree.build(dataset);

    cout << "构建完成，开始挖掘频繁模式..." << endl;

    vector<pair<vector<string>, int>> frequentPatterns;
    vector<string> emptyPrefix;
    tree.mine(emptyPrefix, frequentPatterns);

    // 输出结果
    cout << "========== 挖掘结果 (频繁项集) ==========" << endl;
    for (const auto &p : frequentPatterns)
    {
        cout << "{ ";
        for (const string &s : p.first)
            cout << s << " ";
        cout << "} : " << p.second << endl;
    }

    return 0;
}
```