# Hermes 默认长记忆机制源码解析：它不是本地向量数据库

> 结论先行：Hermes 默认的持久化记忆机制不是向量数据库，也不是 embedding 检索系统，而是“有容量上限的、由 Agent 主动维护的 Markdown 文件记忆”。默认长记忆由 `MEMORY.md` 和 `USER.md` 两个文件组成，启动时读入，渲染成 system prompt 的一部分；会话中通过 `memory` 工具增删改，并立即落盘。历史会话搜索则是另一套 SQLite + FTS5 全文检索系统，也不是向量库。

本文基于本机源码路径：

```text
/Users/pixelkernel/.hermes/hermes-agent
```

主要涉及源码文件：

```text
tools/memory_tool.py
run_agent.py
hermes_cli/config.py
hermes_state.py
agent/memory_manager.py
agent/memory_provider.py
website/docs/user-guide/features/memory.md
website/docs/user-guide/features/memory-providers.md
```

---

## 1. 默认长记忆到底存在哪里？

默认 built-in memory 存在 Hermes home 目录下的：

```text
~/.hermes/memories/MEMORY.md
~/.hermes/memories/USER.md
```

源码位置：`tools/memory_tool.py`

```python
def get_memory_dir() -> Path:
    """Return the profile-scoped memories directory."""
    return get_hermes_home() / "memories"
```

其中 `get_hermes_home()` 是 profile-aware 的。也就是说，如果你使用 Hermes profile，记忆目录会跟随当前 profile 的 `$HERMES_HOME`，不一定永远是主目录下的 `~/.hermes`。

两个文件的职责不同：

| 文件 | 含义 | 默认容量 |
|---|---|---:|
| `MEMORY.md` | Agent 的长期笔记：环境事实、项目约定、工具坑点、工作流经验 | 2200 字符 |
| `USER.md` | 用户画像：偏好、沟通风格、习惯、稳定个人信息 | 1375 字符 |

文档中也明确写了这点：`website/docs/user-guide/features/memory.md`

```markdown
Two files make up the agent's memory:

| File | Purpose | Char Limit |
|------|---------|------------|
| MEMORY.md | Agent's personal notes — environment facts, conventions, things learned | 2,200 chars |
| USER.md | User profile — your preferences, communication style, expectations | 1,375 chars |
```

---

## 2. 文件内部格式：普通 Markdown 文本 + 分隔符

Hermes 默认记忆不是 embedding，不需要向量索引。它就是普通文本条目，条目之间用 `§` 分隔。

源码位置：`tools/memory_tool.py`

```python
ENTRY_DELIMITER = "\n§\n"
```

读文件时：

```python
entries = [e.strip() for e in raw.split(ENTRY_DELIMITER)]
return [e for e in entries if e]
```

写文件时：

```python
content = ENTRY_DELIMITER.join(entries) if entries else ""
```

所以实际文件可能类似：

```markdown
User prefers generated LLM news digest Markdown documents to be in Simplified Chinese.
§
Project ~/code/api uses pytest and uv. Run tests with `uv run pytest`.
§
This machine has Docker Desktop and Homebrew installed.
```

这也是为什么默认 built-in memory 没有“语义召回”概念：它不做 embedding 相似度搜索，而是在每个新 session 开始时直接把全部记忆块塞进 system prompt。

---

## 3. 默认配置：built-in memory 开启，external provider 为空

默认配置在 `hermes_cli/config.py`：

```python
"memory": {
    "memory_enabled": True,
    "user_profile_enabled": True,
    "memory_char_limit": 2200,
    "user_char_limit": 1375,
    # External memory provider plugin (empty = built-in only).
    # Set to a provider name to activate: "openviking", "mem0",
    # "hindsight", "holographic", "retaindb", "byterover".
    # Only ONE external provider is allowed at a time.
    "provider": "",
},
```

关键点：

```text
memory.provider: ""
```

为空表示只使用 built-in memory。也就是：

```text
MEMORY.md + USER.md
```

不会自动启用 mem0、honcho、hindsight、retaindb 等外部记忆 provider。

你可以用下面命令确认当前状态：

