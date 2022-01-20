### 70、爬楼梯

+ 解法一： 动态规划
  ```C++
  class Solution {
  public:
    int climbStairs(int n) {
        int p = 0, q = 0, r = 1;
        for (int i = 1; i <= n; ++i) {
            p = q; 
            q = r; 
            r = p + q;
        }
        return r;
    }
  };
  ```
+ 解法二：
  ```C++
  int climbStairs(int n) {
    int a = 1, b = 1;
    while (n--)
        a = (b += a) - a;
    return a;
  }
  ```

### 22、括号生成

+ 解法一
```C++
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string> res;
        addingpar(res, "", n, 0);
        return res;
    }
    void addingpar(vector<string> &v, string str, int n, int m){
        if(n==0 && m==0) {
            v.push_back(str);
            return;
        }
        if(m > 0){ addingpar(v, str+")", n, m-1); }
        if(n > 0){ addingpar(v, str+"(", n-1, m+1); }
    }
};
```

+ 解法二（此解法的时间复杂度较低）
```C++
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string> ret;
        helper(ret, "", n, n);
        return ret;
    }
    void helper(vector<string> & res, string str, int left, int right){
        if(left == 0 && right == 0){
            res.push_back(str);
            return;
        }
        if(left > 0) helper(res, str + "(", left - 1, right);  
        if(right > left) helper(res, str + ")", left, right - 1);
    }
};
```
> 1. 上述两种解法的要点是左括号数必须与右括号数匹配
> 2. 还有就是递归函数的套路

+ 解法三
```C++
class Solution {
public:
    vector<string>result;
    
    vector<string> generateParenthesis(int n) {
        helper(0,0,n,"");
        return result;
    }

    void helper(int open,int close,int n,string current)
    {
        if(current.length()==n*2)
        {
            result.push_back(current);
            return;
        }
        if(open<n)  helper(open+1,close,n,current+"(");
        if(close<open)  helper(open,close+1,n,current+")");
    }
};
```

+ 方法四：回溯

```C++
class Solution {
    void backtrack(vector<string>& ans, string& cur, int open, int close, int n) {
        if (cur.size() == n * 2) {
            ans.push_back(cur);
            return;
        }
        if (open < n) {
            cur.push_back('(');
            backtrack(ans, cur, open + 1, close, n);
            cur.pop_back();
        }
        if (close < open) {
            cur.push_back(')');
            backtrack(ans, cur, open, close + 1, n);
            cur.pop_back();
        }
    }
public:
    vector<string> generateParenthesis(int n) {
        vector<string> result;
        string current;
        backtrack(result, current, 0, 0, n);
        return result;
    }
};
```

### 98、验证二叉搜索树

+ 解法一： 递归
```C++
class Solution {
public:
    bool helper(TreeNode* root, long long lower, long long upper) {
        if (root == nullptr) {
            return true;
        }
        if (root -> val <= lower || root -> val >= upper) {
            return false;
        }
        return helper(root -> left, lower, root -> val) && helper(root -> right, root -> val, upper);
    }
    bool isValidBST(TreeNode* root) {
        return helper(root, LONG_MIN, LONG_MAX);
    }
};
```
+ 解法二
```C++
bool isValidBST(TreeNode* root) {
    return isValidBST(root, NULL, NULL);
}

bool isValidBST(TreeNode* root, TreeNode* minNode, TreeNode* maxNode) {
    if(!root) return true;
    if(minNode && root->val <= minNode->val || maxNode && root->val >= maxNode->val)
        return false;
    return isValidBST(root->left, minNode, root) && isValidBST(root->right, root, maxNode);
}
```

### 104 二叉树的最大深度

+ 解法一 : 深度优先搜索
```C++
class Solution {
public:
    int maxDepth(TreeNode* root) {
        if (root == nullptr) return 0;
        return max(maxDepth(root->left), maxDepth(root->right)) + 1;
    }
};
```

```C++
int maxDepth(TreeNode *root)
{
    return root == NULL ? 0 : max(maxDepth(root -> left), maxDepth(root -> right)) + 1;
}
```

