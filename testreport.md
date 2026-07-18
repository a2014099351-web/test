# kcalculator 测试用例验证报告

| 项 | 内容 |
|----|------|
| 项目 | kcalculator |
| 对照文档 | [计算器工程版大项目](https://365.kdocs.cn/l/caAYLeSt7OBy) |
| 方法 | 先验证边界 → 再固化实现 → 自动化回归 |
| 自动化目标 | `test_kcalc`（GoogleTest） |
| 最近全量结果 | **85 / 85 PASSED**（Debug，2026-07-18） |
| 行覆盖率 | **86.8%**（1332 / 1534），OpenCppCoverage 0.9.9.0 |

记录格式：验证项 → 现象 → 处理 → 结果。

---

## 一、边界验证（BV-01 ~ BV-10，全员必做）

| 编号 | 验证项 | 现象 | 处理 | 结果 | 自动化用例 |
|------|--------|------|------|------|------------|
| BV-01 | 空输入直接 `=` | 空/`0` 经显示求值得 `0`；纯空表达式返回 `EmptyExpression`；不崩溃、不保留脏状态 | Validator 拒绝空表达式；Controller 展示 Error + 状态栏文案 | **Pass** | `BoundaryValidation.BV01_EmptyAndWhitespace` |
| BV-02 | 连续运算符（如 `1 + * 2`） | 引擎返回 `ConsecutiveOperators`，不产生错误结果 | 求值前校验拦截非法表达式 | **Pass** | `BoundaryValidation.BV02_ConsecutiveOperators_Variants` |
| BV-03 | 连续小数点（如 `1..2`） | Tokenizer 返回 `InvalidDecimal` | 分词阶段拒绝；UI 侧亦限制单小数点 | **Pass** | `BoundaryValidation.BV03_InvalidDecimal_Variants` |
| BV-04 | 除零（`5/0`） | `DivisionByZero`；UI 提示 “Division by zero”；可继续输入 | `KErrorObject` → 状态栏；错误后 Clear/新输入可恢复 | **Pass** | `BoundaryValidation.BV04_DivisionByZero_Variants` |
| BV-05 | Backspace 到空串 | 显示稳定回到 `0` | Controller Backspace 规则：删空则置 `0` | **Pass** | `ControllerBv.BV05_BackspaceToZero` |
| BV-06 | 极长输入（100+ 字符） | 达到上限后拒绝继续追加；返回 `InputTooLong`；UI/逻辑不卡死 | Controller `kMaxInputLength = 100` 截断策略 | **Pass** | `ControllerBv.BV06_DisplayInputTooLong` / `BV06_ExpressionInputTooLong` |
| BV-07 | 括号不匹配（如 `(1+2`） | 返回 `UnmatchedParentheses`；**不**自动补齐右括号 | Validator 明确报错，避免静默改写用户输入 | **Pass** | `BoundaryValidation.BV07_UnmatchedParentheses_Variants` |
| BV-08 | 键盘非法字符 | 字母等在 `keyPressEvent` 中忽略；引擎侧 `1+a` 被拒绝 | 输入过滤 + Tokenizer 双重防护 | **Pass** | `BoundaryValidation.BV08_IllegalCharacter_Variants`（引擎）；键盘过滤：手工 UI 验证 |
| BV-09 | 计算后继续输入 | 数字：开启新输入（覆盖结果）；运算符：以结果续算 | `afterEquals` 状态机切换，行为一致 | **Pass** | `ControllerBv.BV09_*` |
| BV-10 | 清空 `C` / `CE` | CE 只清当前项；C 清表达式与显示 | 语义与 README 一致，Controller 分支实现 | **Pass** | `ControllerBv.BV10_*` / `ControllerCommands.ClearAndClearEntryDismissError` |

### BV 补充说明

- **BV-07**：作业问「是否自动补齐」——本实现选择**报错不补齐**，避免改变用户意图；现象与处理已记录。
- **BV-08**：UI 过滤与引擎校验分工——非法键不进显示；若表达式字符串被注入非法字符，引擎仍拒绝。

---

## 二、验收样例（最小集合）

### 2.1 基础篇样例

| # | 用例 | 期望 | 实测 | 结果 | 依据 |
|---|------|------|------|------|------|
| 1 | `12 + 3 =` | `15` | `15` | **Pass** | `EvalBasics.SimpleAddition` |
| 2 | `12 + 3 - 4 + 5 =` | `16` | `16` | **Pass** | `EvalBasics.ChainedOps` |
| 3 | `12 + 3 * 6 / 2 =` | `21`（标准优先级） | `21` | **Pass** | `EvalBasics.Precedence`；规则见 README |
| 4 | `1234` 连按 Backspace | `123` → `12` → `1` → `0` | 同左 | **Pass** | `ControllerBv.BV05_BackspaceToZero` + UI |
| 5 | `5 / 0 =` | 有错误提示且不崩溃 | `DivisionByZero` + 状态栏提示 | **Pass** | `BoundaryValidation.BV04_*` |

扩展基础能力抽检：

| 用例 | 期望 | 结果 |
|------|------|------|
| `(1+3)*4+2` | `18` | **Pass**（`EvalBasics.Parentheses`） |
| 平方 / 开方 / 求余 / 求倒 | 与引擎 API 一致 | **Pass**（`UnaryApi.SquareSqrtReciprocalNegateModulo`） |

### 2.2 进阶篇样例

| # | 用例 | 期望 | 实测 | 结果 | 依据 |
|---|------|------|------|------|------|
| 1 | `(1 + 3) * 4 + 2 =` | `18` | `18` | **Pass** | `EvalBasics.Parentheses` |
| 2 | 内存：`45*3` → MS；`78*2` → M+；MR | `291` | `291` | **Pass** | `MemoryStore.MsMplusMr` + UI |
| 3 | 非法表达式 `1 + * 2` | 拒绝并提示 | `ConsecutiveOperators` | **Pass** | `BoundaryValidation.BV02_*` |

内存场景补充（对照作业文档说明）：

| 场景 | 步骤摘要 | 结果 |
|------|----------|------|
| 中间结果复用 | `3+5=` → M+；`8-2=` → `×` MR `=` → `48`；MC | **Pass**（手工 + MemoryStore 单测） |
| 累计金额 | `120` M+；`85` M+；`200` M+；MR → `405` | **Pass** |

---

## 三、自动化测试摘要

构建与运行：

```bat
cmake --build build --config Debug --target test_kcalc
build\bin\Debug\test_kcalc.exe
```

| 套件（节选） | 覆盖点 |
|--------------|--------|
| `BoundaryValidation` | BV-01/02/03/04/07/08 |
| `ControllerBv` | BV-05/06/09/10 |
| `EvalBasics` / `EvalEdges` | 四则、优先级、括号 |
| `MemoryStore` / `HistoryStore` | 内存与历史边界 |
| `UnaryApi` / `ScientificApi` / `EngineApi` | 扩展与科学运算 |
| `LengthConverter` / `WeightConverter` / `ConverterController` | 换算 |
| `ControllerCommands` | 控制器命令与错误恢复 |
| `ErrorApi` | 统一错误码 |

**结论**：全量 **85** 项用例通过；作业要求的边界验证与最小验收样例均已覆盖并记录。

---

## 四、OpenCppCoverage 行覆盖率

工具：[OpenCppCoverage](https://github.com/OpenCppCoverage/OpenCppCoverage) 0.9.9.0  
被测程序：`build\bin\Debug\test_kcalc.exe`（须带 PDB）  
统计范围：`src\` + `include\`（排除 autogen / moc）  
报告产出：`coverage\html\index.html`、`coverage\cobertura.xml`（已加入 `.gitignore`）

### 4.1 复现步骤

```bat
cmake --build build --config Debug --target test_kcalc
run_coverage.bat
```

或手动：

```bat
"C:\Program Files\OpenCppCoverage\OpenCppCoverage.exe" ^
  --sources "kcalculator\src" ^
  --sources "kcalculator\include" ^
  --excluded_sources "autogen" ^
  --excluded_sources "moc_" ^
  --modules "test_kcalc" ^
  --export_type "html:coverage\html" ^
  --export_type "cobertura:coverage\cobertura.xml" ^
  --working_dir "build\bin\Debug" ^
  -- "build\bin\Debug\test_kcalc.exe"
```

### 4.2 总体结果（2026-07-18）

| 指标 | 数值 |
|------|------|
| 可执行行 | 1534 |
| 已覆盖行 | 1332 |
| **行覆盖率** | **86.8%** |
| gtest | 85 / 85 PASSED |

说明：`test_kcalc` 只链接 core + controller，**不含** Widgets UI（`ui/`），因此覆盖率反映的是可单测逻辑层，而非整个 GUI。

### 4.3 主要源文件明细

| 文件 | 覆盖行 | 总行 | 覆盖率 |
|------|--------|------|--------|
| `src/core/engine.cpp` | 326 | 347 | 93.9% |
| `src/core/tokenizer.cpp` | 83 | 89 | 93.3% |
| `src/core/validator.cpp` | 58 | 60 | 96.7% |
| `src/core/error.cpp` | 17 | 18 | 94.4% |
| `src/core/memory_store.cpp` | 40 | 53 | 75.5% |
| `src/core/history_store.cpp` | 12 | 12 | 100% |
| `src/core/converter_utils.cpp` | 38 | 42 | 90.5% |
| `src/core/length_converter.cpp` | 21 | 21 | 100% |
| `src/core/weight_converter.cpp` | 21 | 21 | 100% |
| `src/controller/calculator_controller.cpp` | 542 | 685 | 79.1% |
| `src/controller/converter_controller.cpp` | 101 | 113 | 89.4% |

分层汇总（仅 `.cpp`）：

| 层 | 覆盖 / 总行 | 覆盖率 |
|----|-------------|--------|
| `src/core` | 616 / 663 | **92.9%** |
| `src/controller` | 643 / 798 | **80.6%** |
| 合计（含头文件可执行行） | 1332 / 1534 | **86.8%** |

### 4.4 未覆盖原因（摘要）

- **UI 层**（`ui/`、皮肤、快捷键过滤）未链入 `test_kcalc`，不在本次统计内。
- `calculator_controller.cpp` 约 21% 未覆盖：科学模式部分冷路径、少用命令组合等。
- `memory_store.cpp` 约 24.5% 未覆盖：多槽位边角 / 少用 API。

详细行级高亮见 `coverage\html\index.html`。

---

## 五、需求符合性（简表）

| 编号 | 需求 | 结论 |
|------|------|------|
| B-01 ~ B-10 | 基础功能 | 满足 |
| BNF-01 ~ BNF-04 | Layout / 分层 / README | 满足 |
| A-01 ~ A-05 | 键盘、内存、校验、历史、快捷键、gtest、皮肤 | 满足 |
| ANF-01 ~ ANF-03 | 架构、signal/slot、错误对象 | 满足（见 README 设计说明） |

---

## 六、已知说明

1. BV-07 选择「报错、不自动补括号」，与部分计算器「静默补齐」不同，属有意设计。
2. `%` 语义为求余，已在 README 声明。
3. 可执行文件按课程要求单独打包提交，不纳入 git。