```bash
hermes memory status
```

你当前机器上的输出是：

```text
Memory status
────────────────────────────────────────
  Built-in:  always active
  Provider:  (none — built-in only)
```

这说明当前没有启用外部 memory provider。

---

## 4. Agent 初始化时如何加载记忆？

初始化逻辑在 `run_agent.py`。

相关代码：

```python
# Persistent memory (MEMORY.md + USER.md) -- loaded from disk
self._memory_store = None
self._memory_enabled = False
self._user_profile_enabled = False
self._memory_nudge_interval = 10
self._turns_since_memory = 0
self._iters_since_skill = 0
if not skip_memory:
    try:
        mem_config = _agent_cfg.get("memory", {})
        self._memory_enabled = mem_config.get("memory_enabled", False)
        self._user_profile_enabled = mem_config.get("user_profile_enabled", False)
        self._memory_nudge_interval = int(mem_config.get("nudge_interval", 10))
        if self._memory_enabled or self._user_profile_enabled:
            from tools.memory_tool import MemoryStore
            self._memory_store = MemoryStore(
                memory_char_limit=mem_config.get("memory_char_limit", 2200),
                user_char_limit=mem_config.get("user_char_limit", 1375),
            )
            self._memory_store.load_from_disk()
    except Exception:
        pass  # Memory is optional -- don't break agent init
```

流程是：

1. 读取 config 里的 `memory` 配置。
2. 如果 `memory_enabled` 或 `user_profile_enabled` 为 true，就创建 `MemoryStore`。
3. 传入容量限制：`memory_char_limit` 和 `user_char_limit`。
4. 调用 `load_from_disk()` 从 Markdown 文件加载记忆。
5. 如果记忆加载失败，不让 Agent 初始化失败，因为 memory 是 optional。

`MemoryStore.load_from_disk()` 在 `tools/memory_tool.py`：

```python
def load_from_disk(self):
    """Load entries from MEMORY.md and USER.md, capture system prompt snapshot."""
    mem_dir = get_memory_dir()
    mem_dir.mkdir(parents=True, exist_ok=True)

    self.memory_entries = self._read_file(mem_dir / "MEMORY.md")
    self.user_entries = self._read_file(mem_dir / "USER.md")

    # Deduplicate entries (preserves order, keeps first occurrence)
    self.memory_entries = list(dict.fromkeys(self.memory_entries))
    self.user_entries = list(dict.fromkeys(self.user_entries))

    # Capture frozen snapshot for system prompt injection
    self._system_prompt_snapshot = {
        "memory": self._render_block("memory", self.memory_entries),
        "user": self._render_block("user", self.user_entries),
    }
```

这里有一个很重要的设计：

```text
load_from_disk() 不只是加载文件，还会捕获一份 frozen snapshot。
```

这份 frozen snapshot 后面会用于 system prompt 注入。

---

## 5. 记忆如何进入 system prompt？

构造 system prompt 的逻辑在 `run_agent.py`。

```python
if self._memory_store:
    if self._memory_enabled:
        mem_block = self._memory_store.format_for_system_prompt("memory")
        if mem_block:
            prompt_parts.append(mem_block)
    # USER.md is always included when enabled.
    if self._user_profile_enabled:
        user_block = self._memory_store.format_for_system_prompt("user")
        if user_block:
            prompt_parts.append(user_block)
```

`format_for_system_prompt()` 在 `tools/memory_tool.py`：

```python
def format_for_system_prompt(self, target: str) -> Optional[str]:
    """
    Return the frozen snapshot for system prompt injection.

    This returns the state captured at load_from_disk() time, NOT the live
    state. Mid-session writes do not affect this. This keeps the system
    prompt stable across all turns, preserving the prefix cache.
    """
    block = self._system_prompt_snapshot.get(target, "")
    return block if block else None
```

注意这句话：

```text
This returns the state captured at load_from_disk() time, NOT the live state.
```

也就是说，Hermes 的默认记忆采用“冻结快照模式”：