+ 解法二：广度优先搜索
```C++
int maxDepth(TreeNode *root)
{
    if(root == NULL)
        return 0;
    
    int res = 0;
    queue<TreeNode *> q;
    q.push(root);
    while(!q.empty())
    {
        ++ res;
        for(int i = 0, n = q.size(); i < n; ++ i)
        {
            TreeNode *p = q.front();
            q.pop();
            
            if(p -> left != NULL)
                q.push(p -> left);
            if(p -> right != NULL)
                q.push(p -> right);
        }
    }
    
    return res;
}
```

### 111、二叉树的最小深度

+ 解法一：深度优先搜索
```C++
class Solution {
public:
    int minDepth(TreeNode *root) {
        if (root == nullptr) {
            return 0;
        }

        if (root->left == nullptr && root->right == nullptr) {
            return 1;
        }

        int min_depth = INT_MAX;
        if (root->left != nullptr) {
            min_depth = min(minDepth(root->left), min_depth);
        }
        if (root->right != nullptr) {
            min_depth = min(minDepth(root->right), min_depth);
        }

        return min_depth + 1;
    }
};
```

+ 解法二：广度优先搜索
```C++
class Solution {
public:
    int minDepth(TreeNode *root) {
        if (root == nullptr) {
            return 0;
        }

        queue<pair<TreeNode *, int> > que;
        que.emplace(root, 1);
        while (!que.empty()) {
            TreeNode *node = que.front().first;
            int depth = que.front().second;
            que.pop();
            if (node->left == nullptr && node->right == nullptr) {
                return depth;
            }
            if (node->left != nullptr) {
                que.emplace(node->left, depth + 1);
            }
            if (node->right != nullptr) {
                que.emplace(node->right, depth + 1);
            }
        }

        return 0;
    }
};
```
```C++
// 广度优先搜索
// 时间复杂度较低
int minDepth(TreeNode* root) {
    if (root == NULL) return 0;
    queue<TreeNode*> Q;
    Q.push(root);
    int i = 0;
    while (!Q.empty()) {
        i++;
        int k = Q.size();
        for (int j=0; j<k; j++) {
            TreeNode* rt = Q.front();
            if (rt->left) Q.push(rt->left);
            if (rt->right) Q.push(rt->right);
            Q.pop();
            if (rt->left==NULL && rt->right==NULL) return i;
        }
    }
    return -1; 
}
```

```C++
int minDepth(TreeNode* root) {
    if (!root) return 0;
    int L = minDepth(root->left), R = minDepth(root->right);
    return 1 + (min(L, R) ? min(L, R) : max(L, R));
}
```

### 235、二叉搜索树的最近公共祖先
+ 解法一：两次遍历
```C++
class Solution {
public:
    vector<TreeNode*> getPath(TreeNode* root, TreeNode* target) {
        vector<TreeNode*> path;
        TreeNode* node = root;
        while (node != target) {
            path.push_back(node);
            if (target->val < node->val) {
                node = node->left;
            }
            else {
                node = node->right;
            }
        }
        path.push_back(node);
        return path;
    }

    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        vector<TreeNode*> path_p = getPath(root, p);
        vector<TreeNode*> path_q = getPath(root, q);
        TreeNode* ancestor;
        for (int i = 0; i < path_p.size() && i < path_q.size(); ++i) {
            if (path_p[i] == path_q[i]) {
                ancestor = path_p[i];
            }
            else {
                break;
            }
        }
        return ancestor;
    }
};
```
+ 方法二：一次遍历
```C++
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        TreeNode* ancestor = root;
        while (true) {
            if (p->val < ancestor->val && q->val < ancestor->val) {
                ancestor = ancestor->left;
            }
            else if (p->val > ancestor->val && q->val > ancestor->val) {
                ancestor = ancestor->right;
            }
            else {
                break;
            }
        }
        return ancestor;
    }
};
```
--------------------------------------
```C++
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if ((root -> val > p -> val) && (root -> val > q -> val)) {
            return lowestCommonAncestor(root -> left, p, q);
        }
        if ((root -> val < p -> val) && (root -> val < q -> val)) {
            return lowestCommonAncestor(root -> right, p, q);
        }
        return root;
    }
};
```
+ 方法三
```C++
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    while ((root->val - p->val) * (root->val - q->val) > 0)
        root = p->val < root->val ? root->left : root->right;
    return root;
}
```
### 50、pow(x, n)
+ 解法一：快速幂+递归
```C++
class Solution {
public:
    double quickMul(double x, long long N) {
        if (N == 0) {
            return 1.0;
        }
        double y = quickMul(x, N / 2);
        return N % 2 == 0 ? y * y : y * y * x;
    }

    double myPow(double x, int n) {
        long long N = n;
        return N >= 0 ? quickMul(x, N) : 1.0 / quickMul(x, -N);
    }
};
```
---------------------------------------
+ 方法二：快速幂 + 迭代
```C++
class Solution {
public:
    double quickMul(double x, long long N) {
        double ans = 1.0;
        // 贡献的初始值为 x
        double x_contribute = x;
        // 在对 N 进行二进制拆分的同时计算答案
        while (N > 0) {
            if (N % 2 == 1) {
                // 如果 N 二进制表示的最低位为 1，那么需要计入贡献
                ans *= x_contribute;
            }
            // 将贡献不断地平方
            x_contribute *= x_contribute;
            // 舍弃 N 二进制表示的最低位，这样我们每次只要判断最低位即可
            N /= 2;
        }
        return ans;
    }

    double myPow(double x, int n) {
        long long N = n;
        return N >= 0 ? quickMul(x, N) : 1.0 / quickMul(x, -N);
    }
};
```
+ 解法三：
```C++
public class Solution {
    public double pow(double x, int n) {
        long long N = n;

        if (N == 0)
            return 1.0;
        
        if (N < 0) {
            N = -N;
            x = 1/x;
        }

        return (N % 2 == 0) ? pow (x * x, N / 2) : x * pow (x * x, N / 2);
    }
}
```
+ 解法四
```C++
double pow(double x, int n) {
    if (n==0) 
        return 1;
    double t = pow(x,n/2);
    
    if (n%2) {
        return n < 0 ? 1 / x * t * t : x * t * t;
    } else {
        return t * t;
    }
}
```
-------------------------------

