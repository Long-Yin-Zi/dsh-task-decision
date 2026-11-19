# dsh-task-decision 交接文档

> 给接手方（运行在 **Docker 里的 DSH**）的单一入口文档。
> 目标：你读完这一份就能装起来、改得动、知道哪些地方碰不得。
> README 面向使用者（功能 / 安装 / 截图），本文面向维护者（架构 / 状态机 / 排障）。

---

## 0. 先读这三行

1. 这是**一个 DSH 客户端插件**：宿主半部 `index.js` + 浏览器半部 `client.js`，用 `dsh plugin add` 安装。
2. 它做的事：让 AI 每轮结束时把「待用户拍板的问题」登记成一张右侧面板，用户选完一次性回传。
3. 最近 5 个版本（0.8.3 → 0.8.8）几乎都在修同一件事：**别让已经答过的旧拍板反复冒出来**。改这块前先读第 6 节。

---

## 1. 项目是什么

| 项 | 值 |
|---|---|
| 包名 | `dsh-task-decision` |
| 版本 | `0.8.8` |
| 仓库 | https://github.com/Long-Yin-Zi/dsh-task-decision |
| 许可证 | MIT |
| 插件市场 | PR #11976（CI 已通过，等待维护者合并） |
| 宿主依赖 | DSH 客户端插件机制：`dsh.bundle` manifest + `dsh.client`（platform: web） |

---

## 2. 当前快照（接手时请核对）

```
package.json  version = 0.8.8
package.json  dsh.bundle.patch = ./cordis.patch.yml   ← 市场 gate 硬性校验项
package.json  dsh.client.inject = [ui-conversation, ui-sidebar-right, ui-session, dsh-api-remotes]
client.js     约 1990 行（含内联 zh/en 字典与 CSS）
index.js      113 行（GATE_RULES + gate_ask 工具）
locale/*.json 只有 meta（插件市场展示用），界面文案在 client.js 的 zh/en 字典里
```

市场条目文件（在 awesome-dsh-plugin 仓库里）：
`data/plugins/Long-Yin-Zi__dsh-task-decision.yml`，分类 `workflow`。

---

## 3. 目录与文件职责

| 路径 | 职责 | 改动生效方式 |
|---|---|---|
| `index.js` | 宿主半部：注册 `gate_ask` 工具、注入系统提示词 `GATE_RULES` | **重启 DSH 进程** |
| `client.js` | 浏览器半部：解析、探针、右侧 tab、面板、提交、语音、附件 | **重新安装 + 浏览器刷新** |
| `cordis.patch.yml` | 把插件插入 DSH 插件表（`id: dsh-task-decision`） | 改动后需重启 |
| `package.json` | 包名 / 版本 / `dsh.bundle` / `dsh.client` | 改动后需重装 |
| `locale/zh.json`、`en.json` | **仅**插件市场卡片的标题与描述 | 重装 + 刷新 |
| `screenshots.json` | 市场详情页截图清单，指向 `docs/screenshot-panel.png` | 推仓库即生效（市场构建时读取） |
| `docs/screenshot-panel.png` | 面板截图（0.8.3 界面） | — |
| `design/ui-spec.md` | 面板设计规格（历史设计文档，参考用） | — |

---

## 4. 运行原理（一张图）

```
AI 回复结束前调用工具 gate_ask({summary, questions[]})
        ↓  (宿主半部只做登记，无副作用)
聊天流里出现一个 tool 节点（call.name = 'gate_ask'，参数是 JSON）
        ↓
client.js 的 GateProbe（挂在会话页头，零尺寸）解析「本轮」节点
        ↓
setGate() 写入模块级 store（Map: sessionId → row）
        ↓
自动打开右侧栏 tab「本轮工作 & 待拍板」，GatePanel 渲染
        ↓
用户选选项 / 写整题备注 / 加附件 → 「提交拍板」
        ↓
拼【拍板回传】文本 → sessions.send → 面板收起 → AI 接着干
```