```text
新 session 开始时：读取 MEMORY.md / USER.md，生成 system prompt 快照。
会话进行中：memory 工具可以修改磁盘文件。
当前 session 的 system prompt：不会因为中途写入而变化。
下一个 session：重新读取磁盘文件，新记忆才进入 system prompt。
```

这样做的原因在注释里也写了：

```text
preserving the prefix cache
```

也就是保持 system prompt 前缀稳定，提高 LLM provider 的 prompt caching 命中率，避免每次写记忆都导致缓存失效。

---

## 6. system prompt 中的记忆块长什么样？

渲染逻辑在 `MemoryStore._render_block()`：

```python
def _render_block(self, target: str, entries: List[str]) -> str:
    """Render a system prompt block with header and usage indicator."""
    if not entries:
        return ""

    limit = self._char_limit(target)
    content = ENTRY_DELIMITER.join(entries)
    current = len(content)
    pct = min(100, int((current / limit) * 100)) if limit > 0 else 0

    if target == "user":
        header = f"USER PROFILE (who the user is) [{pct}% — {current:,}/{limit:,} chars]"
    else:
        header = f"MEMORY (your personal notes) [{pct}% — {current:,}/{limit:,} chars]"

    separator = "═" * 46
    return f"{separator}\n{header}\n{separator}\n{content}"
```

所以注入到 system prompt 的内容大概是：

```text
══════════════════════════════════════════════
USER PROFILE (who the user is) [10% — 143/1,375 chars]
══════════════════════════════════════════════
User prefers generated LLM news digest Markdown documents to be in Simplified Chinese for easier reading and stored under ~/LLM/ai-news-digest.
```

这就是为什么你在会话开头能看到 `USER PROFILE` 块。

---

## 7. 会话中如何写入记忆？

Agent 通过 `memory` 工具操作 built-in memory。

工具入口在 `tools/memory_tool.py`：

```python
def memory_tool(
    action: str,
    target: str = "memory",
    content: str = None,
    old_text: str = None,
    store: Optional[MemoryStore] = None,
) -> str:
    ...
    if action == "add":
        result = store.add(target, content)
    elif action == "replace":
        result = store.replace(target, old_text, content)
    elif action == "remove":
        result = store.remove(target, old_text)
    else:
        return tool_error(f"Unknown action '{action}'. Use: add, replace, remove", success=False)
```

工具暴露给模型的 schema 里也强调了它的用途：

```python
MEMORY_SCHEMA = {
    "name": "memory",
    "description": (
        "Save durable information to persistent memory that survives across sessions. "
        "Memory is injected into future turns, so keep it compact and focused on facts "
        "that will still matter later.\n\n"
        ...
    )
}
```

`run_agent.py` 中负责把工具调用路由到 `memory_tool()`：

```python
elif function_name == "memory":
    target = function_args.get("target", "memory")
    from tools.memory_tool import memory_tool as _memory_tool
    function_result = _memory_tool(
        action=function_args.get("action"),
        target=target,
        content=function_args.get("content"),
        old_text=function_args.get("old_text"),
        store=self._memory_store,
    )
```

也就是说：

```text
LLM 决定调用 memory 工具
        ↓
run_agent.py 捕获 function_name == "memory"
        ↓
tools.memory_tool.memory_tool(...)
        ↓
MemoryStore.add / replace / remove
        ↓
写入 MEMORY.md 或 USER.md
```

---

## 8. add / replace / remove 的具体行为

### 8.1 add：追加新条目

源码：`MemoryStore.add()`

```python
def add(self, target: str, content: str) -> Dict[str, Any]:
    """Append a new entry. Returns error if it would exceed the char limit."""
    content = content.strip()
    if not content:
        return {"success": False, "error": "Content cannot be empty."}

    # Scan for injection/exfiltration before accepting
    scan_error = _scan_memory_content(content)
    if scan_error:
        return {"success": False, "error": scan_error}

    with self._file_lock(self._path_for(target)):
        # Re-read from disk under lock to pick up writes from other sessions
        self._reload_target(target)

        entries = self._entries_for(target)
        limit = self._char_limit(target)

        # Reject exact duplicates
        if content in entries:
            return self._success_response(target, "Entry already exists (no duplicate added).")

        new_entries = entries + [content]
        new_total = len(ENTRY_DELIMITER.join(new_entries))

        if new_total > limit:
            ...

        entries.append(content)
        self._set_entries(target, entries)
        self.save_to_disk(target)
```

