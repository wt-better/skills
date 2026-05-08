---
name: math-video
description: |
  数学教学视频制作技能，将数学题目转化为带配音的 Manim 动画视频。
  核心工作流：数学分析 → HTML可视化 → 分镜脚本 → TTS音频 → 验证更新 → 脚手架 → Manim代码 → 渲染验证。
  触发条件：用户粘贴数学题图片/文本、需要教学视频、需要数学讲解动画、需要 Manim 教学视频。
description_zh: |
  数学教学视频制作技能，将数学题目转化为带配音的 Manim 动画视频。
---

# 数学视频制作技能

将数学题目转化为带配音的 Manim 动画教学视频。采用"音频驱动画面"的方式确保音画同步。

## 初始化项目

首次使用时，在项目目录运行初始化：

```bash
python {SKILL_DIR}/scripts/init.py [项目目录]
```

这将创建目录结构、拷贝模板和示例文件。

### 依赖安装

```bash
uv venv .venv && source .venv/bin/activate
uv pip install -r {SKILL_DIR}/requirements.txt
```

必要依赖：`manim`, `edge-tts`, `mutagen`, `numpy`, `pillow`

---

## 核心工作流（8步流水线）

```
step1 → step2 → step3 → step4 → step5 → step6 → step7 → step8
                                                    ↑________|  (失败则回到step7)
```

### Step 1: 数学分析 (analyze_problem)

**输入**：题目图片/文本
**输出**：`math_analysis.md`

任务：推导数学事实、建立几何模型、确定画图方法。对于证明题，待证结论可作为已知事实用于分镜。

**关键原则**：绝对不用坐标系解题，必须用定义和几何推理（面积变换、相似、勾股、向量叉积、旋转对称等）。坐标仅用于动画可视化。

### Step 2: HTML可视化 (html_visualization)

**输入**：math_analysis.md
**输出**：`数学_{日期}_{题目简述}.html`

用 HTML+SVG 画图形，展示画图过程和解题流程，标注关键元素。SVG 必须体现绘制顺序（先画什么后画什么）。

### Step 3: 分镜脚本 (storyboard)

**输入**：HTML内容
**输出**：`{日期}_{题目}_分镜.md`

定义视频结构，幕数不限。每幕包含：

| 字段 | 说明 |
|------|------|
| 画面 | 视觉描述 |
| 字幕 | ≤20字，配合画面 |
| 读白 | 详细、口语化、引导思考 |
| 动画 | 带时间戳，用 `→` 标记退场 |
| 目的 | 这一幕要达成什么 |

末尾附音频生成清单表格（时长列留空）：

```markdown
| 幕号 | 文件名 | 读白文本 | 时长 | 说话人 | 情感 |
|------|--------|----------|------|--------|------|
| 1 | audio_001_开场.wav | "大家好..." |  | xiaoxiao | 热情 |
```

**字幕退场约定**：用 `→` 标记退场时机，或 `退场:`/`淡出:` 显式标记，或 `持续X秒`。

详细分镜示例见 [references/storyboard_sample.md](references/storyboard_sample.md)

### Step 4: 生成TTS音频 (generate_tts)

```bash
python {SKILL_DIR}/scripts/generate_tts.py 分镜.md ./audio --voice xiaoxiao
```

**输出**：`audio/audio_{三位幕号}_{幕名}.wav` + `audio/audio_info.json`

可用声音：xiaoxiao(女/默认)、xiaoyi(女)、yunyang(男)、yunjian(男)

### Step 5: 验证音频 (validate_audio)

```bash
python {SKILL_DIR}/scripts/validate_audio.py 分镜.md ./audio
```

验证音频存在性、时长>0、数量匹配，然后回填时长到分镜脚本，更新 `audio_info.json`。

### Step 6: 生成脚手架 (scaffold)

基于模板 [templates/script_scaffold.py](templates/script_scaffold.py) 生成 `script.py` 伪代码框架。