---

## 5. 宿主半部 `index.js`

- `GATE_RULES`：一段注入系统提示词的中文规则。结构：
  1. **硬规则**：什么时候必须调用 `gate_ask`（有做法分歧 / 要动不可逆的东西 / 需补信息 / 需方向选择 / 要写"要不要…"）。
  2. 参数规范：`summary` 3~6 句；`questions[]` 每题 `id/title/prompt/type/options[]`；描述必须脱离对话可读。
  3. 回传格式识别（`q1（标题）: 选 b「…」；整题备注：…`）。
  4. **面板末尾固定附加的「继续深挖，还是直接开干」题**——模型不登记它，只负责识别回传里的【是否继续思考】行。
- `gate_ask` 工具 schema：`{ summary: string, questions: Question[] }`，`execute` 只返回 `{ ok, questions }`，**不写任何副作用**。
- `inject = ['tools', 'systemPrompt']`。

> ⚠️ 继续思考题的**语义在两半之间是契约**：`index.js` 的规则说"面板会自动附加这题"，`client.js` 会用 `continueChoice` 做必答校验并追加 `continueLine()`。两边必须同时改。

---

## 6. 浏览器半部 `client.js`（核心，重点读）

### 6.1 store 的数据结构

模块级 `gates: Map<sessionId, row>`，row 字段：

| 字段 | 含义 |
|---|---|
| `gate` / `key` | 本轮解析出的对象 / `JSON.stringify(gate)` 作为内容指纹 |
| `submittedKey` / `submittedSnapshot` | 已提交的轮次 key 与回放快照（只读回放用） |
| `cancelledKey` | 用户点过「取消本轮」的 key |
| `touchedKey` | 用户**动过**（选过选项/写过备注/选过继续题）的 key |
| `incoming` | 排队中的新一轮（旧轮未处理完时不覆盖，顶部显示「新一轮已到达」） |

`setGate()` 返回值是**三态字符串**，别当布尔用：

| 返回 | 含义 | 后续动作 |
|---|---|---|
| `'same'` | 内容相同 / 已处理过，未写入 | 什么都不做 |
| `'updated'` | 已替换为主 gate | 可能需要自动打开面板 |
| `'queued'` | 旧轮未完成，新一轮进 `incoming` | 顶部提示，等旧轮处理完自动切换 |

### 6.2 三套持久化标记（最容易踩坑，改动必读）

都存在 `localStorage`，按 sessionId 分键，各保留最近 30 个 key：

| 前缀 | 写入时机 | 用途 |
|---|---|---|
| `dsh-task-decision:handled:<sid>` | 提交 / 取消 | 已处理的轮次**不再写回 store、不再弹** |
| `dsh-task-decision:opened:<sid>` | 自动打开面板时 | 弹过的轮次不重复弹（切会话、刷新都不弹） |
| 内存 `registeredKeys` | 探针每次成功解析 | **每轮只登记一次**，防探针反复解析把旧轮写回 |

**为什么需要第三套**：探针会因为聊天节点变化重复解析同一条 `gate_ask`；store 已被新一轮替换后，旧轮再被写回就会被判为"未完成"→ 触发排队 → 顶部提示「新一轮已到达」而面板里是旧题。0.8.8 就是修这个。

### 6.3 探针 `GateProbe`（约 517 行）

- 依赖宿主注入的 `useChat`（读 `legacy.nodes`）与 `useSessionStatus`（读 `running`）。
- `lastTurnNodes()`：**只取最后一条用户消息之后的节点**——本轮 AI 没调用工具时，不会回退到上一轮的调用。
- 解析失败会重试 3 次（600ms 间隔），因为工具节点可能比 `running=false` 晚一拍落盘。
- 自动打开判据：`changed !== 'same'` + 有题 + 未 handled + 未 opened → `markOpened` + `onGate()`。
- `GateToolView`（聊天里的工具卡片）**只写 store，不自动打开面板**。
- 页头「拍板」按钮（`GateLauncher`）常驻，有待办时带角标。

