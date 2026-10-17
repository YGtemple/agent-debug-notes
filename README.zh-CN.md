<div align="right">

[English](README.md) · **[简体中文](README.zh-CN.md)**

</div>

# agent-debug-notes

Codex、Trae、WorkBuddy 三个工具、39 次真实会话里踩过的坑——把最耗时的那些写下来，只耗一次。

<sub>Markdown · 不含代码，每条结论都能回溯到会话</sub>

---

## 这是什么

会话记录很长，而且大量重复。有用的通常只有一句话——真正定位到原因的那一刻。这份笔记就是那句话，外加足够的上下文，让当时的情况能重现。

筛选原则：**只留带具体参数、命令、阈值或复现路径的结论**，不收泛泛之谈。

## ComfyUI 跑在 AMD ROCm 实例上

这是篇幅最大的一块——一台 `gfx1100`，从第一次启动到跑起 LTX-2.3。

### 「重新连接」有三种不同原因

ComfyUI 页面反复重连不是显卡问题，是前端连不上后端，而有三种完全不同的原因，按出现频率排：

| 现象 | 真实原因 | 怎么判断 |
|---|---|---|
| 能打开，一执行就断 | 进程被内存压力杀掉 | `/queue` 和 `/history` 都返回 `Failed to fetch` |
| 生成久了之后断 | 隧道被重置（radeon-tunnel） | ComfyUI 进程还活着 |
| Notebook 单元结束时断 | 前台进程被连带清理 | PID 不存在，`/tmp/comfyui.log` 也没了 |

`/queue` 和 `/history` 都失败，说明后端是真的挂了，不是浏览器缓存。

### 前台启动是大部分问题的根源

用 Notebook 前台进程跑，意味着单元结束或 Jupyter 连接抖动时，ComfyUI 和隧道进程会被一起清理。

```bash
nohup python main.py > /tmp/comfyui.log 2>&1 &
```

之后 Notebook 的启动单元必须一直保持运行——调试时极容易忘。

### 22B BF16 是硬退出

加载 `ltx-2.3-22b-dev.safetensors` 后进程直接消失，日志里**没有 Python traceback**。这是 ROCm 内存压力下的硬退出，不是代码错误。

有效的参数组，按重要程度排：

```bash
# 1. AMD 显存分段（必需）
export PYTORCH_HIP_ALLOC_CONF=expandable_segments:True

# 2. 关闭异步权重换入——gfx1100 上最常见的硬退出路径
python main.py --disable-async-offload --cpu-vae

# 3. 再保守一档
python main.py --lowvram
```

`--disable-async-offload` 是这台机器的关键。AMD 上异步权重换入在 22B 推理时最容易挂。慢，但进程能活下来。

### 模型文件名：大小写与后缀

工作流 JSON 和磁盘上的实际文件对不上：

| 工作流 JSON 里写的 | 磁盘上实际的 |
|---|---|
| `ltx-2.3-22b-dev-FP8.Safetensors` | `ltx-2.3-22b-dev.safetensors` |
| `gemma_3_12B_it_fp4_mixed.safetensors` | `gemma_3_12B_it.safetensors` |

大小写和 `FP8` / `fp4_mixed` 后缀都不同——加载必然失败。

### Connection refused 到底说明什么

```
PingSucceeded : True
TcpTestSucceeded : False
Connection refused
```

这个组合说明：主机可达、本机网络正常、密钥根本没被验证过——云平台侧没有映射 SSH 端口。不是你的问题。

处理顺序，中途不要反复重跑 SSH 命令：

1. 确认实例状态是**运行中**——「准备好了」不等于运行中
2. 停止，等状态变成**已停止**，再启动
3. 等 1–3 分钟，从控制台重新复制 SSH 命令（重建后端口会变）
4. 仍然失败，带着实例号、公网 IP、端口联系平台

### 依赖连锁

用 `--no-deps` 安装会留下二级依赖缺失，失败是一条链：

```
缺 packaging / mako
→ 缺 regex（transformers 的依赖）
→ 缺 charset_normalizer / urllib3（requests 的）
→ 缺 multidict / propcache（yarl 的）
→ 缺 aiohappyeyeballs / aiosignal / attrs / frozenlist（aiohttp 的）
→ tokenizers 版本冲突
```

