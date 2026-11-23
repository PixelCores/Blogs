# SGLang `ir.py` 源码理解笔记

> 目的：这份笔记用于对照 SGLang 源码阅读 `python/sglang/lang/ir.py`，理解 SGLang 前端 DSL 是如何把用户写的 Python Prompt Program 转换为内部表达式，并交给 interpreter / tracer 处理的。

官方仓库：

```text
https://github.com/sgl-project/sglang
```

建议对照阅读的文件：

```text
python/sglang/lang/api.py
python/sglang/lang/ir.py
python/sglang/lang/interpreter.py
python/sglang/lang/tracer.py
```

如果你是从 pip 安装后的包目录里找源码，路径通常不是 `python/sglang/lang/ir.py`，而是类似：

```text
site-packages/sglang/lang/ir.py
```

---

## 1. 先给一个整体结论

`ir.py` 不是在定义一门新语言的传统 AST。

SGLang 并不会重新解析一个 `.sgl` 文件，也不是先把 Python 源码解析成 AST，再编译执行。

它的做法更接近：

```text
Python DSL
  -> 通过 decorator / 函数调用 / 运算符重载 / 状态对象
  -> 构造或提交一组内部表达式节点
  -> interpreter 执行这些节点
  -> backend.generate() / backend.select() 调用模型
```

所以 `ir.py` 里的这些类更像是“提示词程序”的语义操作：

```text
SglFunction       一个可运行的 SGLang 程序
SglExpr           所有表达式节点的基类
SglExprList       多个表达式的序列
SglConstantText   固定文本
SglGen            模型生成
SglSelect         候选选择
SglVariable       变量依赖占位，主要用于 trace/IR graph
SglFork           分叉节点，主要用于 trace/IR graph
SglRoleBegin      chat role 开始
SglRoleEnd        chat role 结束
SglSamplingParams 采样参数
```

一句话概括：

```text
SGLang 前端 = Python DSL + IR 节点 + interpreter/tracer
```

---

## 2. 从一个最小例子开始

用户一般会这样写 SGLang 程序：

```python
import sglang as sgl

@sgl.function
def answer_question(s, question):
    s += sgl.system("You are a helpful assistant.")
    s += sgl.user(question)
    s += sgl.assistant("Answer: " + sgl.gen("answer", max_tokens=128))
    return s["answer"]
```

表面上看，这是普通 Python 代码。

但 SGLang 关心的是这些操作背后的语义：

```text
1. 创建一个 SGLang 函数对象
2. 往 prompt 里追加 system message
3. 往 prompt 里追加 user message
4. 进入 assistant role
5. 追加固定文本 "Answer: "
6. 调用模型生成 answer
7. 把 answer 保存成变量
8. 返回变量 answer
```

这些语义最终会落到 `ir.py` 里的表达式类上。

---

## 3. `SglFunction`：把普通 Python 函数包装成 SGLang 程序

入口在 `api.py`：

```python
def function(func=None, num_api_spec_tokens=None):
    if func:
        return SglFunction(func, num_api_spec_tokens=num_api_spec_tokens)

    def decorator(func):
        return SglFunction(func, num_api_spec_tokens=num_api_spec_tokens)

    return decorator
```

所以：

```python
@sgl.function
def answer_question(s, question):
    ...
```

本质上会变成：

```python
answer_question = SglFunction(answer_question)
```

`ir.py` 里的 `SglFunction.__init__` 会检查用户函数签名：

```python
class SglFunction:
    def __init__(self, func, num_api_spec_tokens=None, bind_arguments=None):
        self.func = func
        self.num_api_spec_tokens = num_api_spec_tokens
        self.bind_arguments = bind_arguments or {}
        self.pin_prefix_rid = None

        argspec = inspect.getfullargspec(func)
        assert argspec.args[0] == "s", 'The first argument must be "s"'
        self.arg_names = argspec.args[1:]
        self.arg_defaults = argspec.defaults if argspec.defaults is not None else []
```

重点是：

```python
assert argspec.args[0] == "s"
```

这解释了为什么 SGLang 函数第一个参数一般写成 `s`。

这个 `s` 不是普通字符串，而是运行时传进去的 `ProgramState`，代表当前 prompt 构造状态。

当你调用：

```python
answer_question.run(question="SGLang 是什么？", backend=runtime)
```

