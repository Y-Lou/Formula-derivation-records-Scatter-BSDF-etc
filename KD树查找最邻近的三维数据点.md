``` c++
/**
 * 快速格子查找器（C++ 高性能版）
 * ================================
 *
 * 问题描述：
 *   给定一个 M×N 矩阵，每个元素包含三维空间中的位置点和法向。以及一个
 *   待查询的三维坐标点及其法向，找出矩阵中法向一致且包含该查询点的
 *   方格子（2×2 相邻矩阵元素围成的四边形），返回四个顶点的信息。
 *
 * 算法流程：
 *   ① KD-Tree 快速查找距离查询点最近的矩阵点 → (i_near, j_near)
 *   ② 枚举最近点周围的至多 4 个候选格子
 *   ③ 逐个检查：法向一致性 → 双线性插值判断点是否在格子内
 *
 * 时间复杂度：
 *   构建索引：O(MN log(MN))，一次性开销
 *   每次查询：O(log(MN))（KD-Tree 查找 + 常数个格子验证）
 *
 * 与 Python 版的区别：
 *   - 使用 std::nth_element 构建 KD-Tree（O(N) 每层，比排序更快）
 *   - 无第三方依赖，仅使用 C++11 标准库
 *   - unique_ptr 自动管理 KD-Tree 节点生命周期
 *
 * 编译：
 *   g++ -std=c++11 -O2 grid_cell_finder_fast.cpp -o grid_cell_finder_fast
 */

#include <algorithm>
#include <array>
#include <cassert>
#include <chrono>
#include <cmath>
#include <iostream>
#include <memory>
#ifdef _WIN32
#include <windows.h>
#endif

#include <set>
#include <stdexcept>
#include <string>
#include <tuple>
#include <vector>

// =========================================================================
// 三维向量结构及其基本运算
// =========================================================================

/// 三维向量（位置 / 法向 / 方向等）
struct Vec3 {
    double x, y, z;

    Vec3() : x(0), y(0), z(0) {}
    Vec3(double x_, double y_, double z_) : x(x_), y(y_), z(z_) {}

    /// 从可索引容器构造（如 vector<double>、array<double,3>）
    template <typename T>
    static Vec3 from(const T& c) {
        return Vec3(c[0], c[1], c[2]);
    }
};

/// 向量点积 a·b
inline double dot(const Vec3& a, const Vec3& b) {
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

/// 向量模长 |v|
inline double norm(const Vec3& v) {
    return std::sqrt(dot(v, v));
}

/// 两点之间欧氏距离的平方（避免开根号，比较时更快）
inline double dist_sq(const Vec3& a, const Vec3& b) {
    double dx = a.x - b.x;
    double dy = a.y - b.y;
    double dz = a.z - b.z;
    return dx * dx + dy * dy + dz * dz;
}

/// 向量归一化 v/|v|，零向量抛出异常
inline Vec3 normalize(const Vec3& v) {
    double n = norm(v);
    if (n < 1e-15) {
        throw std::runtime_error("法向向量长度为零，无法归一化");
    }
    return Vec3(v.x / n, v.y / n, v.z / n);
}

/// 向量减法 a - b
inline Vec3 sub(const Vec3& a, const Vec3& b) {
    return Vec3(a.x - b.x, a.y - b.y, a.z - b.z);
}

/// 向量加法 a + b
inline Vec3 add(const Vec3& a, const Vec3& b) {
    return Vec3(a.x + b.x, a.y + b.y, a.z + b.z);
}

/// 向量数乘 s·v
inline Vec3 scale(const Vec3& v, double s) {
    return Vec3(v.x * s, v.y * s, v.z * s);
}

// =========================================================================
// KD-Tree（三维空间索引，用于 O(log N) 最近邻查找）
// =========================================================================

/**
 * 三维 KD-Tree。
 *
 * 构建过程：
 *   1. 将所有网格点展平为一维数组，每个点关联其扁平索引 k = i*N + j
 *   2. 使用 std::nth_element 在 O(N) 时间内找到当前切分轴坐标的中位数
 *   3. 左子树 ≤ 中位数，右子树 > 中位数
 *   4. 切分轴在 x(0), y(1), z(2) 之间轮换
 *
 * 查询过程：
 *   1. 从根出发，沿切分轴比较走向近侧子树
 *   2. 到达叶子后回溯：若切分面到目标的距离 < 当前最佳距离，
 *      说明远侧子树可能包含更近的点，进入搜索
 *   3. 剪枝条件避免不必要的子树遍历
 *
 * 内存管理：
 *   使用 std::unique_ptr 自动管理节点生命周期，
 *   KDTree 销毁时整棵树自动释放。
 */
class KDTree {
public:
    /**
     * 构建 KD-Tree。
     * @param positions  (M, N) 矩阵，每个元素为三维坐标
     */
    template <typename PosContainer>
    explicit KDTree(const PosContainer& positions)
        : M_(positions.size()), N_(positions[0].size()) {
        // 展平所有点，记录 (坐标, 扁平索引)
        std::vector<PointWithIndex> flat;
        flat.reserve(M_ * N_);
        for (size_t i = 0; i < M_; ++i) {
            for (size_t j = 0; j < N_; ++j) {
                flat.push_back({
                    Vec3::from(positions[i][j]),
                    static_cast<int>(i * N_ + j)
                });
            }
        }
        root_ = build(flat, 0, flat.size(), 0);
    }

    /**
     * 查询距离 target 最近的网格点。
     * @param target  查询点坐标 (3,)
     * @return (row, col, squared_distance)
     */
    std::tuple<int, int, double> query(const Vec3& target) const {
        int best_flat_idx = -1;
        double best_dist_sq = std::numeric_limits<double>::max();
        search_nearest(root_.get(), target, best_flat_idx, best_dist_sq);

        int row = best_flat_idx / static_cast<int>(N_);
        int col = best_flat_idx % static_cast<int>(N_);
        return std::make_tuple(row, col, best_dist_sq);
    }

private:
    struct PointWithIndex {
        Vec3 point;
        int  index;  // 扁平索引 = i * N + j
    };

    struct Node {
        Vec3 point;
        int  flat_index;
        int  axis;  // 0=x, 1=y, 2=z
        std::unique_ptr<Node> left;
        std::unique_ptr<Node> right;
    };

    /// 递归构建子树
    /// @param points  当前点集（会被 nth_element 修改）
    /// @param start   起始下标（含）
    /// @param end     结束下标（不含）
    /// @param depth   当前深度，决定切分轴
    static std::unique_ptr<Node> build(std::vector<PointWithIndex>& points,
                                       size_t start, size_t end, int depth) {
        if (start >= end) return nullptr;

        int axis = depth % 3;
        size_t mid = (start + end) / 2;

        // 用 nth_element 在 O(N) 时间内找到中位数
        // 左侧所有元素 ≤ 中位数，右侧 ≥ 中位数
        std::nth_element(
            points.begin() + start,
            points.begin() + mid,
            points.begin() + end,
            [axis](const PointWithIndex& a, const PointWithIndex& b) {
                double va = (axis == 0) ? a.point.x
                            : (axis == 1) ? a.point.y
                                          : a.point.z;
                double vb = (axis == 0) ? b.point.x
                            : (axis == 1) ? b.point.y
                                          : b.point.z;
                return va < vb;
            });

        auto node = std::unique_ptr<Node>(new Node());
        node->point      = points[mid].point;
        node->flat_index = points[mid].index;
        node->axis       = axis;
        node->left  = build(points, start, mid, depth + 1);
        node->right = build(points, mid + 1, end, depth + 1);
        return node;
    }

    /// 递归最近邻搜索（含剪枝）
    static void search_nearest(const Node* node, const Vec3& target,
                               int& best_idx, double& best_dist_sq) {
        if (!node) return;

        double d2 = dist_sq(node->point, target);
        if (d2 < best_dist_sq) {
            best_dist_sq = d2;
            best_idx     = node->flat_index;
        }

        int axis = node->axis;
        double node_coord = (axis == 0) ? node->point.x
                            : (axis == 1) ? node->point.y
                                          : node->point.z;
        double target_coord = (axis == 0) ? target.x
                              : (axis == 1) ? target.y
                                            : target.z;
        double diff = target_coord - node_coord;

        // 优先搜索近侧子树
        if (diff <= 0) {
            search_nearest(node->left.get(), target, best_idx, best_dist_sq);
            // 剪枝：仅当切分平面到目标距离 < 当前最优距离时搜索远侧
            if (diff * diff < best_dist_sq)
                search_nearest(node->right.get(), target, best_idx, best_dist_sq);
        } else {
            search_nearest(node->right.get(), target, best_idx, best_dist_sq);
            if (diff * diff < best_dist_sq)
                search_nearest(node->left.get(), target, best_idx, best_dist_sq);
        }
    }

    size_t M_, N_;
    std::unique_ptr<Node> root_;
};

// =========================================================================
// 双线性插值求解器（Gauss-Newton 法）
// =========================================================================

/**
 * 求解双线性插值参数 (u, v)，使得：
 *
 *     p ≈ (1-u)(1-v)·v00 + u(1-v)·v10 + (1-u)v·v01 + uv·v11
 *
 * 其中 v00=(i,j), v10=(i+1,j), v01=(i,j+1), v11=(i+1,j+1)
 * 是方格子四个顶点。
 *
 * 数学推导：
 *   f(u,v) = v00 + u(v10-v00) + v(v01-v00) + uv(v00-v10-v01+v11) - p
 *   令 e0 = v10-v00（u方向切线）
 *      e1 = v01-v00（v方向切线）
 *      e2 = v00-v10-v01+v11（twist项）
 *   则 ∂f/∂u = e0 + v·e2,  ∂f/∂v = e1 + u·e2
 *
 * 这是 3 个方程、2 个未知数的超定系统，使用 Gauss-Newton 迭代求解。
 * 每次迭代解正规方程 JᵀJ·Δ = -Jᵀf。
 *
 * @param p     查询点坐标
 * @param v00   格子顶点 (i,   j  )
 * @param v10   格子顶点 (i+1, j  )
 * @param v01   格子顶点 (i,   j+1)
 * @param v11   格子顶点 (i+1, j+1)
 * @param max_iter  最大迭代次数
 * @param tol       收敛容差
 * @return (u, v, converged)
 */
inline std::tuple<double, double, bool> solve_bilinear(
    const Vec3& p,
    const Vec3& v00, const Vec3& v10,
    const Vec3& v01, const Vec3& v11,
    int max_iter = 30, double tol = 1e-10) {
    // 预计算偏导项
    Vec3 e0 = sub(v10, v00);                       // u 方向切线
    Vec3 e1 = sub(v01, v00);                       // v 方向切线
    Vec3 e2 = sub(add(v00, v11), add(v10, v01));   // twist = v00-v10-v01+v11

    double u = 0.5, v = 0.5;  // 从参数域中心开始

    for (int iter = 0; iter < max_iter; ++iter) {
        // ── 残差 f(u,v) = v00 + u·e0 + v·e1 + uv·e2 - p ──
        Vec3 term_linear = add(v00, add(scale(e0, u), scale(e1, v)));
        Vec3 term_twist  = scale(e2, u * v);
        Vec3 f = sub(add(term_linear, term_twist), p);

        // ── Jacobian 矩阵的两列 ──
        Vec3 df_du = add(e0, scale(e2, v));
        Vec3 df_dv = add(e1, scale(e2, u));

        // ── 正规方程 JᵀJ·Δ = -Jᵀf ──
        double a = dot(df_du, df_du);
        double b = dot(df_du, df_dv);
        double d = dot(df_dv, df_dv);
        double det = a * d - b * b;

        if (std::abs(det) < 1e-15)
            return std::make_tuple(u, v, false);  // 奇异矩阵，点可能在格子平面外

        double jtf_u = dot(df_du, f);
        double jtf_v = dot(df_dv, f);

        // Cramer 法则求 Δ = -(JᵀJ)⁻¹(Jᵀf)
        double du = -(d * jtf_u - b * jtf_v) / det;
        double dv = -(a * jtf_v - b * jtf_u) / det;

        u += du;
        v += dv;

        if (du * du + dv * dv < tol * tol)
            return std::make_tuple(u, v, true);
    }
    return std::make_tuple(u, v, false);  // 未收敛
}

// =========================================================================
// 查询结果结构
// =========================================================================

/**
 * 格子查找结果，包含方格子四个顶点的完整信息。
 */
struct GridCellResult {
    int i, j;                            // 格子左上顶点的矩阵索引
    std::vector<std::pair<int, int>> vertices;  // 四个顶点索引
    std::vector<Vec3> positions;          // 四个顶点的三维坐标
    std::vector<Vec3> normals;            // 四个顶点的归一化法向
    int nearest_i, nearest_j;            // 查询点的最近矩阵点
    double bilinear_u, bilinear_v;       // 双线性参数 u, v ∈ [0,1]
};

// =========================================================================
// 核心类：GridCellFinder
// =========================================================================

/**
 * 快速格子查找器。
 *
 * 预先构建 KD-Tree 空间索引，支持对同一矩阵进行多次查询。
 * 每次查询时间复杂度 O(log(M×N))。
 *
 * 使用示例：
 * @code
 *   GridCellFinder finder(positions, normals);
 *   auto result = finder.find(query_point, query_normal);
 *   if (result) {
 *       std::cout << "格子: (" << result->i << "," << result->j << ")\n";
 *   }
 * @endcode
 */
class GridCellFinder {
public:
    /**
     * 初始化查找器，构建空间索引。
     *
     * @tparam PosContainer  外层可索引、内层元素可转为 Vec3 的容器
     * @tparam NormContainer 同上
     *
     * @param positions  (M, N) 矩阵，每个元素为三维位置
     * @param normals    (M, N) 矩阵，每个元素为三维法向（无需预先归一化）
     *
     * @throws std::invalid_argument  矩阵维度不足或形状不匹配
     */
    template <typename PosContainer, typename NormContainer>
    GridCellFinder(const PosContainer& positions, const NormContainer& normals)
        : M_(positions.size()), N_(positions[0].size()),
          kdtree_(positions) {
        if (M_ < 2 || N_ < 2) {
            throw std::invalid_argument(
                "矩阵至少需要 2×2，当前为 " +
                std::to_string(M_) + "×" + std::to_string(N_));
        }
        if (normals.size() != M_ || normals[0].size() != N_) {
            throw std::invalid_argument("normals 维度必须与 positions 一致");
        }

        // 预先归一化所有法向，避免查询时重复计算
        normals_norm_.resize(M_, std::vector<Vec3>(N_));
        for (size_t i = 0; i < M_; ++i) {
            for (size_t j = 0; j < N_; ++j) {
                normals_norm_[i][j] = normalize(Vec3::from(normals[i][j]));
            }
        }

        // 保存位置引用（拷贝副本，因为模板参数可能是临时的）
        positions_.resize(M_, std::vector<Vec3>(N_));
        for (size_t i = 0; i < M_; ++i) {
            for (size_t j = 0; j < N_; ++j) {
                positions_[i][j] = Vec3::from(positions[i][j]);
            }
        }
    }

    /**
     * 查找包含查询点且法向一致的方格子。
     *
     * @param query_point        (3,) 待查询的三维坐标点
     * @param query_normal       (3,) 待查询点的法向
     * @param normal_threshold   法向一致性阈值（点积绝对值），0.9 ≈ 25.8°
     * @param max_search_radius  最大搜索半径（网格步数），超出则回退到最近法向一致格子
     * @return 命中返回结果，否则返回 nullptr
     */
    template <typename QP, typename QN>
    std::unique_ptr<GridCellResult> find(
        const QP& query_point,
        const QN& query_normal,
        double normal_threshold = 0.9,
        int max_search_radius = 5) const {
        Vec3 qp = Vec3::from(query_point);
        Vec3 qn = normalize(Vec3::from(query_normal));

        // ① KD-Tree 查找最近矩阵点
        int near_i, near_j; double _dist; std::tie(near_i, near_j, _dist) = kdtree_.query(qp);

        // ② 从最近点开始逐层扩展搜索
        for (int radius = 0; radius <= max_search_radius; ++radius) {
            auto result = search_at_radius(qp, qn, near_i, near_j,
                                           radius, normal_threshold);
            if (result) return result;
        }

        // ③ 容错回退：返回法向一致且中心最近的格子
        return find_nearest_consistent(qp, qn, near_i, near_j,
                                       max_search_radius, normal_threshold);
    }

    /// 矩阵行数
    int rows() const { return static_cast<int>(M_); }
    /// 矩阵列数
    int cols() const { return static_cast<int>(N_); }

private:
    /**
     * 返回以 (i,j) 为某个角的至多 4 个格子左上角索引。
     * 不验证边界——调用方需自行检查。
     *
     * (i,j) 可以是：
     *   - 左上角 → 返回 (i,   j  )
     *   - 右上角 → 返回 (i-1, j  )
     *   - 左下角 → 返回 (i,   j-1)
     *   - 右下角 → 返回 (i-1, j-1)
     */
    static std::vector<std::pair<int, int>> cells_around(int i, int j) {
        return {
            {i,     j    },
            {i - 1, j    },
            {i,     j - 1},
            {i - 1, j - 1}
        };
    }

    /// 在指定半径范围内搜索匹配的格子
    std::unique_ptr<GridCellResult> search_at_radius(
        const Vec3& qp, const Vec3& qn,
        int center_i, int center_j,
        int radius, double threshold) const {
        // 用 set 记录已检查的格子，避免重复
        std::set<std::pair<int, int>> checked;

        for (int di = -radius; di <= radius; ++di) {
            for (int dj = -radius; dj <= radius; ++dj) {
                // radius>0 时只处理外环，内部已在上轮检查过
                if (radius > 0 &&
                    std::abs(di) < radius && std::abs(dj) < radius)
                    continue;

                int ci = center_i + di;
                int cj = center_j + dj;
                if (ci < 0 || ci >= static_cast<int>(M_) ||
                    cj < 0 || cj >= static_cast<int>(N_))
                    continue;

                for (const auto& _p : cells_around(ci, cj)) {
                    int cell_i = _p.first;
                    int cell_j = _p.second;
                    if (checked.count({cell_i, cell_j})) continue;
                    checked.insert({cell_i, cell_j});

                    auto result = try_cell(cell_i, cell_j, qp, qn,
                                           threshold, ci, cj);
                    if (result) return result;
                }
            }
        }
        return nullptr;
    }

    /// 检查单个格子是否匹配
    std::unique_ptr<GridCellResult> try_cell(
        int i, int j,
        const Vec3& qp, const Vec3& qn,
        double threshold,
        int near_i, int near_j) const {
        // 边界检查：确保四个顶点都在矩阵范围内
        if (i < 0 || j < 0 ||
            i + 1 >= static_cast<int>(M_) ||
            j + 1 >= static_cast<int>(N_))
            return nullptr;

        // ── 法向一致性检查 ──
        const Vec3& n00 = normals_norm_[i][j];
        const Vec3& n10 = normals_norm_[i + 1][j];
        const Vec3& n01 = normals_norm_[i][j + 1];
        const Vec3& n11 = normals_norm_[i + 1][j + 1];

        if (std::abs(dot(n00, qn)) < threshold ||
            std::abs(dot(n10, qn)) < threshold ||
            std::abs(dot(n01, qn)) < threshold ||
            std::abs(dot(n11, qn)) < threshold)
            return nullptr;

        // ── 位置包含检查（双线性插值求解） ──
        const Vec3& v00 = positions_[i][j];
        const Vec3& v10 = positions_[i + 1][j];
        const Vec3& v01 = positions_[i][j + 1];
        const Vec3& v11 = positions_[i + 1][j + 1];

        double u, v; bool converged; std::tie(u, v, converged) = solve_bilinear(qp, v00, v10, v01, v11);
        if (!converged) return nullptr;

        // 允许少量容差 (-0.01 ~ 1.01)，超出则不在格子内
        if (u < -0.01 || u > 1.01 || v < -0.01 || v > 1.01)
            return nullptr;

        u = std::max(0.0, std::min(1.0, u));
        v = std::max(0.0, std::min(1.0, v));

        auto r = std::unique_ptr<GridCellResult>(new GridCellResult());
        r->i = i;  r->j = j;
        r->vertices = {{i, j}, {i + 1, j}, {i, j + 1}, {i + 1, j + 1}};
        r->positions = {v00, v10, v01, v11};
        r->normals   = {n00, n10, n01, n11};
        r->nearest_i = near_i;
        r->nearest_j = near_j;
        r->bilinear_u = u;
        r->bilinear_v = v;
        return r;
    }

    /// 容错回退：在搜索半径内找到法向一致且中心最近的格子
    std::unique_ptr<GridCellResult> find_nearest_consistent(
        const Vec3& qp, const Vec3& qn,
        int center_i, int center_j,
        int radius, double threshold) const {
        std::unique_ptr<GridCellResult> best;
        double best_dist_sq = std::numeric_limits<double>::max();
        std::set<std::pair<int, int>> checked;

        for (int r = 0; r <= radius; ++r) {
            for (int di = -r; di <= r; ++di) {
                for (int dj = -r; dj <= r; ++dj) {
                    if (r > 0 && std::abs(di) < r && std::abs(dj) < r)
                        continue;
                    int ci = center_i + di, cj = center_j + dj;
                    if (ci < 0 || ci >= static_cast<int>(M_) ||
                        cj < 0 || cj >= static_cast<int>(N_))
                        continue;

                    for (const auto& _p : cells_around(ci, cj)) {
                        int cell_i = _p.first;
                        int cell_j = _p.second;
                        if (checked.count({cell_i, cell_j})) continue;
                        checked.insert({cell_i, cell_j});

                        if (cell_i < 0 || cell_j < 0 ||
                            cell_i + 1 >= static_cast<int>(M_) ||
                            cell_j + 1 >= static_cast<int>(N_))
                            continue;

                        const Vec3& n00 = normals_norm_[cell_i][cell_j];
                        const Vec3& n10 = normals_norm_[cell_i+1][cell_j];
                        const Vec3& n01 = normals_norm_[cell_i][cell_j+1];
                        const Vec3& n11 = normals_norm_[cell_i+1][cell_j+1];
                        if (std::abs(dot(n00, qn)) < threshold ||
                            std::abs(dot(n10, qn)) < threshold ||
                            std::abs(dot(n01, qn)) < threshold ||
                            std::abs(dot(n11, qn)) < threshold)
                            continue;

                        const Vec3& p00 = positions_[cell_i][cell_j];
                        const Vec3& p10 = positions_[cell_i+1][cell_j];
                        const Vec3& p01 = positions_[cell_i][cell_j+1];
                        const Vec3& p11 = positions_[cell_i+1][cell_j+1];
                        Vec3 center = scale(add(add(p00, p10), add(p01, p11)), 0.25);

                        double d2 = dist_sq(qp, center);
                        if (d2 < best_dist_sq) {
                            best_dist_sq = d2;
                            auto r = std::unique_ptr<GridCellResult>(new GridCellResult());
                            r->i = cell_i;  r->j = cell_j;
                            r->vertices = {{cell_i, cell_j},
                                          {cell_i+1, cell_j},
                                          {cell_i, cell_j+1},
                                          {cell_i+1, cell_j+1}};
                            r->positions = {p00, p10, p01, p11};
                            r->normals   = {n00, n10, n01, n11};
                            r->nearest_i = ci;
                            r->nearest_j = cj;
                            r->bilinear_u = 0.5;
                            r->bilinear_v = 0.5;
                            best = std::move(r);
                        }
                    }
                }
            }
        }
        return best;
    }

    size_t M_, N_;
    KDTree kdtree_;
    std::vector<std::vector<Vec3>> positions_;
    std::vector<std::vector<Vec3>> normals_norm_;
};

// =========================================================================
// 便捷函数接口
// =========================================================================

/**
 * 一次性快速查找（函数接口）。
 * 内部自动构建 KD-Tree；多次查询请使用 GridCellFinder 类。
 */
template <typename PC, typename NC, typename QP, typename QN>
std::unique_ptr<GridCellResult> find_grid_cell_fast(
    const PC& positions,
    const NC& normals,
    const QP& query_point,
    const QN& query_normal,
    double normal_threshold = 0.9) {
    GridCellFinder finder(positions, normals);
    return finder.find(query_point, query_normal, normal_threshold);
}

// =========================================================================
// 暴力遍历版（仅用于性能对比）
// =========================================================================

/**
 * 暴力遍历所有格子，用于验证快速版结果正确性及性能对比。
 * 时间复杂度 O(M×N)，请勿用于大规模生产数据。
 */
template <typename PC, typename NC, typename QP, typename QN>
std::unique_ptr<GridCellResult> find_grid_cell_bruteforce(
    const PC& positions,
    const NC& normals,
    const QP& query_point,
    const QN& query_normal,
    double normal_threshold = 0.9) {
    int M = static_cast<int>(positions.size());
    int N = static_cast<int>(positions[0].size());
    Vec3 qp = Vec3::from(query_point);
    Vec3 qn = normalize(Vec3::from(query_normal));

    for (int i = 0; i < M - 1; ++i) {
        for (int j = 0; j < N - 1; ++j) {
            Vec3 n00 = normalize(Vec3::from(normals[i][j]));
            Vec3 n10 = normalize(Vec3::from(normals[i+1][j]));
            Vec3 n01 = normalize(Vec3::from(normals[i][j+1]));
            Vec3 n11 = normalize(Vec3::from(normals[i+1][j+1]));

            if (std::abs(dot(n00, qn)) < normal_threshold ||
                std::abs(dot(n10, qn)) < normal_threshold ||
                std::abs(dot(n01, qn)) < normal_threshold ||
                std::abs(dot(n11, qn)) < normal_threshold)
                continue;

            Vec3 v00 = Vec3::from(positions[i][j]);
            Vec3 v10 = Vec3::from(positions[i+1][j]);
            Vec3 v01 = Vec3::from(positions[i][j+1]);
            Vec3 v11 = Vec3::from(positions[i+1][j+1]);

            double u, v; bool ok; std::tie(u, v, ok) = solve_bilinear(qp, v00, v10, v01, v11);
            if (!ok) continue;
            if (u < -0.01 || u > 1.01 || v < -0.01 || v > 1.01)
                continue;

            u = std::max(0.0, std::min(1.0, u));
            v = std::max(0.0, std::min(1.0, v));

            auto r = std::unique_ptr<GridCellResult>(new GridCellResult());
            r->i = i;  r->j = j;
            r->vertices = {{i, j}, {i+1, j}, {i, j+1}, {i+1, j+1}};
            r->positions = {v00, v10, v01, v11};
            r->normals   = {n00, n10, n01, n11};
            r->nearest_i = i;
            r->nearest_j = j;
            r->bilinear_u = u;
            r->bilinear_v = v;
            return r;
        }
    }
    return nullptr;
}

// =========================================================================
// 演示与测试
// =========================================================================

int main() {
    using std::cout;
    using Clock = std::chrono::high_resolution_clock;
    using namespace std::chrono;

#ifdef _WIN32
    SetConsoleOutputCP(CP_UTF8);  // Windows 控制台使用 UTF-8 编码，避免中文乱码
#endif

    cout << "============================================================\n";
    cout << "  快速格子查找器 (C++) — 性能演示\n";
    cout << "============================================================\n";

    // ── 1. 小矩阵正确性验证 ──
    cout << "\n【1】小矩阵正确性验证\n";
    cout << "----------------------------------------\n";

    const int M_small = 4, N_small = 5;
    std::vector<std::vector<std::array<double, 3>>> positions_small(
        M_small, std::vector<std::array<double, 3>>(N_small));
    std::vector<std::vector<std::array<double, 3>>> normals_small(
        M_small, std::vector<std::array<double, 3>>(N_small));

    for (int i = 0; i < M_small; ++i) {
        for (int j = 0; j < N_small; ++j) {
            positions_small[i][j] = {double(i), double(j), 0.0};
            normals_small[i][j]   = {0.0, 0.0, 1.0};
        }
    }

    GridCellFinder finder_small(positions_small, normals_small);

    // 测试 1a：点在格子 (1,2) 内部
    std::array<double, 3> qp{1.3, 2.4, 0.0};
    std::array<double, 3> qn{0.0, 0.0, 1.0};
    auto result = finder_small.find(qp, qn);
    assert(result && "1a 失败：应找到格子");
    assert(result->i == 1 && result->j == 2);
    cout << "  1a ✓ 查询点 (1.3, 2.4, 0.0) → 格子 ("
         << result->i << "," << result->j << ")\n";
    cout << "      双线性参数: u=" << result->bilinear_u
         << ", v=" << result->bilinear_v << "\n";

    // 测试 1b：法向不一致
    auto normals_bad = normals_small;
    normals_bad[1][2] = {1.0, 0.0, 0.0};
    GridCellFinder finder_bad(positions_small, normals_bad);
    auto result_bad = finder_bad.find(qp, qn);
    // 法向不一致的格子不应被选中（可能选其他格子或返回 nullopt）
    assert(!result_bad || result_bad->i != 1 || result_bad->j != 2);
    cout << "  1b ✓ 法向不一致时跳过了有问题的格子\n";

    // 测试 1c：多次查询复用同一个 finder
    for (int k = 0; k < 10; ++k) {
        auto r = finder_small.find(std::array<double,3>{0.7, 1.2, 0.0}, qn);
        assert(r);
    }
    cout << "  1c ✓ 同一 finder 实例支持多次查询\n";

    // ── 2. 大规模矩阵性能对比 ──
    cout << "\n【2】大规模矩阵性能对比 (300×300 = 90,000 个点)\n";
    cout << "----------------------------------------\n";

    const int M_big = 300, N_big = 300;
    cout << "  正在生成 " << M_big << "×" << N_big << " 测试数据...\n";

    std::vector<std::vector<std::array<double, 3>>> positions_big(
        M_big, std::vector<std::array<double, 3>>(N_big));
    std::vector<std::vector<std::array<double, 3>>> normals_big(
        M_big, std::vector<std::array<double, 3>>(N_big));

    for (int i = 0; i < M_big; ++i) {
        for (int j = 0; j < N_big; ++j) {
            // 模拟略有弯曲的曲面
            positions_big[i][j] = {
                double(i) + 0.1 * (j % 10),
                double(j) + 0.1 * (i % 10),
                0.05 * (i + j)
            };
            normals_big[i][j] = {0.0, 0.0, 1.0};
        }
    }

    std::array<double, 3> qp_big{150.3, 150.4, 15.0};
    std::array<double, 3> qn_big{0.0, 0.0, 1.0};

    // ── 快速版 ──
    cout << "  构建 KD-Tree 空间索引...\n";
    auto t0 = Clock::now();
    GridCellFinder finder_big(positions_big, normals_big);
    auto t_build = duration<double, std::milli>(Clock::now() - t0).count();
    cout << "  构建耗时: " << t_build << " ms\n";

    t0 = Clock::now();
    auto result_fast = finder_big.find(qp_big, qn_big);
    auto t_fast = duration<double, std::milli>(Clock::now() - t0).count();
    cout << "  快速查找耗时: " << t_fast << " ms\n";
    if (result_fast) {
        cout << "  快速版结果: 格子 (" << result_fast->i
             << "," << result_fast->j << ")\n";
    }

    // ── 暴力遍历版（性能对比） ──
    cout << "  正在运行暴力遍历版（仅作性能对比）...\n";
    t0 = Clock::now();
    auto result_slow = find_grid_cell_bruteforce(
        positions_big, normals_big, qp_big, qn_big);
    auto t_slow = duration<double, std::milli>(Clock::now() - t0).count();
    cout << "  暴力遍历耗时: " << t_slow << " ms\n";
    if (result_slow) {
        cout << "  慢速版结果: 格子 (" << result_slow->i
             << "," << result_slow->j << ")\n";
    }

    // 验证结果一致
    if (result_fast && result_slow) {
        assert(result_fast->i == result_slow->i &&
               result_fast->j == result_slow->j);
        double speedup = t_slow / t_fast;
        cout << "\n  ✓ 快速版与慢速版结果一致，速度提升 "
             << int(speedup) << "x\n";
    }

    // ── 3. 边界情况测试 ──
    cout << "\n【3】边界情况测试\n";
    cout << "----------------------------------------\n";

    // 3a：查询点在矩阵角点附近
    auto r_corner = finder_small.find(
        std::array<double,3>{0.0, 0.0, 0.0}, qn);
    cout << "  3a " << (r_corner ? "✓ 角点查询命中" : "✗ 角点查询失败")
         << "\n";

    // 3b：查询点离曲面较远
    auto r_far = finder_small.find(
        std::array<double,3>{1.3, 2.4, 10.0}, qn);
    cout << "  3b " << (r_far ? "✓ 离面点返回最近格子" : "✗ 离面点失败")
         << "\n";

    // 3c：法向全不一致
    auto r_bad = finder_small.find(
        std::array<double,3>{1.3, 2.4, 0.0},
        std::array<double,3>{1.0, 0.0, 0.0});
    cout << "  3c " << (!r_bad ? "✓ 法向全不匹配，正确返回 null" : "✗ 异常")
         << "\n";

    // ── 4. 曲面网格测试 ──
    cout << "\n【4】曲面网格测试（正弦波曲面）\n";
    cout << "----------------------------------------\n";

    const int M_curved = 30, N_curved = 30;
    std::vector<std::vector<std::array<double, 3>>> positions_curved(
        M_curved, std::vector<std::array<double, 3>>(N_curved));
    std::vector<std::vector<std::array<double, 3>>> normals_curved(
        M_curved, std::vector<std::array<double, 3>>(N_curved));

    for (int i = 0; i < M_curved; ++i) {
        for (int j = 0; j < N_curved; ++j) {
            positions_curved[i][j] = {
                double(i),
                double(j),
                2.0 * std::sin(i * 0.3) * std::cos(j * 0.3)
            };
        }
    }

    // 用叉积近似计算曲面法向
    for (int i = 0; i < M_curved; ++i) {
        for (int j = 0; j < N_curved; ++j) {
            const auto& p = positions_curved[i][j];
            std::array<double, 3> du_arr, dv_arr;

            if (i < M_curved - 1) {
                const auto& nxt = positions_curved[i+1][j];
                du_arr = {nxt[0]-p[0], nxt[1]-p[1], nxt[2]-p[2]};
            } else {
                const auto& prv = positions_curved[i-1][j];
                du_arr = {p[0]-prv[0], p[1]-prv[1], p[2]-prv[2]};
            }
            if (j < N_curved - 1) {
                const auto& nxt = positions_curved[i][j+1];
                dv_arr = {nxt[0]-p[0], nxt[1]-p[1], nxt[2]-p[2]};
            } else {
                const auto& prv = positions_curved[i][j-1];
                dv_arr = {p[0]-prv[0], p[1]-prv[1], p[2]-prv[2]};
            }

            Vec3 du = Vec3::from(du_arr);
            Vec3 dv = Vec3::from(dv_arr);
            // n = du × dv
            double nx = du.y*dv.z - du.z*dv.y;
            double ny = du.z*dv.x - du.x*dv.z;
            double nz = du.x*dv.y - du.y*dv.x;
            double len = std::sqrt(nx*nx + ny*ny + nz*nz);
            if (len > 1e-15) {
                normals_curved[i][j] = {nx/len, ny/len, nz/len};
            } else {
                normals_curved[i][j] = {0.0, 0.0, 1.0};
            }
        }
    }

    GridCellFinder finder_curved(positions_curved, normals_curved);

    // 取格子 (10,12) 的中心作为查询点
    int vi = 10, vj = 12;
    const auto& v00 = positions_curved[vi][vj];
    const auto& v10 = positions_curved[vi+1][vj];
    const auto& v01 = positions_curved[vi][vj+1];
    const auto& v11 = positions_curved[vi+1][vj+1];

    std::array<double, 3> qp_center = {
        (v00[0] + v10[0] + v01[0] + v11[0]) / 4.0,
        (v00[1] + v10[1] + v01[1] + v11[1]) / 4.0,
        (v00[2] + v10[2] + v01[2] + v11[2]) / 4.0,
    };
    auto qn_center = normals_curved[vi][vj];

    auto r_curved = finder_curved.find(qp_center, qn_center, 0.7);
    if (r_curved) {
        cout << "  曲面查询: 命中格子 (" << r_curved->i
             << "," << r_curved->j << ")\n";
        cout << "  期望格子: (" << vi << "," << vj << ")\n";
    } else {
        cout << "  曲面查询: 未找到（法向阈值可能过严）\n";
    }

    // ── 5. 极端规模测试 ──
    cout << "\n【5】极端规模测试 (1000×1000 = 1,000,000 个点)\n";
    cout << "----------------------------------------\n";

    const int M_xl = 1000, N_xl = 1000;
    cout << "  正在生成 " << M_xl << "×" << N_xl << " 测试数据...\n";

    // 为 1000×1000 构建嵌套容器（约 24 MB）
    std::vector<std::vector<std::array<double, 3>>> pos_xl(
        M_xl, std::vector<std::array<double, 3>>(N_xl));
    std::vector<std::vector<std::array<double, 3>>> nrm_xl(
        M_xl, std::vector<std::array<double, 3>>(N_xl));

    cout << "  正在生成数据...\n";
    for (int i = 0; i < M_xl; ++i) {
        for (int j = 0; j < N_xl; ++j) {
            pos_xl[i][j] = {double(i), double(j), 0.0};
            nrm_xl[i][j] = {0.0, 0.0, 1.0};
        }
    }

    cout << "  构建 KD-Tree 中（可能需要数秒）...\n";
    t0 = Clock::now();

    GridCellFinder finder_xl(pos_xl, nrm_xl);
    auto t_build_xl = duration<double, std::milli>(Clock::now() - t0).count();
    cout << "  构建耗时: " << t_build_xl * 0.001 << " s\n";

    std::array<double, 3> qp_xl{500.3, 500.4, 0.0};
    std::array<double, 3> qn_xl{0.0, 0.0, 1.0};

    // 预热一次（清除缓存效应）
    finder_xl.find(qp_xl, qn_xl);

    t0 = Clock::now();
    const int query_repeat = 100;
    for (int k = 0; k < query_repeat; ++k) {
        finder_xl.find(qp_xl, qn_xl);
    }
    auto t_query_xl = duration<double, std::milli>(Clock::now() - t0).count();
    cout << "  " << query_repeat << " 次查询总耗时: " << t_query_xl << " ms\n";
    cout << "  平均每次查询: " << t_query_xl / query_repeat << " ms\n";

    cout << "\n============================================================\n";
    cout << "  全部测试完成\n";
    cout << "============================================================\n";

    return 0;
}
```