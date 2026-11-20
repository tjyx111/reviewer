# 测试报告：client-rust（W02-a 驱动生命周期阶段）

| 项 | 内容 |
| --- | --- |
| 报告编号 | TR-2026-11-20-01 |
| 被测对象 | `D:\code\lbsprivate\client-rust` |
| 证据基线 | Git HEAD `c6c951d`（2026-10-14，`docs: establish global development gate`），分支 `docs/global-development-gate` |
| 工作区状态 | 17 项未提交/未跟踪；**Git 跟踪的 Rust 文件数为 0**（`crates/` 全部为 untracked） |
| 测试时间 | 2026-11-20 |
| 测试环境 | Windows 10 Pro 10.0.19045 x64；Rust 1.97.1 (8bab26f4f 2026-07-14)；当前会话 `Administrator=True`；`SystemRoot=C:\Windows`；`netfilter2` 服务未安装 |
| 测试角色 | 测试经理（非作者） |
| 参考门禁 | `AGENTS.md` §10/§12/§15；`docs/windows-driver-loader-design.md` §7、§10.4、§10.7 |

## 1. 结论

| 判定项 | 结果 |
| --- | --- |
| 自动化门禁 `scripts/verify.ps1` | **通过**（6/6 步，4.0 s） |
| 单元测试 | **20/20 通过**（client 18、foundation 1、native-library 1），0 失败、0 忽略 |
| W02-a 设计 §10.4 测试要求 | **不通过**（生命周期顺序、回滚、RUNNING 超时、DLL 卸载时机均无证据） |
| W01 设计 T01–T08 | 覆盖 2/8、部分 1/8、缺失 5/8 |
| 真实驱动 E2E | **未执行**（未授权；需管理员 + 可恢复 Windows VM） |
| 是否允许提交/合并 | **否**：测试缺口 + 无 Git diff 可绑定 + 状态文档互相矛盾 |

一句话结论：构建与现有测试全绿，但现有测试证明的范围是“参数、文件复制、哈希、只读 SCM 查询”；W02-a 真正的风险区（注册失败回滚、服务未 RUNNING、清理顺序、DLL 卸载时机）没有任何自动化证据，而其中一条测试在装有 demo 驱动的机器上会停止真实驱动。

## 2. 执行记录

| # | 命令 | 范围 | 结果 | 耗时 |
| --- | --- | --- | --- | ---: |
| 1 | `powershell -NoProfile -ExecutionPolicy Bypass -File scripts/verify.ps1` | 文档、脚本自测、fmt、Release Clippy、Release test、Release build | **通过** | 4021 ms |
| 2 | 同上 `verify-docs` | 25 个文本文件、7 个 Markdown、链接、台账唯一性 | 通过 | 318 ms |
| 3 | 同上 `automation-self-test` | 门禁步骤与 CI 配置自检 | 通过 | 358 ms |
| 4 | 同上 `cargo-fmt` | `cargo fmt --all -- --check` | 通过 | 135 ms |
| 5 | 同上 `cargo-clippy-release` | `--locked --release --workspace --all-targets --all-features -D warnings` | 通过 | 354 ms |
| 6 | 同上 `cargo-test-release` | `--locked --release --workspace --all-features` | 通过，20 tests | 2027 ms |
| 7 | 同上 `cargo-build-release` | `--locked --release --workspace --all-features` | 通过 | 559 ms |
| 8 | `cargo test ... -- --list` | 测试清单 | 20 tests，0 benchmarks | — |
| 9 | `sc query netfilter2` | 本机服务状态 | 未安装（Win32 1060） | — |

原始输出摘要：

```text
Pinned toolchain: rustc 1.97.1 (8bab26f4f 2026-07-14)
Documentation verification passed: 25 text files, 7 Markdown files.
Automation self-test passed.
[verify-docs] passed / [automation-self-test] passed / [cargo-fmt] passed
[cargo-clippy-release] passed / [cargo-test-release] passed / [cargo-build-release] passed
test result: ok. 18 passed; 0 failed; 0 ignored
test result: ok. 1 passed; 0 failed; 0 ignored
test result: ok. 1 passed; 0 failed; 0 ignored
Full S0 verification passed.
```

环境风险提示：本次会话以管理员身份运行门禁。管理员身份会掩盖“权限不足”类分支；GitHub `windows-latest` runner 同样默认管理员，因此 CI 也无法覆盖权限失败路径。

## 3. 测试清单与结果