会进入：

```python
def run(...):
    from sglang.lang.interpreter import run_program

    default_sampling_para = SglSamplingParams(...)
    backend = backend or global_config.default_backend

    return run_program(
        self,
        backend,
        args,
        kwargs,
        default_sampling_para,
        stream,
        use_thread=use_thread,
    )
```

所以 `SglFunction` 的作用是：

```text
把一个普通 Python 函数包装成可运行、可批量运行、可 trace、可 cache 的 SGLang 程序对象。
```

调用链可以理解为：

```text
@sgl.function
  -> SglFunction
  -> SglFunction.run()
  -> interpreter.run_program()
  -> ProgramState / StreamExecutor
  -> backend.generate() / backend.select()
```

需要注意一个源码细节：

```python
def bind(self, **kwargs):
    assert all(key in self.arg_names for key in kwargs)

    new_bind_dict = {**self.bind_arguments, **kwargs}
    return SglFunction(self.func, bind_arguments=new_bind_dict)
```

当前 `bind()` 里重新构造 `SglFunction` 时没有显式传递 `num_api_spec_tokens`。如果后续文章要分析 `bind()`，不要自然假设所有配置都会被完整继承。

---

## 4. `SglExpr`：所有表达式节点的基类

`SglExpr` 是大部分 IR 节点的基类：

```python
class SglExpr:
    node_ct = 0

    def __init__(self):
        self.node_id = SglExpr.node_ct
        self.prev_node = None
        self.pid = None
        SglExpr.node_ct += 1
```

它维护了几个图结构相关字段：

```text
node_id    节点 ID
prev_node  前驱节点
pid        program id / trace 相关标记
```

这些字段在普通 run 模式下不是最核心的执行路径；普通执行更多依赖 `StreamExecutor._execute()` 根据节点类型进行分发。

`node_id` / `prev_node` 更重要的场景是：

```text
tracing
IR graph 打印
表达节点之间的依赖关系
```

`SglExpr` 还重载了 `+`：

```python
def __add__(self, other):
    if isinstance(other, str):
        other = SglConstantText(other)
    assert isinstance(other, SglExpr)

    return self.concatenate_ir(self, other)


def __radd__(self, other):
    if isinstance(other, str):
        other = SglConstantText(other)
    assert isinstance(other, SglExpr), f"{other}"

    return self.concatenate_ir(other, self)
```

这就是为什么下面这种写法可以工作：

```python
"Answer: " + sgl.gen("answer")
```

普通字符串会被包装成：

```python
SglConstantText("Answer: ")
```

然后和 `SglGen("answer")` 拼成：

```python
SglExprList([
    SglConstantText("Answer: "),
    SglGen("answer"),
])
```

也就是说，SGLang 并没有解析 Python 源码，而是借助 Python 的运算符重载在运行时构造表达式。

---

## 5. `SglExprList`：表达式序列

源码结构很简单：

```python
class SglExprList(SglExpr):
    def __init__(self, expr_list: List[SglExpr]):
        super().__init__()
        self.expr_list = expr_list

    def __repr__(self):
        return f"ExprList({self.expr_list})"
```

它表示多个表达式顺序执行。

比如：

```python
sgl.assistant("Answer: " + sgl.gen("answer"))
```

大概会变成：

```python
SglExprList([
    SglRoleBegin("assistant"),
    SglConstantText("Answer: "),
    SglGen("answer"),
    SglRoleEnd("assistant"),
])
```

执行时，interpreter 会展开它：

```python
elif isinstance(other, SglExprList):
    for x in other.expr_list:
        self._execute(x)
```

所以 `SglExprList` 可以理解成 SGLang IR 里的“顺序块”。

---

## 6. `SglConstantText`：固定文本节点

`SglConstantText` 表示一段固定文本：

```python
class SglConstantText(SglExpr):
    def __init__(self, value: str):
        super().__init__()
        self.value = value

    def __repr__(self):
        return f"Constant({repr(self.value)})"
```

用户写：

```python
s += "hello"
```

或者：

```python
"Answer: " + sgl.gen("answer")
```

里面的普通字符串最终会被包装为 `SglConstantText`。

执行时：

```python
if isinstance(other, SglConstantText):
    self._execute_fill(other.value)
```

`_execute_fill()` 会把文本追加到当前 prompt 文本里：

