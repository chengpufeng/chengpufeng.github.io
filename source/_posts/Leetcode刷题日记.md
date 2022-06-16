---
title: Leetcode刷题日记
tags:
  - 刷题
  - Leetcode
date: 2022-06-11 19:16:46
---



Leetcode刷题日记。

不能再摆了！

<!--more-->


# 20220420

## [662. 二叉树最大宽度](https://leetcode-cn.com/problems/maximum-width-of-binary-tree/)

### method1 宽度优先搜索

用unordered_map保存每个节点的pos，如果走左子树pos->2\*pos，如果走右子树pos->2\*pos+1，在同一深度中，更新最大宽度maxWid为R-L+1。需要当心L和R的值会爆炸增长，unsigned long long才装的下，其他值如unsigned long到最后都会溢出。

unsigned long情况下LR输出：

![image-20220420221658157](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206111911597.png)

```c++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int widthOfBinaryTree(TreeNode* root) {
        unsigned long long  maxWid = 0;
        unordered_map<TreeNode*,unsigned long long> node_pos;
        node_pos[root] = 0;
        queue<TreeNode*> qu;
        qu.push(root);
        unsigned long long  L=0,R=0;
        while(!qu.empty())
        {
            int size = qu.size();
            
            bool first = true;
            for(int i = 0 ; i<size ;i++)
            {
                TreeNode* tmp = qu.front();
                qu.pop();
                if(tmp->left)
                {

                    qu.push(tmp->left);
                    node_pos[tmp->left] = node_pos[tmp] * 2;
                    if(first)
                    {
                        R = L = node_pos[tmp->left];
                        first = false;
                    }
                    else{
                        R = node_pos[tmp->left];
                    }
                }
                    
                if(tmp->right)
                {
                    qu.push(tmp->right);
                    node_pos[tmp->right] = node_pos[tmp] * 2 + 1;
                    if(first)
                    {
                        R = L = node_pos[tmp->right];
                        first = false;
                    }
                    else{
                        R = node_pos[tmp->right];
                    }
                }
            
                
            }
            //cout<<L<<" "<<R<<endl;
            maxWid = max(maxWid, R-L+1);
        }
        return maxWid;
    }
};
```

### method2深度优先搜索

用unordered_map储存每层的最左位置，即第一次dfs到depth层时储存，后边每次到这一层更新ans为ans和pos-left+1的最大值，依旧是左子树2\*pos，右子树2\*pos。

```c++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    unsigned long long ans;
    unordered_map<int,unsigned long long> dep_left;
    void dfs(TreeNode* root, int depth,unsigned long long pos)
    {
        if(root == nullptr) return;
        if(dep_left.count(depth))
        {
            ans = max(pos-dep_left[depth]+1, ans);
        }
        else{
            dep_left[depth] = pos;
        }
        
        dfs(root->left, depth + 1,pos * 2);
        dfs(root->right, depth + 1, pos*2 +1);

    }
    int widthOfBinaryTree(TreeNode* root) {
        ans = 1;
        dfs(root,0,0);
        return ans;
    }
};
```

## [388. 文件的最长绝对路径](https://leetcode-cn.com/problems/longest-absolute-file-path/)

遍历

```c++
class Solution {
public:
    int lengthLongestPath(string input) {
        int n = input.size();
        int pos = 0;
        int ans = 0;
        vector<int> level(n);
        while(pos < n)
        {
            //
            int depth = 0;
            while(pos < n && input[pos] == '\t')
            {
                depth++;
                pos++;
            }

            int len = 0;
            bool isFile = false;
            while(pos < n && input[pos] != '\n')
            {
                if(input[pos]=='.')
                    isFile = true;
                pos++;
                len++;
            }
            pos++;//跳过\n
            if(depth > 0)
            {
                len+=level[depth-1] + 1;
            }
            if(isFile)
            {
                ans = max(ans,len);
            }
            else{
                level[depth] = len;
            }
        }
        return ans;
    }
};
```



## [4. 寻找两个正序数组的中位数](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/)

O(n+m)

