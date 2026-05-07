<!-- TOC -->
- [第一部分 C语言语法风格梳理](#第一部分-c语言语法风格梳理)
  - [1.1比对主入口函数（USalign.cpp）](#11比对主入口函数usaligncpp)
  - [1.2 核心算法实现函数](#12-核心算法实现函数)
    - [1.2.1坐标变换与叠加](#121坐标变换与叠加)
    - [1.2.2 序列比对](#122-序列比对)
    - [1.2.3 结构比对核心](#123-结构比对核心)
  - [1.3 IO与工具函数](#13-io与工具函数)
    - [1.3.1内存管理](#131内存管理)
    - [1.3.2 文件读取与解析](#132-文件读取与解析)
    - [1.3.3 结果输出](#133-结果输出)
    - [1.3.4 通用工具](#134-通用工具)
- [第二部分 C风格代码整改为C++风格影响分析(重点关注性能)](#第二部分-c风格代码整改为c风格影响分析重点关注性能)
  - [2.1 正面性能提升](#21-正面性能提升)
  - [2.2 中性性能影响](#22-中性性能影响)
  - [2.3 负面性能影响](#23-负面性能影响)
- [第三部分 US-align 项目基础层优化（不修改算法）](#第三部分-us-align-项目基础层优化不修改算法)
  - [3.1 批量处理无多线程并行](#31-批量处理无多线程并行)
  - [3.2 核心计算循环向量化](#32-核心计算循环向量化)
  - [3.3 Kabsch 算法自实现未优化](#33-kabsch-算法自实现未优化)
  - [3.4 序列比对动态规划缓存效率低](#34-序列比对动态规划缓存效率低)
  - [3.5 IO与PDB解析冗余开销](#35-io与pdb解析冗余开销)
  - [3.6 性能优化总结](#36-性能优化总结)
- [第四部分 US-align编码规范优化](#第四部分-us-align编码规范优化)
  - [4.1 可读性优化](#41-可读性优化)
  - [4.2 可维护性优化](#42-可维护性优化)
  - [4.3 用结构体封装参数，消除超长参数列表](#43-用结构体封装参数消除超长参数列表)
  - [4.4 用RAII(Resource Acquisition Is Initialization, 资源获取即初始化）容器代替手动内存管理，消除内存泄漏](#44-用raiiresource-acquisition-is-initialization-资源获取即初始化容器代替手动内存管理消除内存泄漏)
  - [4.5 补充单元测试，保证修改不破坏原有逻辑](#45-补充单元测试保证修改不破坏原有逻辑)
  - [4.6 健壮性优化](#46-健壮性优化)
- [后续工作计划](#后续工作计划)
  - [5.1 梳理us-align 项目源码架构(~2周左右)](#51-梳理us-align-项目源码架构2周左右)
  - [5.2 针对项目补充单元测试用例，为后续重构和优化结果兜底（待定）。*动代码先完成*](#52-针对项目补充单元测试用例为后续重构和优化结果兜底待定动代码先完成)
<!-- /TOC -->

##

## 第一部分 C语言语法风格梳理

### 1.1比对主入口函数（USalign.cpp）

  -----------------------------------------------------------------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| int TMalign(...) | char*, double**, double[3], double[3][3] | USalign.cpp |
| int MMalign(...) | char*, double**, int**, double[3], double[3][3] | USalign.cpp |
| int SOIalign(...) | char*, double**, int**, int*, double*, double[3], double[3][3] | USalign.cpp |
| int flexalign(...) | char*, double**, double[3], double[3][3] | USalign.cpp |
| int mTMalign(...) | char*, double**, double[3], double[3][3] | USalign.cpp |
| int MMdock(...) | char*, double**, double[3], double[3][3] | USalign.cpp |
| int main(int argc, char *argv[]) | char*[], double**, int*, double[3], double[3][3] | USalign.cpp |
  -----------------------------------------------------------------------

### 1.2 核心算法实现函数

#### 1.2.1坐标变换与叠加

  ---------------------------------------------------------------------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| bool Kabsch(double **x, double **y, int n, int mode, double *rms, double t[3], double u[3][3]) | double**, double*, double[3], double[3][3] | Kabsch.h |
| void do_rotation(double **x, double **x1, int len, double t[3], double u[3][3]) | double**, double[3], double[3][3] | basic_fun.h |
| void transform(double t[3], double u[3][3], double *x, double *x1) | double[3], double[3][3], double* | basic_fun.h |
  ---------------------------------------------------------------------------

#### **1.2.2 序列比对**

  ------------------------------------------------------------------------------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| int NWalign_main(const char *seqx, const char *seqy, ...) | const char*, int* | NWalign.h |
| void trace_back_gotoh(const char *seqx, const char *seqy, int **JumpH, int **JumpV, int **P, ...) | char*, int**, int* | NWalign.h |
| void init_gotoh_mat(int **S, int **JumpH, int **JumpV, int **P, int **H, int **V, ...) | int** | NWalign.h |
| void get_seqID(int *invmap, const char *seqx, const char *seqy, ...) | int*, char* | NWalign.h |
  ------------------------------------------------------------------------------------

#### **1.2.3 结构比对核心**

  ------------------------------------ -------------------------- -------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| int TMalign_main(double **xa, double **ya, char *seqx, char *seqy, ...) | double**, char*, double[3], double[3][3] | TMalign.h |
| int CPalign_main(double **xa, double **ya, char *seqx, char *seqy, ...) | double**, char*, double[3], double[3][3] | TMalign.h |
| int SOIalign_main(double **xa, double **ya, double **xk, double **yk, ...) | double**, char*, int**, int*, double[3], double[3][3] | SOIalign.h |
| int flexalign_main(double **xa, double **ya, char *seqx, char *seqy, ...) | double**, char*, double[3], double[3][3] | flexalign.h |
| int se_main(double **xa, double **ya, char *seqx, char *seqy, ...) | double**, char*, int* | se.h |
| void make_sec(double **x, int len, char *sec) | double**, char* | TMalign.h |
  ------------------------------------ -------------------------- -------------

### 1.3 IO与工具函数

#### **1.3.1内存管理**

  --------------------------------------------- ----------------------------- -------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| template <class A> void NewArray(A ***array, int Narray1, int Narray2) | A***（任意类型二维指针） | basic_fun.h |
| template <class A> void DeleteArray(A ***array, int Narray) | A***（任意类型二维指针） | basic_fun.h |
  --------------------------------------------- ----------------------------- -------------

#### 1.3.2 文件读取与解析

  -------------------------------------------------- -------------- -------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| int read_PDB(const vector<string> &PDB_lines, double **a, char *seq, ...) | double**, char* | basic_fun.h |
| void read_user_alignment(vector<string>&sequence, const string &fname_lign, ...) | char*, FILE* | basic_fun.h |
| void file2chainlist(vector<string>&chain_list, ...) | char*, FILE* | basic_fun.h |
| void file2chainpairlist(vector<string>&chain1_list, vector<string>&chain2_list, ...) | char*, FILE* | basic_fun.h |
  -------------------------------------------------- -------------- -------------

#### **1.3.3 结果输出**

  ------------------------------------------ -------------------- -------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| void output_results(..., double t0[3], double u0[3][3], const char *seqM, ...) | char*, double[3], double[3][3], FILE* | TMalign.h |
| void output_NWalign_results(..., const char *chainID1, const char *chainID2, ...) | char* | NWalign.h |
| void output_flexalign_results(...) | char*, double[3], double[3][3] | flexalign.h |
  ------------------------------------------ -------------------- -------------

#### 1.3.4 通用工具

  ------------------------------------------------ -------------- -------------
| 函数名 | 用到的 C 语言数据结构 | 所在文件 |
|--------|----------------------|----------|
| double dist(double x[3], double y[3]) | double[3] | basic_fun.h |
| double dot(double *a, double *b) | double* | basic_fun.h |
| void assign_sec_bond(int **secx_bond, const char *secx, const int xlen) | int**, char* | SOIalign.h |
| void getCloseK(double **xa, const int xlen, const int closeK_opt, double **xk) | double** | SOIalign.h |
  ------------------------------------------------ -------------- -------------

## 第二部分 C风格代码整改为C++风格影响分析(重点关注性能)

US-align属于计算密集型项目，核心开销在浮点运算、矩阵计算、比对算法，数据结构封装的开销占比极低，性能绝大部分取决于**算法本身**和**编译器向量化优化**。替换后的性能影响可以分为三类。

### 2.1 正面性能提升

1.  **字符串处理优化**：std::string自带小字符串优化（SSO），短字符串（≤15字节）栈上分配，比C风格char\*堆分配开销更小；字符串拼接、查找等操作比手动C实现更高效。

2.  **无额外内存管理开销**：STL容器（vector/array）底层内存分配本质是malloc/free的封装，和项目现有自定义NewArray/DeleteArray模板开销基本一致，且移动语义可避免不必要的内存拷贝。

3.  **IO性能持平（优化后）**：C++ fstream默认同步C
    stdio时比FILE\*慢，但关闭同步后（ios::sync_with_stdio(false);），性能比C风格IO块。

### **2.2 中性性能影响**

**1.
核心计算逻辑**：核心比对算法、Kabsch矩阵变换、坐标运算等核心浮点计算逻辑，替换数据结构后，编译器生成的机器码和C风格几乎完全一致，无性能损失。

2.  **连续内存访问：**vector内存连续，和C风格double\*\*/int\*\*访问效率相同；std::array\<double,3\>替换C固定数组double\[3\]，栈上分配效率完全一致。

**3. 只读字符串传递：**用std::string_view（C++17）替换const
char\*传递只读字符串，无任何额外开销，且自带长度信息，避免重复strlen调用。

### **2.3 负面性能影响**

**1.
Debug模式边界检查：**Debug模式下STL容器默认开启边界检查，有极微小开销，Release模式下完全关闭，无影响。

**2.
极小对象容器开销**：对于长度\<4的极小数组用vector比C数组有极少量元数据开销，可通过std::array避免。

## 第三部分 US-align 项目基础层优化（不修改算法）

### 3.1 批量处理无多线程并行

**现状**：

- US-align批量比对模式（-dir/-dirpair参数）默认单线程执行，每个结构对的比对完全独立无依赖，但没有利用多核CPU并行处理。

**影响场景：**

- 大规模库检索、批量比对任务，可以最大程度的利用计算机算力。

**优化收益**：

- 添加OpenMP（无需手动写OpenMP指令）并行后可获得接近线性的性能提升。

### **3.2 核心计算循环向量化**

核心循环包括：距离计算、TM-score累加、协方差矩阵计算等，这部分官方已经通过开启-O3
-ffast-math已经让编译器自动完成了简单循环的向量化。

**现状：**

- 坐标存储为double\*\*二维指针，行内存不连续，可能会阻碍向量化优化。

- 没有打开-march，可以再配合-march=native适配当前CPU的最高指令集。

  优化方向：

- 二级指针改成一维连续内存存储（double coord\[N\*3\]），保证内存对齐。

- 代码中手动打开AVX2/AVX-512指令集优化

template \<class A\> void NewArray(A \*\*\* array, int Narray1, int
Narray2)

{

    \*array=new A\* \[Narray1\];

    for(int i=0; i\<Narray1; i++) \*(\*array+i)=new A \[Narray2\];

}

上述内存分配函数中，实际上数组中每行的数据之间的地址是连续的。

但是行与行之间的不是连续的。

一个可视化的例子：

假设 A 是 float（占4字节），Narray2 是 4。当执行 new A [4] 时，内存是这样的：

以第 1 行 (Row 1) 为例：

```text
地址: 0x9000   0x9004   0x9008   0x900C
内容: [数据1,0] [数据1,1] [数据1,2] [数据1,3]
      ↑        ↑        ↑        ↑
      |-------- 连续的 4 个 float，地址每次严格 +4 --------|
```

但是，第2行的4个元素的地址可能就是：

```text
地址: 0x3F00   0x3F04   0x3F08   0x3F0C
内容: [数据2,0] [数据2,1] [数据2,2] [数据2,3]
```

两行之间的地址相差较远。

### **3.3 Kabsch 算法自实现未优化**

结构叠加核心的Kabsch算法（求解最优旋转平移矩阵）目前是自实现的简化版，没有利用优化的线性代数库：

**现状：**

- 手写三次方程导致极其昂贵的超越函数调用

sqrth = sqrt(h);

d = atan2(sqrt(d), -g) / 3.0;

sth = sqrth\*sqrt3\*sin(d);

e\[0\] = (spur + cth) + cth;

在现代 CPU 上，一次浮点乘法或加法只需 3-5 个时钟周期，而 atan2、cos、sin
这类超越函数，底层通常需要执行泰勒展开或特定的硬件近似算法，单次调用高达
50\~100 个时钟周期。 这是为了用解析法求 3x3
矩阵的特征值，强行引入了四个数学库函数调用。相比之下，现代标准做法（3x3
Jacobi SVD）全程只需要加减乘除和开方，开销节省很多

- double \*\*x 导致的读取开销

    for (i = 0; i\<n; i++)

    {

        for (j = 0; j \< 3; j++)

        {

            c1\[j\] = x\[i\]\[j\];

            c2\[j\] = y\[i\]\[j\];

两次指针解引用；

外层切换点 i：由于 double \*\*
导致的间接寻址和内存不连续，每次迭代大概率触发 Cache
Miss，阻断了硬件预取。

**影响场景**：

- 长序列蛋白/核酸比对、迭代次数多的高精度比对模式。

**优化收益：**

- 消除指令级并行（ILP）阻塞和流水线停顿。现代 CPU
  是超标量架构，喜欢同时发射多条互不依赖的加减乘指令。atan2/sin/cos
  这种长达几十个周期的复杂指令，会像塞车一样把后面的计算单元堵死（流水线停顿）。换成全是简单乘加的
  Jacobi 后，CPU 可以满负荷运转。

- double \*\* -\> 扁平化连续内存 double
  x\[N\*3\]。小规模数据收益不明显。主要对于大长度蛋白质分子。消除取数据时指令跳转开销，发挥计算机的预取器能力。

### **3.4 序列比对动态规划缓存效率低**

Needleman-Wunsch序列比对的动态规划实现缓存命中率低：

**现状：**

- 动态规划矩阵用int\*\*二维指针实现，每行单独分配，内存不连续，缓存 miss
  率高。

- 分块计算优化，内存访问模式没有利用空间局部性。

**优化收益：**

- 改为连续一维数组存储、分块动态规划实现，序列比对部分可以得到提速。

### **3.5 IO与PDB解析冗余开销**

批量处理场景下，PDB解析和IO的占比会上升.

**现状**：

- 链ID过滤O(n)线性查找开销。代码对每个ATOM/HETATM行都通过std::find在vector\<string\>中线性匹配链ID，单原子查找复杂度O(n)，对大复合物且指定多链过滤的场景，开销占解析总耗时的比例较长。

- 字符串处理冗余开销。

  逐行读取产生临时std::string对象，字段提取通过多次substr()、字符串比较操作，存在大量短生命周期字符串的创建、拷贝和内存分配/释放开销。
  mmCIF格式解析需要先将整行按空格拆分为vector\<string\>，再通过字段名映射查找位置，比PDB格式解析多一层字符串拆分开销。

- IO层固有开销：

  .gz/.bz2压缩文件通过外部gunzip/bzcat进程管道读取，存在进程间通信开销。

  所有文件均采用标准流逐行读取，无内存映射支持，大文件读取时存在频繁的系统调用和内核态/用户态数据拷贝。

  自动格式检测逻辑在识别到mmCIF格式时会递归调用get_PDB_lines重新解析，存在重复文件打开和头部读取的冗余开销。

  **优化收益：**

  优化解析逻辑、用unordered_set做链ID过滤、内存映射IO后，解析部分提提速。

### **3.6 性能优化总结**

  -----------------------
          优化点

    批量处理OpenMP并行

      核心循环向量化

    Kabsch替换BLAS实现

     动态规划缓存优化

        IO解析优化
  -----------------------

上述优化都属于实现层面优化，不修改算法逻辑，理论上对最终比对结果无任何影响。

具体热点函数例如：TMscore8_search、detailed_search、

score_fun8、NWDP_TM、score_matrix_rmsd_sec等还需详查看实现后给出具体优化方案。

## 第四部分 US-align编码规范优化

结合当前US-align源码的实际状况（超大单文件、C/C++混写、大量魔法数字/全局变量、接口不统一），从可读性、可维护性、健壮性三个维度的渐进式重构优化建议如下。

### **4.1 可读性优化**

1\. 消除魔法数字，用枚举/命名常量代替

**现状**：

代码中大量用0/1/2等数字代表比对模式、分子类型、解析选项，比如if (mol_opt
== 1)、if (ter_opt ==0)，完全不知道数字代表的含义。

**优化方案：**

- 用枚举定义所有模式选项：

  enum class MolType { AUTO = 0, PROTEIN = -1, RNA = 1 };

  enum class TerOpt { IGNORE_TER = 0, PARSE_FIRST_TER = 1, PARSE_ALL_TER
  = 2 };

- 用const/constexpr定义常量代替硬编码的数字，比如constexpr int
  MAX_SEQ_LEN = 100000。

**收益：**代码自解释，不需要查文档就能知道判断逻辑的含义，减少参数理解错误。

2.  超长函数拆分，单一职责

**现状**：

- 很多核心函数超过1000行，比如TMalign_main()有1500行，一个函数里做了参数解析、内存分配、迭代比对、结果输出所有逻辑，很难读懂。

**优化方案：**

- 按功能拆分成多个子函数，比如拆分为init_align_context()、generate_initial_alignment()、iterative_align()、output_align_result()，每个函数不超过100行，只做一件事。

**收益：**逻辑清晰，定位问题的时间减少70%，修改某个子逻辑不会影响其他部分。

### **4.2 可维护性优化**

1\. 模块化拆分，头文件与实现分离

**现状**：

当前项目所有源码、头文件、工具、文档、二进制文件全部堆放在根目录下，存在以下问题：

- 无模块化划分，核心算法、基础库、辅助工具、第三方依赖混放

- 可维护性差，查找特定功能需要遍历根目录所有文件

- 构建不规范，编译生成的二进制文件与源码混存

- 功能定位模糊，新用户难以快速区分核心程序和辅助工具

**收益**：

- 清晰的模块边界：核心算法、基础库、工具代码、文档、数据完全分离

- 易于维护扩展：新增算法或工具只需在对应目录下添加，不会污染根目录

- 构建规范：编译产物统一输出到\`build/\`目录，可直接在\`.gitignore\`中全局忽略

- 用户友好：新用户可以快速定位核心程序、工具和文档资源

**整改后目录结构：**

US-align/

├── CMakeLists.txt                     \# 顶层 CMake 构建脚本

├── README.md                          # 项目介绍与使用说明

├── LICENSE                            # 许可证文件

├── usalign/       \# 命名空间隔离

│   ├── core/       \# 核心算法模块

│   │   ├── .cpp    # 核心比对算法的函数实现

│   │   └── .h      # 各对比函数的声明

│   ├── seq/        # 序列比对模块

\|   \|   \|------ .cpp    #具体比对函数的实现

│   │   └── .h      # 比对函数的声明

│   ├── math/       \# 数学与几何模块

│   │   ├── .cpp   \# dist, dot, transform, do_rotation等工具函数的实现

│   │   └── .h      # 函数声明

      \...\...\.....

├── tests/                         \# 单元测试

│   ├── CMakeLists.txt

│   ├── test_kabsch.cpp

│   ├── test_tmscore.cpp

│   ├── test_nw_align.cpp

│   └── \...

│

├── examples/                          # 示例脚本与测试用例

│   ├── *1cpc.pdb*

│   ├── *1mba.pdb*

│   └── run_example.sh

│

└── docs/

    ├── user_guide.md

    └── developer_guide.md

### **4.3 用结构体封装参数，消除超长参数列表**

**现状**：

核心函数参数超长，比如TMalign_main()有18个参数，大量是bool开关，调用的时候只能一个个传参数，很容易传错顺序。

        TMalign_main(xa_h, ya_h, seqx_h, seqy_h, secx_h, secy_h, t0, u0,

            TM1_h, TM2_h, TM3_h, TM4_h, TM5_h, d0_0_h, TM_0_h, d0A_h,
d0B_h,

            d0u_h, d0a_h, d0_out_h, seqM_h, seqxA_h, seqyA_h, do_vec,

            rmsd0_h, L_ali_h, Liden_h, TM_ali_h, rmsd_ali_h, n_ali_h,
n_ali8_h,xlen_h, ylen_h, sequence, Lnorm_ass, d0_scale, i_opt, a_opt,
u_opt, d_opt, fast_opt, mol_type);

**优化方案：**

- 封装AlignOptions结构体，把所有比对选项放在结构体里：

struct AlignOptions

{

MolType mol_type = MolType::AUTO;

TerOpt ter_opt = TerOpt::IGNORE_TER;

bool enable_secondary_structure = false;

int max_iterations = 20;

double tm_score_threshold = 0.5;

};

- 封装AlignContext结构体，把内存缓冲区、中间变量放在里面，避免重复传递参数。

- 函数参数从18个减少到2-3个：int tm_align_main(AlignContext\* ctx, const
  AlignOptions\* opt, AlignResult\* result)。

**收益：**

- 参数传递错误减少，新增选项只需要修改结构体，不需要修改所有函数的参数列表。

### **4.4 用RAII(Resource Acquisition Is Initialization, 资源获取即初始化）容器代替手动内存管理，消除内存泄漏**

**现状：**

- 用自定义的NewArray/DeleteArray手动分配/释放二维数组，很多异常路径下没有释放内存，容易出现内存泄漏，批量运行时间长了会占满内存。

**优化方案：**

- 用std::vector、std::array代替手动分配的double\*\*、int\*\*，自动管理内存，不需要手动释放。

**收益：**

- 完全消除内存泄漏，代码更简洁，不需要写大量的DeleteArray清理逻辑。

### **4.5 补充单元测试，保证修改不破坏原有逻辑**

**现状：**

- 没有单元测试，修改代码后只能靠手动比对几个用例验证正确性，很容易改出隐性bug。

**优化方案**：

- 补充核心模块的单元测试，比如：

Kabsch算法测试：已知坐标输入，验证输出的旋转矩阵和平移向量是否正确。

TM-score计算测试：已知两个结构的比对结果，验证TM-score计算值和预期一致。

序列比对测试：已知两条序列，验证比对结果的序列一致性和预期一致。

**收益：**

- 修改代码后跑一遍单元测试就能知道有没有破坏原有逻辑，回归测试时间从几小时减少到几分钟，发布新版本的风险降低。

***备注：由于us-align需求迭代慢，这一步可以不用搭建测试框架，但针对重要算法函数必须补充单元测试用例，保证后重构、添加新的功能时，不影响已有功能。***

### **4.6 健壮性优化**

**1. 替换直接exit()的错误处理，支持错误恢复**

**现状：**

- 遇到错误（比如PDB解析失败、内存分配失败）直接调用exit()退出整个程序，批量处理的时候遇到一个坏文件就会导致整个任务中断，已经算完的结果也会丢失。

**优化方案：**

- 用错误码代替直接退出，每个函数返回int类型的错误码，0代表成功，非0代表错误，调用方可以根据错误码做处理，比如跳过错误的PDB文件，记录错误日志后继续运行。

**收益：**

- 批量处理的崩溃率降低，不会因为单个坏文件中断整个任务。

2.  **加入参数合法性校验**

    **现状：**

- 没有参数校验，比如传入空指针、负的序列长度、超过范围的阈值，会直接导致段错误，很难定位问题。

**优化方案：**

- 所有公共函数入口加参数校验，比如检查指针是否为空、长度是否为正、枚举值是否在合法范围内，校验失败返回错误码并打印明确的错误信息，比如ERROR:
  序列长度不能为负: -10。

**收益：**

- 段错误减少，错误信息明确，不需要调试就能知道问题出在哪里。

**3. 优化编译配置，提前发现问题**

**现状：**

- Makefile只有基本的编译选项，没有警告检查、Debug/Release区分，很多隐性问题编译的时候发现不了。

**优化方案：**

分Debug和Release模式：

- Debug模式加-g -fsanitize=address -fsanitize=undefined -Wall
  -Wextra，编译时检查所有警告，运行时检测内存泄漏、越界、未定义行为。

- Release模式加-O3 -march=native -flto，开启最高优化。

- 开启-Werror，把警告当成错误，编译不通过，提前发现代码问题。

**收益：**

- 编译阶段就能发现隐性bug，不需要等到运行时崩溃才发现。

##  后续工作计划

### **5.1 梳理us-align 项目源码架构(\~2周左右)**

- 整理出us-align项目源码架构流程图

- 根据文献整理主要算法的实现原理，整理算法源码流程

- 梳理完成后输出一份详细的技术实现方案。

- 根据算法现状，提出响应优化建议，并给出具体可落地的实施方案。

### **5.2 针对项目补充单元测试用例，为后续重构和优化结果兜底（待定）。*动代码先完成***

- 主要针对重点算法函数、各个比对函数。

  **5.3 将项目中C语言编码风格的代码片段转为为C++（待定）**

将第一部分梳理出来的全部点，等价转化为C++语法。只做语法层面的替换。

**5.4 优化源码项目可读性问题（\~2周）**

- 整改超大函数，使每个函数的功能更加聚焦，每个函数尽量不超过100行。

- 函数入参数量整理，每个函数入参保持在5个左右。

- 项目源码组织形式整改。按照模块化改造。

**5.5 启动性能优化**

1\. 输出us-align 火焰图

根据火焰分析工具性能瓶颈

3.  针对数值计算部分采用GPU加速技术

    柔性结构比对、mtmalign整合到us-align