### 6.4 面板 `GatePanel`（约 736 行）

- 所有状态用 React hooks；**全部 hooks 必须写在任何 early return 之前**（0.7.9 就是因为语音 hooks 放在 `if (!gate) return` 后面，触发 React #300「Rendered fewer hooks than expected」，表现为面板整片空白）。
- 外层包 `GatePanelBoundary`（错误边界）：渲染抛错时显示错误文本 + 版本号，而不是空白。
- 顶层结构：`wg-head`（标题/计数/状态）→ 可选 `wg-incoming`（新一轮提示条）→ `wg-scroll`（摘要 + 题目 + 继续题）→ `wg-foot`（提交/取消）。
- 0 题时只显示「本轮无待拍板」信息卡，**不渲染继续题和提交栏**。

### 6.5 提交与回传

- `buildReturn()` 生成纯文本回传；`buildPromptContent()` 生成带附件的内容块数组（`text` + `image` + `file`）。
- 发送：`sendToSession(ctx.get('sessions'), sessionId, content)`。
- 防重复提交两道闸：同步锁 `sendLock` + `submittedRow` 已提交判断。
- 提交后 `markSubmitted()` → 写 `handled` + 若有 `incoming` 自动切换 → 收起侧栏。

### 6.6 语音（整题备注旁的麦克风）

链路：`MediaRecorder` 录音 → `decodeAudioData` → `OfflineAudioContext` 重采样到 **16kHz 单声道** → `encodeWav()` → base64 → `ctx.get('remote.speech').transcribe({ audioBase64 })` → 追加到备注。

- 取值必须用 `ctx.get('remote.speech')`（可选查找），**直接写 `ctx.remote.speech` 会被 Cordis 的 `fiber.inject` 拦下**（报 `cannot get property 'remote.speech' without inject`）。
- 录音中：按钮变红色停止方块 + 秒数；转写中：方形外框 + 内部小环自转（`.wg-mic-spin`）+「转写中…」。
- 依赖宿主启用语音转写服务；没有时提示「当前 DSH 未启用语音转写服务」。浏览器需要 https 或 localhost 才能拿到麦克风权限。

### 6.7 附件

- 点「添加附件」或备注框里 `Ctrl+V` 粘贴；拖拽不可用（DSH 在 `document` 冒泡阶段接管了 `dragover/drop`，插件收不到）。
- 上传走宿主 `ctx.get('fileUpload').upload(sessionId, blob, name, signal, onProgress)` → `receiptId`。
- 图片随消息内联，普通文件存宿主附件库。

---

## 7. 开发与调试

### 7.1 两种改动，生效方式不同

| 改的是 | 怎么生效 |
|---|---|
| `client.js`（面板、探针、语音、CSS） | 重新安装插件 → 浏览器 `Ctrl+Shift+R` |
| `index.js`（`GATE_RULES`、工具 schema） | **重启 DSH 进程**（系统提示词在启动时注入） |
| `package.json` 的 `dsh.*` | 重装（宿主会重新生成客户端 boot manifest） |

### 7.2 安装（Docker 里）

```bash
# 从 GitHub 源码装（容器需能访问 github.com）
dsh plugin --profile <profile-name> add github:Long-Yin-Zi/dsh-task-decision

# GitHub 不通时：codeload 直链（不走 git 探测）
dsh plugin --profile <profile-name> add https://codeload.github.com/Long-Yin-Zi/dsh-task-decision/tar.gz/refs/heads/main

# 开发调试：直接装本地目录（改完重装即可）
dsh plugin --profile <profile-name> add /path/to/task-decision
```

装完确认：`~/.dsh/profiles/<name>/package.json` 的 `dependencies` 里有 `dsh-task-decision`。

### 7.3 冒烟测试

