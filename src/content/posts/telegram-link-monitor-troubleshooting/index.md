---
title: "Telegram 群链接监控机器人：从建起来到排掉五个真实的坑"
published: 2026-09-18
description: "群里的链接要自动转发、要定时催人、还要能手动收工。工具不到 600 行，却在 tkinter 的线程模型、cmd 的编码和 Telethon 的 update 协议上各摔了一跤，这篇是完整复盘。"
tags: ["Telegram", "Python", "Telethon", "tkinter", "SQLite", "排错"]
category: 自动化
draft: false
---

需求很具体：值班群里每天固定几个时间段会有人发一条短链（形如 `https://sck.io/l/XXXXXXXX`），我要把它**原样转发**到另一个群、并且不暴露原发送者；如果某个时间段一直没人发，就每隔一段时间在群里 @ 提醒一次。

写起来不难，大概 600 行 Python：`main.py` 跑 Telethon 负责监听和转发，`gui.py` 用 tkinter 做配置编辑和状态面板，数据落在 SQLite。但从「能跑」到「敢让它 7×24 挂着」，中间踩了五个坑。这篇按时间顺序复盘，每个坑都按 **现象 → 排查 → 根因 → 修复** 写。

## 一、先把设计说清楚

### 任务模型：每任务每天一行

核心表是 `task_runs`，**一个任务在一个自然日只有一行**：

| 字段 | 含义 |
|------|------|
| `run_id` | 主键，`任务名\|2026-09-18` |
| `start_at` / `end_at` | 这个时间窗口的实际绝对时间 |
| `status` | `waiting` / `found` / `done` / `expired` |
| `found_count` / `required` | 已收到几条 / 这个窗口需要几条 |
| `urls` | JSON 数组，存全部链接和各自的时间 |
| `last_reminder_at` / `reminder_count` | 提醒节流用 |

用「每任务每天一行」而不是「每次收到链接插一行」，是因为要回答的问题天然是**窗口级**的：「这个时间段的任务做完了吗？」「上次提醒是什么时候？」一行就能表达，不用聚合。

### 消费式分配

一条链接只归一个任务：把所有 `waiting` 且当前处于时间窗口内的任务按 `start_at` 升序排，取**最早开始且还没收满**的那个。这样时间段重叠时，先开始的任务先被满足，符合直觉。

### 两个容易写错的边界

**跨午夜**：`end <= start` 就认为窗口跨天，`19:00 → 01:00` 的结束时间加一天。配套还有一个「逻辑日期」函数——凌晨 00:30 收到的链接，应该算**前一天**那个窗口的，否则每天半夜都会凭空多出一个任务。

**消息去重**：单独一张 `processed_messages` 表记 `(chat_id, message_id)`。Telethon 断线重连后可能重放更新，内存里的集合重启就没了，必须落库。

## 二、坑 1：界面点不动，但什么都不报

**现象**：程序启动后弹出「选择登录方式」窗口，两个按钮点了没反应，右上角 × 也关不掉。整个窗口像死了一样。

**排查**：这种「完全无响应且无报错」的情况，先怀疑回调抛异常。用 AST 把类里的方法定义按行号列出来：

```python
for node in ast.walk(ast.parse(src)):
    if isinstance(node, ast.ClassDef) and node.name == 'LoginDialog':
        for f in node.body:
            if isinstance(f, ast.FunctionDef):
                print(f'  def {f.name}  -> line {f.lineno}')
```

输出：

```
  def __init__  -> line 61
  def _choose  -> line 94
  def _cancel  -> line 98
  def _choose  -> line 120   ← 重复了
  def _cancel  -> line 124   ← 重复了
```

**根因**：这个类里混进了一段旧版本的残留代码，`_choose` / `_cancel` 被定义了两次。**Python 里后定义的静默覆盖前面的**，而后面那份引用的是一个不存在的属性 `self.win`（旧版本用的是 `self.win`，新版改成了 `self.root`）。于是每次点击都是：

```python
AttributeError: 'LoginDialog' object has no attribute 'win'
```

不用开窗口就能复现——绕过 `__init__` 直接调方法：

```python
d = object.__new__(gui.LoginDialog)
d.result = None
gui.LoginDialog._choose(d, 'bot')   # AttributeError: ... no attribute 'win'
```

**为什么一个报错都没看到**：启动脚本用的是 `pythonw.exe`（无控制台）。tkinter 的回调异常走 `report_callback_exception`，默认是往 `sys.stderr` 打印——而 `pythonw` 下 `sys.stderr` 是 `None`，异常就这么凭空消失了。窗口于是表现为「卡死」。

**修复**：删掉重复定义，同时给每个 Tk 根窗口挂上自己的异常出口：