### 169、多数元素

+ 方法一：哈希表

```C++
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        unordered_map<int, int> counts;
        int majority = 0, cnt = 0;
        for (int num: nums) {
            ++counts[num];
            if (counts[num] > cnt) {
                majority = num;
                cnt = counts[num];
            }
        }
        return majority;
    }
};

// 哈希
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        unordered_map<int, int> counter;
        for (int num : nums) {
            if (++counter[num] > nums.size() / 2) {
                return num;
            }
        }
        return 0;
    }
};
```

+ 方法二：分治

```C++
class Solution {
    int count_in_range(vector<int>& nums, int target, int lo, int hi) {
        int count = 0;
        for (int i = lo; i <= hi; ++i)
            if (nums[i] == target)
                ++count;
        return count;
    }
    int majority_element_rec(vector<int>& nums, int lo, int hi) {
        if (lo == hi)
            return nums[lo];
        int mid = (lo + hi) / 2;
        int left_majority = majority_element_rec(nums, lo, mid);
        int right_majority = majority_element_rec(nums, mid + 1, hi);
        if (count_in_range(nums, left_majority, lo, hi) > (hi - lo + 1) / 2)
            return left_majority;
        if (count_in_range(nums, right_majority, lo, hi) > (hi - lo + 1) / 2)
            return right_majority;
        return -1;
    }
public:
    int majorityElement(vector<int>& nums) {
        return majority_element_rec(nums, 0, nums.size() - 1);
    }
};

// 分治
class Solution {
public:
    int majorityElement(vector<int>& nums) {
        return majority(nums, 0, nums.size() - 1);
    }
private:
    int majority(vector<int>& nums, int l, int r) {
        if (l == r) {
            return nums[l];
        }
        int m = l + (r - l) / 2, lm = majority(nums, l, m), rm = majority(nums, m + 1, r);
        if (lm == rm) {
            return lm;
        }
        return count(nums.begin() + l, nums.begin() + r + 1, lm) > count(nums.begin() + l, nums.begin() + r + 1, rm) ? lm : rm;
    }
}; 
```

### 17、电话号码的字母组合

+ 解法一：回溯