特点：

1. 去掉首尾空白。
2. 空内容拒绝。
3. 写入前做安全扫描。
4. 加文件锁。
5. 锁内重新读磁盘，避免多会话并发写覆盖。
6. 精确重复条目不重复添加。
7. 超过容量限制则拒绝。
8. 成功后立即保存到磁盘。

### 8.2 replace：用子串匹配替换

源码：`MemoryStore.replace()`

```python
matches = [(i, e) for i, e in enumerate(entries) if old_text in e]

if not matches:
    return {"success": False, "error": f"No entry matched '{old_text}'."}

if len(matches) > 1:
    unique_texts = set(e for _, e in matches)
    if len(unique_texts) > 1:
        return {
            "success": False,
            "error": f"Multiple entries matched '{old_text}'. Be more specific.",
            "matches": previews,
        }
```

它不是用 ID，而是用 `old_text` 子串查找条目。

例如原条目：

```text
User prefers Simplified Chinese for LLM news digest Markdown documents.
```

可以用：

```text
old_text = "Simplified Chinese"
```

如果匹配多条，就要求更具体。

### 8.3 remove：同样用子串匹配删除

源码：`MemoryStore.remove()`

```python
matches = [(i, e) for i, e in enumerate(entries) if old_text in e]
```

删除逻辑和 replace 类似：

```text
匹配 0 条：报错
匹配多条不同内容：要求更具体
匹配 1 条：删除并落盘
```

---

## 9. 写入安全：文件锁 + 原子替换

Hermes 默认记忆虽然是 Markdown 文件，但写入不是简单 `open(..., "w")` 粗暴覆盖，而是做了并发安全处理。

### 9.1 文件锁

源码：`MemoryStore._file_lock()`

```python
lock_path = path.with_suffix(path.suffix + ".lock")
...
if fcntl:
    fcntl.flock(fd, fcntl.LOCK_EX)
else:
    msvcrt.locking(fd.fileno(), msvcrt.LK_LOCK, 1)
```

它会在目标文件旁边使用 `.lock` 文件：

```text
MEMORY.md.lock
USER.md.lock
```

Unix/macOS/Linux 下使用 `fcntl.flock`；Windows 下尝试用 `msvcrt.locking`。

### 9.2 写临时文件 + fsync + atomic replace

源码：`MemoryStore._write_file()`

```python
fd, tmp_path = tempfile.mkstemp(
    dir=str(path.parent), suffix=".tmp", prefix=".mem_"
)
try:
    with os.fdopen(fd, "w", encoding="utf-8") as f:
        f.write(content)
        f.flush()
        os.fsync(f.fileno())
    atomic_replace(tmp_path, path)
```

这意味着：

1. 先在同目录写一个临时文件。
2. flush。
3. fsync。
4. 原子替换目标文件。

源码注释解释了为什么不直接 `open("w")`：

```python
Previous implementation used open("w") + flock, but "w" truncates the
file *before* the lock is acquired, creating a race window where
concurrent readers see an empty file. Atomic rename avoids this:
readers always see either the old complete file or the new one.
```

这点设计很关键：读者永远看到旧完整文件或新完整文件，不会看到写到一半的文件。

---

## 10. 安全扫描：防止把 prompt injection 写入 system prompt

因为默认记忆会进入 system prompt，所以 Hermes 在写入前做了轻量安全扫描。

源码：`tools/memory_tool.py`

```python
_MEMORY_THREAT_PATTERNS = [
    # Prompt injection
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    ...
    # Exfiltration via curl/wget with secrets
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_curl"),
    (r'wget\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_wget"),
    ...
]
```

扫描函数：

