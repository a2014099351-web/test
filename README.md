# 单位转换引擎

跨平台 C++ 命令行单位转换引擎，完整覆盖「任务1：基础转换引擎」与「任务2：模块化进阶」两项专题要求。

---

## 专题知识点体现

### 任务1：命令行单位转换引擎

| 要求 | 本项目实现 | 对应代码 |
|------|-----------|----------|
| 输入格式 `<数值> <单位> to <单位>` | 词法分析器识别 `to` 关键字，解析「数值+单位」对 | `KExprLexer`、`KExprCalculator` |
| 至少 5 类单位 | 内置 **length / weight / time / area / speed** 五大类及全部参考单位 | `KDefaultUnits::registerBuiltins()` |
| `help` 命令 | 输出语法说明与已注册分类、单位列表 | `KConversionService::buildHelpText()` |
| `quit` / `exit` 命令 | 退出主循环 | `KConversionService::processLine()` |
| 基本错误提示 | 未知单位、跨类运算、语法错误、JSON 错误等统一错误码 | `ErrorCode`、`KErrorMessages` |
| 多用类组织代码 | 领域模型、注册表、转换引擎、表达式、CLI 门面均独立成类 | 见下方「核心类职责」 |

**内置单位（任务1 最低要求）**

| 类别 | 基准单位 | 已支持符号 |
|------|----------|-----------|
| length | m | m, km, cm, mm, mile, ft, inch |
| weight | kg | kg, g, mg, lb, oz |
| time | s | s, min, h, day |
| area | m2 | m2, km2, cm2, ft2 |
| speed | m/s | m/s, km/h, mile/h |

**转换核心思路**：每个单位维护相对基准单位的系数 `toBaseFactor`，换算公式为：

```
结果 = 输入值 × 原单位系数 ÷ 目标单位系数
```

由 `KConvertEngine::convert()` 实现，表达式场景下先归一到基准单位求和，再换算到目标单位。

---

### 任务2：模块化与进阶能力

| 要求 | 本项目实现 | 设计说明 |
|------|-----------|----------|
| 动态库 / 静态库模块化拆分 | **common**（静态库）→ **unitcore**（默认 DLL/SO，可切静态）→ **unitconvert**（可执行文件） | 按**职责层次**拆分：工具层 / 核心算法层 / 交互层 |
| Windows & Linux 跨平台 | `platform.h` 抽象导出宏；`KStopwatch` 分平台计时；`KPathUtil` 处理路径分隔符；CMake 统一构建 | 同一套源码，`build_win.bat` / `build_linux.sh` 一键编译 |
| 表达式计算 | `5 km + 3 mile to m`：词法分析 → 同类项加减 → 统一转换 | `KExprLexer` 分词 + `KExprCalculator` 求值 |
| JSON 配置驱动 | 启动加载 `units_config.json`；运行时 `load <路径>` 热加载；失败回退内置单位 | `KJsonConfigParser`（nlohmann/json）+ `KJsonUnitLoader` |
| 性能评测与优化 | `bench` 命令对比优化前后耗时；默认哈希查表，bench 时可切线性扫描 | `KBenchmarkRunner` + `KStopwatch` |
| 批量转换 | 读测试文件逐条执行，写出 `result.json` | `KBatchRunner` |

**模块化拆分依据**

```
unitconvert.exe                ← 仅负责 stdin/stdout 循环，不含业务算法
        │
        ▼
   unitcore.dll / .so          ← 单位注册、换算、表达式、JSON 合并、性能评测、批量
        │
        ▼
   common.lib                   ← 错误码、JSON 解析、文件读取、计时、路径工具
```

- **common**：与业务无关的通用能力，可被多个模块复用，编译为静态库减少链接复杂度。
- **unitcore**：单位转换领域核心，对外暴露 `KConversionService` 等 API，默认编译为动态库便于独立测试与替换。
- **unitconvert**：极薄的 CLI 入口（`main.cpp`），只负责控制台编码、读行、调用 `KConversionService`。
- **test_uce**：GoogleTest 可执行文件，链接 `unitcore` + `common`，用于逻辑层回归。

**目录结构（节选）**

```
unitconvert/
  ├── CMakeLists.txt
  ├── README.md
  ├── testreport.md
  ├── config/                 # units_config.json / testcases.txt 等
  ├── include/                # common / unitcore 公共头文件
  ├── src/
  │   ├── common/
  │   ├── unitcore/
  │   └── unitconvert/        # main.cpp
  ├── test/gtest/             # GoogleTest 用例
  └── third_party/nlohmann/   # json 单头文件
```