```c++
class Solution {
public:
    int getKthElement(vector<int>& nums1, vector<int>& nums2,int k)
    {
        int cur1 = 0;
        int cur2 = 0;
        if(nums1.size() == 0) return nums2[k];
        if(nums2.size() == 0) return nums1[k];
        while(cur1<nums1.size() && cur2<nums2.size() && cur1 + cur2 <k)
        {
            if(nums1[cur1] < nums2[cur2])
            {
                cur1++;
            }
            else{
                
                cur2++;
            }
        }
        if(cur1 == nums1.size())
        {
            while(cur1 + cur2 < k)
            {
                cur2++;
            }
            return nums2[cur2];
        }
        if(cur2 == nums2.size())
        {
            while(cur1 + cur2 < k)
            {
                cur1++;
            }
            return nums1[cur1];
        }
        return nums1[cur1] < nums2[cur2] ? nums1[cur1] : nums2[cur2];

    }
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        int size1 = nums1.size();
        int size2 = nums2.size();
        int sum = size1 + size2;
        if(sum % 2)
        {
            return (double) getKthElement(nums1,nums2, sum/2);
        }
        else{
            return ((double) getKthElement(nums1,nums2, sum/2-1) + (double) getKthElement(nums1,nums2, sum/2))/2;
        }
    }
};

```



O(log(n+m))

```c++
class Solution {
public:
    int getKthElement(const vector<int>& nums1, const vector<int>& nums2, int k) {

        //二分法
        int m1 = nums1.size();
        int m2 = nums2.size();
        
        int c1 = 0,c2 = 0;
        while(true)
        {
            //边界情况
            if(c1 == m1) return nums2[c2 + k - 1];
            if(c2 == m2) return nums1[c1 + k - 1];
            if(k == 1)
            {
                return min(nums1[c1],nums2[c2]);
            }

            //
            int nc1 = min(c1 + k/2 - 1, m1 - 1);
            int nc2 = min(c2 + k/2 - 1, m2 - 1);
            if(nums1[nc1] <= nums2[nc2])
            {
                k -= nc1 - c1 + 1;
                c1 = nc1 + 1;
                
            }
            else{
                k -= nc2 - c2 + 1;
                c2 = nc2 + 1;
            }



        }
    }

    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        int totalLength = nums1.size() + nums2.size();
        if (totalLength % 2 == 1) {
            return getKthElement(nums1, nums2, (totalLength + 1) / 2);
        }
        else {
            return (getKthElement(nums1, nums2, totalLength / 2) + getKthElement(nums1, nums2, totalLength / 2 + 1)) / 2.0;
        }
    }
};
```

# 20220422

## 网易游戏1面

```c++
//
/*
由空地和墙组成的迷宫中有一个球。球可以向上下左右四个方向滚动，但在遇到墙壁前不会停止滚动。当球停下时，可以选择下一个方向。
给定球的起始位置，目的地和迷宫，找出让球停在目的地的最短距离。距离的定义是球从起始位置（不包括）到目的地（包括）经过的空地个数。如果球无法停在目的地，返回 -1。
迷宫由一个0和1的二维数组表示。 1表示墙壁，0表示空地。你可以假定迷宫的边缘都是墙壁。起始位置和目的地的坐标通过行号和列号给出。
*/
#include <bits/stdc++.h>
using namespace std;
vector<int> dx{0, 0, 1, -1};
vector<int> dy{1, -1, 0, 0};
int rowS, colS;
int rowD, colD;

int main()
{
    int N;
    while (cin >> N)
    {
        if (N == -1)
            break;

        vector<vector<int>> maze(N, vector<int>(N));
        for (int i = 0; i < N; i++)
        {
            for (int j = 0; j < N; j++)
                cin >> maze[i][j];
        }

        cin >> rowS >> colS;
        cin >> rowD >> colD;
        vector<vector<bool>> visited(N, vector<bool>(N, false));
        queue<pair<int, int>> qu;
        qu.push({rowS, colS});
        int ans = INT_MAX;
        vector<vector<int>> steps(N, vector<int>(N, INT_MAX));
        steps[rowS][colS] = 0;
        bool res = false;
        while (!qu.empty())
        {
            int curX = qu.front().first;
            int curY = qu.front().second;
            qu.pop();

            for (int i = 0; i < 4; i++)
            {
                int nx = curX, ny = curY;
                visited[nx][ny] = true;
                int step = steps[nx][ny];
                while (nx + dx[i] >= 0 && nx + dx[i] < N && ny + dy[i] >= 0 && ny + dy[i] < N && maze[nx + dx[i]][ny + dy[i]] != 1 && !visited[nx + dx[i]][ny + dy[i]])
                {

                    step++;
                    nx = nx + dx[i];
                    ny = ny + dy[i];
                    visited[nx][ny] = true;
                    steps[nx][ny] = min(steps[nx][ny], step);
                }
                if (nx == rowD && ny == colD)
                {
                    ans = min(ans, steps[nx][ny]);
                    visited[nx][ny] = false;
                    res = true;
                }
                else if (nx != curX || ny != curY)
                    qu.push({nx, ny});
            }
        }
        if (res)
            cout << ans << endl;
        else
        {
            cout << -1 << endl;
        }
    }
}
```