```python
def _scan_memory_content(content: str) -> Optional[str]:
    """Scan memory content for injection/exfil patterns. Returns error string if blocked."""
    # Check invisible unicode
    for char in _INVISIBLE_CHARS:
        if char in content:
            return f"Blocked: content contains invisible unicode character U+{ord(char):04X} (possible injection)."

    # Check threat patterns
    for pattern, pid in _MEMORY_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            return f"Blocked: content matches threat pattern '{pid}'. Memory entries are injected into the system prompt and must not contain injection or exfiltration payloads."

    return None
```

这不是完整的安全沙箱，但能防止一些明显危险的记忆被持久化到 system prompt 中。

---

## 11. 容量控制：为什么默认记忆很小？

默认限制：

```text
MEMORY.md: 2200 chars
USER.md:   1375 chars
```

这不是 token 级别限制，而是字符数限制。源码注释说明：

```python
Character limits (not tokens) because char counts are model-independent.
```

当新增条目超过限制时，`add()` 会返回错误：

```python
if new_total > limit:
    current = self._char_count(target)
    return {
        "success": False,
        "error": (
            f"Memory at {current:,}/{limit:,} chars. "
            f"Adding this entry ({len(content)} chars) would exceed the limit. "
            f"Replace or remove existing entries first."
        ),
        "current_entries": entries,
        "usage": f"{current:,}/{limit:,}",
    }
```

这个设计说明 built-in memory 不是“无限历史记忆”，而是“常驻 system prompt 的精简事实集合”。它适合保存：

```text
稳定偏好
长期项目约定
环境事实
工具坑点
用户明确要求记住的事实
```

不适合保存：

```text
完整聊天记录
大段日志
长代码片段
临时任务状态
低价值流水账
```

---

## 12. 默认 built-in memory 和 session_search 的区别

Hermes 还有一个 `session_search` 工具，可以检索过去会话。但它和默认长记忆不是一回事。

### 12.1 built-in memory

```text
位置：~/.hermes/memories/MEMORY.md / USER.md
实现：Markdown 文件
进入上下文方式：每个 session 启动时直接注入 system prompt
容量：很小，固定上限
用途：关键事实常驻上下文
是否向量库：否
```

### 12.2 session_search

`hermes_state.py` 文件开头说明：

```python
"""
SQLite State Store for Hermes Agent.

Provides persistent session storage with FTS5 full-text search, replacing
the per-session JSONL file approach. Stores session metadata, full message
history, and model configuration for CLI and gateway sessions.

Key design decisions:
- WAL mode for concurrent readers + one writer (gateway multi-platform)
- FTS5 virtual table for fast text search across all session messages
- Compression-triggered session splitting via parent_session_id chains
- Batch runner and RL trajectories are NOT stored here (separate systems)
- Session source tagging ('cli', 'telegram', 'discord', etc.) for filtering
"""
```

默认 SQLite 文件：

```python
DEFAULT_DB_PATH = get_hermes_home() / "state.db"
```

所以历史会话存在：

```text
~/.hermes/state.db
```

它建立了普通 FTS5 表：

```python
FTS_SQL = """
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content
);
```

还建立了 trigram FTS5 表，专门改善 CJK 等语言的子串搜索：

```python
# Trigram FTS5 table for CJK substring search.  The default unicode61
# tokenizer splits CJK characters into individual tokens, breaking phrase
# matching.  The trigram tokenizer creates overlapping 3-byte sequences so
# substring queries work natively for any script (CJK, Thai, etc.).
FTS_TRIGRAM_SQL = """
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts_trigram USING fts5(
    content,
    tokenize='trigram'
);
```

也就是说，session_search 是：

```text
SQLite + FTS5 全文检索
```

不是向量数据库。

---

## 13. 外部 memory provider：什么时候才可能有向量库？

Hermes 支持外部 memory provider。相关抽象在：

```text
agent/memory_provider.py
agent/memory_manager.py
plugins/memory/
```

文档 `website/docs/user-guide/features/memory-providers.md` 写道：

```markdown
Hermes Agent ships with 8 external memory provider plugins that give the agent persistent, cross-session knowledge beyond the built-in MEMORY.md and USER.md. Only one external provider can be active at a time — the built-in memory is always active alongside it.
```