`.tmp-verify/smoke.mjs` **不在版本库里**（`.tmp-verify/` 被 `.gitignore` 忽略）。它做的事：在 Node 里用桩 React + 桩 `window.__ModuleLoader__` 加载 `client.js`，渲染面板并断言 70+ 条（渲染、布局、提交回传文本、排队、0 题、handled/opened 记录等）。

接手后如果还想跑它，两条路：

1. 从原开发机拷一份 `.tmp-verify/smoke.mjs`；
2. 或按上面断言清单自己写——`client.js` 末尾导出了调试对象 `window.__GATE_DEBUG__`（含 `normalizeGate / setGate / getGate / markSubmitted / cancelGate / isHandled / isOpened / lastTurnNodes / markTouched / gates`），在浏览器控制台或 Node 桩里都能直接调。

### 7.4 调试开关

浏览器控制台可用：

```js
// 看某个会话的当前 row 与三套标记
__GATE_DEBUG__.gates.get('<sessionId>')
JSON.parse(localStorage.getItem('dsh-task-decision:handled:<sessionId>') || '[]')
JSON.parse(localStorage.getItem('dsh-task-decision:opened:<sessionId>') || '[]')
```

---

## 8. Docker 部署注意事项

| 事项 | 说明 |
|---|---|
| 客户端插件加载 | 插件由浏览器端 `window.__DSH_BOOT__` manifest 加载，**与容器无关**；只要 `dsh web` 正常生成 manifest 即可 |
| 已验证环境 | 仅 **0.2.1-rc 系列桌面端**验证过；容器化 `dsh web` 属未验证场景，装完后先做第 9 节的验收 |
| 宿主版本 | `dsh.client.inject` 里列了 `@deepseek-ai/dsh-api-remotes` 等包，DSH 版本过旧（没有这些客户端包）会导致插件加载失败 |
| 语音 | 需要宿主启用语音转写插件（`speech-to-text` 一类）；容器里通常没有麦克风设备，**录音发生在浏览器端**，容器只负责接收 base64 转写 |
| 浏览器安全上下文 | 麦克风要求 `https://` 或 `localhost`；用 IP 直连 http 时语音会失败 |
| 附件存储 | 走宿主 `fileUpload`，文件落在容器内 harness 目录（注意容器重启后的持久化） |
| GitHub 访问 | 容器内 `dsh plugin add github:...` 需要出网；不通时用 codeload 直链或把仓库拷进容器用本地目录装 |

---

## 9. 排障手册（现象 → 先查什么）

| 现象 | 先查 | 常见原因 |
|---|---|---|
| 右侧栏没有「本轮工作 & 待拍板」 | 插件是否真的加载 | 没重装 / profile 里没这个依赖 / DSH 没重启 |
| 面板整片空白 | 是否能看到版本号或错误文本 | 0.7.9 之前的 hooks 顺序问题（#300）；CSS 高度为 0 |
| 有拍板但不自动弹 | localStorage `opened` 是否有该 key | 这轮弹过（设计如此）；或 AI 本轮没调用 `gate_ask` |
| 答完新拍板又冒出旧拍板 | `handled` / `registered` 是否生效 | 低于 0.8.8；或刷新后内存 `registeredKeys` 重置（属正常，旧轮会被 `handled` 拦住） |
| 顶部提示「新一轮已到达」但内容是旧题 | `row.incoming` 与 `row.gate` | 0.8.8 之前的重复写回；升级后应消失 |
| 语音报「未启用语音转写服务」 | 宿主是否装语音插件 | 容器版 DSH 没启用 speech 服务 |
| 语音报 `without inject` | 代码里是否用了 `ctx.remote` | 必须是 `ctx.get('remote.speech')` |
| 提交后 AI 没反应 | `sessions.send` 是否成功 | 会话 id 变化 / 消息构造异常（`buildPromptContent`） |
| 继续题没选不能提交 | — | **设计如此**（必答），不是 bug |

---

## 10. 红线（不要碰）