```C++
class Solution {
public:
    vector<string> letterCombinations(string digits) {
        vector<string> combinations;
        if (digits.empty()) {
            return combinations;
        }
        unordered_map<char, string> phoneMap{
            {'2', "abc"},
            {'3', "def"},
            {'4', "ghi"},
            {'5', "jkl"},
            {'6', "mno"},
            {'7', "pqrs"},
            {'8', "tuv"},
            {'9', "wxyz"}
        };
        string combination;
        backtrack(combinations, phoneMap, digits, 0, combination);
        return combinations;
    }

    void backtrack(vector<string>& combinations, const unordered_map<char, string>& phoneMap, const string& digits, int index, string& combination) {
        if (index == digits.length()) {
            combinations.push_back(combination);
        } else {
            char digit = digits[index];
            const string& letters = phoneMap.at(digit);
            for (const char& letter: letters) {
                combination.push_back(letter);
                backtrack(combinations, phoneMap, digits, index + 1, combination);
                combination.pop_back();
            }
        }
    }
};
```
-------------------------
+ 解法二

```C++
class Solution {
public:
    const vector<string> pad = {
        "", "", "abc", "def", "ghi", "jkl",
        "mno", "pqrs", "tuv", "wxyz"
    };
    
    vector<string> letterCombinations(string digits) {
        if (digits.empty()) return {};
        vector<string> result;
        result.push_back("");
        
        for(auto digit: digits) {
            vector<string> tmp;
            for(auto candidate: pad[digit - '0']) {
                for(auto s: result) {
                    tmp.push_back(s + candidate);
                }
            }
            result.swap(tmp);
        }
        return result;
    }
};
```
### 8、八皇后
+ 解法一：
```C++
class Solution {
public:
    vector<vector<string>> solveNQueens(int n) {
        auto solutions = vector<vector<string>>();
        auto queens = vector<int>(n, -1);
        auto columns = unordered_set<int>();
        auto diagonals1 = unordered_set<int>();
        auto diagonals2 = unordered_set<int>();
        backtrack(solutions, queens, n, 0, columns, diagonals1, diagonals2);
        return solutions;
    }

    void backtrack(vector<vector<string>> &solutions, vector<int> &queens, int n, int row, unordered_set<int> &columns, unordered_set<int> &diagonals1, unordered_set<int> &diagonals2) {
        if (row == n) {
            vector<string> board = generateBoard(queens, n);
            solutions.push_back(board);
        } else {
            for (int i = 0; i < n; i++) {
                if (columns.find(i) != columns.end()) {
                    continue;
                }
                int diagonal1 = row - i;
                if (diagonals1.find(diagonal1) != diagonals1.end()) {
                    continue;
                }
                int diagonal2 = row + i;
                if (diagonals2.find(diagonal2) != diagonals2.end()) {
                    continue;
                }
                queens[row] = i;
                columns.insert(i);
                diagonals1.insert(diagonal1);
                diagonals2.insert(diagonal2);
                backtrack(solutions, queens, n, row + 1, columns, diagonals1, diagonals2);
                queens[row] = -1;
                columns.erase(i);
                diagonals1.erase(diagonal1);
                diagonals2.erase(diagonal2);
            }
        }
    }

    vector<string> generateBoard(vector<int> &queens, int n) {
        auto board = vector<string>();
        for (int i = 0; i < n; i++) {
            string row = string(n, '.');
            row[queens[i]] = 'Q';
            board.push_back(row);
        }
        return board;
    }
};
```

### 102、二叉树的层序遍历

+ 方法一：广度优先搜索

```C++
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector <vector <int>> ret;
        if (!root) {
            return ret;
        }

        queue <TreeNode*> q;
        q.push(root);
        while (!q.empty()) {
            int currentLevelSize = q.size();
            ret.push_back(vector <int> ());
            for (int i = 1; i <= currentLevelSize; ++i) {
                auto node = q.front(); q.pop();
                ret.back().push_back(node->val);
                if (node->left) q.push(node->left);
                if (node->right) q.push(node->right);
            }
        }
        
        return ret;
    }
};
```
------------------------
+ 解法二

```C++
vector<vector<int>> ret;

void buildVector(TreeNode *root, int depth)
{
    if(root == NULL) return;
    if(ret.size() == depth)
        ret.push_back(vector<int>());
    
    ret[depth].push_back(root->val);
    buildVector(root->left, depth + 1);
    buildVector(root->right, depth + 1);
}

vector<vector<int> > levelOrder(TreeNode *root) {
    buildVector(root, 0);
    return ret;
}
```