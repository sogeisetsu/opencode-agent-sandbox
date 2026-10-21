# 删了会话，数据库为什么不缩小：freelist、VACUUM、auto_vacuum

现象：把聊天记录删了很多，本地库文件（`~/.local/share/opencode/opencode.db`）大小一点没变。

## 先看清现状（只读）

```sql
PRAGMA page_size;          -- 页大小，通常 4096
PRAGMA page_count;         -- 文件总页数
PRAGMA freelist_count;     -- 空闲页数
PRAGMA auto_vacuum;        -- 0=NONE / 1=FULL / 2=INCREMENTAL
PRAGMA journal_mode;
PRAGMA wal_autocheckpoint;
```

文件大小 ≈ `page_size × page_count`。实测样本（库在 GB 量级）：`page_size=4096`、`auto_vacuum=0`、`journal_mode=wal`，而空闲页占一半以上（约 54%）——也就是说文件里一半是已删数据的空壳。

看"活跃数据到底占多少"，按 `part` 类型分组统计比看总大小有用。注意 **`part` 表没有 `type` 列**（列只有 `id / message_id / session_id / time_created / time_updated / data`），类型存在 `data` 的 JSON 里：

```sql
SELECT json_extract(data,'$.type') AS t, count(*), sum(length(data))
FROM part GROUP BY t ORDER BY 3 DESC;
```

直接写 `SELECT type ...` 会报 `no such column: type`。实测把 `sum(length(data))` 从大到小排，典型顺序是 `tool` → `file` → `reasoning` → `text` → `step-finish` → `step-start` → `patch` → `compaction` → `agent` → `subtask`；其中 `file` 就是上面那篇说的图片（base64 落库的那些）。只统计 tool/file/reasoning/text 四类会漏掉 step-* 等类型，不过它们的体积通常小一两个数量级。

## 删除的真实语义

- **行确实被删掉了**：会话删除是硬删除加级联，`session` / `message` / `part` 都没有 `time_deleted` 之类的软删除字段；实测删除后孤立的 `part`、`message` 计数都是 0。
- **文件不会缩小**：SQLite 默认只把腾出来的页登记进 **freelist**，文件高水位不动。要缩得把尾部之后的页全往前搬，它不做。
- freelist 里的空页**可以被任意新页复用**，不限于原来那种数据。所以"只删不增"不会让文件继续涨，但也不会回落；增长来自后续新增内容。
- 空页里的**旧字节仍在文件里**，有被恢复出来的可能（隐私层面要单独考虑，见 `github-credentials/sqlite-wal-是什么.md`）。

## VACUUM 做什么

建一个紧凑的新库，只复制活跃页，丢弃空闲页与碎片，然后替换旧文件。

- 不删数据：活着的记录（包括图片）原样搬过去。
- 需要**接近等大的临时空间**：原库和新库同时存在。
- **原地 `VACUUM` 是写操作，要拿独占锁**：别的连接只要握着会挡写的锁（比如一个没提交的读事务）就失败，实测报 `database is locked`。
- **`VACUUM INTO '<new.db>'` 对源库只读**，不需要独占锁——实测另一个连接开着未提交读事务时它照样成功。它把内容写进新文件，原库完全不动，校验通过再替换，所以更安全。

草图：[`空闲页与VACUUM.excalidraw`](./空闲页与VACUUM.excalidraw)（OpenChamber 的 Files 视图里点开就是画布）

## auto_vacuum 三种模式

| 值 | 行为 | 代价 |
|---|---|---|
| `NONE`(0) | 默认。删除只进 freelist，文件永不缩 | 无 |
| `FULL`(1) | 每次提交时把尾部空闲页搬回并截断，文件随删除自动缩 | 写放大、删大块时一次性卡顿、少量 pointer-map 开销 |
| `INCREMENTAL`(2) | 只有显式 `PRAGMA incremental_vacuum` 才收缩 | 应用不调用就无效 |

从 `NONE` 改 `FULL` **必须跑一次 VACUUM 才生效**（标记持久写在库头里）。两个实测过的坑：

- 只执行 `PRAGMA auto_vacuum=FULL;` 而不跟 `VACUUM;`，再查还是 `0`，**不报任何错**——等于没开。
- `VACUUM INTO` 出来的新库**继承源库的 auto_vacuum**（源库是 0，新库就是 0）。所以要开自动收缩，得在新库上再 `PRAGMA auto_vacuum=FULL;` 然后 `VACUUM;`——多一遍全量重写，代价要提前算。

## 收缩流程（破坏性，按顺序做）

本机（Windows）默认**没有 `sqlite3` CLI**，`Get-Command sqlite3` 无结果；用 Python 自带的 `sqlite3` 模块。下面两个脚本存成文件再跑，别在 `python -c` 里塞长代码（PowerShell 会拆引号）。

`db-inspect.py`（只读巡检，随时能跑）：

