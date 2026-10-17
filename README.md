# agent-debug-notes · Agent 调试经验笔记

> 一句话简介：从 Codex、Trae、WorkBuddy 三个 Agent 的 39 个会话记录里，把反复踩过的坑整理成可复用的排查手册。

内容来源：`.codex/sessions`（39 个 rollout 会话）、`.trae-cn/memory`（45 个 session_memory，743 条 learned）、`.workbuddy`（项目会话）。

筛选标准：**只保留带具体参数、命令、阈值、复现路径的结论**，不收录观点感悟。每条都标注了原始场景，方便回溯。

---

## 一、ComfyUI 云端部署（AMD ROCm 实例）

这部分占了大头——一台 Radeon Cloud 的 `gfx1100` AMD GPU 实例，从启动到跑 LTX-2.3 的完整踩坑链。

### 1.1 「重新连接」的三层原因（最常见，按出现频率排序）

ComfyUI 页面反复显示「重新连接」，**不是显卡问题**，是前后端连接断开。按这个顺序排查：

| 现象 | 真实原因 | 判断方法 |
|---|---|---|
| 页面能开，一执行就断 | **进程被显存压力杀掉** | `/queue`、`/history` 都返回 `Failed to fetch` |
| 长时间生成后断 | **临时隧道被重置**（radeon-tunnel） | ComfyUI 进程还活着（`ps -p <PID>`） |
| Notebook 单元结束就断 | **前台进程被连带清理** | PID 不存在，`/tmp/comfyui.log` 也没了 |

**关键判断**：如果 `/queue` 和 `/history` 都 `Failed to fetch`，说明后端连接真的断了，不是浏览器缓存。

### 1.2 前台启动是万恶之源

最初的启动方式是 Notebook 前台进程，导致 Jupyter 单元结束或连接波动时，**ComfyUI 和隧道进程被一起清理**。

解决：改成独立后台会话启动，脱离 Notebook 单元生命周期：

```bash
nohup python main.py > /tmp/comfyui.log 2>&1 &
```

**教训**：Notebook 的启动单元格必须一直保持运行——停止该单元格会关闭服务。这在调试时极容易忘。

### 1.3 22B BF16 模型显存硬退出

加载 `ltx-2.3-22b-dev.safetensors`（22B BF16）后进程直接消失，日志里**没有 Python traceback**——这是 ROCm/显存压力下的硬退出，不是代码错误。

有效参数组合（按优先级）：

```bash
# 1. AMD 显存分段（必配）
export PYTORCH_HIP_ALLOC_CONF=expandable_segments:True

# 2. 关闭异步权重换入 —— gfx1100 上这是最容易硬退出的路径
python main.py --disable-async-offload --cpu-vae

# 3. 进一步保守
python main.py --lowvram
```

其中 `--disable-async-offload` 是这台实例的关键——AMD 上异步权重换入在 22B 推理时最容易硬退出。慢一些，但进程不被打掉。

### 1.4 模型文件名大小写陷阱

工作流 JSON 里的模型名和云端实际文件名不一致：

| 工作流 JSON 里写的 | 云端实际 |
|---|---|
| `ltx-2.3-22b-dev-FP8.Safetensors` | `ltx-2.3-22b-dev.safetensors` |
| `gemma_3_12B_it_fp4_mixed.safetensors` | `gemma_3_12B_it.safetensors` |

大小写和 `FP8`/`fp4_mixed` 后缀不一致，加载必然失败。

### 1.5 SSH 端口 Connection refused 的判据

```
PingSucceeded : True
TcpTestSucceeded : False
Connection refused
```

**这个组合说明**：IP 可达、本机网络正常、密钥根本没机会被验证——是云平台侧 SSH 端口没映射，不是你的问题。

按这个顺序处理，别重复试 SSH 命令：

1. 平台页面确认实例状态是「运行中」——「准备好了」不等于运行中
2. 点「停止」，等状态变「已停止」，再点「启动」
3. 等 1–3 分钟，重新复制页面上的 SSH 命令（实例重建后端口会变）
4. 仍失败就联系平台客服，附上实例号、公网 IP、端口

### 1.6 依赖安装的连环坑

`--no-deps` 安装会留下二级依赖缺失，然后是一长串连锁：

```
缺 packaging / mako
→ 缺 regex（transformers 的依赖）
→ 缺 charset_normalizer / urllib3（requests 的依赖）
→ 缺 multidict / propcache（yarl 的依赖）
→ 缺 aiohappyeyeballs / aiosignal / attrs / frozenlist（aiohttp 的依赖）
→ tokenizers 版本冲突
```

**两个具体坑**：

- `tokenizers==0.23.0` **在 PyPI 上没有正式版**，只有 rc。降级装 `0.22.2` 而不是找 0.23.0。
- 系统 Python 受 PEP 668 管理，直接 pip install 会被拦。加 `--break-system-packages` 绕过（容器环境可接受）。

### 1.7 启动参数的正确写法

```bash
# 错误：--disable-custom-nodes 不是有效参数
# 正确：--disable-all-custom-nodes
```