```python
self.text_ += value
```

所以 `SglConstantText` 的作用很直接：

```text
往当前 prompt 状态里追加一段固定文本。
```

---

## 7. `SglGen`：模型生成节点

`SglGen` 是最核心的表达式之一。

源码简化后是：

```python
class SglGen(SglExpr):
    def __init__(
        self,
        name: Optional[str] = None,
        max_new_tokens: Optional[int] = None,
        min_new_tokens: Optional[int] = None,
        n: Optional[int] = None,
        stop: Optional[Union[str, List[str]]] = None,
        temperature: Optional[float] = None,
        top_p: Optional[float] = None,
        dtype: Optional[type] = None,
        regex: Optional[str] = None,
        json_schema: Optional[str] = None,
        ...
    ):
        super().__init__()
        self.name = name
        self.sampling_params = SglSamplingParams(...)

    def __repr__(self):
        return f"Gen('{self.name}')"
```

用户写：

```python
s += sgl.gen("answer", max_tokens=128, temperature=0.7)
```

`api.py` 里的 `gen()` 会返回：

```python
return SglGen(
    name,
    max_tokens,
    min_tokens,
    n,
    stop,
    stop_token_ids,
    stop_regex,
    temperature,
    top_p,
    top_k,
    ...
)
```

注意：`SglGen` 本身不直接调用模型。它只是一个表达式节点，含义是：

```text
执行到这里时，请调用模型生成一段文本。
生成结果保存到变量 name 里。
生成参数使用 sampling_params。
```

真正调用模型的是 `interpreter.py` 里的 `_execute_gen()`：

```python
def _execute_gen(self, expr: SglGen):
    sampling_params = self._resolve_sampling_params(expr.sampling_params)
    name = expr.name

    comp, meta_info = self.backend.generate(
        self,
        sampling_params=sampling_params,
    )

    if isinstance(comp, list):
        self.text_ += comp[0]
    else:
        self.text_ += comp

    self.variables[name] = comp
    self.meta_info[name] = meta_info
    self.variable_event[name].set()
```

这段逻辑做了几件事：

```text
1. 合并默认采样参数和当前 gen 节点的局部采样参数
2. 调用 backend.generate()
3. 把生成结果追加到当前 prompt 文本 text_
4. 把生成结果保存到 variables[name]
5. 保存 meta_info
6. 通知等待这个变量的逻辑：变量已经 ready
```

所以 `SglGen` 可以理解成：

```text
一条“生成指令”。
它既改变当前 prompt 状态，也产出一个命名变量。
```

这是 SGLang DSL 和普通字符串模板最大的区别之一。

普通模板只是拼文本；SGLang 的 `gen()` 是一个带副作用的语义节点：

```text
生成文本 + 追加上下文 + 保存变量
```

---

## 8. `SglSelect`：候选选择节点

`SglSelect` 表示从固定候选项中选择一个结果。

源码简化如下：

```python
class SglSelect(SglExpr):
    def __init__(
        self,
        name: str,
        choices: List[str],
        temperature: float,
        choices_method: ChoicesSamplingMethod,
    ):
        super().__init__()
        self.name = name
        self.choices = choices
        self.temperature = temperature
        self.choices_method = choices_method

    def __repr__(self):
        return f"Select({self.name}, choices={self.choices}, choices_method={self.choices_method})"
```

用户可以显式写：

```python
s += sgl.select(
    "sentiment",
    choices=["positive", "negative", "neutral"],
)
```

也可以通过 `gen(choices=...)` 间接得到 `SglSelect`。

`api.py` 里：

```python
def gen(..., choices=None, choices_method=None, ...):
    if choices:
        return SglSelect(
            name,
            choices,
            0.0 if temperature is None else temperature,
            token_length_normalized if choices_method is None else choices_method,
        )
```

严格说，是当 `choices` 非空、truthy 时，`gen()` 返回 `SglSelect`；如果 `choices` 是 `None` 或空列表，则不会走这个分支。

执行时：

```python
def _execute_select(self, expr: SglSelect):
    choices_decision = self.backend.select(
        self, expr.choices, expr.temperature, expr.choices_method
    )

    if expr.name is not None:
        name = expr.name
        self.variables[name] = choices_decision.decision
        self.meta_info[name] = choices_decision.meta_info
        self.variable_event[name].set()

    self.text_ += choices_decision.decision
```