```python
self.root.report_callback_exception = self._on_error

def _on_error(self, exc_type, exc_value, exc_tb):
    detail = "".join(traceback.format_exception(exc_type, exc_value, exc_tb))
    messagebox.showerror("界面错误", detail, parent=self.root)
```

**教训**：**「无响应且无日志」几乎总是异常被吞了**，而不是真的死锁。无控制台运行时，一定要自己兜住异常；否则你会花几个小时去找一个根本不存在的死锁。

## 三、坑 2：后台线程碰 tkinter

程序结构是「后台线程跑 asyncio 事件循环，主线程跑 tkinter」。日志转发最初这么写：

```python
def write(self, msg):
    self.app.root.after(0, self.app.append_log, msg.strip())   # 从子线程调 Tk
```

`root.after` 看起来像个「安全投递」，其实它仍然是在**跨线程操作 Tcl 解释器**。Tkinter 不是线程安全的，这种写法在负载上来以后会偶发卡住甚至崩掉，而且**无法稳定复现**——正是最难查的那类问题。

**修复**：不再让子线程碰任何 Tk 对象，改成「子线程只写队列，主线程定时取」：

```python
def run_in_ui_thread(app, fn, timeout=10.0):
    """把 fn 排到主线程执行，并等待返回结果。"""
    box, done = {}, threading.Event()

    def _wrapper():
        try:
            box["value"] = fn()
        except Exception as e:
            box["error"] = e
        finally:
            done.set()

    app.ui_queue.put(_wrapper)
    if not done.wait(timeout):
        raise TimeoutError("GUI 主线程无响应")
    if "error" in box:
        raise box["error"]
    return box.get("value")
```

主线程侧一个 150ms 的泵，同时干两件事：执行排队的 UI 任务、批量刷日志。

```python
def _pump(self):
    while True:
        try:
            fn = self.ui_queue.get_nowait()
        except queue.Empty:
            break
        fn()
    # 日志攒一批再一次性 insert，避免每行都触发一次重绘
    self.root.after(150, self._pump)
```

手机号登录要弹窗收验证码，那个回调也是从 asyncio 线程发起的，同样走这个队列。

顺带修了一个会让界面「顿一下」的小问题：GUI 每 5 秒读一次 SQLite 显示状态，用的是默认 5 秒 busy timeout。如果此时后台正在写库，主线程最多会干等 5 秒——**界面看起来就是卡住了**。把连接的 `timeout` 调到 2 秒，宁可这次读不到、下次再读。

## 四、坑 3：`start.bat` 里不能写中文

**现象**：双击 `start.bat` 启动，Windows 弹「应用程序无法启动」。

**排查**：先别急着改代码，**先确认这个弹窗是谁发的**。Python 脚本出错只会打印 traceback，绝不会说「应用程序无法启动」——这句话是 Windows 对 **exe** 说的（典型于 `0xc0000142`、并行配置错误、.NET 崩溃）。

于是分头取证：

1. 手动按资源管理器的方式跑一遍 `start.bat`，进程**正常起来了**；
2. 查系统事件日志，近 7 天**没有任何** python / 本程序启动失败的记录，倒是有几个完全无关的 `GCUService.exe`、`Marvis.exe`、`DllHost.exe` 崩溃；
3. 直接把当前屏幕上所有窗口标题枚举出来看（比截图更准，也不受分辨率影响）。

第 3 步用 P/Invoke 就够了，不需要任何截图工具：

```csharp
[DllImport("user32.dll")] static extern bool EnumWindows(EnumProc cb, IntPtr p);
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
static extern int GetWindowTextW(IntPtr h, StringBuilder s, int n);
```

结果很清楚：屏幕上根本没有那个对话框，程序也确实是启动成功的。**这个报错属于另一个程序，和本工程无关**——如果顺着「双击 start.bat 报错」的描述去猜，方向从一开始就错了。

不过这轮排查里确实发现了一个**真**问题：我顺手给 `start.bat` 加中文提示时，脚本自己坏了。这台机器控制台代码页是 **936**，而 cmd.exe 解析批处理时对多字节字符的处理有缺陷——每行**行首字符会被吃掉**，然后整行报 `'cho' is not recognized`。UTF-8 和 GBK 两种编码我都试了，**都会坏**。

**修复**：`start.bat` 保持**纯 ASCII**，中文只出现在 Python 打印的内容里。同时补上路径校验，失败时把原因说清楚再 `pause`，避免黑窗一闪什么都看不到：

```bat
set "PY=%~dp0.venv\Scripts\python.exe"
if not exist "%PY%" goto novenv
start "Telegram Monitor" "%PY%" "%~dp0gui.py"
exit /b 0

:novenv
echo [ERROR] Python venv not found:
echo     %PY%
pause
```

**教训**：两个。第一，**先确认错误是谁报的**，别顺着用户的描述猜；第二，**`.bat` 里别放非 ASCII 字符**，这个坑值得写进项目文档。