| # | 测试 | 层次 | 验证内容 | 结果 |
| ---: | --- | --- | --- | --- |
| 1 | `rejects_bad_arguments` | 单元 | 参数个数、相对路径、源文件不存在 | 通过 |
| 2 | `accepts_existing_driver_file` | 单元 | 合法两参数被接受 | 通过 |
| 3 | `reports_missing_nfapi_exports` | 集成（真实 DLL） | 对 `winhttp.dll` 解析 5 导出全部失败 | 通过 |
| 4 | `queries_service_state` | 集成（真实 SCM） | 缺失服务报错、`EventLog` 可查询 | 通过 |
| 5 | `detects_existing_service` | 集成（真实 SCM） | 缺失→false、`EventLog`→true | 通过 |
| 6 | `queries_service_config` | 集成（真实 SCM） | `EventLog` 可查询、缺失服务报错 | 通过 |
| 7 | `formats_checklist_result` | 单元 | OK/ERR 文本与 `file_expectation` | 通过 |
| 8 | `filters_related_drivers` | 单元 | 名称过滤（**死代码**） | 通过 |
| 9 | `converts_wide_string` | 单元 | UTF-16 读取（**死代码**） | 通过 |
| 10 | `normalizes_service_image_paths` | 单元 | 2 种 ImagePath 形式 | 通过 |
| 11 | `computes_sha256` | 单元 | `"abc"` 已知向量 | 通过 |
| 12 | `copies_and_deletes_driver_file` | 单元（临时目录） | 复制/同哈希跳过/异哈希拒绝/源缺失/父目录缺失/删除语义 | 通过 |
| 13 | `refuses_existing_service` | 集成（真实 SCM） | 服务已存在时拒绝复制且不落盘 | 通过 |
| 14 | `stops_missing_service` | 集成（真实 SCM） | 缺失服务停止失败 | 通过 |
| 15 | `register_and_unregister_status` | 单元（fake 函数指针） | 返回值 0/非 0 映射 | 通过 |
| 16 | `builds_driver_destination` | 单元 | 目标路径后缀 | 通过 |
| 17 | `waits_for_missing_service_until_error` | 集成（真实 SCM） | 缺失服务立即报错 | 通过 |
| 18 | `cleanup_skips_unregister_when_stop_fails` | 集成（真实 SCM，**有副作用风险**） | 停止失败时不调用注销 | 通过 |
| 19 | `workspace_test_harness_is_active` | 单元 | 基础 crate 包名/版本 | 通过 |
| 20 | `loads_winhttp_and_rejects_bad_input` | 集成（真实 DLL） | 相对路径拒绝、`winhttp.dll` 加载、单符号查找 | 通过 |

代码量分布：`crates/app/client/src/main.rs` 共 1093 行，其中 `#[cfg(test)]` 测试模块约 236 行（21.6%），生产代码约 857 行（含约 66 行当前不执行的死代码）。

## 4. 覆盖矩阵（对照已确认设计）

### 4.1 W01 加载器设计 §7

| ID | 要求 | 实际对应 | 判定 |
| --- | --- | --- | --- |
| W01-T01 | 含中文/空格的绝对路径加载并解析已知导出 | 无 | **缺失** |
| W01-T02 | 相对路径拒绝且不搜索 CWD/PATH | `rejects_bad_arguments`、`loads_winhttp_and_rejects_bad_input` | 覆盖 |
| W01-T03 | 10 个必需导出全部解析后才成功 | 无（代码当前只解析 5 个） | **缺失** |
| W01-T04 | 缺一个必需导出整体失败，错误含符号名 | 无（`winhttp` 是全部缺失，不是“缺一个”） | **缺失** |
| W01-T05 | 不存在/不可访问/非 DLL 分别可分类 | 仅覆盖相对路径与缺失服务，`load_exact` 的不存在/非 DLL 未测 | 部分 |
| W01-T06 | 成功/失败路径均释放模块，无句柄泄漏 | 依赖 `Drop`，无断言、无句柄计数 | 部分 |
| W01-T07 | Windows x64 Release 构建 | 门禁 `cargo-build-release` | 覆盖 |
| W01-T08 | 符号清单与 ABI fixture/C probe 一致 | 无 | **缺失** |

### 4.2 W02-a 设计 §10.4

