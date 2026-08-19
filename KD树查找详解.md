# KD树查找原理
给定一个构建于一个样本集的$kd$树，下面的算法可以寻找距离某个点 $p$最近的$k$个样本。
1. 设 $L$ 为一个有$k$ 个空位的列表，用于保存已搜寻到的最近点。  
2. 根据$p$的坐标值和每个节点的切分向下搜索（也就是说，如果树的节点是按照$x_r = a$进行切分，并且$p$的$r$坐标小于$a$，则向左枝进行搜索；反之则走右枝）。  
3. 当达到一个底部节点时，将其标记为访问过。如果$L$里不足$k$个点，则将当前节点的特征坐标加入$L$；如果$L$不为空并且当前节点的特征与$p$的距离小于$L$里最长的距离，则用当前特征替换掉 $L$中离$p$最远的点。  
4. 如果当前节点不是整棵树最顶端节点，执行(2-1)；反之，输出$L$，算法完成。  
	1. 向上爬一个节点。如果当前（向上爬之后的）节点未曾被访问过，将其标记为被访问过，然后执行(3-1)和(3-1)；如果当前节点被访 问过，再次执行 (2-1)。  
		1. 如果此时$L$里不足$k$个点，则将节点特征加入$L$；如果$L$中已满$k$个点，且当前节点与 $p$的距离小于$L$ 里最长的距离，则用节点特征替换掉$L$中离最远的点。  
		2. 计算$p$和当前节点切分线的距离。如果该距离大于等于$L$中距离$p$最远的距离，**并且** $L$中已有$k$个点，则在切分线另一边不会有更近的点，执行(3)；如果该距离小于$L$中最远的距离，**或者** $L$中不足$k$个点，则切分线另一边可能有更近的点，因此在当前节点的另一个枝从 (1) 开始执行。