**必须包含的结构**：

```python
COLORS = {
    'background': '#1a1a2e',   # 深蓝背景
    'primary': '#4ecca3',      # 青色
    'secondary': '#e94560',    # 红色
    'highlight': '#ffc107',    # 黄色
    'text': '#ffffff',         # 白色
}

SCENES = [
    # (幕号, 幕名, 音频文件名, 时长秒数)
    (1, "开场", "audio_001_开场.wav", None),
]
```

**必须实现的函数**：
- `calculate_geometry()` - 计算所有几何元素，始终使用2D坐标(z=0)
- `assert_geometry()` - 验证几何正确性 + 画布范围检查
- `define_elements()` - 定义Manim图形对象（不创建动画）
- `construct()` - 主流程
- `play_scene_X()` - 每幕动画

**assert_geometry 验证内容**：
1. 几何条件验证：基于题目条件（等边、中点、直角等），相对误差1e-6或绝对误差1e-4，assert消息用中文带具体数值
2. 画布范围验证：计算所有元素包围盒 → 验证在画布内(FRAME_WIDTH=14.2, FRAME_HEIGHT=8, 边距0.5-1.0) → 验证中心在视觉中心区(x∈[-1.4,1.4], y∈[-0.8,0.8])

### Step 7: 实现代码 (implement)

根据分镜+audio_info.json填充脚手架。关键规则：

1. **音频必须**：每幕第一行 `self.add_sound(f"audio/{audio_file}")`
2. **时长匹配**：动画时长 ≥ 音频时长，不够用 `self.wait()` 补
3. **高亮对应**：配音提到什么，画面就高亮什么
4. **字幕退场**：用 `show_subtitle_timed()` 或 `show_subtitle_with_audio()`，幕结束前所有文字必须退场

**绕过LaTeX依赖**：全部用 `Text()` 代替 `MathTex()`，使用Unicode替代：
- x² → x² (Unicode上标)
- 分数 → Unicode分数
- 度 → °
- 角 → ∠
- 根号 → √

### Step 8: 检查与渲染 (check_and_render)

```bash
# 推荐：完整流水线
python {SKILL_DIR}/scripts/render.py

# 或分步执行
python {SKILL_DIR}/scripts/check.py          # 代码结构检查
manim -pqh script.py MathScene               # 渲染
```

渲染质量：`-q h`(1080p60默认), `-q k`(4K), `-q m`(720p30)

**渲染后验证**：几何正确、高亮同步、字幕清晰、动画流畅、每幕有音频。失败则回到Step 7修复。

---

## 核心原则速查

| 原则 | 说明 |
|------|------|
| 数学先行 | 先建立正确数学模型再画图 |
| 音频必须 | 每幕必须调用 `self.add_sound()` |
| 音画同步 | 动画时长 ≥ 音频时长 |
| 高亮对应 | 配音提到的元素必须高亮 |
| 最小验证 | assert_geometry 验证题目条件+画布范围 |
| 渐进抽象 | HTML可视化 → 分镜脚本 → Manim代码 |
| 无LaTeX | 用 Text() + Unicode，不用 MathTex() |
| 字幕退场 | 幕结束前所有文字必须退场，避免残留 |

---

## 文件结构参考

```
项目目录/
├── math_analysis.md         # Step 1 输出
├── 数学_日期_题目.html      # Step 2 输出
├── 日期_题目_分镜.md        # Step 3 输出
├── audio_list.csv           # Step 4 输入
├── script.py                # Step 6-7 输出
├── audio/
│   ├── audio_001_开场.wav
│   ├── audio_002_xxx.wav
│   └── audio_info.json
└── media/                   # Manim 渲染输出
```

## 其他资源

- 脚手架模板：[templates/script_scaffold.py](templates/script_scaffold.py)
- 完整示例：[templates/script_example.py](templates/script_example.py)
- 分镜示例：[references/storyboard_sample.md](references/storyboard_sample.md)