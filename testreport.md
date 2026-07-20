# 单位转换引擎 测试用例验证报告

| 项 | 内容 |
|----|------|
| 项目 | unitconvert |
| 方法 | 手工 CLI 验收 + GoogleTest 逻辑层回归 + OpenCppCoverage 行覆盖率 |
| 自动化目标 | `test_uce`（GoogleTest） |
| 最近全量结果 | **33 / 33 PASSED**（Debug） |
| 行覆盖率 | **82.6%**（795 / 963），OpenCppCoverage 0.9.9.0（2026-07-20） |
| 构建 | VS 2019 / C++17 / vcpkg GTest |

记录格式：验证项 → 现象 → 处理 → 结果。

---

## 一、手工 CLI 验收

| 用例 ID | 场景 | 预期 | 结果 | 备注 |
|---------|------|------|------|------|
| TC-01 | `1 km to m` | 输出 `1000 m` | **Pass** | `ConvertEngine.LengthKmToM` / `ServiceTest.ConvertViaProcessLine` |
| TC-02 | `5.5 km to m` | 输出 `5500 m` | **Pass** | `ExprCalculator.SimpleConvert` |
| TC-03 | `3 km + 200 m to km` | 输出 `3.2 km` | **Pass** | `ExprCalculator.CompoundExpression` |
| TC-04 | `1 h + 30 min + 30 s to s` | 输出 `5430 s` | **Pass** | `ExprCalculator.TimeCompoundToSeconds` |
| TC-05 | `help` | 列出语法与已注册分类/单位 | **Pass** | 含 JSON 合并的 `volume` |
| TC-06 | `quit` / `exit` | 退出主循环 | **Pass** | `ServiceTest.QuitAndExit` |
| TC-07 | `5 xyz to m` | 语法/非法输入错误提示 | **Pass** | 词法阶段 `SyntaxError` |
| TC-08 | `1 km to kg` | 跨类别错误 | **Pass** | `ConvertEngine.CrossCategoryRejected` |
| TC-09 | 启动加载 `units_config.json` | 可用 `yd`、`volume` | **Pass** | `ServiceTest.JsonExtendedUnitYard` |
| TC-10 | `load units_config_test.json` | 热加载 `data`/`KB` 等 | **Pass** | `ServiceTest.LoadHotReload` |
| TC-11 | `bench` | 输出线性/哈希对比与加速比 | **Pass** | `ServiceTest.BenchProducesReport` |
| TC-12 | 批量 `testcases.txt` | 写出 `result.json`，含通过/失败 | **Pass** | `ServiceTest.BatchRunFromTestcasesFile` |

---

## 二、自动化测试摘要

构建与运行：

```bat
cmake -S . -B build -G "Visual Studio 16 2019" -A x64
cmake --build build --config Debug --target test_uce
build\Debug\test_uce.exe
```

关闭测试：`cmake -S . -B build -DUCE_BUILD_TESTS=OFF ...`

| 套件 | 覆盖点 | 用例数 |
|------|--------|--------|
| `Registry` | 内置五类、查找、覆盖、长符号优先排序 | 4 |
| `ConvertEngine` | 长度/重量换算、未知单位、跨类拒绝 | 5 |
| `ExprLexer` | 简单分词、`mile` 长符号优先 | 2 |
| `ExprCalculator` | 单值/复合/时间表达式、缺 `to`、跨类、未知符号 | 6 |
| `JsonParser` | 字符串/文件解析、缺失文件、非法 JSON | 4 |
| `JsonLoader` | 合并 `units_config.json` / `units_config_test.json` | 2 |
| `ServiceTest` | help/quit/换算/JSON/load/bench/batch | 9 |
| `ServiceUninit` | 未 initialize 时拒绝 processLine | 1 |

**结论**：全量 **33** 项用例通过；逻辑层（注册表/换算/词法求值/JSON/CLI 门面/bench/batch）已自动化回归。`main.cpp` 交互循环不在 `test_uce` 内。

---

## 三、OpenCppCoverage 行覆盖率

工具：[OpenCppCoverage](https://github.com/OpenCppCoverage/OpenCppCoverage) 0.9.9.0  
被测程序：`build\Debug\test_uce.exe`（链接 `unitcore.dll`）  
统计范围：`src\` + `include\`（排除 `test\` / `third_party\`）


### 3.1 总体结果（2026-07-20）

| 指标 | 数值 |
|------|------|
| 可执行行 | 963 |
| 已覆盖行 | 795 |
| **行覆盖率** | **82.6%** |
| gtest | 33 / 33 PASSED |

说明：`test_uce` **不含** CLI 入口（`src/unitconvert/main.cpp`），覆盖率反映可单测逻辑层（common + unitcore），而非完整交互进程。

### 3.2 主要源文件明细（`.cpp`，按文件去重）

| 文件 | 覆盖行 | 总行 | 覆盖率 |
|------|--------|------|--------|
| `src/unitcore/conversionservice.cpp` | 138 | 152 | 90.8% |
| `src/unitcore/exprlexer.cpp` | 101 | 123 | 82.1% |
| `src/unitcore/batchrunner.cpp` | 66 | 76 | 86.8% |
| `src/unitcore/exprcalculator.cpp` | 65 | 92 | 70.7% |
| `src/common/jsonconfigparser.cpp` | 61 | 89 | 68.5% |
| `src/unitcore/benchmarkrunner.cpp` | 54 | 58 | 93.1% |
| `src/unitcore/unitregistry.cpp` | 41 | 42 | 97.6% |
| `src/unitcore/defaultunits.cpp` | 33 | 33 | 100% |
| `src/common/pathutil.cpp` | 32 | 36 | 88.9% |
| `src/unitcore/jsonunitloader.cpp` | 31 | 44 | 70.5% |
| `src/common/stopwatch.cpp` | 23 | 28 | 82.1% |
| `src/unitcore/convertengine.cpp` | 22 | 25 | 88.0% |
| `src/common/fileutil.cpp` | 14 | 14 | 100% |
| `src/common/errorcode.cpp` | 12 | 20 | 60.0% |
| `src/unitcore/unit.cpp` | 3 | 3 | 100% |
| `src/unitcore/unitcategory.cpp` | 1 | 1 | 100% |

分层汇总（含头文件可执行行；按文件去重后合计 **717 / 857 ≈ 83.7%**，与 OCC 汇总 795/963 的差异来自 `jsonconfigparser.cpp` / `fileutil.cpp` / `uce_log.h` 在 `test_uce`/`unitcore` 双模块各计一次）：

| 层 | 覆盖 / 总行 | 覆盖率 |
|----|-------------|--------|
| `include/unitcore` | 17 / 18 | **94.4%** |
| `src/unitcore` | 555 / 649 | **85.5%** |
| `src/common` | 142 / 187 | **75.9%** |
| `include/common` | 3 / 3 | **100%** |
| OCC 汇总（含双模块重复行） | 795 / 963 | **82.6%** |

### 3.3 未覆盖原因（摘要）

- **CLI 入口**（`src/unitconvert/main.cpp`）未链入 `test_uce`，不在本次统计内。
- `exprcalculator.cpp` / `exprlexer.cpp`：部分语法错误分支与冷路径（如非法数字、运算符组合）未全部命中。
- `jsonconfigparser.cpp`：部分非法 JSON 字段/非法系数分支未覆盖。
- `jsonunitloader.cpp`：空 name/base、非法 factor、base 与内置不一致等警告路径未全覆盖。
- `errorcode.cpp`：部分错误码文案分支未在断言中逐一触达。
- `batchrunner.cpp`：写结果文件失败等 IO 错误分支未模拟。

---