```python
import sqlite3, sys
db = sys.argv[1]
con = sqlite3.connect(f"file:{db}?mode=ro", uri=True)
for p in ("page_size", "page_count", "freelist_count", "auto_vacuum", "journal_mode"):
    print(p, con.execute("PRAGMA " + p).fetchone()[0])
size = con.execute("PRAGMA page_size").fetchone()[0] * con.execute("PRAGMA page_count").fetchone()[0]
print("file MB", round(size / 1048576, 1))
for row in con.execute("SELECT json_extract(data,'$.type'), count(*), sum(length(data)) FROM part GROUP BY 1 ORDER BY 3 DESC"):
    print(row)
for t in ("session", "message", "part"):
    print(t, con.execute("SELECT count(*) FROM " + t).fetchone()[0])
```

`db-vacuum.py`（产出新库，不动原库）：

```python
import sqlite3, sys, os
src, dst = sys.argv[1], sys.argv[2]
if os.path.exists(dst):
    sys.exit("目标已存在，换一个名字")
con = sqlite3.connect(f"file:{src}?mode=ro", uri=True)
con.execute("VACUUM INTO ?", (dst,))     # 源库只读，不需要独占锁
con.close()
chk = sqlite3.connect(dst)
print("integrity:", chk.execute("PRAGMA integrity_check").fetchone()[0])
if len(sys.argv) > 3 and sys.argv[3] == "autovacuum":
    chk.execute("PRAGMA auto_vacuum=FULL")
    chk.execute("VACUUM")                # 必须紧跟这一次 VACUUM，否则等于没设
    print("auto_vacuum:", chk.execute("PRAGMA auto_vacuum").fetchone()[0])
chk.close()
```

步骤：

1. **完全退出应用**（托盘退出，别只停后台服务——服务会被重新拉起）。停不干净，第 5 步的文件替换会撞句柄。
2. 备份：把 `.db` 复制到另一个盘（`Copy-Item -LiteralPath <db> -Destination <另一块盘>`）。
3. 收缩到**同目录**下的新文件（同卷改名才是原子的；跨盘复制不原子）：
   ```powershell
   python db-vacuum.py "<db>" "<db>.new.db"
   ```
   想同时开自动收缩就加第三个参数 `autovacuum`。机制上这一步不需要独占锁，但建议仍在停服状态下做：它读到的是快照，跑的过程中新写入不会进新库。
4. 校验：脚本打印的 `integrity: ok`，再用 `db-inspect.py` 对比新旧库的 session / message / part 计数。顺带一提，`VACUUM INTO` 出来的新库是 rollback journal 模式（不继承源库的 WAL），应用下次打开会自己切回 WAL，不是问题。
5. **同卷替换**（先挪走旧的三个文件，再放新的）：
   ```powershell
   $db = '<db 的绝对路径>'
   Move-Item -LiteralPath $db "$db.old"
   foreach ($s in '-wal','-shm') { if (Test-Path -LiteralPath "$db$s") { Move-Item -LiteralPath "$db$s" "$db$s.old" } }
   Move-Item -LiteralPath "$db.new.db" $db
   ```
   **`-wal` 和 `-shm` 必须一起挪走**——旧 WAL 套到新库上是整个流程唯一真正有破坏力的点（详见 `github-credentials/sqlite-wal-是什么.md`）。
6. 启动应用抽查几个老会话，稳定几天再删 `.old`。

预期：空闲页占比高时，体积能掉到活跃数据的量级。

## 上游状态（避免白等官方命令）

仓库已迁到 `anomalyco/opencode`，相关 issue（2026-10-22 查 `gh api` 的状态）：

| issue | 内容 | 状态 |
|---|---|---|
| #47022 | 库两周涨到 ~72 GB | closed |
| #53760 | `event` 表 9.03 GB，消息内容只有 1.09 GB | closed |
| #33356 | `event` 表无限增长，无 retention / compaction | open |
| #16101 | 会话生命周期管理、统一存储回收 | open |

结论：官方知道这个问题，但**没有自动回收**。当时那版 CLI（v2.0.32）顶层 `--help` 里**没有** `vacuum` / `prune` / `db`，`session` 子命令只有 `list / delete / export / import`；`db` 子命令只存在于 dev 分支规格。清理只能自己做（子命令清单一节见 `opencode-host/托管OpenCode-CLI的定位与子命令.md`）。

## 已验证 / 推测

- **已验证**（2026-10-22 复核）：`part` 无 `type` 列、类型要用 `json_extract` 取、实际类型清单与体积排序；删除是硬删加级联、孤立 part/message 为 0；freelist 语义与文件不缩；本机 `auto_vacuum=0` / `page_size=4096` / WAL；`PRAGMA auto_vacuum=FULL` 不带 `VACUUM` 是空操作；`VACUUM INTO` 继承源库的 auto_vacuum、但不继承 WAL（产出库是 rollback journal）；原地 VACUUM 被读事务挡住会 `database is locked`，`VACUUM INTO` 不会；`sqlite3` CLI 不存在、Python 3.12 可用；四个 issue 的状态；CLI 顶层无维护命令、`session` 四个子命令。
- **推测**：`incremental_vacuum` 模式下应用不会主动调用（只是没看到调用，没审完源码）；压缩后能落到活跃数据的 1~1.5 倍这个量级（按 freelist 比例估，未实测整库替换）。