启用方式：

```bash
hermes memory setup
hermes memory status
hermes memory off
```

或者手动配置：

```yaml
memory:
  provider: openviking   # or honcho, mem0, hindsight, holographic, retaindb, byterover, supermemory
```

外部 provider 的抽象接口在 `agent/memory_provider.py`：

```python
class MemoryProvider(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        """Short identifier for this provider (e.g. 'builtin', 'honcho', 'hindsight')."""

    @abstractmethod
    def is_available(self) -> bool:
        ...

    @abstractmethod
    def initialize(self, session_id: str, **kwargs) -> None:
        ...

    def system_prompt_block(self) -> str:
        return ""

    def prefetch(self, query: str, *, session_id: str = "") -> str:
        return ""

    def queue_prefetch(self, query: str, *, session_id: str = "") -> None:
        ...

    def sync_turn(self, user_content: str, assistant_content: str, *, session_id: str = "") -> None:
        ...

    @abstractmethod
    def get_tool_schemas(self) -> List[Dict[str, Any]]:
        ...
```

这套接口支持：

```text
system_prompt_block：把 provider 的静态信息加入 system prompt
prefetch：每轮前召回相关记忆
queue_prefetch：为下一轮预取
sync_turn：把当前对话写入 provider
get_tool_schemas：暴露 provider 自己的工具
handle_tool_call：路由 provider 工具调用
```

`MemoryManager` 负责管理这些 provider：

```python
class MemoryManager:
    """Orchestrates the built-in provider plus at most one external provider.

    The builtin provider is always first. Only one non-builtin (external)
    provider is allowed.  Failures in one provider never block the other.
    """
```

虽然注释里说“built-in provider plus external provider”，从当前源码实际路径看，默认 `MEMORY.md / USER.md` 这一套主要是通过 `MemoryStore` 和 `memory` 工具处理；external provider 则由 `MemoryManager` 管理，并作为 additive layer 挂在 built-in memory 旁边。

`MemoryManager.add_provider()` 里限制只能有一个外部 provider：

```python
if not is_builtin:
    if self._has_external:
        ...
        logger.warning(
            "Rejected memory provider '%s' — external provider '%s' is "
            "already registered. Only one external memory provider is "
            "allowed at a time. Configure which one via memory.provider "
            "in config.yaml.",
            provider.name, existing,
        )
        return
    self._has_external = True
```

当 `memory.provider` 不为空时，`run_agent.py` 会加载外部 provider：

```python
_mem_provider_name = mem_config.get("provider", "") if mem_config else ""

if _mem_provider_name:
    from agent.memory_manager import MemoryManager as _MemoryManager
    from plugins.memory import load_memory_provider as _load_mem
    self._memory_manager = _MemoryManager()
    _mp = _load_mem(_mem_provider_name)
    if _mp and _mp.is_available():
        self._memory_manager.add_provider(_mp)
```

所以：

```text
默认 provider 为空：不会加载外部 provider。
provider = mem0 / honcho / hindsight 等：才会进入外部记忆插件体系。
```

是否使用 embedding、向量库、知识图谱、远端服务，要看具体 provider 的实现。比如文档中对 Honcho 的说明包括 semantic search 和 server-side data；Mem0 等 provider 也可能使用向量化/语义检索。但这不是 built-in memory 的默认行为。

---

## 14. 一张流程图：默认记忆生命周期

```text
┌─────────────────────────────┐
│ ~/.hermes/memories/          │
│   MEMORY.md                  │
│   USER.md                    │
└──────────────┬──────────────┘
               │ session start
               ▼
┌─────────────────────────────┐
│ MemoryStore.load_from_disk() │
│ - read files                 │
│ - split by \n§\n             │
│ - deduplicate                │
│ - render frozen snapshot     │
└──────────────┬──────────────┘
               │ build system prompt
               ▼
┌─────────────────────────────┐
│ system prompt                │
│ - MEMORY block               │
│ - USER PROFILE block         │
└──────────────┬──────────────┘
               │ during conversation
               ▼
┌─────────────────────────────┐
│ LLM decides to call memory   │
│ action: add/replace/remove   │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ run_agent.py routes tool     │
│ to tools.memory_tool         │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ MemoryStore mutates entries  │
│ - scan injection patterns    │
│ - lock file                  │
│ - reload target from disk    │
│ - check char limit           │
│ - temp file + fsync          │
│ - atomic replace             │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ MEMORY.md / USER.md updated  │
│ immediately on disk          │
└──────────────┬──────────────┘
               │ next session
               ▼
┌─────────────────────────────┐
│ new snapshot includes change │
└─────────────────────────────┘
```

