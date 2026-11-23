# 给 AI 的落地指南：搭建免费「AI 自动剪视频三件套」并出片

你是用户的 AI 编程助手。用户把这份文档发给你，是希望你**全权负责**在他电脑上装好免费剪辑三件套（**本地语音转写 + FFmpeg + HyperFrames**），并带他用一条真实口播视频跑通「自动剪口误 → 加字幕与动效 → 导出成片」的全流程。

> **协作铁律**：安装依赖、写脚本、排错、渲染、验收全由你来做，不要让用户手敲命令。只有在「需要安装系统授权/弹窗点同意」「确认删哪些口误」「挑选动效风格」「确认成片效果」时，才停下来请用户拍板。

---

## 1. 先查清用户电脑

动手装任何东西前，先用终端查清下面 6 项，再向用户汇报一行结论并开始安装：

1. **操作系统与芯片架构**：`uname -sm`（Mac/Linux）或 Windows 架构（x64/arm64）。
   - **重要**：如果是 Intel 芯片 Mac（`Darwin x86_64`），FunASR（SenseVoice）官方不支持，直接只装 Whisper 路线，不要强装 FunASR。
2. **Python 环境**：检查 `python3 --version`（或 Windows `python --version`），需要 **3.9 及以上**。
3. **Node.js 环境**：检查 `node -v` 和 `npx -v`，HyperFrames 需要 **Node.js 20+**。
4. **FFmpeg**：检查 `ffmpeg -version` 与 `ffprobe -version` 是否已装。
5. **中文字体**：查找系统里现成的黑体文件（如思源黑体 / Noto Sans CJK SC / 苹方 / 微软雅黑 `.ttf` 或 `.otf`），记下路径备用（HyperFrames 渲染中文字幕必须把字体文件放进工程目录）。
6. **网络情况**：简单判断访问 Hugging Face / npm 是否通畅；如果国内直连较慢，Python 包走国内镜像源，语音模型优先走 ModelScope（魔搭）或 HF 镜像（`HF_ENDPOINT=https://hf-mirror.com`）。

---

## 2. 核心原理（不能改反）

整套免费剪辑拆成**两段独立任务 + 三个免费工具**：

1. **本地语音转写（Whisper / SenseVoice Small）**：把视频里的声音转成带时间戳的逐字稿，纯本地运行、不花 API 钱。
   - **选型建议**：
     - **精准剪口误 / 逐字高亮字幕（主推）**：用 **Whisper**（Apple Silicon Mac 优先 `mlx-whisper`，Windows/Linux/Intel Mac 用 `faster-whisper`，模型推荐 `large-v3-turbo` 或轻量 `small`/`medium` 视显卡和内存定），支持**字级时间戳（word timestamps）**。
     - **中文极速转写 / 复核错字（可选加装）**：**SenseVoice Small**（通过 `funasr` + `modelscope` 国内直连下载极快，中文识别准），适合快速出整段逐字稿或拿来跟 Whisper 对照纠错；但它默认输出段级时间，逐字卡点仍以 Whisper 为准。
2. **FFmpeg（剪口误 + 规整视频）**：
   - 不要让 AI 凭感觉盲剪，而是先把逐字稿编号展示给用户确认「删哪些口误、重录、超过 0.4 秒的长停顿」，算出**保留时间区间列表**，再由 FFmpeg 一次性剪切拼接，并在切点加极短音频淡入淡出（15~30ms）防爆音。
3. **HyperFrames（加动效 + 字幕 + 合并成片）**：
   - 用 HTML/CSS/GSAP 写动画，由 HyperFrames 逐帧渲染成 MP4。一切画面都是时间的函数，用来给粗剪好的口播视频加**动态字幕、标题弹出、侧边要点卡片、数据图表**，或者直接做纯动画科普片段。

---

## 3. 工作流程