`SglSelect` 和 `SglGen` 的区别：

```text
SglGen:
    开放式生成，模型可以生成任意文本。

SglSelect:
    受限选择，只能从 choices 里选一个。
```

例如：

```python
s += "这个评论的情感是："
s += sgl.select("label", ["正面", "负面", "中性"])
```

这类节点适合分类、路由、打标签、判断等任务。

---

## 9. `SglVariable`：变量依赖占位，主要用于 trace

`SglVariable` 源码很短：

```python
class SglVariable(SglExpr):
    def __init__(self, name: str, source):
        super().__init__()
        self.name = name
        self.source = source

    def __repr__(self):
        return f"Variable('{self.name}', source=%{self.source.node_id})"
```

这里需要特别注意：

```text
普通 run 模式下，s["answer"] 通常直接拿到实际生成出来的字符串。
trace 模式下，因为不会真的调用模型，所以 s["answer"] 会返回 SglVariable 占位节点。
```

普通 run 模式里：

```python
def __getitem__(self, name):
    return self.get_var(name)


def get_var(self, name):
    return self.stream_executor.get_var(name)
```

再往下：

```python
def get_var(self, name):
    if name in self.variable_event:
        self.variable_event[name].wait()
    return self.variables[name]
```

也就是说，普通运行时读变量，拿的是变量表里的真实值。

但是 trace 模式里不同。

`tracer.py` 中，遇到 `SglGen` 时不会调用模型，而是创建一个变量占位：

```python
def _execute_gen(self, expr: SglGen):
    name = expr.name if expr.name is not None else "gen_" + str(len(self.variables))
    new_node = SglVariable(name, source=expr)
    self.variables[name] = new_node
    self._append_node(expr)
```

然后 `get_var()` 返回一个 `SglVariable`：

```python
def get_var(self, name):
    ret = self.arguments.get(name, None)
    if ret is not None:
        return ret

    v = self.variables[name]
    return SglVariable(v.name, v.source)
```

所以 `SglVariable` 更准确的理解是：

```text
在 trace / IR graph 中，表示某个变量来自哪个生成节点或选择节点。
```

它不是传统 AST 里的简单 Identifier，而是带来源依赖的变量引用：

```text
answer 变量
  -> 来自某个 SglGen("answer") 节点
```

这对 trace、graph 打印、prefix 分析、依赖关系表达很重要。

---

## 10. `SglFork`：分叉节点，主要用于 trace 图

`SglFork` 源码：

```python
class SglFork(SglExpr):
    def __init__(self, number: int, position_ids_offset=None):
        super().__init__()
        self.number = number
        self.position_ids_offset = position_ids_offset

    def __repr__(self):
        return (
            f"Fork(%{self.prev_node.node_id}, number={self.number}, "
            f"position_ids_offset={self.position_ids_offset})"
        )
```

它表示：

```text
从当前状态分叉出多个后续执行分支。
```

但这里也要注意普通运行和 trace 的区别。

普通运行路径下，用户写：

```python
branches = s.fork(3)
```

实际走的是 `ProgramState.fork()`：

```python
def fork(self, size=1, position_ids_offset=None):
    stream_executors = self.stream_executor.fork(size, position_ids_offset)
    states = [ProgramState(x) for x in stream_executors]
    state_group = ProgramStateGroup(states, self)
    return state_group
```

再往下，`StreamExecutor.fork()` 会复制当前执行状态：

```python
for i in range(size):
    exes[i].variables = dict(self.variables)
    exes[i].text_ = str(self.text_)
    exes[i].messages_ = list(self.messages_)
    exes[i].cur_role = self.cur_role
    exes[i].cur_role_begin_pos = self.cur_role_begin_pos
    exes[i].fork_start_text_pos = len(self.text_)
    exes[i].images_ = list(self.images_)
```

也就是说，普通运行时的 fork 本质是：

```text
复制当前 StreamExecutor 状态，得到多个可以继续独立执行的 ProgramState。
```

而在 trace 模式中，才会显式构造 `SglFork` 节点：

```python
def fork(self, size: int = 1, position_ids_offset=None):
    fork_node = SglFork(size)
    fork_node.prev_node = self.last_node

    states = [
        TracerProgramState(self.backend, self.arguments, self.only_trace_prefix)
        for _ in range(size)
    ]

    for i in range(size):
        node = SglGetForkItem(i)
        node.prev_node = fork_node
        states[i].last_node = node
        ...
```