## [396. 旋转函数](https://leetcode-cn.com/problems/rotate-function/)

```c++
class Solution {
public:
    int maxRotateFunction(vector<int>& nums) {
        int init = 0;
        int mesh = 0;
        for(int i = 0 ; i<nums.size() ; i++)
        {
            init += i*nums[i];
            mesh += nums[i];
        }
        int ans = init;
        for(int i = 0 ; i<nums.size() ; i++)
        {
            init += mesh;
            init -= nums[nums.size()-1-i]*(nums.size());
            ans = max(init,ans);
        }
        return ans;

        

    }
};
```

## [103. 二叉树的锯齿形层序遍历](https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal/)

```c++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
        vector<TreeNode*> vec;
        if(root == nullptr) return {};
        vec.push_back(root);
        int cnt = 1;
        vector<vector<int>> ans;
        while(!vec.empty())
        {
            int n = vec.size();
            vector<int> tmp;
            vector<TreeNode*> mid;
            if(cnt % 2)
            {
                for(int i = 0 ; i<n ; i++)
                {
                    TreeNode* tmpt = vec.back();
                    vec.pop_back();
                    tmp.push_back(tmpt->val);
                    if(tmpt->left)
                    {
                        mid.push_back(tmpt->left);
                    }
                    if(tmpt->right)
                    {
                        mid.push_back(tmpt->right);
                    }
                }
                cnt++;
                swap(mid,vec);
            }
            else{
                for(int i = 0 ; i<n ; i++)
                {
                    TreeNode* tmpt = vec.back();
                    vec.pop_back();
                    tmp.push_back(tmpt->val);
                    if(tmpt->right)
                    {
                        mid.push_back(tmpt->right);
                    }
                    if(tmpt->left)
                    {
                        mid.push_back(tmpt->left);
                    }
                }
                cnt++;
                swap(mid,vec);
            }
            ans.push_back(tmp);
        }
        return ans;
    }
};
```

双端队列deque

```c++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
        if(root == nullptr) return {};
        deque<TreeNode*> deq;
        deq.push_back(root);
        vector<vector<int>> ans;
        int cnt = 1;

        while(!deq.empty())
        {
            int n = deq.size();
            vector<int> tmpofans;
            if(cnt % 2)
            {
                
                for(int i = 0 ; i<n ; i++)
                {
                    TreeNode* tmp = deq.front();
                    deq.pop_front();
                    tmpofans.push_back(tmp->val);
                    if(tmp->left)
                    {
                        deq.push_back(tmp->left);
                    }
                    if(tmp->right)
                    {
                        deq.push_back(tmp->right);
                    }
                }
                cnt++;
            }
            else{
                for(int i = 0 ; i<n ; i++)
                {
                    TreeNode* tmp = deq.back();
                    deq.pop_back();
                    tmpofans.push_back(tmp->val);
                    if(tmp->right)
                    {
                        deq.push_front(tmp->right);
                    }
                    if(tmp->left)
                    {
                        deq.push_front(tmp->left);
                    }
                }
                cnt++;
            }
            ans.push_back(tmpofans);
        }
        return ans;
    }
};
```

## [16. 最接近的三数之和](https://leetcode-cn.com/problems/3sum-closest/)

```c++
class Solution {
public:
    int threeSumClosest(vector<int>& nums, int target) {
        sort(nums.begin(),nums.end());
        int ans;
        int dev = INT_MAX;
        for(int i = 0 ; i<nums.size()-2;  i++)
        {
            int a = nums[i];
            int cb = i+1;
            int cc = nums.size()-1;
            while(cb < cc)
            {
                int num = nums[cb] + nums[cc] + a;
                if(abs(num - target)<dev)
                {
                    ans = num;
                    dev = abs(num - target);
                }
                if(num < target)
                {
                    cb++;
                }
                else{
                    cc--;
                }
            }
            
        }
        return ans;
    }
};
```





# 20220424

## [LCP 56. 信物传送](https://leetcode-cn.com/problems/6UEx57/)