两个具体值得记住的：

- `tokenizers==0.23.0` **在 PyPI 上没有正式版**，只有 rc。降级到 `0.22.2`，别去找 0.23.0。
- 系统 Python 受 PEP 668 管理，直接 `pip install` 会被拒。`--break-system-packages` 可以绕过去（容器里可接受）。

### 参数名不是看着的那样

```bash
# 错：--disable-custom-nodes 不是有效参数
# 对：--disable-all-custom-nodes
```

启动成功的标志是日志里出现 `To see the GUI go to: http://0.0.0.0:8188`。

### 从 Jupyter 访问

把 JupyterLab 地址里的 `/lab` 换成 `/proxy/8188/`。

`/proxy/` 404 通常意味着缺 `jupyter-server-proxy`，或者反向代理没转发该路径。配置在 `/opt/venv/etc/jupyter`。

### 销毁实例前抢救数据

执行命令不代表文件能挺过销毁——实例的临时磁盘通常会被删除。

顺序：先 `df -h` 和 `ls -lah /workspace`，看平台有没有持久化目录（`/data`、`/mnt`、`/persistent`、`/root/autodl-tmp`），打包复制过去，**最后**再销毁。

## Cocos Creator / 微信小游戏

### 构建前先确认平台支持

在跑 Creator 3.8.8 构建之前，确认构建工具支持 `wechatgame`。输出到 `NewProject/build/wechatgame`；构建只产出构建产物和引擎缓存，永远不碰玩法源码。

### 分包约束

- `ch2-8` 分包 `isBundle=true`，**主包对分包零引用**
- 主包 123 张纹理，要盯着 4 MB 上限
- 主纹理 ≤ 1.5 MB

### 玩法注册表怎么校验

重构检查器会找每个玩法子类上的 `onQuestionStart` / `onAnswer` / `onResult`——三个方法齐了才算「已实现」。

## 响应式布局

### 输入框必须 16px

```css
input { font-size: 16px; }  /* 防止 iOS Safari 聚焦时缩放 */
```

低于 16px，iOS 在输入框获得焦点时会缩放整个页面。

### 横向裁切看起来像内容丢失

在 `html/body` 上设 `overflow-x: hidden`，又配上依赖横向滚动的导航，结果就是「内容在，但手机上看不见」。

正解是真正的单列回流加完整换行——不是裁切。

### 三个能挡住溢出的属性

```css
.card      { overflow-wrap: break-word; }
.longtext  { word-break: break-all; }
.container { min-width: 0; }            /* flex / grid 子项必需 */
```

`min-width: 0` 是最常被漏掉的那个。flex 或 grid 子项默认 `min-width: auto`，会被内部的长文本撑开。

### 别信 320px 的截图工具

截图工具会把浏览器的最小布局宽度裁成输出尺寸，所以它验证不了窄屏布局。用设备模拟模式，否则你会把裁切造成的假象读成页面回归。

### 断点

- 320px 及以下：更小的边距与字号
- 390px：单列，联系方式纵向排列
- 桌面：两列项目栅格

## 两个值得记下的产品决策

### 文字输入 → 录音优先

文字优先的版本能证明「有一条诊断链路」。它评价不了语速、停顿、口头禅、是否像在读稿、语感是否自信。

于是录音成为主入口，文字输入降为对文稿做结构分析的备用手段。

### 泛化场景 → 聚焦路演

不做四个平行的表达场景，只做一个：练路演。理由是校园里正在办自己的比赛，那些队伍第一天就是可触达的用户，而这比一个通用的说话工具更能说明机构为什么要支持。

## 题库产品笔记

- 顺序模式优先出没做过的题
- 随机模式和模拟考试跳过最近看过的 120 道
- 非套题模式按题干 + 选项 + 答案去重；套题模式保留重复（一套卷还原原卷）

## 来源

| 来源 | 数量 | 内容 |
|---|---|---|
| `.codex/sessions/` | 39 个 rollout | 大部分调试过程，含完整诊断链 |
| `.trae-cn/memory/` | 45 个会话记忆 | 743 条结构化 `learned` |
| `.workbuddy/projects/` | 项目会话 | 交付记录 |

## 说明

只收录可复用的技术结论。主机名、端口、实例号已替换为占位符；不含 API 密钥与个人信息。
