# 特有概念

-  **Motion算子：**除了常见的数据库操作（例如表扫描，联接等）之外，Greenplum数据库还有一种名为motion的算子。motion用于在segment之间移动元组。
-  **Slice：**为了在查询执行期间实现最大的并行度，Greenplum将查询计划的工作划分为slices。slice是计划中可以独立进行处理的部分。查询计划会为motion生成slice，motion的每一侧都有一个slice。
-  **Gang：**属于同一个slice但是运行在不同的segment上的进程，称为gang。

- QE / QD 