**JSON 配置机制**

- 格式：`{ "categories": [ { "name", "base", "units": [ { "symbol", "to_base" } ] } ] }`
- 启动查找顺序：当前目录 → 可执行文件同目录 → `config/units_config.json`
- 启动时：`initialize()` → 内置五类 → 合并 JSON（可扩展新分类如 `volume`，可覆盖已有单位如 `yd`）
- 运行时：`load config/units_config_test.json` 热加载，路径支持相对 cwd 与 exe 同目录
- 样例：`config/units_config.json`（追加 yd、volume）、`config/units_config_test.json`（data、pressure 等新类）

**性能优化对比（`bench` 命令）**

在 CLI 中输入 `bench`，程序默认执行混合负载（单值转换），对比：

| 阶段 | 实现方式 |
|------|----------|
| 优化前 | `KUnitRegistry::findUnitLinear()` 线性扫描单位表 |
| 优化后 | `unordered_map` 哈希 O(1) 查找；词法缓冲复用；系数启动预注册 |

输出示例：

Windows下：
（Debug）
```
> bench
性能基准（迭代 10000 次，每轮含 4 组 convert / findUnit）
  为放大差异，临时注册表另灌入 100 个假单位（不污染正式表）
  优化前（线性查找）: 40.616 ms，平均 0.004062 ms/轮
  优化后（哈希查找）: 37.464 ms，平均 0.003746 ms/轮
  加速比: 1.084x
```
（Release）
```
> bench
性能基准（迭代 10000 次，每轮含 4 组 convert / findUnit）
  为放大差异，临时注册表另灌入 100 个假单位（不污染正式表）
  优化前（线性查找）: 1.680 ms，平均 0.000168 ms/轮
  优化后（哈希查找）: 0.682 ms，平均 0.000068 ms/轮
  加速比: 2.464x
```
Linux下
（Debug）
```
> bench
性能基准（迭代 10000 次，每轮含 4 组 convert / findUnit）
  为放大差异，临时注册表另灌入 100 个假单位（不污染正式表）
  优化前（线性查找）: 56.002 ms，平均 0.0056  ms/轮
  优化后（哈希查找）: 6.591 ms，平均 0.000659 ms/轮
  加速比: 8.497x
```
（Release）
```
> bench
性能基准（迭代 10000 次，每轮含 4 组 convert / findUnit）
  为放大差异，临时注册表另灌入 100 个假单位（不污染正式表）
  优化前（线性查找）: 3.698 ms，平均 0.00037 ms/轮
  优化后（哈希查找）: 0.558 ms，平均 0.000056 ms/轮
  加速比: 6.631x
```
> 具体数值因机器而异，请在本地运行 `bench` 获取实测数据。
```
---

### 核心类职责

| 类 | 职责 |
|----|------|
| `KUnit` / `KUnitCategory` | 单位与分类领域模型 |
| `KUnitRegistry` | 单位注册、哈希查找、分类索引；bench 时可切线性查找 |
| `KConvertEngine` | 单值单位换算 |
| `KExprLexer` | 表达式词法分析（长符号优先匹配） |
| `KExprCalculator` | 复合表达式求值 |
| `KConversionService` | CLI 门面：命令分发、help、load、bench |
| `KBatchRunner` | 批量用例执行与 `result.json` 输出 |
| `KDefaultUnits` | 内置五大类单位注册 |
| `KJsonConfigParser` / `KJsonUnitLoader` | JSON 解析与合并注册 |
| `KBenchmarkRunner` | 性能基准对比 |
| `KFileUtil` / `KPathUtil` / `KStopwatch` | 文件读取、路径解析、高精度计时 |

### 关键代码设计评价

- **基准单位归一化**：所有换算先归一到分类基准，表达式可直接做加减，避免为每对单位单独建换算表。
- **注册表模式**：单位定义与换算逻辑解耦，内置 + JSON 均通过 `registerUnit()` 注入，符合开闭原则。
- **错误码而非异常**：CLI 场景以返回 `ErrorCode` 为主，控制流清晰，避免异常开销。
- **RAII 资源管理**：全程 `std::string` / `vector` / `ifstream`，无手动 `new/delete`。
- **跨平台导出宏 `UNITCORE_API`**：unitcore 作为 DLL/SO 时正确导出符号，静态链接时自动为空。
- **可测性**：逻辑集中在 `unitcore`/`common`，由 `test_uce` 做回归；验收细节见 `testreport.md`。
