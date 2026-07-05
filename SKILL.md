---
name: cangjie-skill
description: Distill a book into a coherent set of executable skills OR a hands-on manual (Chinese-scanned-book specialist). Use when the user asks to "拆书" / "蒸馏一本书" / "把 XX 书做成 skill" / "turn a book into skills" — i.e. wants a book's frameworks, principles, and methodologies extracted into atomic, reusable Claude skills OR a Chinese-scanned-book-derived MARKDOWN HANDBOOK(用户最常需要的形态)。 内置 阶段 -1 快速体检 + RapidOCR 中文硬选型 + 产出路径分支决策(skill 手册 MD)。 2026-07 实战复盘已内化到流程中。 NOT for simple summarization, book reviews, or role-playing as the author (that is nuwa-skill's job).
---

# cangjie-skill — 把一本书蒸馏成一组可执行 skills 的元 skill

## 使命

把一本书里沉淀的方法论,拆解成一组**原子化、可被 agent 在真实场景下调用**的 skills,让读者真正用起来。

**边界**:
- ✅ 做: 方法论 / 决策框架 / 清单 / 原则 / 概念体系的蒸馏
- ❌ 不做: 书摘 / 读后感 / 作者人设角色扮演 (后者请用 nuwa-skill)

## 核心方法论: RIA-TV++

一个四阶段 + 并行提取 + 三重验证 + darwin 兼容测试的流水线。详见 `methodology/00-overview.md`。

```
阶段 0: Adler 整书理解     → BOOK_OVERVIEW.md
阶段 1: 5 个 agent 并行提取 → 候选方法论单元池
阶段 1.5: 三重验证筛选       → 通过的单元
阶段 2: RIA++ 构造 skill     → 每个 skill 的 SKILL.md
阶段 3: Zettelkasten 链接    → INDEX.md
阶段 4: 压力测试 (darwin 兼容) → test-prompts.json + 回炉淘汰
```

## 何时调用此 skill

用户说类似:
- "帮我拆《穷查理宝典》"
- "把毛选蒸馏成 skill"
- "distill this book into skills: <path>"
- "我想把这本书的方法论做成可用的 skill"
- "蒸馏这一本 <pdf 路径>"(更常见形式,直接给文件路径)

## 输入要求(**必须从用户处确认**)

在开始前**必须**从用户处确认三件事.三件缺一不可:
1. **书的文本来源**: PDF / EPUB / TXT 文件路径(包含扫描版 PDF).
2. **书名 + 作者 + 出版年**: 用于目录命名和审计.
3. ★**期望的产出形态 (必问)**:
   - **A. 可执行 skill**(默认): 输出 `~/skills/<slug>/SKILL.md` 给 agent 调用
   - **B. 人读 markdown 手册**: 输出 `<同名_实操手册.md>` 中包含 R/I/A1/A2/E/B 结构
   - **C. 两份都要**: 一份 skill,一份 md 手册
   
   ⚠️ 不要自己猜测用户期望 — 2026 年的实战中,用户经常"想做 skill"但实际只需要一份 md 手册.这个决策影响整个流程路径(手册不需要阶段 1 并行 extractor,也不需要阶段 4 压力测试).
4. **是否首次试点**: 如果用户是第一次用 cangjie-skill,建议先拆 1 本验证流程再批量.

## 阶段 -1 : PDF / 源文件体检(强制执行,5 分钟)

用户给 PDF 后,**不要**直接分块读或 OCR.先做体检,判断"这本书是文本本还是扫描本" + "文字密度分布在哪里".

### -1.1 体检清单(顺序执行)

```bash
# A. 基础信息
pdfinfo <path.pdf>                          # 页数 / Producer (DuXiu / SuperStar = 扫描本)
# B. 文本层探测
pdftotext <path.pdf> - | head -200 | wc -l  # 文字量 < 50 行 = 扫描本走 OCR 路径
# C. 目录页 OCR 定位(只 OCR 前 8-15 页目录)
pdftotext -f 1 -l 15 <path.pdf> | grep "目\s*录" -A 30  # 文本本目录
# 或 OCR 目录页: pypdfium2 + RapidOCR / tesseract 目录页
```