和leetcode 另一道题是一样的，简单总结一下就是：
变异的最短路径搜索，与最短路径搜索的区别是最短路径搜索每走一步路径长度+1，而本题中，移动方向与传送带方向相同时，移动的路径长度+0 因此是一个 01 BFS
可以使用deque双端队列这种数据结构进行01BFS，因为对于一个节点而言，当出现移动路径长度+0的情况（移动方向与传送带方向相同时），不是将节点加入队列尾，而是直接加入队列头部加入当前遍历队列进行遍历，而对于 移动方向与传送带方向不相同的 情况，将节点加入队列尾，加入下一次进行遍历

下面是leetcode的另一个01BFS的最路径搜索的题目
https://leetcode-cn.com/problems/minimum-cost-to-make-at-least-one-valid-path-in-a-grid/solution/by-s9lssloubq-pvi0/

作者：S9LsSLoUbq
链接：https://leetcode-cn.com/problems/6UEx57/solution/by-s9lssloubq-c6nw/
来源：力扣（LeetCode）
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

```c++
class Solution {
public:
    int ver(char a){
        if(a=='^') return 0;
        else if(a=='v') return 1;
        else if(a=='<') return 2;
        else return 3;
    }
    int conveyorBelt(vector<string>& matrix, vector<int>& start, vector<int>& end) {
        int tmp[4][2]={{-1,0},{1,0},{0,-1},{0,1}};
        int m=matrix.size(),n=matrix[0].size();
        int bx=start[0],by=start[1];
        int ex=end[0],ey=end[1];
        deque<pair<int,int>> que;
        que.emplace_back(bx,by);
        vector<vector<bool>> dist(m,vector<bool>(n,0));
        int count=-1;
        while(!que.empty()){
            int size=que.size();
            ++count;
            for(int i=0;i<size;++i){
                int x=que.front().first,y=que.front().second;
                que.pop_front();
                if(x==ex && y==ey) return count;
                if(dist[x][y]) continue;
                dist[x][y]=1;
                for(int k=0;k<4;++k){
                    int xx=x+tmp[k][0],yy=y+tmp[k][1];
                    if(xx<0 || xx>=m || yy<0 || yy >=n)
                        continue;
                    if(k==ver(matrix[x][y])){
                        que.emplace_front(xx,yy);
                        --i;
                    } 
                    else que.emplace_back(xx,yy);
                }
            }
        }return count;
    }
};

```

## [868. 二进制间距](https://leetcode-cn.com/problems/binary-gap/)

```c++
class Solution {
public:
    int binaryGap(int n) {
        int hold = -1;
        int dev = 0;
        for(int i = 0 ; n>>i != 0 ; i++)
        {
            int bit = (n>>i) & 1;
            if(bit){
                if(hold == -1)
                {
                    hold = i;
                    continue;
                }
                dev = max(dev,i-hold);
                hold = i;
            }                
        }
        return dev;
    }
};
```


# 20220425

## [398. 随机数索引](https://leetcode-cn.com/problems/random-pick-index/)

蓄水池抽样或哈希表unordered_map<int,vector<int>>

```c++
class Solution {
    vector<int> &nums;
public:
    Solution(vector<int> &nums) : nums(nums) {}

    int pick(int target) {
        int ans;
        for (int i = 0, cnt = 0; i < nums.size(); ++i) {
            if (nums[i] == target) {
                ++cnt; // 第 cnt 次遇到 target
                if (rand() % cnt == 0) {
                    ans = i;
                }
            }
        }
        return ans;
    }
};


/**
 * Your Solution object will be instantiated and called as such:
 * Solution* obj = new Solution(nums);
 * int param_1 = obj->pick(target);
 */
```

# 20220429

## [427. 建立四叉树](https://leetcode.cn/problems/construct-quad-tree/)

