# 踩坑记录（PITFALLS）

> 每次踩坑先查这里。新增一条的格式：**现象 → 根因 → 正确做法（在哪）**。
> 标注 ✅ 表示已验证修复，⚠️ 表示已知但未根治。

## 环境

| # | 现象 | 根因 | 做法 |
|---|---|---|---|
| E1 | 所有 shell 命令 fork 失败（errno 11 / 0xC0000142） | 本机 Git Bash 不可用 | 全部走 PowerShell；输出中文先 `[Console]::OutputEncoding=[Text.Encoding]::UTF8` |
| E2 | `python -m http.server` 起的服务约 2 分钟后消失 | 后台任务被回收 | 每次要 reload 前重启；长任务分批验证 |
| E3 | HuggingFace 直连超时 | 外网被墙 | `hf-mirror.com` 与 `modelscope.cn` 均可用；下载器已做 HF → hf-mirror → 魔搭 自动换源 ✅ |
| E4 | `Start-Process` 报「已添加了具有相同键的项」，`Get-ChildItem Env:` 同样报错；headless Chrome 启动后卡死无输出 | 环境里同时存在大小写不同的同名代理变量（如 `HTTPS_PROXY` 与 `https_proxy`），.NET 按 Key 建字典时冲突 | 会话内先 `foreach($n in 'http_proxy','https_proxy','all_proxy','no_proxy'){ [Environment]::SetEnvironmentVariable($n,$null,'Process') }` 清掉小写重复；浏览器仍挂起时改用 `new Function` 语法检查 + 截图回退验证 |
| E5 | agent-browser 每次 CLI 调用都是**全新的无头浏览器**：页面变 about:blank、IndexedDB/localStorage 不跨调用保留、`reload` 会丢 URL | CLI 无驻留会话 | 「打开 → 操作 → 断言 → 截图」必须链在**同一条 PowerShell 命令**里（`open; sleep; eval; sleep; eval`）；跨命令的页面状态别指望 |

## 单文件架构

| # | 现象 | 根因 | 做法 |
|---|---|---|---|
| A1 | 改一处不知道会不会影响别处 | 24k 行、16 个 script 块、同名函数靠加载顺序覆盖 | 低风险改法：同一 IIFE 内、`init();` 之前插新的同名函数声明；DOM 用 `setTimeout(cvInstall,0)` 在全部脚本后克隆节点丢掉旧监听 |
| A2 | 分享包模板被改坏 | `SHARE_TMPL` 里有同名 `$`/`addMsg`/`chatLog`/`sendBtn` | 所有编辑限定在模板字符串之外 |
| A3 | 改了文档里的行号全对不上 | 文档是旧版本写的 | 以 `grep` 定位为准，别信历史行号 |
| A4 | 同一件事三套实现：模型库面板 / 模型抽屉 / 权重页各有列表与「选用」，激活模型的代码抄了 3 份 | 历史迭代叠加 | 已收进「模型中心」：列表唯一（`ModelCenter.refresh`），选用唯一（`T.adoptRecord`），对话源唯一（`T.selectChatSource`，写回 `#chatWeightSelect`）。新增模型入口直接调这三个，不要再抄 ✅ |
| A5 | 模型中心首屏渲染时 `TLX_MODELS` 还没加载，开源权重显示 0 | 脚本加载顺序 | `listOfflineModels()` 对未就绪返回 `[]`；`TLX_MODELS.boot()` 里延迟 600ms 补一次 `ModelCenter.refresh()` ✅ |
| A6 | 空状态对「有没有模型」判断错误（有模型却显示上手路径） | `cvEmpty` 跑在 `loadChatSettings` 的异步续段里，而 `refreshChatSelect`（cvInstall）更晚——`chatWeightSelect` 尚未填充，`chatSrc()` 是空串 | `cvHasChatSource()` 把空串按默认源 `__current__` 处理（`chatSrc() || '__current__'`）✅ |

## 推理引擎