```
原始口播 MP4
   │
   ▼
[1] FFmpeg 抽 16k 单声道 WAV + 规整关键帧（-g 30 防卡帧）
   │
   ▼
[2] 本地转写（Whisper 出字级时间 / 可选 SenseVoice 复核错字）
   │
   ▼
[3] 输出带编号逐字稿 → 标出疑似口误/重复/长停顿 → 【停下来请用户确认】
   │
   ▼
[4] FFmpeg 按确认后的保留区间剪切拼接（加 20ms 音频淡化防爆音）→ 输出粗剪版.mp4
   │
   ▼
[5] HyperFrames 建工程 → 挂载粗剪版.mp4 → 加字幕/弹出大字/侧边卡片 → check & 截帧自查
   │
   ▼
[6] 【展示预览/关键帧请用户确认】→ HyperFrames 渲染导出最终成片.mp4
```

---

## 4. 关键格式约定

### 4.1 转写结果（`transcript.json`）
统一存成简单结构，方便后续剪辑和加字幕复用：
```json
[
  { "id": 1, "start": 0.42, "end": 2.85, "text": "今天教大家怎么用免费工具剪视频", "words": [{"w": "今天", "t0": 0.42, "t1": 0.78}] }
]
```

### 4.2 剪辑保留清单（`keep_segments.json`）
AI 与用户确认删哪些编号后，只把**要保留的区间**写入清单（每段前后各留 0.05~0.08 秒呼吸余量，防止吃字）：
```json
[
  { "start": 0.35, "end": 4.10 },
  { "start": 5.20, "end": 12.80 }
]
```

### 4.3 HyperFrames 工程要点
- 初始化：`npx hyperframes init <工程目录>`
- 根合成 `index.html`：根节点 `data-duration` 等于成片总秒数；原片用 `<video class="clip" data-start="0" data-duration="..." data-has-audio="true">`；字幕、标题卡、侧边栏各写成独立子合成（`compositions/*.html`）或清晰分组。
- 每个合成必须注册暂停的 GSAP 时间线：`window.__timelines["<id>"] = tl`。
- 渲染前必跑检查与抽帧：`npx hyperframes check --json` → `npx hyperframes snapshot --at 1,10,25` → 确认无遮挡错位后再 `npx hyperframes render -o 最终成片.mp4`。

---

## 5. 实现细节与常见踩坑（实测总结）

| 常见坑 | 正确处理方式 |
|---|---|
| **剪完听着「啪啪」爆音** | 拼接每段音频时加 `afade=t=in:st=0:d=0.02` 与尾端 `afade=t=out`（20ms 极短淡入淡出），并用 `concat` 滤镜统一重编码，不要直接 `-c copy` 硬拼。 |
| **剪完开头结尾「吃字」** | 保留区间前移 `0.05s`、后延 `0.08s`；相邻两段间隔小于 `0.25s` 时直接合并成一段，别切得太碎。 |
| **HyperFrames 渲染画面卡帧/音画不同步** | 喂给 HyperFrames 之前，先用 FFmpeg 把粗剪视频转成固定帧率且每秒一个关键帧：`ffmpeg -i in.mp4 -r 30 -g 30 -c:v libx264 -pix_fmt yuv420p -c:a aac out.mp4`。 |
| **HyperFrames 中文字幕变成方块或回退字体** | 渲染器不认系统字体名！必须把 `.ttf`/`.otf` 中文字体文件拷到工程的 `assets/fonts/`，并在 CSS 里用 `@font-face` 指向 `assets/fonts/xxx.otf`。 |
| **HyperFrames 动效闪烁或每次渲染不一样** | 严禁使用 `Math.random()`、`Date.now()`、`setTimeout`、`setInterval` 或 CSS `transition`；全部动画必须写在 GSAP 时间线上，且优先用 `fromTo`。 |
| **Whisper 结尾静音乱编字幕** | Whisper 在空白静音处容易幻觉出「谢谢观看/请点赞订阅」；开启 VAD 过滤（`vad_filter=True`）或在后处理中剔除无声区间的幻觉句。 |
| **国内下载 Whisper 模型慢/超时** | 终端临时设置 `HF_ENDPOINT=https://hf-mirror.com` 后再跑首次下载；如果是 Apple Silicon Mac 用 `mlx-whisper` 或 Windows 用 `faster-whisper` 均生效。 |
| **Intel Mac 装 FunASR / SenseVoice 报错** | Intel Mac（`x86_64`）跳过 SenseVoice，直接使用 `faster-whisper` 即可。 |
| **3~5 分钟视频剪得慢** | 先拿视频前 30 秒跑通全流程并让用户验收样式，满意后再跑全片，避免返工浪费时间。 |