## 五、坑 4：需求变了——一个时间段要发两条链接

上线几天后新需求来了：**同一个时间段里可能先后要发两条链接**。

原来的模型是「一条链接消费一个任务」：任务收到第一条就变 `found`，从此不再接收。第二条链接打过来时，日志只会写一句「当前没有需要消费链接的任务」，然后被丢掉。

**改法**：给任务加一个「需要几条」`required`，任务在窗口内可以连续收多条，收满才自动完成。

```yaml
- name: "任务2"
  start: "17:00"
  end: "23:00"
  required: 2        # 这个时间段要发两条
```

数据库加三列，并且**迁移必须幂等 + 回填历史数据**（线上库里有已经跑完的记录）：

```python
def migrate_schema(conn):
    cols = {row[1] for row in conn.execute("PRAGMA table_info(task_runs)")}
    if "required" not in cols:
        conn.execute("ALTER TABLE task_runs ADD COLUMN required INTEGER NOT NULL DEFAULT 1")
    if "found_count" not in cols:
        conn.execute("ALTER TABLE task_runs ADD COLUMN found_count INTEGER NOT NULL DEFAULT 0")
    if "urls" not in cols:
        conn.execute("ALTER TABLE task_runs ADD COLUMN urls TEXT")
    # 老数据：状态是 found 的，视为已收到 1 条
    conn.execute("UPDATE task_runs SET found_count=1 WHERE status='found' AND found_count=0")
```

用 SQLite 的 `ALTER TABLE ADD COLUMN` 加列是 O(1) 的元数据操作，不重写表；判断 `PRAGMA table_info` 保证重复执行安全。主进程和 GUI 启动时都会调一次，谁先起来谁迁移。

写入改成「先读、再算、再写」，全部在同一个锁里完成：

```python
async def record_found(self, run_id, message_id, url, found_at):
    async with self.lock:
        row = self.conn.execute(
            "SELECT status, required, found_count, urls FROM task_runs WHERE run_id=?", (run_id,)
        ).fetchone()
        if row is None or row["status"] != "waiting":
            return None                      # 任务已经完成/过期，不再接收

        required = max(1, int(row["required"] or 1))
        found_count = int(row["found_count"] or 0) + 1
        urls = json.loads(row["urls"]) if row["urls"] else []
        urls.append({"url": url, "found_at": found_at, "message_id": message_id})

        completed = found_count >= required
        self.conn.execute(
            "UPDATE task_runs SET found_count=?, urls=?, url=COALESCE(url, ?), "
            "status=? WHERE run_id=? AND status='waiting'",
            (found_count, json.dumps(urls, ensure_ascii=False), url,
             "found" if completed else "waiting", run_id),
        )
        self.conn.commit()
        return found_count, required, completed
```

几个细节值得说：

- `url` 这个旧字段用 `COALESCE` **保留第一条**，只做兼容展示；完整列表进 `urls`（JSON），不破坏老代码。
- 返回 `None` 表示「任务在读表和写库之间被完成了」，调用方据此**不转发**，避免并发下多发一条。
- 提醒逻辑一行没改就自动正确了：提醒的条件本来就是「状态是 `waiting`」，而 `required: 2` 只收到 1 条时状态仍是 `waiting`，于是继续催——正好是想要的行为。

**再补一个手动出口**。收到 1 条但另一条今天不发了，得能提前收工，于是 GUI 加了「手动完成任务」：选中一行 → 状态置 `done` → 本窗口不再接收链接、不再提醒；下个时间段自动重开。「重置选中任务」可以撤回。表格新增「进度」列（`1/2`），双击一行看这个任务收到的**全部**链接和时间。

**上线验证**：第二天看库里，那条 `required: 2` 的任务是 `found_count: 2, required: 2`，中间提醒过 1 次——两条都收到了，功能生效。

## 六、坑 5：Telethon 每几十秒刷一行日志

**现象**：控制台稳定地冒这种行，间隔几十秒：

```
2026-09-11 13:58:21,494 | INFO | Got difference for channel 2064760796 updates
2026-09-11 13:59:26,134 | INFO | Got difference for channel 2064760796 updates
2026-09-11 13:59:55,163 | INFO | Got difference for channel 2064760796 updates
```

**先定位**：这不是自己打的日志，`grep` 一下 telethon 源码就找到了出处——`telethon/client/updates.py:474`：

```python
updates, users, chats = self._message_box.apply_channel_difference(...)
if updates:
    self._log[__name__].info('Got difference for channel %d updates', ...)
```

**为什么会这样**：Telegram 的更新流是**带序号的**，用来保证「一条消息都不漏」：

- 账号级一份 `pts` / `qts` / `seq` / `date`；
- **每个频道各有一份独立的 `pts`**——频道的消息计数本来就是各自独立的序列，一个全局计数器根本对不上。

