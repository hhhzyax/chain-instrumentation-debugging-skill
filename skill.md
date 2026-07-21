---
name: chain-instrumentation-debugging
description: Use when a bug has been traced through static code analysis but the root cause remains hidden—every code path looks correct in isolation yet the runtime behavior is wrong. The methodology adds targeted instrumentation across a call chain to capture the full data flow, identifying exactly where and why correct state turns incorrect.
---

# 链式插桩调试

## 概述

当 bug 发生在多步调用链中（特别是涉及递归或多次遍历同一批数据），静态代码分析无法定位根因时，在调用链的每个关键节点插入带"身份标签"的临时日志，用日志重建完整的数据流，找到"正确的状态是在哪一步被谁改坏的"。

**核心原则：** 每条日志必须能回答三个问题——处理的是什么？做了什么决策？是谁（哪个调用路径）做的？

## 何时使用

- 已通读所有相关代码，每条路径单独看都"逻辑正确"，但运行时结果就是不对
- 数据经过多个处理阶段，不确定中间态在哪个阶段被改变
- 同类型的一批对象中只有部分出问题（空间分布、ID 分布等有明显模式）
- 同一个函数被多条调用路径触发，需要区分每次调用来自哪里

**不适用：** 有明确堆栈的单步错误、类型/编译错误、一眼就能定位的简单 bug。

## 五步法

### 第一步：理解整条链路

从触发条件到最终效果，理清数据经过了哪些函数、哪些分支、哪些循环。不要跳过"看起来没问题"的环节——bug 往往就在你跳过的地方。

这一步的目标不是画精确的调用图，而是回答：**数据从入口到出口，到底经过了谁的手？**

### 第二步：给每个节点加"身份日志"

给调用链上每个关键函数加一个额外的 `caller` 参数（不影响原逻辑），日志格式统一：

```
[阶段标签] 对象标识 关键属性 决策结果 caller=来源
```

```javascript
// 入口
log(`[ENTER] total=${items.length} filter=${activeFilter}`)

// 判定函数
function isMatch(item, filter) {
  const result = compute(item, filter)
  log(`[MATCH] id=${item.id} key=${item.key} result=${result}`)
  return result
}

// 操作函数——核心：加 caller 参数追踪调用来源
function apply(item, visible, caller = 'main') {
  log(`[APPLY] id=${item.id} visible=${visible} prev=${item.state} caller=${caller}`)
  visible ? item.show() : item.hide()
  // 递归子元素时传递身份
  item.children.forEach(child => apply(child, visible, 'recurse'))
}
```

### 第三步：加"谁触发了它"探头

当你看到某个对象被错误处理，但不知道是哪个上游调用导致的——在可疑的触发方加探头，记录触发方身份和被它影响的目标：

```javascript
function apply(item, visible, caller = 'main') {
  log(`[APPLY] id=${item.id} visible=${visible} prev=${item.state} caller=${caller}`)
  visible ? item.show() : item.hide()

  // 探头：记录父是谁 + 它的子有哪些（仅在你关心的类型出现时 log）
  const targets = item.children.filter(c => c.type === TARGET_TYPE)
  if (targets.length > 0) {
    log(`[PARENT] parentType=${item.type} parentId=${item.id} ` +
        `childIds=${targets.map(c => c.id)} myDecision=${visible}`)
  }
  targets.forEach(child => apply(child, visible, 'recurse'))
}
```

### 第四步：追踪数据流，找"翻转点"

在日志中搜索你关心的那个对象的 ID，追踪它的完整生命周期。找到状态从"正确"变成"错误"的那一步——那就是根因所在的那层调用。

### 第五步：VCS 清理 + 最小修复

```bash
git checkout -- .   # 扔掉所有调试日志，回到干净状态
```

基于日志证据施加修复。优先同时修两处：
- **防御层**（通用保护）：让下游代码不盲目信任上游数据
- **源头层**（阻止发生）：让产生错误数据的逻辑不再产生

## 快速参考

| 阶段 | 动作 | 关键点 |
|------|------|--------|
| 1. 理解链路 | 理清数据经过哪些函数、分支、循环 | 不要跳过"看起来没问题"的环节 |
| 2. 加身份 | 统一日志格式，关键函数加 caller 参数 | 这是区分不同调用路径的唯一方式 |
| 3. 加探头 | 在合适节点记录上下文信息 | 能回答"谁触发的、影响了谁"即可 |
| 4. 找翻转 | 追踪目标对象的完整日志链路 | 找到状态从正确变错误的那一步 |
| 5. 清理 | VCS 回退所有调试代码 | 比手工回退更干净、更安全 |

## 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 日志不带调用来源标识 | 分不清是外层循环还是内层递归干的 | 每个关键函数加 caller/stage 参数 |
| 只在目标对象上打日志 | 不知道谁触发了对它的处理 | 在触发方加日志，记录"谁→处理了→谁" |
| 手工逐行回退调试代码 | 残留死代码、引入新 bug | `git checkout` 或 `git stash` 一步到位 |
| 只修直接症状不修源头 | 脏数据还在，其他路径可能再次触发 | 防御层 + 源头层同时修 |
| 全量日志不加过滤 | 关键信息被淹没 | `console.clear()` 后触发一次，只关注目标对象 |

## 与 system debugging 的关系

此技能是 `systematic-debugging` 的补充。`systematic-debugging` 定义四阶段流程（根因→模式→假设→实施），本技能提供**第一阶段中收集运行时证据的具体技术**——当静态分析走不通时，如何高效地在调用链上插桩收集证据。