---

## 6. 落地顺序（请严格按此顺序带用户走）

### 第一步：安装三件套（自动完成，遇系统弹窗提醒用户点允许）
1. **FFmpeg**：
   - macOS：`brew install ffmpeg`（若未装 Homebrew 先提醒用户确认安装，或直接下载静态二进制放入 `~/.local/bin`）
   - Windows：`winget install Gyan.FFmpeg`（装完确认新环境变量或直接取其绝对路径）
2. **Node.js 20+ 与 HyperFrames**：
   - 检查 `node -v`，不足 20 则用 `brew install node` / `winget install OpenJS.NodeJS.LTS`。
   - 跑 `npx hyperframes --version` 确认能拉起 CLI（顺带可帮用户安装官方 HyperFrames 技能说明，方便以后写动效更稳）。
3. **本地语音转写（建独立虚拟环境，不污染系统 Python）**：
   - 建议在用户工作目录下建 `.venv` 虚拟环境：
     - Apple Silicon Mac：`pip install mlx-whisper`（首次转写自动下模型，记得配 `HF_ENDPOINT=https://hf-mirror.com` 如需加速）。
     - Windows / Linux / Intel Mac：`pip install faster-whisper`。
   - 可选加装中文复核引擎 **SenseVoice Small**：`pip install funasr modelscope`（仅在非 Intel Mac 且用户希望多一层中文校对时加装）。

### 第二步：小范围跑通验收（必须用用户的一段真实视频）
1. 请用户把一段 **30 秒～3 分钟的口播 MP4** 拖进对话框或告诉你路径（如果用户手头暂时没有，提醒他用手机随手录 15 秒带一两句故意说错的话来测）。
2. 你自动执行：转写 → 标出口误和超过 0.4 秒的停顿 → 发给用户看一眼确认删不删 → FFmpeg 剪出粗剪版。
3. 接着用 HyperFrames 给这段粗剪版加上：
   - 底部清晰中文大字幕（白字黑描边或高亮重点词）；
   - 开头 3 秒一个醒目主题弹出大字；
   - 中间顺手加 1 张侧边要点卡片。
4. 跑 `npx hyperframes check --json` 并截 3 张关键帧图给用户确认，确认没问题后渲染输出 `最终成片.mp4`，并把文件所在路径清楚告诉用户。

### 第三步：沉淀成用户的专属剪辑规则（以后一句话就干活）
- 跑通第一条后，询问用户个人偏好（例如：停顿超过几秒剪、喜欢什么字幕配色/字号、要不要默认加开头大字），把这些偏好写进工作目录的 `剪辑规则.md`（或注册成当前 AI 助手的本地 Skill）。以后用户只要把新视频丢给你说「按我的剪辑规则剪一下」，直接复用这套流程。

---

## 7. 排错与自救清单

- **命令找不到（`ffmpeg`/`node` not found）**：Windows 刚装完 `winget` 需要刷新 PATH 或直接用绝对路径调用；macOS 检查 `/opt/homebrew/bin` 或 `~/.local/bin` 是否在 PATH 中。
- **HyperFrames 渲染报浏览器/Chromium 缺失**：按提示执行其自带的浏览器安装命令，或指定本机已安装的 Chrome 路径。
- **显卡内存不足（OOM）**：`faster-whisper` 切换 `compute_type="int8"` 并改用 `small` 或 `medium` 模型，纯 CPU 也能稳跑。
- **用户想彻底清理/卸载**：只需删除新建的工作目录（含 `.venv`）及 `~/.cache/huggingface` 下的模型缓存即可，不会弄乱系统。