```c++
#include <bits/stdc++.h>
using namespace std;

// Definition for a QuadTree node.
class Node
{
public:
    bool val;
    bool isLeaf;
    Node *topLeft;
    Node *topRight;
    Node *bottomLeft;
    Node *bottomRight;

    Node()
    {
        val = false;
        isLeaf = false;
        topLeft = NULL;
        topRight = NULL;
        bottomLeft = NULL;
        bottomRight = NULL;
    }

    Node(bool _val, bool _isLeaf)
    {
        val = _val;
        isLeaf = _isLeaf;
        topLeft = NULL;
        topRight = NULL;
        bottomLeft = NULL;
        bottomRight = NULL;
    }

    Node(bool _val, bool _isLeaf, Node *_topLeft, Node *_topRight, Node *_bottomLeft, Node *_bottomRight)
    {
        val = _val;
        isLeaf = _isLeaf;
        topLeft = _topLeft;
        topRight = _topRight;
        bottomLeft = _bottomLeft;
        bottomRight = _bottomRight;
    }
};

class Solution
{
public:
    bool leaf(vector<vector<int>> &grid, int rowS, int rowE, int colS, int colE)
    {
        int val = grid[rowS][colS];
        for (int i = rowS; i < rowE; i++)
        {
            for (int j = colS; j < colE; j++)
            {
                if (grid[i][j] != val)
                {
                    return false;
                }
            }
        }
        return true;
    }
    Node *constructSub(vector<vector<int>> &grid, int rowS, int rowE, int colS, int colE)
    {

        int tlen = rowE - rowS;

        if (tlen == 0)
        {
            return nullptr;
        }
        Node *cur = new Node();
        if (leaf(grid, rowS, rowE, colS, colE))
        {
            cur->val = grid[rowS][colS];
            cur->isLeaf = true;
        }
        else
        {
            int dev = tlen / 2;
            cur->isLeaf = false;
            cur->val = true; // random
            cur->topLeft = constructSub(grid, rowS, rowS + dev, colS, colS + dev);
            cur->topRight = constructSub(grid, rowS, rowS + dev, colS + dev, colE);
            cur->bottomLeft = constructSub(grid, rowS + dev, rowE, colS, colS + dev);
            cur->bottomRight = constructSub(grid, rowS + dev, rowE, colS + dev, colE);
        }
        return cur;
    }
    Node *construct(vector<vector<int>> &grid)
    {
        return constructSub(grid, 0, grid.size(), 0, grid[0].size());
    }
};
```

# 20220502

## [591. 标签验证器](https://leetcode.cn/problems/tag-validator/)

```c++
class Solution {
public:
    const string cdata_pre = "![CDATA[";
    const string cdata_end = "]]>";
    bool isValid(string code) {
        stack<string> stk;
        int cur = 0;
        int end = code.size();
        bool res = true;
        while(cur < end)
        {
            if(code[cur] == '<')
            {
                if(cur == end-1)
                {
                    return false;
                }
                if(code[cur + 1]=='!')
                {
                    if(stk.empty()) return false;
                    string iscdata_pre = code.substr(cur+1,8);
                    if(cdata_pre == iscdata_pre)
                    {
                        int tail = code.find(cdata_end,cur);
                        if(tail == string::npos) return false;
                        cur = tail + 1;
                    }
                    else{
                        return false;
                    }
                }
                else if(code[cur + 1]== '/')
                {
                    if(stk.empty()) return false;
                    int tail = code.find('>',cur);
                    if(tail == string::npos) return false;
                    string send = code.substr(cur+2, tail-(cur + 2));
                    if(stk.top() != send)
                    {
                        return false;
                    }
                    stk.pop();
                    
                    cur = tail+1;
                    if(stk.empty() && cur != end)
                    {
                        return false;
                    }
                }
                else if(cur < end)
                {
                    int nextl = code.find('>',cur);
                    if(nextl == string::npos) return false;
                    string sbeg = code.substr(cur+1,nextl-(cur+1));
                    if(sbeg.size()<=0 || sbeg.size()>9) return false;
                    if (!all_of(sbeg.begin(), sbeg.end(), [](unsigned char c) { return isupper(c); })) 
                    {
                        return false;
                    }
                    stk.push(move(sbeg));
                    cur = nextl+1;
                }
            }
            else{
                if(stk.empty())
                {
                    return false;
                }
                cur++;
            }
        }
        return stk.empty();
    }
};
```

