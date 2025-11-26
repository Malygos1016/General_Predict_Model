# General_Predict_Model
输入数据，根据拟合类型计算出参数，再根据参数计算出温度漂移修正系数，用于解决传感器温度漂移问题。

# C#调用的实际方式
public static class GpmNative
{
    private const string DllName = "GPM.dll";

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern void GPM_ClearBuffer();

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern int GPM_AddSample(double x, double y, double T);

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern int GPM_FitFromBuffer(int innerOrder, int outerOrder);

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern double GPM_PredictCached(double x, double T);
}

// 1. 开始一轮新的采样
GpmNative.GPM_ClearBuffer();

// 2. 每采到一个点，就丢给 DLL： (自变量1, 自变量2, 温度)
GpmNative.GPM_AddSample(x1, y1, T1);
GpmNative.GPM_AddSample(x2, y2, T2);
// ... 想加多少就加多少

// 3. 采完之后，告诉 DLL 去拟合
int err = GpmNative.GPM_FitFromBuffer(innerOrder: 2, outerOrder: 2);
if (err != 0)
{
    // TODO：在 UI 上提示 “拟合失败，错误码 = err”
}

// 4. 之后随时预测：给当前 (x, T)，得到 y
double y_est = GpmNative.GPM_PredictCached(x_now, T_now);

# GPM.dll 使用方法与说明

> 版本：v0.1（两层多项式拟合 + 内部缓冲）  
> 说明：输入数据，根据拟合类型计算出参数，再根据参数计算出温度漂移修正系数，用于解决传感器温度漂移问题。 
> `y = f(x, T)`，并在运行时根据 `(x, T)` 预测 `y`。

---

## 1. 整体思路

应用的场景可以抽象成：

- 有很多温度点：`T0, T1, T2, ...`
- 每个温度下都有一堆数据点 `(x, y)`，比如：
  - `T = -10℃` 时有：`(x0, y0), (x1, y1), ...`
  - `T = 0℃` 时有：`(x0, y0), (x1, y1), ...`
  - …

GPM.dll 会做两件事：

1. **内层拟合（对 x）**

   对每个固定温度 `T_i`，拟合一条多项式曲线：

   \[
   y(x;T_i) \approx a_0(T_i) 
             + a_1(T_i)x 
             + \dots 
             + a_n(T_i)x^n
   \]

   这里 `n = innerOrder`，由用户指定阶数。

2. **外层拟合（对 T）**

   再把每个系数 `a_k(T_i)` 看成随温度变化的一组点，拟合：

   \[
   a_k(T) \approx c_{k0} 
              + c_{k1}T 
              + \dots 
              + c_{k,m}T^m
   \]

   这里 `m = outerOrder`，由用户指定阶数。

最终形成一个统一模型：

\[
y(x, T) \approx 
\sum_{k=0}^{n} \left(
  \sum_{j=0}^{m} c_{kj} T^j
\right) x^k
\]

所有 `c_{kj}` 会被压扁存进一个数组 `model[]` 里。  
DLL 会把这些细节都管好，你只需要喂 `(x, y, T)`，最后用 `(x, T)` 就能预测 `y`。

---

## 2. 函数总览

GPM.dll 分两层接口：

- **底层接口**：一次性传大数组（`temps/counts/xs/ys`），适合批处理、工具化。
- **高层接口（推荐）**：像状态机一样一点点 `AddSample(x, y, T)`，DLL 内部缓冲并拟合，更好用。

### 2.1 高层接口（推荐使用）

```cpp
// 1. 清空内部缓冲与模型
void GPM_ClearBuffer();

// 2. 追加一条样本数据 (x, y, T)
int GPM_AddSample(double x, double y, double T);

// 3. 使用当前所有样本数据，执行两层拟合并缓存模型
int GPM_FitFromBuffer(int innerOrder, int outerOrder);

// 4. 使用“缓存的模型”进行预测：传入 x 和 T，返回 y
double GPM_PredictCached(double x, double T);
```

**在 C# 端你可以做到：**

1. `GPM_ClearBuffer()` 开新一轮；
2. 每采到一点，`GPM_AddSample(x, y, T)`；
3. 样本采集完，`GPM_FitFromBuffer(innerOrder, outerOrder)` 做一次拟合；
4. 运行阶段：随时 `GPM_PredictCached(x, T)` 得到 `y`。

---

### 2.2 底层接口（可选，用于高级控制 / 工具）

```cpp
// 简单加法测试
double Add(double a, double b);

// 单层多项式拟合（SVD 实现）
int FitPolynomial(
    const double* xs,
    const double* ys,
    int           n,
    int           order,
    double*       outCoeffs);

// 两层拟合（一次性传 temps/counts/xs/ys）
int GPM_FitTwoLayer(
    const double* temps,
    const int*    counts,
    const double* xs,
    const double* ys,
    int           numTemps,
    int           innerOrder,
    int           outerOrder,
    double*       outModel,
    int           outModelLen);

// 使用指定模型预测 y(x, T)
double GPM_Predict(
    double        x,
    double        T,
    const double* model,
    int           innerOrder,
    int           outerOrder);
```