---

## 15. 和“向量数据库型记忆”的核心差异

| 维度 | Hermes 默认 built-in memory | 向量数据库型记忆 |
|---|---|---|
| 存储 | Markdown 文件 | 向量索引 + 文档库 |
| 路径 | `~/.hermes/memories/*.md` | provider 自定义 |
| 检索 | 不检索，启动时全量注入 | 根据 query 做 embedding 相似度搜索 |
| 容量 | 很小，默认 2200 + 1375 字符 | 通常可很大 |
| 更新方式 | Agent 主动调用 `memory` 工具增删改 | 可能自动抽取、自动 embedding、自动索引 |
| 注入方式 | system prompt frozen snapshot | 每轮动态 recall / prefetch |
| 依赖 | 无外部依赖 | 依赖 embedding model / DB / 服务 |
| 默认是否启用 | 是 | 否 |

---

## 16. 实用建议

### 16.1 如果你关心隐私和可控性

保持：

```yaml
memory:
  provider: ""
```

这样默认只使用本地 Markdown 文件，不接入外部服务。

可以查看：

```bash
hermes memory status
```

如果看到：

```text
Provider: (none — built-in only)
```

说明没有外部 provider。

### 16.2 如果你想手动检查记忆内容

直接打开：

```bash
open ~/.hermes/memories
```

或：

```bash
$EDITOR ~/.hermes/memories/MEMORY.md
$EDITOR ~/.hermes/memories/USER.md
```

注意：文件格式依赖 `\n§\n` 分隔。如果手动编辑，尽量保留这个分隔方式。

### 16.3 如果你想彻底关闭 built-in memory

可以在配置中关闭：

```yaml
memory:
  memory_enabled: false
  user_profile_enabled: false
```

或者用：

```bash
hermes config edit
```

修改后需要新 session 才会体现。

### 16.4 如果你想要语义记忆/向量召回

那就不是默认 built-in memory 的职责了，需要启用外部 provider：

```bash
hermes memory setup
```

然后选择 honcho、mem0、hindsight、retaindb 等 provider。启用后是否本地建库、远端存储、使用 embedding，要逐个看 provider 的 README 和配置。

---

## 17. 总结

Hermes 默认长记忆的设计非常朴素，但很实用：

```text
Markdown 文件持久化
+ 小容量人工/Agent curated facts
+ session start 注入 system prompt
+ 会话中通过 memory 工具写入
+ 文件锁和原子替换保证可靠写入
+ 安全扫描避免明显 prompt injection 持久化
```

它并不会默认创建本地向量数据库。

Hermes 中容易混淆的三种“记忆/回忆”能力应该分开理解：

1. built-in memory

```text
~/.hermes/memories/MEMORY.md
~/.hermes/memories/USER.md
```

用于关键长期事实，默认启用，不是向量库。

2. session_search

```text
~/.hermes/state.db
SQLite + FTS5 / trigram FTS5
```

用于搜索过去会话，不是向量库。

3. external memory providers

```text
honcho / mem0 / hindsight / holographic / retaindb / byterover / supermemory / openviking
```

可选启用，可能使用语义搜索、embedding、知识图谱、本地或远端存储。只有这类 provider 才可能引入向量数据库或类似系统。

因此，如果你的问题是：

> Hermes 默认会不会在本地构建一个向量数据库来管理长记忆？

答案是：

```text
不会。默认只是本地 Markdown 文件 + system prompt 注入；历史会话搜索使用 SQLite FTS5；向量/语义记忆只属于可选外部 provider 的范畴。
```