### -1.2 决策矩阵

| 体检结果 | 后续路径 |
|---|---|
| 文本本 + 文字密度高 | 直接 Read/Grep/Read 文本 |
| 文本本 + 只有目录页,后面全图 | **只 OCR 目录+标题章节**,跳过图谱区(常见于图谱/画集) |
| ★**扫描本 + 中文 → 首选 RapidOCR**,这是 2026 年 7 月实测最佳链 | pypdfium2(scale=1.8~2.2) + `rapidocr_onnxruntime`(中文 95%+ 识别率,无云依赖) |
| 扫描本 + 英文 | pdftoppm + tesseract(eng) |
| markitdown(来自用户建议) | ⚠️ 默认不含 OCR!需要外接 Azure Doc Intelligence / LLM Vision.无云账号时改用 RapidOCR. |
| 其他 OCR | PaddleOCR 4.x(企业级复杂需求) / tesseract(遗留系统兼容性) |

### -1.3 文字密度采样决策(**最关键一步**)

**不要无差别 OCR 整本**.目录页后,按以下规则抽样 OCR 每章头 2-3 页:

| 章节类型(目录识别) | OCR 价值 | 处理策略 |
|---|---|---|
| 文字密集章节(文说/论文/经验谈) | 90% | **精读 OCR** |
| 表格主导(版别表/数据) | 40% | **OCR 表头+关键行**,跳过图片 |
| 图谱/图鉴(一页一图) | 5% | **直接放弃 OCR**,保留"使用原书索引 pXXX" |
| 序/跋/出版信息 | 100% | OCR 全段 |

**实战案例(2026-07)**:一本 408 页钱币书,207–408 全图谱.若一脚踢下去 OCR 全册需 ~6 小时+多数空输出;用上述抽样法只 OCR p1–61 一页一座 + p209–386 标本密度区 ~2 小时,成功率 95%+.省去 60% 以上 OCR 时间.

**操作**:
```
1. OCR 目录页(5 min) → 列出章节表
2. 用户确认要拆哪几章(5 min)
3. 仅 OCR 这些章节 + 每个 sampled 3 页验证
4. 完成后再进入阶段 0
```

## 输出结构

```
books/<book-slug>/
├── BOOK_OVERVIEW.md           # 阶段 0 产出: 主旨/骨架/术语/批判
├── INDEX.md                   # 阶段 3 产出: skill 总览 + 引用图
├── candidates/                # 阶段 1 产出: 原始候选池 (审计用)
├── rejected/                  # 阶段 1.5 淘汰的单元 + 原因 (审计用)
├── <skill-slug-1>/
│   ├── SKILL.md
│   └── test-prompts.json      # darwin-skill 兼容格式
├── <skill-slug-2>/
│   └── ...
```

## 执行流程 (严格按顺序)

### 阶段 0 — 整书理解

1. 读取用户提供的书本文本。大文件分块阅读。
2. 执行 `methodology/01-stage0-adler.md` 中的 Adler 四步 (结构 / 解释 / 批判 / 应用)。
3. 按 `templates/BOOK_OVERVIEW.md.template` 填充,写入 `books/<slug>/BOOK_OVERVIEW.md`。
4. 把产出展示给用户确认:"骨架我理解对了吗?有没有你希望重点突出的方向?" 得到确认再进入阶段 1。

### 阶段 1 — 5 个 sub-agent 并行提取

**并行** spawn 5 个 Task sub-agents(使用 Agent 工具,一次调用中发起 5 个):

| sub-agent | 读取的 prompt | 产出 |
|---|---|---|
| 框架提取器 | `extractors/framework-extractor.md` | 决策框架 / 思维模型 |
| 原则提取器 | `extractors/principle-extractor.md` | 原则 / 清单 / 规则 |
| 案例提取器 | `extractors/case-extractor.md` | 作者在书中亲自使用过的实例 |
| 反例提取器 | `extractors/counter-example-extractor.md` | 书中警告的失败模式 |
| 术语提取器 | `extractors/glossary-extractor.md` | 关键概念词典 |

每个 sub-agent 独立读书、独立提取、独立输出到 `books/<slug>/candidates/<type>.md`。