服务端推来的更新带着 pts，客户端一旦发现序号跳号，就**必须**调 `updates.GetChannelDifference` 把中间那段补回来，否则消息就永久丢了。补到内容时，上面那行日志就出现了。

所以「每个频道一份状态」不是 Telethon 多此一举，是协议本身如此。状态还持久化在 session 文件里——打开 `phone_session.session` 能看到一张 `update_state` 表，里面是 1 行账号级 + 12 行频道级。

关键点在于：**你的程序只关心一个群，但 Telethon 不能只同步那一个**。跳过的频道会让全局序号对不齐，最后连你要的群也开始丢更新。

**能关掉吗**：三个候选开关都查了。

| 开关 | 结论 |
|------|------|
| `receive_updates=False` | 唯一「真关」的开关，但它关掉的是**整个更新接收**，源码注释写明 event handlers 将不会工作——等于把程序废掉 |
| `catch_up=False` | 已经是默认值，且只管启动时要不要补历史，与这个循环无关 |
| 只对个别频道关 | **没有这个 API**。`end_channel_difference` 的 reason 只有 `TEMPORARY_SERVER_ISSUES` 和 `BANNED` 两个 |

硬去 monkeypatch 内部状态强行为不关心的频道「结束差异」，会让那些频道的 pts 错乱，而且 `_updates` 是内部模块、升级就可能失效——为了省几个很小的 RPC 冒这个险不划算。

**结论：关不掉，只能过滤。** 而且只需要过滤**这一条**，同模块下真正的告警（被踢出频道、拉取差异失败、网络中断）必须保留：

```python
class _DropChannelDifferenceNoise(logging.Filter):
    def filter(self, record):
        return not record.getMessage().startswith("Got difference for channel")

logging.getLogger("telethon.client.updates").addFilter(_DropChannelDifferenceNoise())
```

顺带一提：这行日志其实说明**连接是健康的、更新在正常收**。真正该处理的是它把日志刷满——而不是它本身。

## 七、几个可复用的工程习惯

### 1. 端到端测试可以不碰网络

监听转发的逻辑写在 `main()` 内部的闭包里，直接测不到。但可以把 `TelegramClient` 换成假对象，跑**真实的 `main()`**：

```python
class FakeClient:
    def on(self, builder):
        def deco(fn):
            self.handlers.append(fn)
            return fn
        return deco

    async def run_until_disconnected(self):
        await asyncio.sleep(0.3)          # 等 scheduler 建好当天的任务行
        for msg in type(self).inbox:
            for handler in self.handlers:
                await handler(FakeEvent(msg))
```

用一个临时 `config.yaml` + 临时数据库，喂三条消息进去，断言「前两条被转发到群二、第三条不转发、库里 `2/2`」。这条测试直接覆盖了这次改动的核心，比手点界面可靠得多。

> 有个小细节：第一版 fake 里忘了在触发事件前 `sleep`，scheduler 还没来得及建当天的任务行，导致三条链接全被判定「无人消费」。**测试挂了先怀疑测试**。

### 2. 让错误可见

- 无控制台运行时（`pythonw` / 双击启动），**一定**要自己兜异常；
- 后台线程的异常不能只在日志里躺着，要能看见；
- 启动脚本失败要 `pause`，别让窗口一闪而过。

### 3. 迁移要幂等、要回填

`PRAGMA table_info` 判断 + `ALTER TABLE ADD COLUMN` + 老数据回填，三件套写全。任何时刻（主进程、GUI、升级到一半断电）重新执行都不该出问题。

### 4. 证据优先于猜测

这次最省时间的一步是「先确认弹窗是谁发的」。事件日志、进程枚举、数据库现状——这些是可验证的事实；「用户描述里说的那个原因」只是一个假设。两者冲突时，信事实。

### 5. 把坑写进文档

项目根目录放一个 `AGENTS.md`，里面记着：`.bat` 必须纯 ASCII、`MessageMediaWebPage` 必须过滤（Telegram 给链接自动生成网页预览卡片，`send_message` 的 `file=` 处理不了这个类型）、配置热重载只对部分字段生效需要重启……这些「反直觉但会再犯」的点，写下来收益最高。

## 总结

- **无响应 + 无日志 = 异常被吞了**，尤其是 `pythonw` 环境下，先补异常出口再谈别的；
- **Tkinter 只有主线程能碰**，跨线程一律走队列；
- **`.bat` 只写 ASCII**，中文交给程序打印；
- **需求变更先改数据模型**：加 `required` 而不是加分支；状态机（`waiting/found/done/expired`）一清楚，提醒、过期、统计全都自然成立；
- **Telethon 的 channel difference 关不掉**，那是协议层面的可靠性机制，过滤噪音即可；
- **能被自动化验证的核心逻辑就别靠手点**，一个假客户端能覆盖大半。
