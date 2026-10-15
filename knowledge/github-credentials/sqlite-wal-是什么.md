# SQLite 的 `-wal` / `-shm` 是什么，为什么不能手删

## 机制

`opencode.db` 这类库开的是 **WAL 模式**（write-ahead log）。写不是直接改主库，而是先追加到 `-wal`：

> 草图：[`WAL-写入与checkpoint.excalidraw`](./WAL-写入与checkpoint.excalidraw)（OpenChamber 的 Files 视图里点开就是画布）

写入先追加到 `-wal`，读取看到的是"主库 + 未合并增量"的合并视图，checkpoint 时才把 `-wal` 的页合并回主库。

三个文件一组：`.db` 主库 / `-wal` 未合并增量 / `-shm` 共享内存索引（运行时用，进程全退出后可重建）。同时存在是**正常状态**，不是垃圾。

① 写先追加到 `-wal`；② checkpoint 时才合并回主库 —— 中间这段时间，新数据只在 `-wal` 里。

## 认现状（只读）

```bash
ls -l <db>.db <db>-wal <db>-shm      # main 通常很大，wal 是"还没合并的增量"
```

本沙箱实测：主库 ~1.19 GB、`-wal` ~10 MB、`-shm` 32 KB。刚写进去的数据就在 `-wal` 里。

## 为什么不能手删 `-wal`

- 它里面是**已提交但还没合并**的数据：直接删 = 丢最近的写入，甚至损坏数据库。
- 句柄被占用时也删不掉（Windows 上会报 "being used by another process"）。

## 正确做法

1. 让**所有**使用该库的进程退出（释放句柄）；
2. 再做 checkpoint，把 `-wal` 合并回主库并截断：
   ```sql
   PRAGMA wal_checkpoint(TRUNCATE);
   ```
3. 想让"敏感内容"从库里消失，是删**记录**（或 vacuum 整理），不是删 WAL 文件。

## 和"删干净"的关系

WAL 会把写入放大成多份副本：同一段内容可能同时出现在 `.db`、`.db-wal`（还有 free pages 里的残留）。所以清理凭据残留时，**主库和 WAL 要一起考虑**，且必须先停进程再动。