| 要求 | 实际对应 | 判定 |
| --- | --- | --- |
| 复制成功/跳过/失败 | `copies_and_deletes_driver_file` | 覆盖 |
| `nf_registerDriver` 失败回滚 | 仅 fake 返回值映射；未断言服务/文件残留 | 部分 |
| 服务未 RUNNING（含超时） | 无；`waits_for_missing_service_until_error` 只覆盖“查询立即出错” | **缺失** |
| `nf_init` 失败回滚 | 尚未实现该轮 | N/A |
| `nf_free → stop → unregister → delete` 调用顺序 | 无 | **缺失** |
| 停止失败不注销 / 注销失败不删除 | 仅“停止失败跳过注销”一条 | 部分 |
| 注销后 DLL 才卸载 | 无 | **缺失** |
| 参数：缺失、相对、源不存在 | `rejects_bad_arguments` | 覆盖 |

### 4.3 清单设计 §10.7

| 要求 | 实际对应 | 判定 |
| --- | --- | --- |
| SHA-256 已知向量 | `computes_sha256` | 覆盖 |
| expected/actual 比较逻辑 | `formats_checklist_result` | 覆盖 |
| 日志写入失败注入 | 无 | **缺失** |
| 相关驱动/同文件服务偏差项 | 枚举未接线，无测试 | **缺失** |
| VM 三场景（干净/预存/额外驱动） | 未执行 | 未执行 |

## 5. 失败分支覆盖（生产代码）