1. **不改包名 `dsh-task-decision`、不改仓库名**：市场条目里的 `url` 和安装命令都指向它。
2. **不改 `cordis.patch.yml` 里的 `id: dsh-task-decision`**：改了等于换插件身份。
3. **不删 `package.json` 的 `dsh.bundle`**：市场 gate 会直接拒（只声明 `dsh.client` 不可安装）。
4. **`client.js` 里所有 hooks 必须写在任何 early return 之前**（React #300 的坑，0.7.9 踩过）。
5. **`index.js` 与 `client.js` 的「继续思考题」契约要同步改**（一处加、另一处必须跟着）。
6. **改界面文案要同时改 zh/en 两份字典**，并注意有些文案被冒烟断言按字面校验。
7. 本项目只做"登记 + 回传"，**不要在这里加网络请求、读写用户文件、外发数据**（README 的权限承诺）。

---

## 11. 待办（接手后按需推进）

- [ ] 市场 PR #11976 等维护者合并；合并后确认 `workflow` 分类里出现条目。
- [ ] 0.8.8 的「旧拍板不再写回」待实际使用验证；若复现，取 `handled` / `opened` 的 localStorage 内容定位。
- [ ] AI 主动调用率：`GATE_RULES` 已加硬规则，仍可能漏；可考虑客户端兜底（检测正文里有问句但没调工具时提示）。
- [ ] 可选：发布 npm 包（市场可显示下载量）；可选：给仓库准备版本无关的 tarball。
- [ ] 历史遗留：本地目录名仍叫 `task-decision`（与包名不一致，不影响功能）。

---

## 12. 版本历史（关键节点）

| 版本 | 内容 |
|---|---|
| 0.5.0 | 已提交只读回放、清理死样式、补齐中英文字典 |
| 0.6.0 | 每题附件（粘贴 + 按钮） |
| 0.6.2 | 「取消本轮」按钮 + 滚动优化 |
| 0.7.0 | 改名 `dsh-task-decision`；每轮末尾固定「继续思考」题 |
| 0.7.1–0.7.2 | 整题备注语音输入；按钮图标化 |
| 0.7.3–0.7.5 | 修 remote 注入（`ctx.get('remote.speech')`） |
| 0.7.6 | 市场卡片标题/描述改英文 |
| 0.7.7 | 面板空白可诊断（错误边界 + 版本号 + 最小高度） |
| 0.7.9 | 修 React #300（语音 hooks 顺序） |
| 0.8.0 | 语音停止后立即进入转写态 |
| 0.8.1 | 麦克风并入「添加附件」同一行 |
| 0.8.2 | 0 题不显示继续题/提交栏 |
| 0.8.3 | 未提交时新一轮排队（顶部提示 + 手动切换） |
| 0.8.4 | 0 题不排队；转写中图标改方形外框 |
| 0.8.5 | localStorage 记 handled；切会话/刷新不重复弹 |
| 0.8.6 | 自动打开改按 key 持久化（修"该弹不弹"） |
| 0.8.7 | 只解析本轮节点；选过即视为处理过；提示词加硬规则 |
| 0.8.8 | 每轮只登记一次 + handled 短路（修旧拍板写回） |

---

## 13. 验收清单（装到 Docker 后按顺序过）

1. 右侧栏按钮「拍板」在；发一条会引出拍板的消息，面板能自动弹出。
2. 选一个选项 + 写一句备注 → 提交 → 会话里出现【拍板回传】，面板收起。
3. 同一会话再跑一轮 → 只显示新一轮，不出现上一轮。
4. 切到别的会话再切回来 → 面板不自动弹；点页头「拍板」能看到已提交回放。
5. 点「取消本轮」→ 显示已取消；再跑一轮 → 不重复提示旧的。
6. （有语音服务时）点麦克风说一句 → 停止后出现「转写中…」→ 文字进备注。
7. 看不到版本号 / 面板空白 → 按第 9 节排障。