因此文章里最好这样描述：

```text
SglFork 在 trace/IR 图中表达从当前节点分叉出多个执行分支；普通运行时的 s.fork() 主要通过复制 StreamExecutor 状态实现。
```

`fork` 之后通常会配合 `join()`。

`ProgramStateGroup.join()` 默认模式是：

```python
def join(self, mode: str = "gather_variable"):
    if mode == "gather_variable":
        # Copy variables back
        ...
    elif mode == "concate_and_append":
        self.src_state += SglConcateAndAppend(self.states)
        ...
```

两个模式的区别是：

```text
gather_variable:
    把子分支中新产生的变量收集回父状态。

concate_and_append:
    把子分支文本/KV cache 相关内容拼回父状态。
```

所以 fork/join 不只是“多路生成”这么简单，它还涉及：

```text
变量如何回收到父状态
文本和 KV cache 是否拼回主状态
```

---

## 11. `SglRoleBegin` / `SglRoleEnd`：chat role 边界

对于 chat model 来说，prompt 不只是一个字符串，还可能是 OpenAI chat API 风格的 messages：

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."},
]
```

SGLang 用两个 IR 节点表示 role 边界：

```python
class SglRoleBegin(SglExpr):
    def __init__(self, role: str):
        super().__init__()
        self.role = role

    def __repr__(self):
        return f"RoleBegin({self.role})"


class SglRoleEnd(SglExpr):
    def __init__(self, role: str):
        super().__init__()
        self.role = role

    def __repr__(self):
        return f"RoleEnd({self.role})"
```

用户写：

```python
s += sgl.user("你好")
```

`api.py` 里会变成：

```python
def user(expr=None):
    return _role_common("user", expr)


def _role_common(name, expr=None):
    if expr is None:
        return SglExprList([SglRoleBegin(name), SglRoleEnd(name)])
    else:
        return SglExprList([SglRoleBegin(name), expr, SglRoleEnd(name)])
```

所以：

```python
sgl.user("你好")
```

大致会变成：

```python
SglExprList([
    SglRoleBegin("user"),
    SglConstantText("你好"),
    SglRoleEnd("user"),
])
```

执行 role begin：

```python
def _execute_role_begin(self, expr: SglRoleBegin):
    assert self.cur_role is None, "Nested roles are not allowed."

    self.cur_role = expr.role

    prefix, _ = self.chat_template.get_prefix_and_suffix(expr.role, self.messages_)
    self._execute_fill(prefix, prefix=True)
    self.cur_role_begin_pos = len(self.text_)
```

执行 role end：

```python
def _execute_role_end(self, expr: SglRoleEnd):
    self.cur_role = None

    new_text = self.text_[self.cur_role_begin_pos :].lstrip()

    _, suffix = self.chat_template.get_prefix_and_suffix(expr.role, self.messages_)
    self._execute_fill(suffix)

    self.messages_.append({"role": expr.role, "content": new_text})
```

所以 Role 节点同时维护两套状态：

```text
text_      用于 completion 风格的 prompt 文本
messages_  用于 chat API 风格的 messages
```

这也是 SGLang 可以适配 completion backend 和 chat backend 的原因之一。

---

## 12. `SglSamplingParams`：生成参数对象

`SglSamplingParams` 保存生成参数：

```python
@dataclasses.dataclass
class SglSamplingParams:
    max_new_tokens: int = 128
    min_new_tokens: int = 0
    n: int = 1
    stop: Union[str, List[str]] = ()
    stop_token_ids: Optional[List[int]] = ()
    stop_regex: Optional[Union[str, List[str]]] = ()
    temperature: float = 1.0
    top_p: float = 1.0
    top_k: int = -1
    min_p: float = 0.0
    frequency_penalty: float = 0.0
    presence_penalty: float = 0.0
    ignore_eos: bool = False
    return_logprob: Optional[bool] = None
    json_schema: Optional[str] = None

    dtype: Optional[str] = None
    regex: Optional[str] = None