### 阶段 1.5 — 三重验证筛选

读取 `methodology/03-stage1.5-triple-verify.md`,对每个候选单元执行:

- **V1 跨域**: 书中至少 2 个独立段落有佐证?
- **V2 预测力**: 能用它回答一个书里没明说的新问题吗?
- **V3 独特性**: 不是任何聪明人都会说的常识吗?

通过的进入阶段 2。不通过的写入 `books/<slug>/rejected/` 并附原因 — 保留审计轨迹,也允许用户事后捞回。

### 阶段 2 — RIA++ 构造 skill

对每个通过的单元,按 `templates/SKILL.md.template` 填充:

- **R (Reading)**: 原文引用 ≤150 字/段
- **I (Interpretation)**: 用自己的话重写方法论骨架 (避免照搬译本)
- **A1 (Past Application)**: 书中作者用过的案例
- **A2 (Future Trigger)** ★: 用户在什么情境下会需要这个 → skill 的 `description` 字段
- **E (Execution)**: 1-2-3 可执行步骤
- **B (Boundary)**: 什么时候不适用 / 来自阶段 0 批判阶段的作者盲点

细则见 `methodology/04-stage2-ria-plus.md`。

### 阶段 3 — Zettelkasten 链接

按 `methodology/05-stage3-zettelkasten.md`:
1. 找出 skill 之间的引用关系 (A 依赖 B / A 对比 B / A 组合 B)
2. 在每个 SKILL.md 末尾补"相关 skills"段
3. 按 `templates/INDEX.md.template` 生成 `INDEX.md` (含引用图 mermaid)

### 阶段 4 — 压力测试 (darwin 兼容)

对每个 skill 按 `methodology/06-stage4-pressure-test.md`:
1. 设计 5–10 条测试 prompt,按 `templates/test-prompts.json.template` 写入 `test-prompts.json`
2. 至少包括 3 类: **应调用** / **不应调用 (诱饵)** / **边界模糊**
3. 优先用独立 sub-agent 盲测每条 prompt,由主流程对照预期统计结果,**未过的回炉重做阶段 2** — 不做"表面修补"
4. 全部通过后通知用户: "已完成,可一键喂给 darwin-skill 自动进化"

## 质量红线 (违反则阻止输出)

1. 每个 skill 必须通过**全部**三重验证
2. 每个 skill 必须有完整的 R / I / A1 / A2 / E / B 六段
3. 原文引用 ≤150 字/段
4. 每个 skill 必须有 `test-prompts.json`,且包含诱饵测试 (不应调用的场景)
5. `description` 字段必须明确 trigger 条件,不能只是"一个关于 X 的 skill"

## 与 nuwa-skill / darwin-skill 的生态定位

- **nuwa-skill**: 蒸馏人 (思维方式 / 表达 DNA)
- **cangjie-skill** (本 skill): 蒸馏书 (方法论 / 框架 / 原则)
- **darwin-skill**: 进化任意 skill

三者咬合: 本 skill 输出的 `test-prompts.json` 严格遵循 darwin-skill 格式,以便产出的 skill 可直接接入 darwin 做自动进化。

## 产出路径分支决策

根据"输入要求 3"用户选的产出形态,流程分为两条路径:

### 路径 A: 可执行 skill(默认)

严格按阶段 0→1→1.5→2→3→4 执行不变.

### 路径 B: markdown 手册(用户写作"我只要 md" / "人读的")

简化流程,跳过阶段 1 并行 extractor / 阶段 3 链接 / 阶段 4 压力测试(这些是给 agent 用的).只保留:

- **仍然要做**:阶段 0(Adler 骨架) + 候选单元(单人脑内做,不 spawn 5 个 agent) + 三重验证 + RIA++ 六段.
- **产出结构**:
  ```
  <book-slug>/
  ├── BOOK_OVERVIEW.md        # 骨架
  ├── <book>_实操手册.md      # 主产出(板块 I / II / III ...)
  └── src/                    # 章节 OCR 底稿(可选,审计用)
  ```
- **关键调整**: 原文引用页码用 pXXX 标注即可,不需要精确到 test-prompts; 不需要 darwin 兼容.
- **实战案例**:2026 年 7 月《钱币收藏研究文图集》蒸馏就是手册模式,产出 5 板块通用 + 1 技术半两断代手册.