| # | 现象 | 根因 | 做法 |
|---|---|---|---|
| I1 | 本机训练模型一对话就崩（`createBuffer of null` / `subarray is not a function`） | `TLX_BRIDGE.wrapForInfer` 按 CPU 语义实现（直接读 `logits.data.subarray`），而 `pickOps()` 有 GPU 时返回 WebGPU 算子，`.data` 是 GPUBuffer | 本机通道**强制用 `__SPARE__.cpuOps`** ✅ |
| I2 | 多轮对话越聊越乱 | `chatContextParts` 已拼全量 prompt，引擎会话却没清空，历史被重复喂入 | 每轮 `session.reset()`，上下文只由调用方拼一次 ✅ |
| I3 | 训完再对话还是旧权重 | 会话按 `_ver` 缓存，权重更新后没变 key | 缓存键必须带 `_ver`；本机先 `Backend.ensureSynced()` ✅ |

## GPU 训练 / 微调

| # | 现象 | 根因 | 做法 |
|---|---|---|---|
| T1 | loss 一直不变 | `P[name] = W + B·A` 只建一次图节点，A/B 更新后 ΔW 仍是旧值 | 每步 forward 前 `rebind()` 重算 ΔW（动态图）✅ |
| T2 | `WeakMap` 报错，信息完全不指向真因 | `adamwStep` 收到 `grad === null` 的参数（该层这轮没参与前向） | 只把 `grad` 非空的参数喂给优化器 ✅ |
| T3 | loss 卡在 `ln(V)` 附近不动 | `createLlmModel(..., {zeroInit:true})` 全零权重 → 梯度全 0 | 测试用随机初始化；真实模型无所谓（权重来自 safetensors） |
| T4 | 加载模型后手动算 CE，损失读回全 0 | 怀疑 GPU 工作区池（arena）把 buffer 回收复用了 | 训练前 `if (workspaceStats().active) workspaceEnd()` ⚠️ 未复测 |
| T5 | 训练结束偶发推理变慢/爆显存 | 忘了关构图 | `try/finally` 里必须 `setGraphBuilding(false)` ✅ |
| T6 | 小模型 SFT「练了没用」 | `sftStep` 按 `maxSeqLen`（默认 48）截断序列，120 token 的回答被砍掉 2/3 | 生成长度按窗口收敛 + 导入数据时提示超窗对数 ✅ |

## 数据与存储

| # | 现象 | 根因 | 做法 |
|---|---|---|---|
| D1 | 刷新后界面空白，但模型仍"记得"上下文 | 历史存了 localStorage，却没有重绘逻辑 | 会话存 IndexedDB，启动即重绘 ✅ |
| D2 | 魔搭下载报「大小不符（期望 1，实际 659）」 | 魔搭不返回 Content-Length，`Range: bytes=0-0` 探测拿到的 `content-length: 1` 被当成文件大小 | `cl > 1` 才认；未知大小记 `size:0` 并跳过校验，下载后用 OPFS 实际字节回写 ✅ |
| D3 | 进度条永远 0% | 同上，总大小未知 | 未知时只报「已下载字节」，不给假百分比 ✅ |
| D4 | `indexedDB.deleteDatabase` 有时不生效 | 还有连接没关（blocked） | 删除前关掉连接；测试完刷新页面再断言 |
| D5 | 删除模型后重新加载还是旧的 | `TLX_MODELS.cache` 内存缓存没清 | `remove()` 里同时 `delete cache[key]` ✅ |

## 验证方法（省时间）

1. 语法：把每个 `<script>` 抽出来 `new Function(code)`，一遍就能抓括号/模板字符串问题。
2. 没有外网也能验全链路：用 Python 生成一个合成 Qwen2 小模型（`config.json` + `tokenizer.json` + `model.safetensors`），配一个支持 Range 的本地服务器，就能跑通「下载 → OPFS → 垫片 → 加载 → 训练」。
3. 判断 GPU 是否真可用：`__ENGINE__.api.gpu.ensure()` 返回 `ok:true` 且 `deviceInfo()` 非空；否则后面全是 null 报错。
4. 测损失别只看 UI 数字，直接 `ops.download(loss.data,1)` 读回真值；两次读数一样就说明图没重建（T1）。