| 函数 | 分支 | 状态 |
| --- | --- | --- |
| `query_service_state` | SCM 打开失败 / 服务打开失败 / 查询失败 / 成功 | 仅“服务打开失败”与“成功” |
| `service_exists` | 1060 → false / 其他错误 / 存在 | 全部覆盖（1060、存在、缺失服务报错） |
| `query_service_config` | 打开失败 / 大小无效 / 查询失败 / 成功（含空路径） | 仅“打开失败”与“成功” |
| `sha256_file` | 读失败 / 成功 | 覆盖（读失败经 `copy_driver` 间接） |
| `copy_driver` | 服务存在 / 源哈希失败 / 目标元数据失败 / 同哈希 / 异哈希 / 复制 | 除“目标元数据失败”外覆盖 |
| `copy_file` | 复制成功 / 复制失败后删除成功 / 删除 NotFound / 删除失败 | 部分（删除失败分支未测） |
| `delete_driver` | 未复制 / 删除成功 / NotFound / 其他错误 | 除“其他错误”外覆盖 |
| `stop_service` | SCM 失败 / 打开失败 / Control 失败 / 1062 / 轮询失败 / STOPPED / 超时 | **仅“打开失败”** |
| `register_driver` / `unregister_driver` | 0 / 非 0 | 覆盖（fake） |
| `wait_until_running` | 查询错误 / RUNNING / 超时 | 仅“查询错误” |
| `cleanup_driver` | 停止失败 / 注销失败 / 删除失败 / 全部成功 | 仅“停止失败跳过注销” |
| `load_functions` | 全部存在 / 缺一个 / 全部缺失 | 仅“全部缺失” |
| `NativeLibrary` | 相对路径 / 不存在 / 加载失败 / 成功 / 符号存在/缺失 | 除“不存在、加载失败”外覆盖 |
| 清单模块（`unix_now`、`append_checklist_log`、`checklist_line`、`startup_checklist`、`shutdown_checklist`） | 正常/异常 | **未覆盖** |
| `normalize_image_path` | `\SystemRoot\`、`\System32\`、`system32\`、`\??\`、`%SystemRoot%`、带引号 | 仅前 3 类中的 2 类 |

## 6. 测试缺陷与风险

| # | 严重级别 | 问题 | 影响 | 建议 |
| --- | --- | --- | --- | --- |
| T1 | 高 | `cleanup_skips_unregister_when_stop_fails`（main.rs:1086）调用真实 `cleanup_driver`，内部硬编码停止 `netfilter2` | 在安装了 demo 驱动的开发机/VM 上运行 `cargo test` 会真正停止系统过滤驱动；本会话为管理员，具备该权限。当前本机未安装该服务，所以测试“碰巧”无害 | 删除该测试或把 stop/unregister/delete 改为注入函数指针；测试不得触碰真实服务 |
| T2 | 高 | 生命周期核心路径（顺序、回滚、RUNNING、卸载时机）无自动化测试 | W02-a 的主要风险完全未被验证，门禁绿色具有误导性 | 建立离线 fake 边界，按 §10.4 补测 |
| T3 | 中 | 6 个测试依赖真实 SCM / `System32\winhttp.dll` | 跨机器非确定性；历史已出现 487；CI 环境不可控 | 归为一个 `#[ignore]` 集成测试或 VM 用例，默认运行只保留纯逻辑测试 |
| T4 | 中 | 清单功能（设计仍“待确认”）无失败注入与日志测试 | “清单失败不改变主流程但必须可见”的承诺无证据 | 先确认设计；保留则补测，否则回退 |
| T5 | 低 | 3 个测试验证死代码（`filters_related_drivers`、`converts_wide_string`）与平凡逻辑 | 消耗每轮 100 行预算与维护面 | 与死代码一并删除/延后 |
| T6 | 中 | `normalize_image_path` 缺少 `\??\`、`%SystemRoot%` 用例 | 真实 ImagePath 形式可能产生假 ERR | VM 取证后补表驱动测试 |
| T7 | 中 | 无负向用例验证 `load_exact` 对“不存在文件/非 DLL”的错误分类（W01-T05） | 加载失败路径证据不足 | 补 2 个用例 |
| T8 | 低 | 测试无端到端退出码断言（`main` 不可测） | 退出码 1/2 的契约无证据 | 把 `main` 拆成可测的自由函数或 VM 验收覆盖 |
| T9 | 低 | `TestDirectory::drop` 清理失败仅打印 | 资源清理失败在测试结果中不可见 | 显式清理或计入失败 |

## 7. 未执行 / 无法执行的测试

1. 真实 `netfilter2.sys` 安装、启动、停止、注销、删除（需管理员 + 可恢复 VM + 独立授权）。
2. `nf_init` + 16+2 callback 注册与回调静止屏障（该轮尚未实现）。
3. 非管理员权限路径（当前环境与 CI runner 均为管理员）。
4. 清单日志写入失败注入、相关驱动枚举、截断上限。
5. 退出码端到端断言。
6. 崩溃/断电后残留状态恢复。

## 8. 未验证的关键假设（需 VM 取证）

1. `nf_registerDriver` 返回非零时，服务是否可能已被创建（决定注册失败路径能否安全删除文件）。
2. `nf_unRegisterDriver` 只删除服务、不停止服务（来自反汇编，未在真实驱动验证）。
3. 驱动服务的实际启动类型是否为 `SERVICE_SYSTEM_START`。
4. 驱动服务 ImagePath 的实际形式（`\SystemRoot\…`、`\??\…` 或相对 `system32\…`）。
5. demo 驱动停止耗时是否稳定在 10 s deadline 内。

## 9. 重测计划与准入条件

### 9.1 下一轮代码前必须补齐（P0，离线可执行）

| 项 | 断言 |
| --- | --- |
| 注入式生命周期测试 | `nf_free → stop → unregister → delete → FreeLibrary` 顺序；每步失败短路 |
| 注册失败回滚 | 无残留服务、无残留本次复制文件、退出码 1 |
| RUNNING 等待 | fake 状态 `missing → running` 成功；持续非 running 到 deadline 失败 |
| 停止“不存在” | 视为已停止并继续后续步骤 |
| `delete_driver` 失败 | 稳定错误并参与退出码 |
| `load_exact` 负向 | 不存在文件、非 DLL 文件的可分类错误 |
| 清单（若保留） | 写失败只告警、主流程结果不变；条目上限与 `truncated` |
| 测试隔离 | 源码约束测试：测试不得打开/停止真实 `netfilter2` |

### 9.2 VM 验收（需单独授权）

1. 干净快照 + 管理员：安装后 RUNNING；清单逐项 OK；Enter 后服务与文件消失；退出码 0。
2. 预存服务 / 预存同名不同哈希文件：拒绝覆盖，原文件保留。
3. 环境存在其他 `netfilter|wfp|accel|vpn` 驱动：清单 ERR，退出码不变。
4. 非管理员：注册失败明确，退出码 1，无残留文件。
5. stdin 提前关闭：清理路径执行并输出关闭清单。
6. 每次测试前恢复快照，串行执行，记录驱动停止耗时。

### 9.3 出口条件

- P0 项全部自动化通过；
- `scripts/verify.ps1` 通过且结果绑定到一个已审查的提交（当前无跟踪的 Rust 文件，无法绑定）；
- 设计 §10.4 测试矩阵与台账状态一致；
- VM 冒烟结果作为附加证据归档，不替代 P0。

## 10. 残余风险登记

| 风险 | 状态 |
| --- | --- |
| 代码未提交，所有测试结论无法绑定 SHA | 未解除 |
| 487 / mtime 刷新 workaround 根因未证明 | 未解除 |
| 管理员身份掩盖权限失败路径 | 未解除 |
| 测试停真实驱动（T1） | 未解除，需代码修改 |
| 真实驱动失败语义（§8 假设） | 未验证 |
| 无签名/版本/哈希校验（本阶段有意） | 已接受，禁止进入生产制品 |