### 路径 C: 两份都要

先跑路径 A(skill)→ 再追加路径 B(md 手册).顺序不能反.

## 文件操作纪律(2026 实战复盘)

### 改文件前

```bash
# 大改前必须:
git add -A && git commit -m "checkpoint before <改动描述>"
# 或对项目 node :
cp <file> <file>.bak.$(date +%y%m%d%H%M)
```

### 改文件后

```bash
# Grep 验标题不缺号:
grep -nE "^(##|###) " <output.md>
# 验字数没丢区块:
wc -l <output.md>   # 跟心理预期差距不能 >20%
```

### Write 调用注意

- **路径用 ASCII 文件名**:`D:\books\<slug>_dating.md` 稳;`D:\要读书\<中文>.md` 可能解析出错.
- **每次 Write 后立刻 ls 校验**:不要盲信调用成功.
- **写入大文件用 cat join**:分 p1/p2/p3 写小文件,最后 `cat p1 p2 p3 > final.md`,避免单 Write 内容过大被截断.
- **heredoc 中中文引号**,改用单引号包裹 python 脚本:`cat > /tmp/x.py <<'PYEOF'` (引号保护 heredoc).

## 调用惯例(2026 修订版)

### 核心规则

- ★**永远先试点 1 本**,除非用户明确说"批量".
- ★**阶段 -1 PDF 体检必做**.5 分钟体检能省去后续 60%+ 的 OCR 浪费.
- ★**阶段 0 后主动问用户 B 路径(skill 还是 md)**.不要默认走技能路径.
- ★**静默改为有声**:阶段间 + 每章节完成时主动汇报进度(含字数/候选单元数).
- **不凭记忆拆书**,没文本就停下来问.
- **保留审计轨迹**:candidates/ 和 rejected/ 都要留.

### 实战复盘(2026-07 钱币书蒸馏的教训,需内化)

1. **OCR 工具探索走了回头路,因为没先做 PDF 体检**.先 `pdfinfo` + `pdftotext` 5 秒判断类型,而不是探索 6 个 PATH 后才发现"扫描本该走 OCR".
2. **OCR 整本浪费**:一本 408 页书 OCR 了 200+ 页图谱章节才发现"图谱 OCR 价值只有 5%".正确做法是**先 OCR 目录页 → 挑文字密集章节 OCR**,丢弃图谱区.
3. **板块二整章丢失**:一次大改把 `## 二、收藏决策树 + 正文 90 行` 吞掉,2 小时核查才发现.根因:**没用 git** → **grep 验标题没做** → **用户早发现**.
4. **markitdown 评估过于武断**:README 说它"支持 OCR",读者容易误用为 markitdown 自带 OCR.实际上它需要 Azure / LLM Vision 后端,**中文批量 OCR 应该首选 RapidOCR**.评测第三方工具前先答"它解决了啥 / 依赖啥 / 我环境缺啥".
5. **Write 路径带中文三次失败**:中文文件名被解析歪曲.稳用 ASCII 路径.

### 时间预算参考(文本本 300 字 PDF)

| 阶段 | 耗时 |
|---|---|
| 体检 + 输出决策(必做) | 5-10 min |
| 章节抽样 OCR | 20-40 min |
| 阶段 0 Adler 骨架 | 15-20 min |
| 候选+验证(手册模式人工做 / skill 模式 agent spawn) | 10 min / 20-30 min |
| RIA++ 手册 | 40-60 min |
| **总计(手册)** | **~2 小时** |
| **总计(skill,含并行压力测试 4 章)** | **~4-5 小时** |

### 一旦出错的止损规则

| 故障 | 处理 |
|---|---|
| 某章 OCR 失败率 >50% | **直接跳过**,改用目录页 + 用户补述 |
| 代理商 spawn 失败切回人工 | 单人做 triple-verify,不强行上框架 |
| 1 小时还没读完 | 汇报用户"建议只拆最精华 3 章" |
| 用户反悔产出路径 | 已经产出 skill → 切手册模式只需要改 SKILL.md header,不需要重写 |


