# 代码审查：binlog_large_commit_threshold (commit-by-rename)

## 功能概述

当大事务的 binlog cache 落盘到 `#binlog_cache_files` 且大小不小于 `binlog_large_commit_threshold` 时，通过将 cache 文件 rename 为新 binlog 文件完成提交，避免拷贝阻塞其他提交。参考 MDEV-32014、dbsgogogo/binlog/prompt.md。

## 变更范围

| 模块 | 文件 | 说明 |
|------|------|------|
| 系统层 | include/my_sys.h, mysys/mf_iocache.cc | IO_CACHE 增加 create_file_callback/create_file_arg，首次落盘时可选自定义建文件 |
| Binlog 核心 | sql/binlog.h, sql/binlog.cc | m_binlog_cache_files_dir、init_binlog_cache_files_dir、commit_by_rename、get_binlog_cache_* |
| Binlog 缓存 | sql/binlog_ostream.h, sql/binlog_ostream.cc | reserve 区、create_file_callback、has_renameable_file、flush_sync_and_close_file、reset(file==-1) |
| 变量与入口 | sql/mysqld.h, sql/mysqld.cc, sql/sys_vars.cc | opt_binlog_large_commit_threshold 默认 128MB |
| MTR | binlog_nogtid/t/binlog_large_commit_threshold.test | 覆盖 threshold=0、小事务、spill<阈值、commit_by_rename、rollback 后 reset |

## 审查结论

- **设计**：commit-by-rename 路径清晰，仅在 threshold>0、事务 cache、has_renameable_file 且 size>=threshold 时进入；否则走原有写路径。
- **线程安全**：commit_by_rename 内通过 LOCK_log 保护，与 open_binlog/new_file 等一致。
- **错误与回滚**：init_binlog_cache_files_dir 失败仅禁用 commit-by-rename，不中断 binlog 打开；commit_by_rename 各步失败会返回 true，上层可走正常写路径。
- **存储层**：IO_CACHE 的 create_file_callback 扩展合理，mf_iocache 仅在 file==-1 且 callback 非空时调用，兼容现有用法。
- **测试**：binlog_large_commit_threshold.test 覆盖 6 类场景（含 threshold=0、不 spill、spill<阈值、commit_by_rename、reset 后二次大事务、rollback 后 truncate+reserve），并引用 have_log_bin/have_binlog_format_row 及变量存在性检查，结构清晰。

## Patch 说明

- **文件**：`mysql-server-binlog-large-commit-threshold.patch`
- **包含文件**（仅 binlog 特性相关）：
  - include/my_sys.h, mysys/mf_iocache.cc
  - sql/binlog.cc, sql/binlog.h, sql/binlog_ostream.cc, sql/binlog_ostream.h
  - sql/mysqld.cc, sql/mysqld.h, sql/sys_vars.cc
  - mysql-test/suite/binlog_nogtid/t/binlog_large_commit_threshold.test
  - mysql-test/suite/binlog_nogtid/r/binlog_large_commit_threshold.result
- **应用**：在 mysql-server 根目录执行 `git apply mysql-server-binlog-large-commit-threshold.patch`。