如果你以后想做“离线拟合工具 exe”，直接用底层接口会更灵活。

---

## 3. 高层接口详细说明（你主要用这一组）

### 3.1 `GPM_ClearBuffer`

```cpp
void GPM_ClearBuffer();
```

- **作用**：清空内部采样缓冲和已有模型。
- **何时调用**：每次开始新的一轮采样之前。

调用后：

- 之前通过 `GPM_AddSample` 加入的所有 `(x, y, T)` 点会被清空；
- 之前拟合出的模型无效，需要重新拟合。

---

### 3.2 `GPM_AddSample`

```cpp
int GPM_AddSample(double x, double y, double T);
```

- **输入参数**：
  - `x`：自变量 1（如厚度、距离等）
  - `y`：目标值 / 自变量 2（如电压、测量值等）
  - `T`：温度（或你选择作为“外层变量”的参数）

- **行为**：
  - DLL 内部会把 `(x, y, T)` 存进一个全局缓冲 `g_samples` 中；
  - 本函数不做拟合，只是存数据；
  - 每次调用相当于 `push_back({T, x, y})`。

- **返回值**：
  - `0`：成功
  - `<0`：失败（例如内部异常，当前实现简单返回 `-1`）

> 你可以连续调用很多次，只要存得下内存，DLL 就会帮你存着。

---

### 3.3 `GPM_FitFromBuffer`

```cpp
int GPM_FitFromBuffer(int innerOrder, int outerOrder);
```

- **输入参数**：
  - `innerOrder`：内层多项式阶数（`y` 对 `x`）
    - 1 → 一次：`a0 + a1 x`
    - 2 → 二次：`a0 + a1 x + a2 x²`
    - 建议 1 ~ 4 之间
  - `outerOrder`：外层多项式阶数（`a_k` 对 `T`）
    - 同上，建议 1 ~ 4

- **内部做了什么**（自动完成的数据处理流程）：
  1. 检查缓冲是否有数据，没有则返回错误；
  2. 把所有样本按温度 `T` 排序；
  3. 根据温度值把样本分组，构造：
     - `temps[]`：不同温度值
     - `counts[]`：每个温度下的数据点数
     - `xs[]/ys[]`：按温度拼接的扁平数组
  4. 调用底层 `GPM_FitTwoLayer(...)` 做两层拟合；
  5. 把拟合好的模型参数 `model[]` 缓存在 DLL 内部：
     - `g_model`：保存所有 `c_kj`
     - 记录 `g_innerOrder` / `g_outerOrder`
     - 将 `g_modelValid` 标记为 `true`。

- **返回值**：
  - `0`：成功，模型已准备好，可以用 `GPM_PredictCached`
  - `-1`：没有采样数据
  - `-2`：阶数不合法（非 1~4）
  - `-3`：分组后数据异常（一般不会出现，除非样本为 0）
  - 其它负数：底层拟合错误，例如：
    - 内层点数不足
    - 温度点太少
    - SVD 拟合失败（极端病态）

> 简单理解：  
> **采样阶段只调用 GPM_AddSample，采完之后调用 GPM_FitFromBuffer，一次性把所有数据变成一个“统一模型”。**

---

### 3.4 `GPM_PredictCached`

```cpp
double GPM_PredictCached(double x, double T);
```

- **输入参数**：
  - `x`：要预测的自变量 1
  - `T`：要预测时对应的温度（或外层变量）

- **内部做的事**：
  1. 检查是否已经成功调用过 `GPM_FitFromBuffer` 并得到有效模型；
  2. 如果模型有效：
     - 根据缓存的 `g_model`、`g_innerOrder`、`g_outerOrder`，
       - 先算出当前温度下的每个 `a_k(T)`；
       - 再计算 `y(x, T) = Σ a_k(T) x^k`；
  3. 返回 `y`。

- **返回值**：
  - 正常情况下：预测的 `y`；
  - 如果模型无效（没拟合过或拟合失败）：当前实现返回 `NaN`（`quiet_NaN()`）。

> 用法就一句话：  
> **前面辛苦拟合出来的模型，全部藏在 DLL 内部了，之后你只需要喂 `(x, T)`，问它要 `y` 就行。**

---

## 4. 典型使用流程（C# 视角）

### 4.1 P/Invoke 声明示例

```csharp
using System;
using System.Runtime.InteropServices;

public static class GpmNative
{
    private const string DllName = "GPM.dll";

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern void GPM_ClearBuffer();

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern int GPM_AddSample(double x, double y, double T);

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern int GPM_FitFromBuffer(int innerOrder, int outerOrder);

    [DllImport(DllName, CallingConvention = CallingConvention.Cdecl)]
    public static extern double GPM_PredictCached(double x, double T);
}
```

### 4.2 一轮完整流程（伪代码）