# 查找案例
![[Pasted image 20260518141919.png]]
![[Pasted image 20260518141932.png]]
![[Pasted image 20260518141944.png]]
![[Pasted image 20260518141954.png]]
![[Pasted image 20260518142005.png]]
![[Pasted image 20260518142017.png]]
![[Pasted image 20260518142027.png]]
![[Pasted image 20260518142037.png]]
![[Pasted image 20260518142156.png]]
![[Pasted image 20260518142318.png]]
![[Pasted image 20260518142435.png]]
![[Pasted image 20260518142509.png]]
![[Pasted image 20260518142546.png]]
![[Pasted image 20260518142557.png]]
![[Pasted image 20260518142610.png]]
![[Pasted image 20260518142622.png]]
![[Pasted image 20260518142632.png]]
![[Pasted image 20260518142654.png]]
![[Pasted image 20260518142722.png]]
![[Pasted image 20260518142731.png]]
![[Pasted image 20260518142759.png]]
![[Pasted image 20260518142811.png]]
![[Pasted image 20260518142821.png]]
![[Pasted image 20260518142853.png]]
![[Pasted image 20260518142914.png]]
![[Pasted image 20260518142956.png]]
![[Pasted image 20260518143008.png]]
![[Pasted image 20260518143033.png]]
![[Pasted image 20260518143047.png]]
# KD树C++实现
``` c++
// 最邻近/k临近
#include <vector>
#include <algorithm>
#include <queue>
#include <cmath>
#include <limits>
#include <iostream>

// KD 树节点
template<typename T>
struct KDNode {
    int idx;                // 点在数据数组中的索引
    int axis;               // 分裂轴
    KDNode* left;
    KDNode* right;
    KDNode() : idx(-1), axis(-1), left(nullptr), right(nullptr) {}
};
// KD 树主类
template<typename T>
class KDTree {
public:
    // 构造函数：传入维度、数据指针（扁平存储，长度为 n * dim）
    KDTree(int dim, const std::vector<T>& data, int n)
        : dim_(dim), data_(data), n_(n) {
        // 建立索引数组
        std::vector<int> indices(n_);
        for (int i = 0; i < n_; ++i) indices[i] = i;
        root_ = build(indices, 0);
    }
    ~KDTree() {
       free(root_);
    }
    // 最近邻搜索（单个）
    int nearest(const std::vector<T>& query, T& best_dist) const {
        if (!root_) return -1;
        int best_idx = -1;
        best_dist = std::numeric_limits<T>::max();
        nearestSearch(root_, query, best_idx, best_dist);
        return best_idx;
    }
    // k 近邻搜索（返回索引列表和距离平方列表）
    void knnSearch(const std::vector<T>& query, int k,
        std::vector<int>& indices,
        std::vector<T>& dists) const {
        using Pair = std::pair<T, int>;  // <dist^2, index>
        // 最大堆，保留距离最大的在顶部
        std::priority_queue<Pair> pq;
        knnSearch(root_, query, k, pq);
        // 提取结果（距离从小到大）
        indices.clear();
        dists.clear();
        while (!pq.empty()) {
            indices.push_back(pq.top().second);
            dists.push_back(pq.top().first);
            pq.pop();
        }
        std::reverse(indices.begin(), indices.end());
        std::reverse(dists.begin(), dists.end());
    }
private:
    int dim_;
    int n_;
    std::vector<T> data_;  // 扁平存储：data_[i * dim_ + j]
    KDNode<T>* root_;
    // 构建树（递归）
    KDNode<T>* build(std::vector<int>& indices, int depth) {
        if (indices.empty()) return nullptr;
        int axis = depth % dim_;
        // 按照当前轴排序
        std::sort(indices.begin(), indices.end(),
            [this, axis](int a, int b) {
                return data_[a * dim_ + axis] < data_[b * dim_ + axis];
            });
        int mid = indices.size() / 2;
        KDNode<T>* node = new KDNode<T>();
        node->idx = indices[mid];
        node->axis = axis;
        std::vector<int> leftIndices(indices.begin(), indices.begin() + mid);
        std::vector<int> rightIndices(indices.begin() + mid + 1, indices.end());
        node->left = build(leftIndices, depth + 1);
        node->right = build(rightIndices, depth + 1);
        return node;
    }
    // 释放树
    void free(KDNode<T>* node) {
        if (!node) return;
        free(node->left);
        free(node->right);
        delete node;
    }
    // 递归最近邻搜索
    void nearestSearch(KDNode<T>* node, const std::vector<T>& query,
        int& best_idx, T& best_dist) const {
        if (!node) return;
        int idx = node->idx;
        // 当前点到查询点的距离平方
        T dist = 0;
        for (int i = 0; i < dim_; ++i) {
            T diff = data_[idx * dim_ + i] - query[i];
            dist += diff * diff;
        }
        if (dist < best_dist) {
            best_dist = dist;
            best_idx = idx;
        }
        int axis = node->axis;
        T diff_axis = data_[idx * dim_ + axis] - query[axis];
        // 先搜索更近的一侧
        KDNode<T>* first = (diff_axis <= 0) ? node->left : node->right;
        KDNode<T>* second = (diff_axis <= 0) ? node->right : node->left;
        nearestSearch(first, query, best_idx, best_dist);
        // 若另一侧可能有更近点，再搜索
        if (diff_axis * diff_axis < best_dist) {
            nearestSearch(second, query, best_idx, best_dist);
        }
    }
    // 递归 k 近邻搜索
    void knnSearch(KDNode<T>* node, const std::vector<T>& query, int k,
        std::priority_queue<std::pair<T, int>>& pq) const {
        if (!node) return;
        int idx = node->idx;
        // 计算距离
        T dist = 0;
        for (int i = 0; i < dim_; ++i) {
            T diff = data_[idx * dim_ + i] - query[i];
            dist += diff * diff;
        }
        // 插入候选堆
        if (pq.size() < k) {
            pq.emplace(dist, idx);
        }
        else if (dist < pq.top().first) {
            pq.pop();
            pq.emplace(dist, idx);
        }
        //std::priority_queue插入数据后会自动根据优先级排序
        //std::priority_queue::pop 移除优先队列的最顶部元素
        int axis = node->axis;
        T diff_axis = data_[idx * dim_ + axis] - query[axis];
        KDNode<T>* first = (diff_axis <= 0) ? node->left : node->right;
        KDNode<T>* second = (diff_axis <= 0) ? node->right : node->left;
        knnSearch(first, query, k, pq);
        // 剪枝判断-----pq.top()返回最大值
        if (pq.size() < k || diff_axis * diff_axis < pq.top().first) {
            knnSearch(second, query, k, pq);
        }
    }
};
// ---------------------- 示例用法 ----------------------
int main() {
    // 示例：2 维点集，使用 double 类型
    int dim = 3;
    std::vector<double> data = {
        1.0, 2.0,
        3.0, 4.0,
        5.0, 6.0,
        7.0, 8.0,
        9.0, 10.0,
        11.0,12.0
    };
    int n = data.size() / dim;  // 5 个点  
    KDTree<double> kdtree(dim, data, n); 
    std::vector<double> query = { 4.0, 5.0 ,6.0};  
    // 最近邻搜索
    double best_dist;
    int best_idx = kdtree.nearest(query, best_dist);
    std::cout << "最近邻索引: " << best_idx << " 距离平方: " << best_dist << std::endl;
    std::cout << "坐标: (" << data[best_idx * dim] << ", " << data[best_idx * dim + 1]
        << ", " << data[best_idx * dim + 2] << ")" << std::endl;  
    // k 近邻搜索
    std::vector<int> indices;
    std::vector<double> dists;
    int k = 3;
    kdtree.knnSearch(query, k, indices, dists);
    std::cout << "\n" << k << " 个最近邻:\n";
    for (int i = 0; i < indices.size(); ++i) {
        std::cout << "索引 " << indices[i] << ", 距离平方 " << dists[i]
            << ", 坐标 (" << data[indices[i] * dim] << ", " << data[indices[i] * dim + 1]
            << ", " << data[indices[i] * dim + 2]  << ")\n";
    } 
    return 0;
}
```
## 三维点KD树查找
[[KD树查找最邻近的三维数据点]]
[[三维空间下的面积插值法]]