![微信图片_20220504093015](https://raw.githubusercontent.com/bevancheng/imgrepo/main/202206111912290.jpg)

# 20220504

##  [1823. 找出游戏的获胜者](https://leetcode-cn.com/problems/find-the-winner-of-the-circular-game/)

```c++
class Solution {
public:
    int fun(int n, int k)
    {
        if(n == 1)
        {
            return n;
        }
        int x = fun(n-1,k);
        return (k+x-1)%n+1;
    }
    int findTheWinner(int n, int k) {
        return fun(n,k);
    }
};
```

## [390. 消除游戏](https://leetcode-cn.com/problems/elimination-game/)

等差数列

$step_{k+1} = step_k * 2$

$cnt_{k+1} = floor[cnt_k/2]$

k=0开始，k为偶数，从左往右删，k为奇数，从右往左删

k even：

​		$cnt_k$ even，$a_1^{k+1} = a_1^{k} + step_k,\quad a_n^{k+1} = a_n^{k}-step_k$

​		$cnt_k$ odd，$a_1^{k+1} = a_1^{k} + step_k,\quad a _n^{k+1} = a_n^{k}$

k odd:

​		$cnt_k$ even，$a_1^{k+1} = a_1^{k},\quad a_n^{k+1} = a_n^{k}-step_k$

​		$cnt_k$ odd，$a_1^{k+1} = a_1^{k} + step_k,\quad a_n^{k+1} = a_n^{k}-step_k$

```c++
class Solution {
public:
    int lastRemaining(int n) {
        int cnt = n;
        int step = 1;
        int a1 = 1;
        int k = 0;

        while(cnt>1)
        {
            if(k % 2 == 0)
            {
                a1 = a1 + step;
            }
            else{
                if(cnt % 2)
                {
                    a1 = a1 + step;
                }

            }
            cnt /=2;
            step*=2;
            k++;
        }
        return a1;
    }
};
```

约瑟夫环思想

定义$f[i]$为连续序列$[1,i]$中进行起始从左往右,再从右往左的轮流换向删除后,最终左边剩余的编号(实际上就是最后剩余的编号);

定义$f^\prime[i]$为在连续序列$[1,i]$中进行起始右往左,再左往右的轮流换向删除,最终右边剩余的编号(实际上就是最后剩余的编号).

删除过程有对称性,剩余编号再连续序列中也有对称性.

即$f[i] + f^\prime[i] = i+1$

如果起始对$[1,2,3,...,i]$ 进行从左往右消除,得到序列$[2,4,6,...,x]$

新序列长度$\lfloor i/2 \rfloor$

重新编号,继续保持连续序列的定义,$[1,2,3,...,\lfloor i/2 \rfloor]$

然后进行从右往左间隔删,得到$f^\prime[\lfloor i/2\rfloor]$

则$f[i] = f^\prime[\lfloor i/2 \rfloor]*2$

从两个公式,可得递推公式

$f[i] = 2* (\lfloor i/2\rfloor + 1 - f[\lfloor i/2 \rfloor])$

出口$f[1] = 1$

```c++
class Solution {
public:
    int lastRemaining(int n) {
        if(n == 1) return 1;
        return 2*(n/2 + 1 - lastRemaining(n/2));
    }
};
```

## [354. 俄罗斯套娃信封问题](https://leetcode-cn.com/problems/russian-doll-envelopes/)

不能用O(n^2)的动态规划，会超时。

用贪心+二分O(nlogn)

关键是排序过程中，按宽度升序，宽度相等时按高度降序。之后只需要按高度插入就行了，如[[1,1],[2,3],[4,6],[4,5],[4,4],[4,3],[4,2],[6,6]]是排序之后的结果，先插入了[4,6]，之后就不会再插入[4,5]了，当遍历到[4,5]时，$envelopes[i][1]<dvec.back()$，进入else二分，dvec={1,3,6}，找到第一个大于5的值6，替换为{1,3,5}，之后遇到[4,2]，替换3，但替不替换都不会影响[6,6]的插入。（但会引起排序的混乱，宽度信息无法保存，但是是正确的），只有数组中最后一个数才有意义，这决定了是否插入新的数据，而数组的长度才是我们要得到的。

```c++
class Solution {
public:
    int maxEnvelopes(vector<vector<int>>& envelopes) {
        sort(envelopes.begin(),envelopes.end(),[](auto& a, auto& b){
            if(a[0] == b[0])
            {
                return a[1] > b[1];//////////////关键
            }
            return a[0] < b[0];
        });
        vector<int> dvec;
        dvec.push_back(envelopes[0][1]);
        for(int i = 1 ; i<envelopes.size() ; i++)
        {
            if(envelopes[i][1]>dvec.back())
            {
                dvec.push_back(envelopes[i][1]);
            }
            else{
                int l =0, r= dvec.size()-1;
                while(l<=r)
                {
                    int mid = l + (r-l)/2;
                    if(envelopes[i][1] > dvec[mid])
                    {
                        l = mid + 1;
                    }
                    else{
                        r = mid-1;
                    }
                }
                dvec[l] = envelopes[i][1];
            }
        }
        
        return dvec.size();
    }
};
```

# 20220505

## 每日 [713. 乘积小于 K 的子数组](https://leetcode-cn.com/problems/subarray-product-less-than-k/)

滑动窗口

```c++
class Solution {
public:
    int numSubarrayProductLessThanK(vector<int>& nums, int k) {
        if(k <=1){
            return 0;
        }
        int product = 1;
        int ans = 0;
        //right所指的数必须为连续子数组的最后一个数，每一次ans加上right为最后数的子数组数目，总体O(n)
        //如果product大于等于k了，就往前推left，即product/=nums[left]
        for(int left = 0,right = 0; left<nums.size() && right<nums.size() ; right++)
        {
            product*=nums[right];
            while(product >= k)
            {
                product/=nums[left];
                left++;
            }
            ans+=right-left+1;

        }
        return ans;
    }
};
```

需要搞通背包问题了

字节3题背包，4题删除连续子数组获得最长连续递增数组

# 20220508

## [442. 数组中重复的数据](https://leetcode.cn/problems/find-all-duplicates-in-an-array/)

重点是避免冲突，以及循环中需要让

```c++
#include <bits/stdc++.h>
using namespace std;
class Solution
{
public:
    void printVec(vector<int> &nums)
    {
        for (int c = 0; c < nums.size(); c++)
        {
            cout << nums[c] << " ";
        }
        cout << endl;
    }
    vector<int> findDuplicates(vector<int> &nums)
    {
        vector<int> res;
        //printVec(nums);
        //printf("\n");
        for (int i = 0; i < nums.size(); i++)
        {
            while (nums[nums[i] - 1] != nums[i])
            {
                swap(nums[i], nums[nums[i] - 1]);
                //cout<<i<<" ";
                //printVec(nums);
            }
        }
        for (int i = 0; i < nums.size(); i++)
        {
            if (nums[i] != i + 1)
            {
                res.push_back(nums[i]);
            }
        }
        return res;
    }
};

int main()
{
    vector<int> nums{4, 3, 2, 7, 8, 2, 3, 1};
    Solution sol;
    sol.findDuplicates(nums);
}
```



# 20220611

#### [926. 将字符串翻转到单调递增](https://leetcode.cn/problems/flip-string-to-monotone-increasing/)

如果一个二进制字符串，是以一些 0（可能没有 0）后面跟着一些 1（也可能没有 1）的形式组成的，那么该字符串是 单调递增 的。

给你一个二进制字符串 s，你可以将任何 0 翻转为 1 或者将 1 翻转为 0 。

返回使 s 单调递增的最小翻转次数。

解：

动态规划，`dp[i][0]`存储第i个字符为0时前i字符串为递增所需的最小反转次数，`dp[i][1]`为变为1时的情况。

边缘条件`dp[-1][0] = dp[-1][1] = 0`，

对于第i个字符是0，它的前边必须是0；对于第i字符是1，前边可以是0也可以是1，因此

状态转移方程

```
dp[i][0] = dp[i-1][0] + II(s[i]=='1') //当前字符是1时必须变为0
dp[i][1] = min(dp[i-1][0],dp[i-1][1]) + II(s[i] == '0') //选择前缀字符串中最小的反转次数，以此为基础进行状态转移
```



```c++
class Solution {
public:
    int minFlipsMonoIncr(string &s) {
        vector<int> dp0(s.size()+1,0);
        vector<int> dp1(s.size()+1,0);
        for(int i = 1 ; i<=s.size() ; i++){
            dp0[i] = dp0[i-1] + (s[i-1] == '1'? 1:0);
            dp1[i] = min(dp0[i-1],dp1[i-1]) + (s[i-1] =='0'?1 :0);
        }
        return min(dp0.back(),dp1.back());
    }
};
```

状态只依赖于前一个值，所以可以将空间复杂度从O(n)降到O(1)

```c++
class Solution {
public:
    int minFlipsMonoIncr(string &s) {
        int a0 = 0;
        int a1 = 0;
        for(char c : s){
            int n0 = a0;
            int n1 = min(a0,a1);
            if(c == '0'){
                n1++;
            }else{
                n1++;
            }
            a0 = n0;
            a1 = n1;
        }
        return min(a0,a1);
    }
};

```

# tips

数据结构
单调栈 monotone_stack.go
单调队列 monotone_queue.go
二维单调队列
双端队列 deque.go
堆（优先队列）heap.go
支持修改、删除指定元素
并查集 union_find.go
点权
边权（种类）
持久化
回滚操作（动态图连通性）
稀疏表（ST 表）sparse_table.go
树状数组 fenwick_tree.go
线段树 segment_tree.go
延迟标记（懒标记）
动态开点
线段树合并
线段树分裂
持久化（主席树）
左偏树（可并堆）leftist_tree.go
笛卡尔树 cartesian_tree.go
二叉搜索树公共方法 bst.go
Treap treap.go
伸展树 splay.go
动态树 LCT link_cut_tree.go
红黑树 red_black_tree.go
替罪羊树 scapegoat_tree.go
k-d 树 kd_tree.go
珂朵莉树（ODT）
数组版 odt.go
平衡树版 odt_bst.go
根号算法（分块）sqrt_decomposition.go
莫队算法 mo.go
普通莫队
带修莫队
回滚莫队
树上莫队
字符串 strings.go
字符串哈希
KMP
最小循环节
扩展 KMP（Z algorithm）
最小表示法
最长回文子串
Manacher 算法
后缀数组（SA）
字典树 trie.go
持久化
AC 自动机
异或字典树 trie01.go
持久化
Hack：构造一组数据，最大化树上的节点数
数学
数论 math.go
最大公因数（GCD）
类欧几里得算法 ∑⌊(ai+b)/m⌋
Pollard-Rho 质因数分解算法
埃氏筛
线性筛
欧拉函数
原根
扩展 GCD
二元一次不定方程
逆元
线性求逆元
中国剩余定理（CRT）
扩展中国剩余定理
离散对数
大步小步算法（BSGS）
扩展大步小步算法
二次剩余
Jacobi 符号
组合数学
卢卡斯定理
卡特兰数
默慈金数
那罗延数
斯特林数
第一类斯特林数（轮换）
第二类斯特林数（子集）
贝尔数
莫比乌斯函数
数论分块
杜教筛
容斥原理
快速傅里叶变换 FFT math_fft.go
快速数论变换 NTT math_ntt.go
包含多项式全家桶（求逆、开方等等）
快速沃尔什变换 FWT math_fwt.go
连分数、佩尔方程 math_continued_fraction.go
线性代数 math_matrix.go
矩阵相关运算
高斯消元
行列式
线性基
数值分析 math_numerical_analysis.go
自适应辛普森积分
拉格朗日插值
计算几何 geometry.go
线与点
线与线
圆与点
最小圆覆盖
随机增量法
固定半径覆盖最多点
圆与线
圆与圆
圆与矩形
最近点对
多边形与点
凸包
最远点对
旋转卡壳
半平面交
博弈论 games.go
SG 函数
动态规划 dp.go
背包
0-1 背包
完全背包
多重背包
分组背包
树上背包（依赖背包）
字典序最小方案
线性 DP
最大子段和
LCS
LPS
LIS
狄尔沃斯定理
LCIS
长度为 m 的 LIS 个数
本质不同子序列个数
区间 DP
环形 DP
状压 DP
旅行商问题（TSP）
高维前缀和（SOS DP）
插头 DP
数位 DP
倍增优化 DP
斜率优化 DP（CHT）
树形 DP
树的直径个数
在任一直径上的节点个数
树上最大独立集
树上最小顶点覆盖
树上最小支配集
树上最大匹配
换根 DP（二次扫描法）
图论 graph.go
链式前向星
欧拉回路和欧拉路径
无向图
有向图
割点
割边（桥）
双连通分量（BCC）
v-BCC
e-BCC
最短路
Dijkstra
SPFA（队列优化的 Bellman-Ford）
差分约束系统
Floyd-Warshall
Johnson
0-1 BFS（双端队列 BFS）
字典序最小最短路
最小环
最小生成树（MST）
Kruskal
Prim
单度限制最小生成树
次小生成树
曼哈顿距离最小生成树
最小差值生成树
最小树形图
朱刘算法
二分图判定（染色）
二分图找奇环
二分图最大匹配
匈牙利算法
带权二分图最大完美匹配
Kuhn–Munkres 算法
拓扑排序
强连通分量（SCC）
Kosaraju
Tarjan
2-SAT
基环树
最大流
Dinic
ISAP
HLPP
最小费用最大流
SPFA
Dijkstra
三元环计数
四元环计数
仙人掌
树上问题 graph_tree.go
直径
重心
点分治
最近公共祖先（LCA）
倍增
ST 表
Tarjan
树上差分
树链剖分（重链剖分，HLD）
长链剖分
树上启发式合并（DSU）
树分块
Prufer 序列
其他
位运算笔记 bits.go
bitset
区间位运算 trick（含 GCD）
二分 三分 sort.go
0-1 分数规划
整体二分
搜索 search.go
枚举排列
枚举组合
生成下一个排列
康托展开
逆康托展开
折半枚举
超大背包问题
枚举子集
Gosper’s Hack
随机算法 rand.go
模拟退火
杂项A common.go
算法思路整理
前缀和
二维前缀和
二维差分
离散化