启动成功的标志是日志出现 `To see the GUI go to: http://0.0.0.0:8188`。

### 1.8 通过 Jupyter 代理访问

在 JupyterLab 地址栏把 `/lab` 替换成 `/proxy/8188/` 即可访问 ComfyUI。

但 `/proxy/` 404 通常是因为**缺 jupyter-server-proxy 扩展**，或者反向代理没转发该路径。装扩展即可，配置文件在 `/opt/venv/etc/jupyter`。

### 1.9 实例销毁前的数据抢救

**仅仅执行命令不能保证「摧毁实例」后文件还在**——实例里的临时磁盘通常会被删除。

抢救顺序：先 `df -h` 和 `ls -lah /workspace` 看平台是否提供持久化目录（`/data`、`/mnt`、`/persistent`、`/root/autodl-tmp`），确认后再打包、复制到持久化目录，**最后**才点摧毁。

---

## 二、Cocos Creator / 微信小游戏

### 2.1 构建前先确认平台支持

执行 Creator 3.8.8 微信构建前，先确认构建工具支持 `wechatgame` 平台：

```bash
# 输出到 NewProject/build/wechatgame
# 构建只产生产物和引擎缓存，不改玩法源码
```

### 2.2 分包与主包约束

- `ch2-8` 分包 `isBundle=true`，**主包零跨分包引用**
- 主包含 123 张纹理时要注意 4MB 限制
- 主纹理 ≤1.5MB（见 word-sprite-quest-tooling 的 build.js 检查）

### 2.3 玩法注册表的判定方式

校验器判定玩法子类「是否只实现三方法」的方式值得注意——重构检查脚本会扫 `onQuestionStart` / `onAnswer` / `onResult` 三个抽象方法是否齐全。

---

## 三、响应式布局（移动端）

### 3.1 搜索框必须设 16px

```css
input { font-size: 16px; }  /* 避免 iPhone 聚焦时自动放大页面 */
```

小于 16px 时，iPhone 聚焦输入框会触发页面自动缩放。

### 3.2 横向裁切 = 内容「消失」的错觉

给 `html/body` 加 `overflow-x: hidden` 或 `overflow-x: clip`，配合依赖横向滑动的导航，会造成「内容其实存在，但手机上看不到」。

**正确做法**：真正的单列回流 + 完整换行，而不是横向裁切。

### 3.3 防溢出的三个属性

```css
.card      { overflow-wrap: break-word; }
.longtext  { word-break: break-all; }   /* 或 overflow-wrap: anywhere */
.container { min-width: 0; }            /* flex/grid 子项必须 */
```

`min-width: 0` 是 flex/grid 布局里最容易被漏的一条——子项默认 `min-width: auto`，会被内部长文本撑宽。

### 3.4 320px 的截图工具不可信

截图工具会把浏览器最小布局宽度裁切为输出尺寸，**不能作为严格的手机验证依据**。改用浏览器设备模拟模式（DevTools Device Mode），否则会把裁切结果误判为页面回归。

### 3.5 断点策略

- 320px 及以下：更小的边距和字号
- 390px：单列操作 + 纵向联系方式
- 桌面端：保持双列项目布局

---

## 四、产品定位的两次关键收敛

这两次不是技术问题，但记录了对产品方向的判断过程。

### 4.1 文字输入 → 录音优先

最初版本用文字输入做主入口，实际只能证明「有诊断流程」，不能评价：

- 语速是否过快
- 停顿是否自然
- 口头禅是否过多
- 是否像背稿
- 临场表达是否流畅

**结论**：录音才是主入口，文字输入降级为「备用入口」主要用于文稿结构分析。

低成本实现：浏览器 `MediaRecorder` 本地录音回放 + 时长/活动量/停顿估算 + 转写补充。

### 4.2 泛化场景 → 聚焦路演

不再平铺做四类表达场景，收束为：

> **智言：面向大学生创新创业路演的 AI 语音表达训练平台**

理由：学校正在组织创新大赛，参赛团队就是第一批真实用户，比泛泛做英语学习更容易被认定有校内培育价值。

---

## 五、题库产品的两项优化

### 5.1 重复题的处理

- 顺序刷题：优先展示未做题
- 随机刷题/模拟考试：避开最近 120 道已看题，降低连续重复
- 非套题模式按「题干 + 选项 + 答案」去重；套题模式保留同题多次出现（还原试卷语境）

### 5.2 部署

GitHub Pages 工作流 `.github/workflows/pages.yml`，静态单文件直接部署，成本 0。

---

## 附：记录来源

| 来源 | 数量 | 说明 |
|---|---|---|
| `.codex/sessions/` | 39 个 rollout | 主要踩坑过程，含完整诊断链 |
| `.trae-cn/memory/` | 45 个 session_memory | 743 条 learned，结构化摘要 |
| `.workbuddy/projects/` | 项目会话 | 交付过程记录 |

---

## 说明

本笔记只收录**可复用的技术结论**，不含：
- 具体服务器 IP、端口、实例号（已在文中替换为占位符）
- 任何 API Key、token
- 个人隐私信息

每条结论都来自真实调试过程，非推测。