```

它的作用不是只保存参数，还负责把 SGLang 自己的参数转换成不同 backend 的参数格式。

比如 OpenAI backend：

```python
def to_openai_kwargs(self):
    return {
        "max_tokens": self.max_new_tokens,
        "max_completion_tokens": self.max_new_tokens,
        "n": self.n,
        "stop": self.stop or None,
        "temperature": self.temperature,
        "top_p": self.top_p,
        "frequency_penalty": self.frequency_penalty,
        "presence_penalty": self.presence_penalty,
    }
```

这里可以看出一个设计意图：

```text
SGLang DSL 层希望提供统一生成参数；
具体 backend 支持哪些参数、字段名叫什么，由 adapter 转换处理。
```

---

## 13. interpreter 如何执行这些节点

核心分发逻辑在 `interpreter.py` 的 `_execute()`：

```python
def _execute(self, other):
    if isinstance(other, str):
        other = SglConstantText(other)

    assert isinstance(other, SglExpr), f"{other}"

    if isinstance(other, SglConstantText):
        self._execute_fill(other.value)
    elif isinstance(other, SglGen):
        self._execute_gen(other)
    elif isinstance(other, SglSelect):
        self._execute_select(other)
    elif isinstance(other, SglExprList):
        for x in other.expr_list:
            self._execute(x)
    elif isinstance(other, SglRoleBegin):
        self._execute_role_begin(other)
    elif isinstance(other, SglRoleEnd):
        self._execute_role_end(other)
    elif isinstance(other, SglVariable):
        self._execute_variable(other)
    elif isinstance(other, SglVarScopeBegin):
        self._execute_var_scope_begin(other)
    elif isinstance(other, SglVarScopeEnd):
        self._execute_var_scope_end(other)
    elif isinstance(other, SglCommitLazy):
        self._execute_commit_lazy_operations(other)
    elif isinstance(other, SglConcateAndAppend):
        ...
    elif isinstance(other, SglSeparateReasoning):
        self._execute_separate_reasoning(other)
    else:
        raise ValueError(f"Unknown type: {type(other)}")
```

这段代码是理解 SGLang 前端 IR 的关键。

它说明：

```text
ir.py 定义节点类型；
interpreter.py 根据节点类型执行不同语义。
```

所以看 `ir.py` 时，不要只看 class 定义，还要顺着 `interpreter.py` 里的 `_execute_xxx()` 看每个节点真正产生的副作用。

---

## 14. tracer 如何记录这些节点

普通 run 模式会真的执行生成。

trace 模式则不会真的调用模型，而是记录表达式和依赖关系。

`SglFunction.__call__()` 里会判断当前是否处于 tracing scope：

```python
def __call__(self, *args, **kwargs):
    from sglang.lang.tracer import TracingScope

    tracing_scope = TracingScope.get_current_scope()
    if tracing_scope is None:
        return self.run(*args, **kwargs)
    else:
        kwargs["backend"] = tracing_scope.tracer_state.backend
        return self.trace(*args, **kwargs)
```

`trace_program()` 里创建 `TracerProgramState`：

```python
def trace_program(program, arguments, backend):
    if backend is None:
        backend = BaseBackend()

    dummy_arguments = {
        name: SglArgument(name, None)
        for name in program.arg_names
        if name not in arguments
    }
    arguments.update(dummy_arguments)
    arguments.update(program.bind_arguments)

    tracer = TracerProgramState(backend, arguments, only_trace_prefix=False)
    with TracingScope(tracer):
        tracer.ret_value = program.func(tracer, **arguments)
    return tracer
```

trace 模式下遇到 `SglGen`：

```python
def _execute_gen(self, expr: SglGen):
    name = expr.name if expr.name is not None else "gen_" + str(len(self.variables))
    new_node = SglVariable(name, source=expr)
    self.variables[name] = new_node
    self._append_node(expr)
```

遇到 `fork()`：

```python
fork_node = SglFork(size)
fork_node.prev_node = self.last_node
```

所以：

```text
SglVariable / SglFork 在 trace/IR graph 语境下尤其重要。
```

这也是自审后需要修正的重点：

```text
不要把 SglVariable 和 SglFork 说成普通运行路径里一定会直接执行的核心节点。
更准确地说：
    SglGen / SglSelect / SglRoleBegin / SglRoleEnd / SglConstantText 是普通执行路径中更直接的表达式节点；
    SglVariable / SglFork 在 trace/IR graph 语境下更重要。