```csharp
// 1. 开始新一轮采集 / 拟合
GpmNative.GPM_ClearBuffer();

// 2. 采样阶段：每得到一个样本点就喂给 DLL
foreach (var sample in collectedSamples)
{
    double x = sample.X;  // 自变量1
    double y = sample.Y;  // 自变量2 / 输出
    double T = sample.T;  // 温度

    int errAdd = GpmNative.GPM_AddSample(x, y, T);
    if (errAdd != 0)
    {
        // TODO: 提示“追加样本失败，errAdd”
    }
}

// 3. 采样结束，执行两层拟合
int innerOrder = 2;  // 内层二次
int outerOrder = 2;  // 外层二次

int errFit = GpmNative.GPM_FitFromBuffer(innerOrder, outerOrder);
if (errFit != 0)
{
    // TODO: 提示“拟合失败，错误码 = errFit”
}
else
{
    // 4. 运行阶段：只要有 (x, T)，就可以预测 y
    double xNow = 1.5;
    double tNow = 5.0;
    double yEstimated = GpmNative.GPM_PredictCached(xNow, tNow);
}
```

---

## 5. 底层接口简要说明（可选阅读）

如果以后你想自己控制数据分组、直接传大数组，可以用：

### 5.1 `FitPolynomial`

- 输入：`xs[]/ys[]` + 点数 `n` + 阶数 `order`
- 输出：`outCoeffs[]`（长度 `order+1`）
- 数学形式：`y ≈ a0 + a1 x + ... + a_order x^order`

### 5.2 `GPM_FitTwoLayer`

- 输入：
  - `temps[]`：不同温度点，长度 `numTemps`
  - `counts[]`：每个温度下的点数，长度 `numTemps`
  - `xs[]/ys[]`：按温度拼接的点数据，长度 = 总点数之和
  - `innerOrder`/`outerOrder`：内层/外层阶数
- 输出：
  - `outModel[]`：长度 = `(innerOrder+1)*(outerOrder+1)`
- 输出布局：
  - 对于每个 k = 0..innerOrder：
    - `a_k(T) = c_k0 + c_k1 T + ... + c_k,outerOrder T^outerOrder`
    - 对应：
      - `outModel[k*(outerOrder+1) + j] = c_kj`

### 5.3 `GPM_Predict`

- 输入：
  - `x, T`
  - `model[]`：上面说的 `outModel[]`
  - `innerOrder`, `outerOrder`
- 输出：
  - `y(x, T)`。

---

## 6. 编译与平台说明（C++ 侧）

- 编译器：Visual Studio（MSVC）
- 目标类型：**动态库 (.dll)**
- 依赖：
  - [Eigen](https://eigen.tuxfamily.org/)（头文件库）
    - 在项目属性的「C/C++ → 常规 → 附加包含目录」中加入 Eigen 根目录。
- 平台：
  - 建议统一使用 **x64**（C# 工程也设置为 x64）。
- 导出约定：
  - 所有导出函数使用 `extern "C"` + `__declspec(dllexport)`；
  - C# 端使用 `CallingConvention.Cdecl`。

---

## 7. 一些使用上的小建议

1. **多项式阶数的选择**
   - 内层 `innerOrder`：建议 2 或 3，避免过拟合；
   - 外层 `outerOrder`：一般 1 或 2 就够，多了容易抖。

2. **数据点数量**
   - 每个温度至少需要 `innerOrder+1` 个点；
   - 温度点个数至少需要 `outerOrder+1` 个（否则外层拟合不出来那么高阶）。

3. **温度分组的比较**
   - DLL 内部分组用的是一个小阈值 `1e-9`：
     - 认为 `|T_i - T_j| <= 1e-9` 就是同一温度；
   - C# 端最好保证相同温度的数值一致（比如直接用相同 double）。

4. **线程安全**
   - 当前版本使用 **全局静态缓冲**（`g_samples` / `g_model`），**不是线程安全**；
   - 在一个进程里请只在单线程中使用这些接口，或者加你自己的互斥锁。

5. **错误处理**
   - 所有 `int` 返回值 `<0` 都表示异常情况；
   - `GPM_PredictCached` 在模型无效时返回 `NaN`，C# 可通过 `double.IsNaN()` 检查。

---

## 8. 总结

- **采集阶段**：  
  `GPM_ClearBuffer()` → 之后多次 `GPM_AddSample(x, y, T)`
- **拟合阶段**：  
  `GPM_FitFromBuffer(innerOrder, outerOrder)`
- **运行阶段**：  
  多次 `GPM_PredictCached(x, T)` → 得到 `y`

把 C# 端作为“UI + 数据来源”，  
GPM.dll 主要作为数据处理，  
两边职责非常清晰，后面要改模型形式（比如不用多项式，换别的函数形状），基本都只用改 DLL 内部就行。
调试的时候，如果遇到任何一个阶段报错（尤其是 `err` 返回值），可以先在 C# 端打印出来，再根据错误码分析是哪一环节的假设不成立。
