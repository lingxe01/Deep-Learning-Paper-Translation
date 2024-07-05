[【3D视觉】PointNet和PointNet++ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336496973)

# PointNet：

![image-20231130164646206](C:\Users\ling\AppData\Roaming\Typora\typora-user-images\image-20231130164646206.png)

在PointNet这篇文章中确实没有做到像CNN那样逐层提取局部特征。我们知道在CNN中，一个点会与周围若干点进行加权求和（具体取决于卷积核大小），然后获取一个新的点，随着网络层数加深，深层网络的一个点对应原始图像的一个映射区域，这就是感受野的概念。但是本文做的特征提取都是点之间独立进行的，这势必会造成一些问题，至于具体的问题解决，作者在PointNet++展开了说明。

# PointNet++：

![image-20231130164703345](C:\Users\ling\AppData\Roaming\Typora\typora-user-images\image-20231130164703345.png)



**问题1：一个点云图往往有非常多的点，这会造成计算量过大而限制模型使用，如何解决**

```python
def farthest_point_sample(point, npoint):
    """
    Input:
        xyz: pointcloud data, [N, D]
        npoint: number of samples
    Return:
        centroids: sampled pointcloud index, [npoint, D]
    """
    N, D = point.shape
    xyz = point[:,:3]
    centroids = np.zeros((npoint,))
    distance = np.ones((N,)) * 1e10
    farthest = np.random.randint(0, N)
    for i in range(npoint):
        centroids[i] = farthest
        centroid = xyz[farthest, :]
        dist = np.sum((xyz - centroid) ** 2, -1)
        mask = dist < distance
        distance[mask] = dist[mask]
        farthest = np.argmax(distance, -1)
    point = point[centroids.astype(np.int32)]
    return point
```

这段代码实现了PointNet++论文中的FPS算法（最远点采样算法）。

这个最远点采样算法（FPS）的流程如下：

- （1）随机选择一个点作为**初始点**作为**已选择采样点**；
- （2）计算**未选择采样点集**中每个点与**已选择采样点集**之间的距离distance，将距离最大的那个点加入已选择采样点集，
- （3）更新distance，一直循环迭代下去，直至获得了目标数量的采样点。

这段代码定义了一个名为 `farthest_point_sample` 的函数，用于在给定的点云上执行最远点采样。最远点采样是一种基于点的空间分布选择子集的技术，在计算机视觉或三维数据分析等各种应用中都很有用。

以下是该函数的详细解释：

- **输入参数**：
  - `xyz`：具有形状 `[B, N, 3]` 的点云数据，其中 `B` 是批次大小，`N` 是点的数量，`3` 表示（x、y、z）坐标。
  - `npoint`：要采样的点的数量。

- **输出**：
  - `centroids`：表示采样点的索引的张量，形状为 `[B, npoint]`。

- **实现**：
  1. `device = xyz.device`：获取输入点云所在的设备（CPU 或 GPU）。
  2. `B, N, C = xyz.shape`：提取批次大小（`B`）、点的数量（`N`）和坐标数量（`C`）。
  3. `centroids = torch.zeros(B, npoint, dtype=torch.long).to(device)`：初始化一个张量，用于存储采样点的索引。
  4. `distance = torch.ones(B, N).to(device) * 1e10`：初始化一个张量，用于存储距离，将初始距离设置为一个较大的值。
  5. `farthest = torch.randint(0, N, (B,), dtype=torch.long).to(device)`：为每个批次随机选择一个起始点。
  6. `batch_indices = torch.arange(B, dtype=torch.long).to(device)`：创建一个包含批次索引的张量。
  7. 重复执行 `npoint` 次：
     - `centroids[:, i] = farthest`：将当前最远点分配给采样点的张量。
     - `centroid = xyz[batch_indices, farthest, :].view(B, 1, 3)`：提取当前最远点的坐标。
     - `dist = torch.sum((xyz - centroid) ** 2, -1)`：计算所有点与当前质心之间的平方欧几里得距离。
     - `mask = dist < distance`：更新距离较近的点的距离。
     - `distance[mask] = dist[mask]`：更新距离。
     - `farthest = torch.max(distance, -1)[1]`：选择具有最大距离的点作为下一个最远点。

- **返回**：
  - `centroids`：采样点的索引。

该函数实际上实现了一个最远点采样算法，确保所选的点在输入点云中分布均匀。

**问题2：如何将点集划分为不同的区域，并获取不同区域的局部特征？**

**问题3：点云不均匀的时候，在密集区域学习出来的特征可能不适合稀疏区域，这个问题应该如何解决？**

![image-20231130165604699](C:\Users\ling\AppData\Roaming\Typora\typora-user-images\image-20231130165604699.png)

MSG的代码实现：

```python
        B, N, C = xyz.shape
        S = self.npoint
        new_xyz = index_points(xyz, farthest_point_sample(xyz, S))
        new_points_list = []
        for i, radius in enumerate(self.radius_list):
            K = self.nsample_list[i]
            group_idx = query_ball_point(radius, K, xyz, new_xyz)
            grouped_xyz = index_points(xyz, group_idx)
            grouped_xyz -= new_xyz.view(B, S, 1, C)
            if points is not None:
                grouped_points = index_points(points, group_idx)
                grouped_points = torch.cat([grouped_points, grouped_xyz], dim=-1)
            else:
                grouped_points = grouped_xyz

            grouped_points = grouped_points.permute(0, 3, 2, 1)  # [B, D, K, S]
            for j in range(len(self.conv_blocks[i])):
                conv = self.conv_blocks[i][j]
                bn = self.bn_blocks[i][j]
                grouped_points =  F.relu(bn(conv(grouped_points)))
            new_points = torch.max(grouped_points, 2)[0]  # [B, D', S]
            new_points_list.append(new_points)

        new_xyz = new_xyz.permute(0, 2, 1)
        new_points_concat = torch.cat(new_points_list, dim=1)
```