```

---

## 15. 整体执行流程图

以这个程序为例：

```python
@sgl.function
def answer_question(s, question):
    s += sgl.system("You are a helpful assistant.")
    s += sgl.user(question)
    s += sgl.assistant("Answer: " + sgl.gen("answer", max_tokens=128))
    return s["answer"]
```

运行流程可以理解为：

```text
@sgl.function
  -> 创建 SglFunction

answer_question.run(...)
  -> SglFunction.run()
  -> interpreter.run_program()
  -> 创建 StreamExecutor
  -> 创建 ProgramState
  -> 调用用户函数 program.func(state, ...)

用户函数内部：
  s += sgl.system(...)
    -> SglExprList([SglRoleBegin, SglConstantText, SglRoleEnd])
    -> StreamExecutor.submit(expr)
    -> StreamExecutor._execute(expr)

  s += sgl.user(question)
    -> SglExprList([SglRoleBegin, SglConstantText, SglRoleEnd])
    -> 更新 text_ / messages_

  s += sgl.assistant("Answer: " + sgl.gen("answer"))
    -> SglExprList([
         SglRoleBegin("assistant"),
         SglConstantText("Answer: "),
         SglGen("answer"),
         SglRoleEnd("assistant"),
       ])
    -> 执行 SglGen
    -> backend.generate()
    -> self.text_ += generated_text
    -> self.variables["answer"] = generated_text

  return s["answer"]
    -> 从变量表读取 answer
```

---

## 16. 一个更适合写进文章的版本

如果要放到你的 SGLang 源码浅析文章里，可以这样写：

```text
其中 `ir.py` 里定义了 SGLang 前端 DSL 的核心中间表示。`SglFunction` 负责把一个普通 Python 函数包装成可运行、可批量运行、可 trace 的 SGLang 程序；`SglGen` 表示一次模型生成操作；`SglSelect` 表示一次受限候选选择；`SglVariable` 主要在 tracing/IR 图中表示生成变量与其来源节点之间的依赖关系；`SglFork` 则主要在 trace 图中表达从当前状态分叉出多个后续执行分支，而普通运行时的 `s.fork()` 实际是通过复制 `StreamExecutor` 状态实现的。

这些类并不是传统意义上对 Python 语法的 AST 解析。SGLang 并不会重新解析一门 `.sgl` 语言，而是借助 Python 本身的函数调用、装饰器、运算符重载和状态对象，把用户写下的 `s += "Answer: " + sgl.gen("answer")` 这类代码转换成一组内部表达式或直接提交给 interpreter 执行。随后 interpreter 根据这些节点依次更新 prompt 状态、调用后端模型、保存变量、处理 chat role，并在需要时支持 trace、batch、stream 和 fork/join。
```

---

## 17. 阅读源码时的建议顺序

建议不要直接从 `ir.py` 从头读到尾，而是按下面顺序：

```text
1. api.py
   看用户 API 如何创建 IR 节点：
   function / gen / select / system / user / assistant

2. ir.py
   看 IR 节点的数据结构：
   SglFunction / SglExpr / SglGen / SglSelect / SglVariable / SglFork

3. interpreter.py
   看普通 run 模式如何执行节点：
   ProgramState / StreamExecutor / _execute / _execute_gen / _execute_select

4. tracer.py
   看 trace 模式如何记录节点和依赖：
   TracerProgramState / SglVariable / SglFork / prev_node
```

这样更容易建立完整心智模型：

```text
api.py        用户写法入口
ir.py         表达式定义
interpreter.py 真实执行
tracer.py     记录图结构和依赖
```

---

## 18. 最后总结

`ir.py` 的价值不在于“复杂的数据结构”，而在于它把 LLM Prompt Program 里的关键语义抽象了出来：

```text
文本追加
模型生成
受限选择
变量引用
角色边界
状态分叉
采样参数
```

这些节点让 SGLang 可以做到：

```text
1. 用户用 Python 写自然的 Prompt Program
2. 前端 DSL 不绑定具体模型后端
3. interpreter 可以统一执行不同表达式
4. tracer 可以记录程序结构和依赖关系
5. runtime 可以支持 batch / stream / fork / prefix cache 等能力
```

所以理解 `ir.py` 的关键不是把它当成传统 AST，而是把它当成 SGLang Prompt Program 的“语义指令集”。
