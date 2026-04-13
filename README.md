# 姓名：任雅文-学号：23354131-第三次人工智能编程作业


## 1. 任务拆解与 AI 协作策略

本次作业包含数据预处理、时间分析、统计建模、可视化及文件导出等多个任务。为了保证代码的可控性与正确性，我没有一次性让 AI 生成全部代码，而是采用了**分阶段拆解 + 逐步验证**的策略。

具体流程如下：

1. **任务1（数据预处理）**

   * 先让 AI 生成基础的数据读取与清洗代码
   * 手动检查字段解析是否正确（特别是“交易时间”）
   * 补充异常值（ride_stops=0）删除逻辑

2. **任务2（时间分布分析）**

   * 明确要求 AI 必须使用 numpy 完成条件统计
   * 单独生成柱状图代码，并手动检查是否使用 matplotlib（避免误用 seaborn）

3. **任务3（线路分析）**

   * 强制 AI 按题目要求生成指定函数签名
   * 再补充 seaborn 可视化部分

4. **任务4（PHF计算）**

   * 这是最复杂部分，分三步让 AI 完成：
     1）找高峰小时
     2）做5分钟/15分钟聚合
     3）计算 PHF
   * 手动检查公式是否正确

5. **任务5（文件导出）**

   * 让 AI 生成批量文件输出逻辑
   * 自己检查路径和文件命名规范

6. **任务6（热力图）**

   * 先生成 Top10 统计
   * 再生成 heatmap，并补充中文标签

👉 总体策略：
**AI负责生成代码框架，我负责检查逻辑、修正错误和保证符合评分标准。**

---

## 2. 核心 Prompt 迭代记录

### 初代 Prompt：

> 请用 pandas 完成公交刷卡数据分析，并画出时间分布图

---

### AI 生成的问题：

1. ❌ 使用 seaborn 绘制柱状图（违反任务要求，必须用 matplotlib）
2. ❌ 没有使用 numpy 做时间段统计
3. ❌ PHF 计算直接写死高峰小时（不符合“自动识别”要求）
4. ❌ analyze_route_stops 函数签名不符合题目要求

---

### 优化后的 Prompt：

> 请严格按照以下要求生成代码：
> 1）任务2必须使用 numpy 进行条件统计
> 2）柱状图必须使用 matplotlib.pyplot
> 3）任务3函数 analyze_route_stops 的函数签名必须完全一致
> 4）任务4必须先自动计算高峰小时，再进行5分钟和15分钟聚合（使用 resample）
> 5）所有图必须包含中文标签和标题

---

### 优化结果：

✔ 成功使用 numpy 完成统计
✔ matplotlib 柱状图符合要求
✔ PHF 计算逻辑正确
✔ 函数签名完全符合评分标准

---

## 3. Debug 记录

### 报错现象：

运行代码时报错：

```python
KeyError: '交易时间'
```

同时打印 df.info() 发现：

```text
Data columns (total 1 columns):
'交易类型,交易时间,交易卡号,...'
```

---

### 问题原因：

CSV 文件实际是 **逗号分隔**，但代码中使用了：

```python
pd.read_csv('ICData.csv', sep='\t')
```

导致 pandas 没有正确解析列，所有数据被读入一列。

---

### 解决过程：

1. 修改读取方式：

```python
df = pd.read_csv('ICData.csv', sep=None, engine='python')
```

2. 去除列名空格：

```python
df.columns = df.columns.str.strip()
```

3. 增强时间解析鲁棒性：

```python
df['交易时间'] = pd.to_datetime(df['交易时间'], errors='coerce')
```

---

### 最终结果：

✔ 数据成功分列
✔ 时间字段解析正确
✔ 后续分析正常运行

---

## 4. 人工代码审查（逐行中文注释）

以下为任务4 PHF计算核心代码及人工注释：

```python
# 统计每个小时的刷卡量（只统计上车记录）
hour_counts = df_on['hour'].value_counts()

# 找到刷卡量最大的小时（自动识别高峰）
peak_hour = hour_counts.idxmax()

# 获取该小时的总刷卡量
peak_count = hour_counts.max()

# 输出高峰小时信息
print(f"高峰小时：{peak_hour}:00~{peak_hour+1}:00，刷卡量：{peak_count}")

# 筛选出高峰小时内的所有数据
peak_df = df_on[df_on['hour'] == peak_hour].copy()

# 将时间列设为索引，方便后续按时间重采样
peak_df.set_index('交易时间', inplace=True)

# ======================
# 5分钟粒度统计
# ======================

# 按5分钟时间窗口统计刷卡次数
count_5 = peak_df.resample('5min').size()

# 找到最大5分钟刷卡量
max_5 = count_5.max()

# 找到该最大值对应的时间段
max_5_time = count_5.idxmax()

# 计算PHF5（公式：高峰小时总量 / (12 × 最大5分钟流量)）
PHF5 = peak_count / (12 * max_5)

# ======================
# 15分钟粒度统计
# ======================

# 按15分钟时间窗口统计
count_15 = peak_df.resample('15min').size()

# 找最大15分钟流量
max_15 = count_15.max()

# 对应时间
max_15_time = count_15.idxmax()

# 计算PHF15（公式：高峰小时总量 / (4 × 最大15分钟流量)）
PHF15 = peak_count / (4 * max_15)

# 输出结果
print(f"最大5分钟刷卡量（{max_5_time}）：{max_5}")
print(f"PHF5 = {peak_count} / (12 × {max_5}) = {PHF5:.4f}")

print(f"最大15分钟刷卡量（{max_15_time}）：{max_15}")
print(f"PHF15 = {peak_count} / (4 × {max_15}) = {PHF15:.4f}")
```

---

# ✅ 总结

本次作业通过人机协作完成，AI主要负责代码生成与结构搭建，而我重点参与：

* 数据解析与错误定位
* 关键算法（PHF）逻辑验证
* 可视化规范调整
* Debug 与结果校验

最终实现了从原始数据到分析结论的完整流程，符合课程对“理解 + 协作”的要求。
