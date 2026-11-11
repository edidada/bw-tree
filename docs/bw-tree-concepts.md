# Bw-tree 概念入门（dbgroup-nagoya-u 实现）

这篇文档面向第一次接触本仓库的读者，解释 Bw-tree 是什么、本实现由哪些部件组成、以及怎么编译运行。读完后再去看 `include/` 下的头文件会容易很多。

> 本仓库 fork 自 [dbgroup-nagoya-u/bw-tree](https://github.com/dbgroup-nagoya-u/bw-tree)（名古屋大学 DB 组的研究用实现）。当前分支 `edidada/main` 只增加了本文和 `.github/workflows/build-and-test.yml`，没有改动库代码。

## 1. Bw-tree 是什么

Bw-tree 是微软研究院 2013 年提出的**无锁 B+ 树索引**，面向多核 CPU 与大内存/闪存。传统 B+ 树改页时要加 latch（闩锁），高并发下根节点和内部节点会成为瓶颈；Bw-tree 用两个技巧绕开它：

- **间接寻址**：节点之间引用逻辑 ID（PID），通过一张可用 CAS 更新的**映射表（Mapping Table）**找到物理地址；
- **增量更新**：不改写原页，而是往页面头部追加一条 **Delta Record**（增量记录），再用一次 CAS 把映射表指针切到新链头。

于是写操作 = 分配一小块内存 + 一次原子指令，读者完全不需要等锁。

本实现的定位是**复现论文并测量性能**，因此是 **header-only 的 C++17 库**，靠 CMake 的 `INTERFACE` target 对外提供，附带一组 GoogleTest 单元测试。

## 2. 结构（Structure）：源码与概念的对应

| 文件 | 概念 |
| --- | --- |
| `include/bw_tree/bw_tree.hpp` | `BwTree` 主类，对外的 `Read` / `Write` / `Insert` / `Update` / `Delete` 接口 |
| `include/bw_tree/component/mapping_table.hpp` | 映射表：PID → 物理地址 |
| `include/bw_tree/component/logical_ptr.hpp` | 逻辑指针：映射表里的一个可 CAS 单元（也被 GC 用来保护旧版本） |
| `include/bw_tree/component/delta_chain.hpp` | 增量链：从链头遍历、收集 delta、判断是否需要 consolidation |
| `include/bw_tree/component/fixlen/delta_record.hpp` | 定长键/值的增量记录类型 |
| `include/bw_tree/component/varlen/delta_record.hpp` | 变长键/值的增量记录类型 |
| `include/bw_tree/component/fixlen/node.hpp` | 定长版本的基础页（Base Node） |
| `include/bw_tree/component/varlen/node.hpp` | 变长版本的基础页 |
| `include/bw_tree/component/varlen/metadata.hpp` | 变长数据的元数据布局 |
| `include/bw_tree/component/record_iterator.hpp` | 记录级迭代器 |
| `include/bw_tree/component/common.hpp` | 公共类型与常量 |
| `include/bw_tree/utility.hpp` | 工具函数（如 Log2） |
| `external/memory-manager` | 外部 GC 子模块，基于 epoch 的内存回收 |

### 六个关键词

- **Mapping Table / PID**：树里存的是逻辑页号，查表才拿到地址；CAS 映射表项即原子发布新版本。
- **Delta Chain**：一串"在旧状态上做了什么"的记录，挂在基础页前面；读时自链头向下合并出当前视图。
- **Consolidation（页合并/压实）**：链太长时把基础页与所有 delta 压成一个新的紧凑基础页，再 CAS 替换。
- **SMO（Structure Modification Operation）**：split / merge 这类改树形状的操作；无法一次 CAS 完成，拆成多步并靠 help-along 让后来者帮忙收尾。
- **Epoch-based GC**：被替换的旧页不能立刻 delete（可能还有读者），按 epoch 延迟回收，由 `external/memory-manager` 负责。
- **fixlen / varlen**：定长与变长键值两套实现，是本仓库同时提供的两个 Bw-tree 变体。

## 3. 优势（Strengths）与代价

| 优势 | 原因 |
| --- | --- |
| latch-free，多核伸缩性好 | 冲突收敛到映射表上的一次 CAS |
| 读不被写阻塞 | 读者只看版本快照 |
| 对闪存友好 | 追加小增量而非就地重写整页，写放大低 |
| 便于研究调参 | 页大小、合并阈值、重试次数等都做成 CMake 选项 |

代价：增量链过长会拖慢读、SMO 协议复杂、CAS 失败要重试、GC 需要额外机制。

## 4. 可调参数（CMake 选项）

在 `CMakeLists.txt` 里以 `CACHE STRING` 暴露，含义：

| 选项 | 默认值 | 含义 |
| --- | --- | --- |
| `BW_TREE_PAGE_SIZE` | `1024` | 页大小（字节） |
| `BW_TREE_DELTA_RECORD_NUM_THRESHOLD` | `(2 * Log2(kPageSize / 256))` | 增量数超过它就触发 consolidation |
| `BW_TREE_MAX_DELTA_RECORD_NUM` | `64` | 增量数超过它时让线程等待 |
| `BW_TREE_MIN_NODE_SIZE` | `kPageSize / 16` | 节点低于它就触发 merge |
| `BW_TREE_MAX_VARIABLE_DATA_SIZE` | `128` | 预期的最大变长数据长度 |
| `BW_TREE_RETRY_THRESHOLD` | `10` | 防止忙等的最大重试次数 |
| `BW_TREE_SLEEP_TIME` | `10` | 防忙等的睡眠时间（微秒） |
| `BW_TREE_BUILD_TESTS` | `OFF` | 是否编译单元测试 |
| `DBGROUP_TEST_THREAD_NUM` | `8` | 单测的工作线程数 |
| `DBGROUP_TEST_RANDOM_SEED` | `0` | 固定随机种子，便于复现 |
| `DBGROUP_TEST_EXEC_NUM` | `1E5` | 每线程执行次数 |
| `DBGROUP_TEST_OVERRIDE_MIMALLOC` | `OFF` | 用 mimalloc 接管全部内存分配 |

## 5. 编译与运行

```bash
sudo apt update && sudo apt install -y build-essential cmake
git submodule update --init --recursive     # 必须：external/memory-manager、index-fixtures
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBW_TREE_BUILD_TESTS=ON
cmake --build build --parallel 2
ctest --test-dir build -C Release --output-on-failure
```

`test/` 下共 9 个 GoogleTest 目标，覆盖从底层部件到整树行为：

`logical_ptr_test`、`mapping_table_test`、`metadata_test`、`delta_record_test`、`node_test`、`bw_tree_varlen_test`、`bw_tree_fixlen_test`、`bw_tree_varlen_multi_thread_test`、`bw_tree_fixlen_multi_thread_test`。

本分支的 CI 会在 `edidada/**` 分支 push 时自动执行上面的流程（含 `submodules: recursive`），可用 `gh run list` / `gh run view --log` 查看日志。

## 6. 与其他 Bw-tree 实现的区别

| 实现 | 语言/构建 | 特点 |
| --- | --- | --- |
| 本仓库（dbgroup-nagoya-u） | C++17 / CMake，header-only | 研究用途，定长+变长两套实现，参数可调，单元测试齐全 |
| Open BwTree（wangziqi2013，本账号的 `edidada/BwTree`） | C++11 / Makefile | 与 Peloton 配套，含 ABORT delta、Bloom Filter、预分配等改进 |
| TUM 教学版（flode/BwTree） | C++14 / CMake，可执行 benchmark | 学生项目，代码量小，适合读主流程 |

三者都遵循同一篇论文，术语是通用的；差别主要在 GC 实现、SMO 细节和工程取舍。

## 7. 参考

- Levandoski, Lomet, Sengupta. *The Bw-Tree: A B-tree for new hardware platforms.* ICDE 2013.
- 本仓库 `README.md` 的 Build / Usage 章节。
- `.github/workflows/unit_tests.yaml`（上游 CI，4 种页大小的矩阵）。
