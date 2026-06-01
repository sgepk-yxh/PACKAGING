# Issue 打包标准化 Prompt（通用版 — 适配多种 AI Coding 工具）

> **适用工具**：Cursor / Windsurf / Cline / Claude Code / Copilot Chat / 其他任何 AI Coding 工具
>
> **与内部版差异**：本版本去除了 Cursor 专属的 transcript 导出和 Local History 还原步骤，改为基于 git commit 的 init/final 构造方式。适用于"已知 base commit + AI 工具实现功能"的出题场景。
>
> **路径约定**：本文档中所有 `$ISSUE_ROOT` 均指 Issue 打包的输出根目录。
> 默认值为**当前工作目录**下的 `./case_collection_issue_output/`（即执行打包命令时所在目录的子文件夹）。
> 你也可以通过在打包前 `export ISSUE_ROOT=<你想要的绝对或相对路径>` 来自定义位置。
> **禁止**使用系统根目录下的路径（如 `/data/issue/`），以避免权限问题。
>
> 打包流程的**第一步操作**必须先初始化该变量并创建目录：
> ```bash
> ISSUE_ROOT="${ISSUE_ROOT:-./case_collection_issue_output}"
> mkdir -p "$ISSUE_ROOT"
> ```

将工作过程整理为结构化的 Issue 题目，保存到 `$ISSUE_ROOT/` 目录，并提交到远程仓库的独立分支（一题一分支）供评测使用。

> **提交仓库**：`https://github.com/Qinyun718/external_manual_dataset.git`
>
> **分支命名**：`<你的名字拼音>/<task-slug>`，如 `huangming/add-stepper-nuxtui`

> **任务类型说明**（三选一，打包流程因类型而异）：
>
>
> | 类型         | 典型场景                   | init/final             | 答案形式            |
> | ---------- | ---------------------- | ---------------------- | --------------- |
> | **新功能实现**  | 代码缺少某个能力，需添加           | init=功能未实现，final=功能已实现 | 改动后输出包含期望内容     |
> | **Bug 修复** | 代码有缺陷，需纠正              | init=有缺陷，final=修复后     | 改动后能正常运行        |
> | **代码理解**   | 理解某段代码的行为/流程/设计意图，回答问题 | 只有 `code/`（单一快照）       | ISSUE.md 中的期望答案 |
>
>
> 本轮出题以**新功能实现**为主。前两种类型（代码修改类）打包流程相同，标记为 `[修改类]`；第三种单独标记为 `[理解类]`，差异之处会显式注明。
>
> **评测池通用要求**：
>
> - 三种题型 **同等重要**；应严格根据用户评测目标和题目本质选择类型，不做优先级区分
> - 题目必须同时满足：**可在干净 Docker 中复现、Issue 包自包含、验证标准客观、上下文完整、init/final 差异聚焦**
> - 任何依赖宿主机绝对路径、缺失 import 依赖、引用 CDN/外网资源、手工伪造静态结果的题包，**默认视为不合格，不允许提交远端**

---

## 前置准备：基于 base commit 构造 init 状态

> **与内部版的区别**：内部版通过 Cursor transcript 和 Local History 自动还原 init 基线。本版本直接基于 git commit 构造——init 就是 base commit 的状态，final 就是你实现后的状态。

### 第一步：Clone 仓库到 base commit

```bash
# 从候选题清单中获取 repo 地址和 base_commit
REPO_URL="https://github.com/<owner>/<repo>.git"
BASE_COMMIT="<候选题清单中的 commit hash>"
TASK_SLUG="<候选题清单中的 task_slug>"

git clone "$REPO_URL" "work-$TASK_SLUG"
cd "work-$TASK_SLUG"
git checkout "$BASE_COMMIT"
```

### 第二步：验证 base 状态健康

```bash
# 安装依赖（根据技术栈选择）
# Node.js 项目：
pnpm install  # 或 npm ci / yarn --frozen-lockfile

# Go 项目：
go mod download

# Java/Kotlin 项目：
./gradlew build  # 或 mvn compile

# 跑一次测试确认 base 状态是健康的
pnpm test  # 或 go test ./... / ./gradlew test
```

### 第三步：确认功能当前不存在

按候选题的验收步骤执行，确认预期功能在 base commit 中**确实不存在**。

- 如果功能已存在 → 报告给负责人，题目作废
- 如果 base 状态测试有大量失败 → 弃题换题

---

## Step 0：打包门禁（硬性要求）

在开始复制文件前，先判断题目是否满足以下硬性要求；**任意一条不满足，就不要继续打包，先补齐题目或缩小范围**。

1. **题型匹配**
  - 三种题型同等重要，必须按题目本质归类：Bug 修复 / 新功能实现 / 代码理解
  - 不要把代码理解题误打成代码编辑题，也不要把代码修改题误打成理解题
2. **单题聚焦**
  - 一道题只考察一个核心问题或一组强相关问题
  - 若一段真实用户对话里有多轮**强相关**追加需求，应尽量保留为同一道题，不要为了"切小题"把上下文和难度拍扁
  - 若一个分支里混入多个**明显无关**改动，才考虑拆题；拆题前优先与用户确认是拆分还是保留
  - 不要把一次真实对话机械拆成很多个"小需求题"，这种做法通常会降低题目难度、损失真实协作感
3. **Issue 包必须自包含**
  - 复现与验证默认只能依赖 `$ISSUE_ROOT/` 内的文件
  - 不允许依赖宿主机路径（如 `/Users/...`、`/data/...` 的原项目目录）、本地私有数据库、共享磁盘、手工预放文件
  - 若原始数据很大，优先裁剪出"最小可复现样本"一并打包；**不要**因为文件大就直接留外部路径
  - 打包过程默认把**用户原工作区视为只读源目录**；所有"回退 / 还原 / 删除 / 改动前版本恢复"都只能发生在 `$ISSUE_ROOT/` 或其他临时副本内
  - **禁止**在用户原工作区执行 `git checkout --`、`git restore`、`git reset`、`rm`、覆盖写回等会改变当前状态的命令
  - 如需拿"改动前版本"，只允许使用**只读**方式导出：如 `git show`、`git diff`、`git archive`，或在 `$ISSUE_ROOT/` 副本中反向应用 patch；**不要**切换原仓分支、不要在原工作区来回试错
  - 如果无法在不触碰原工作区的前提下安全构造 `init/`，应先停止打包并补充备份 / 历史信息，而不是继续冒险操作
4. **依赖闭包完整**
  - 所有运行入口直接/间接 import 到的源码、配置、模板、提示词、静态资源、样例数据，都必须在 issue 包中
  - `environment.yml` 必须覆盖运行入口及其依赖链所需的所有三方包
  - **不要为了减小包体而删掉关键上下文**：只要这些文件会显著影响定位、理解、实现路径或真实任务难度，就应该保留
5. **验证必须客观**
  - 必须能通过命令行输出、测试、生成文件内容、结构化字段等方式客观区分 init 和 final
  - 尽量避免"只看页面好不好看""只凭主观阅读文案"这类弱验证
6. **验收必须操作化**
  - `ISSUE.md` 中的每个期望结果都必须能写成：**运行什么 -> 检查哪里 -> 应看到什么**
  - 不允许只写"修好了 / 正确了 / 支持了 / 更完整了 / 更合理了"这类抽象描述
  - 不要把根因分析、实现过程、中间日志、内部重构动作写成验收项
  - **期望结果应优先描述"主验收路径"**，自动化 verifier 是辅助验证；详见 Step 4.5
7. **验收标准必须泛化**
  - verifier 的断言必须验证**功能行为**，而非**实现细节**
  - 题面中只应约束用户真实会提出的技术要求（如 applicationId、SDK 版本、使用某框架），不应规定具体类名、函数名、变量名、文件路径、UI 文案等实现细节
  - 若 verifier 中出现了硬编码的类名/函数名/字段名/文件路径/UI 文本，必须逐项审视：**这个名称是用户在需求中明确指定的，还是出题者从参考实现中抄过来的？** 后者必须改为泛化检查
  - 详细的泛化分层指导见 Step 4.5
8. **输出必须真实生成**
  - 若交付物是 HTML / 报告 / 图表 / JSON 汇总，必须由 final 代码根据 package 内输入数据真实生成
  - 禁止把写死结果的静态 HTML/JSON 当成"实现产物"直接塞进 final 冒充完成
9. **离线友好**
  - Issue 包在 Docker 内验证时，除安装 conda/pip 包外，不应再依赖外网
  - 前端可视化若依赖 JS/CSS/font，优先内嵌或 vendor 到仓库；不要默认引用 CDN
10. **init/final 差异可解释**
  - `init` 与 `final` 的差异必须集中在题目真正考察的实现上
  - 如果 diff 中混入大量无关文件、格式化噪音、自动生成文件，先清理再打包
11. **答案必须闭合**
  - 若题目目标包含"最难 / 最好 / 更合理 / 更完整 / 更美观 / 更像人工"这类开放表述，必须额外补齐 **canonical 答案、确定性排序规则、reference output、结构化 rubric** 中的至少一种
  - 若没有唯一 oracle，或评测者无法仅凭题包判断"到底算不算对"，该题默认不合格，不进入标准评测池
12. **结果必须可重复**
  - 凡是涉及随机采样、排序并列、时间窗口、LLM-as-judge、缓存命中、外部服务返回的题，必须固定 seed / 输入 / mock / 时间基线
  - 同一份 issue 包重复运行，init 现象和 final 验证结果应稳定一致；否则不允许提交

---

## Step 1：用 AI 工具实现功能

使用你熟悉的 AI Coding 工具（Cursor / Windsurf / Cline / Claude Code / Copilot Chat 均可）来实现候选题的需求。

### 操作步骤

1. 在 AI 工具中打开 `work-$TASK_SLUG` 项目
2. 把候选题的 `issue_description` 发给 AI
3. AI 生成代码后，按 `acceptance_steps` 验收
4. 不通过 → 把失败信息反馈给 AI，继续迭代
5. 全部通过 → 跑回归测试确认无破坏
6. 记录交互轮数，进入打包步骤

### 导出会话历史（可选但推荐）

不同 AI 工具的会话记录位置：

| 工具 | 会话记录位置 |
|------|------------|
| Cursor | `~/.cursor/projects/<workspace>/agent-transcripts/<uuid>.jsonl` |
| Windsurf | `~/.codeium/windsurf/memories/` 下的对话文件 |
| Claude Code | `~/.claude/projects/` 下的 JSONL 文件 |
| Cline | VS Code 扩展数据目录下的对话记录 |
| Copilot Chat | VS Code 中无直接文件，可手动复制对话内容 |

如果能导出，复制到 `$ISSUE_ROOT/session_history.jsonl`（或 `.txt`）。这不是必须的，但有助于质检人员理解上下文。

### [修改类] 理解改动内容

1. 修改了哪个文件的哪些行？diff 是什么？
2. 改动前存在什么问题或缺少什么能力？根本原因/需求是什么？
3. 用户观察到了什么现象（异常输出、功能缺失、预期效果未出现）？
4. 改动后的预期行为是什么？

### [理解类] 理解考察内容

1. 要考察的核心对象是什么（函数、类、模块、数据流、执行流程……）？
2. 考察目的是什么（理解行为？理解设计意图？识别边界条件？追踪数据流向……）？
3. 需要回答哪些具体问题？（列出所有问题，后续原样写入 ISSUE.md）
4. 理解这段代码需要哪些上下文（调用方、被调用方、配置文件、测试文件……）？

---

## Step 2：分析代码库依赖结构

**必须仔细分析，不能遗漏任何依赖：**

1. **找到项目根目录**，列出涉及的所有子目录
2. **按技术栈分析依赖，不要只看 Python**
  - **Python**：检查跨目录 import、`sys.path.insert`、入口脚本、测试、模板、prompt、样例数据
  - **JavaScript / TypeScript**：检查 `package.json`、lockfile、workspace 配置、`tsconfig.json`、构建脚本、`vite/webpack/next` 配置、前端静态资源；若 Docker 内不能访问 npm registry，必须准备离线 cache 或 vendor 依赖
  - **Java / Kotlin / Scala**：检查 `pom.xml` / `build.gradle*` / wrapper、JDK 版本、资源目录、annotation processor、测试资源；不要假设宿主机已有 Maven/Gradle 缓存
  - **Go**：检查 `go.mod` / `go.sum` / `go.work`、`vendor/`、`//go:embed` 资源、CGO 依赖；若离线验证，优先 `go mod vendor`
  - **Rust**：检查 `Cargo.toml` / `Cargo.lock`、workspace 成员、`build.rs`、feature flag、生成代码；若离线验证，优先 `cargo vendor`
  - **C / C++**：检查 `Makefile` / `CMakeLists.txt` / `compile_commands.json`、头文件、链接库、运行时数据、编译器版本；不要只复制 `.cpp` 而漏掉 include / config / schema
  - **Shell / Ruby / Perl / R / Lua 等脚本语言**：检查 shebang、外部命令依赖、平台差异（GNU/BSD 工具）、脚本中引用的数据文件和模板
3. **分析 shell 脚本的工作目录**：脚本运行时会 `cd` 到哪里？`PYTHONPATH` / `sys.path` 如何设置？
4. **关注 `sys.path.insert`**：凡是把上级目录加入 `sys.path` 的，对应模块目录都要包含
5. **分析运行时非代码依赖**：配置文件、prompt 模板、HTML 模板、静态资源、fixture、样例数据、README 提示、shell helper 都要纳入依赖清单
6. **分析硬编码路径**：默认不允许依赖 issue 包外的绝对路径；若原始数据过大，优先裁剪最小可复现样本并一起打包，只有用户明确接受外部依赖时才允许保留，且这类题默认不进入标准评测池
7. **注意代码中的绝对路径和外链**，尽可能换成相对路径或本地资源；重点扫描 `/Users/`、`/data/`、`http://`、`https://`
8. **静态扫描高风险引用**：检查代码和产物中是否有以下内容：
  - 宿主机绝对路径
  - CDN / 外部字体 / 外部 JS
  - 指向未复制模块或未包含数据的路径
  - 直接写死最终答案/最终统计结果的文件
9. **静态扫描构建产物与占位实现**
  - 检查是否错误提交了 `dist/`、`build/`、`.next/`、`target/`、编译后二进制、打包产物，且这些文件并不是由题目要求显式考察的对象
  - 检查是否存在 `/path/to/your/...`、`TODO`、`FIXME`、`your_file_here`、空 verifier、只验证"文件存在"的占位逻辑

> **[理解类] 额外要求**：分析范围要超出被考察代码本身——
>
> - 谁调用了被考察的函数/类？（调用方代码也要包含，帮助理解使用场景）
> - 被考察代码依赖了哪些底层模块？（帮助评测者追溯实现细节）
> - 有没有测试文件？（测试即规格，是最直接的行为说明）
> - 若 `python -c "import <核心模块>"` 在独立环境中会失败，说明代码快照不完整，不能打包

---

## Step 3：确定复制范围

### [修改类] 必须包含

- 与 Bug/功能相关的所有源码目录（完整，排除 `__pycache__/`）
- shell 脚本（入口脚本、配置脚本）
- 所有被跨目录 import 的模块
- 所有运行入口间接依赖的非代码资源：配置、模板、静态资源、prompt、fixture、样例数据
- 与核心问题强相关的上游/下游模块，即使严格意义上"跑起来暂时没 import 到"，只要它们能显著影响定位与理解，也建议一并保留
- Issue 当天的运行日志（现象的直接证据）
- 与 Issue 相关模型的分析结果 JSON（断点续评/已完成的证据）
- 近期小日志（≤5MB），提供历史上下文
- **用户需求中明确引用的参考文件**（如"参考 xxx 的格式"中的 xxx，评测人员需要它才能判断输出是否正确）
- **验证所需的输入样例与对照文件**（基线 txt、参考 JSONL、示例 Excel、模板 HTML 等）
- 若产物是 HTML / 报告 / 图表：生成它所需的本地前端依赖或 vendor 资源

### [理解类] 必须包含（范围比修改类更宽）

- **核心模块**：直接被考察的文件/目录（完整，排除 `__pycache__/`）
- **调用方代码**：所有调用了考察目标的上层模块——帮助评测者理解使用场景
- **被调用模块**：考察目标依赖的底层模块——帮助评测者追溯实现细节
- **测试文件**：针对考察目标的单元测试或集成测试——行为规格最直接的体现
- **典型输入/输出样例**：日志、JSON 结果文件等——帮助验证对代码行为的理解
- **配置文件**：影响代码行为的 yaml/json/env 配置

> 原则：**宁多勿少**。评测者只能看 workspace 里的文件，上下文不够就无法作答。
>
> 复制范围的目标不是"压到最小可运行"，而是**尽量保留真实任务上下文与原始难度**；不要因为少拷几个目录就把题目做成低难度 toy task。

### 两种类型都不包含

- 任何 >5MB 的无关历史日志
- `__pycache__/` 目录
- 指向宿主机路径的符号链接
- 依赖 issue 包外文件才能成立的"半成品题包"

---

## Step 4：创建目录结构

### [修改类] 目录结构

```
$ISSUE_ROOT/
├── ISSUE.md              ← Issue 描述 + 复现步骤
├── reproduce.sh          ← 一键安装脚本（conda 环境 + workspace）
├── environment.yml       ← conda 环境定义
├── run-tests.sh          ← 可选；存在稳定自动化断言时提供
├── changes.diff          ← unified diff（patch 格式）
├── session_history.jsonl  ← 可选；AI 工具的完整对话记录
├── tests/
│   └── test_outputs.py   ← 可选；自动化测试 / verifier
├── init/                 ← base commit 的完整代码库快照
│   ├── <主代码目录>/
│   └── <依赖目录>/
├── final/                ← 实现功能后的代码库快照
│   ├── <主代码目录>/
│   └── <依赖目录>/
└── workspace/            ← reproduce.sh 运行后自动生成，用于实际复现
```

**关键要求：**

- `init/` 和 `final/` 必须尽量保留原项目相对路径，不能为了省事把文件平铺成失真目录
- 只要运行入口通过相对路径引用了某个文件，该文件在 `init/` 和 `final/` 中都必须位于同样的相对位置
- 若题目依赖样例数据、模板、静态资源，它们也必须进入快照，而不是只复制代码文件
- 整个打包过程必须把**用户原工作区视为只读输入**
- 任何"改动前版本恢复 / 新增文件删除 / 代码片段替换"都只能发生在 `$ISSUE_ROOT/init`、`$ISSUE_ROOT/final` 或其他临时副本中
- 永远不要为了构造 `init/` 而在用户原工作区执行 `git restore`、`git checkout --`、`git reset`、`rm` 或覆盖写回

**操作步骤：**

```bash
# 初始化打包目录
ISSUE_ROOT="${ISSUE_ROOT:-./case_collection_issue_output}"
mkdir -p "$ISSUE_ROOT"

# ==================== 构造 init ====================
# init = base commit 的状态
mkdir -p "$ISSUE_ROOT/init"
git stash  # 暂存你的改动
rsync -a --exclude='.git/' --exclude='node_modules/' --exclude='__pycache__/' \
  --exclude='.gradle/' --exclude='build/' --exclude='target/' \
  . "$ISSUE_ROOT/init/"
git stash pop

# ==================== 构造 final ====================
# final = 实现功能后的状态（当前工作目录）
mkdir -p "$ISSUE_ROOT/final"
rsync -a --exclude='.git/' --exclude='node_modules/' --exclude='__pycache__/' \
  --exclude='.gradle/' --exclude='build/' --exclude='target/' \
  . "$ISSUE_ROOT/final/"

# ==================== 生成 diff ====================
diff -ruN "$ISSUE_ROOT/init" "$ISSUE_ROOT/final" > "$ISSUE_ROOT/changes.diff" || true

# ==================== 验证 diff 合理性 ====================
# 确认 diff 只包含你改动的文件
diff -rq "$ISSUE_ROOT/init" "$ISSUE_ROOT/final"
```

**必须验证：**

- init 和 final 的 diff 只包含被改动的文件，无意外差异
- 被修改文件：init 中能找到改动前的代码内容
- 新增文件：init 中不应存在该文件

---

### [理解类] 目录结构

```
$ISSUE_ROOT/
├── ISSUE.md                  ← 理解任务描述 + 考察问题列表
├── reproduce.sh              ← 一键安装脚本（conda 环境 + workspace）
├── environment.yml           ← 可选；仅当理解题确实需要运行代码辅助分析时提供
├── run-checks.sh             ← 可选；仅当需要做结构完整性检查或运行 smoke 时提供
├── code/                     ← 需要理解的完整代码快照（含充足上下文）
│   ├── <核心模块>/            ← 直接被考察的代码
│   ├── <调用方代码>/          ← 帮助理解使用场景
│   ├── <被调用模块>/          ← 帮助追溯实现细节
│   └── <测试 & 样例数据>/     ← 帮助验证对代码行为的理解
└── workspace/                ← reproduce.sh 运行后自动生成
```

**操作要点：**

- 按 Step 3 的范围，用 `rsync -a --exclude='__pycache__/'` 将核心模块、调用方、被调用模块等复制到 `code/` 对应子目录
- 复制完成后，对每个考察问题手动确认相关代码都在 `code/` 里

---

## Step 4.5：先定义主验收路径与辅助验证（写 ISSUE.md 之前）

**核心原则：**

- 在写 `ISSUE.md` 之前，先把题目的验收边界定义清楚
- **修改类**：应先写清至少 1 条用户视角 / 评测员视角的主验收路径；若存在稳定自动化断言，再补 verifier
- **理解类**：应先落成"问题 -> 结论 -> 代码依据"的答案映射，不强求自动化测试
- 如果复制范围已经确定，却仍然写不出稳定主验收路径（修改类）或稳定答案映射（理解类），说明题目目标、oracle 或样例还不够闭合，应回到 Step 0 修题

### 验收标准泛化性分层（修改类必读）

verifier 断言的泛化程度直接决定题目质量。以下按优先级从高到低排列——**优先使用高层验证，只在高层不可行时才降级**：


| 层级           | 验证方式                 | 泛化性   | 适用场景            | 示例                                          |
| ------------ | -------------------- | ----- | --------------- | ------------------------------------------- |
| **L1 运行时行为** | 程序能跑、能启动、能产出正确结果     | ★★★★★ | 所有题型的首选         | 构建成功、程序可启动、命令输出正确结果、页面可交互                   |
| **L2 结构/模式** | 搜索注解、基类继承、依赖声明、控件类型  | ★★★★  | 验证架构约束          | 存在特定注解/装饰器、存在继承特定基类的子类、构建文件含特定依赖、UI 有特定类型控件 |
| **L3 语义特征**  | 搜索语义关键词、正则模式         | ★★★   | 验证功能存在性         | 代码中存在与特定功能相关的关键词、正则模式能匹配到目标逻辑               |
| **L4 精确匹配**  | 检查具体文件名、类名、函数名、UI 文本 | ★★    | **仅限用户明确指定的约束** | 用户明确指定的 applicationId、SDK 版本、输出文件名等         |


**泛化性自检清单（每个 verifier 断言都要过一遍）：**

1. 这个断言检查的是**行为**还是**命名**？
  - 行为（"App 能启动""数据库能创建"）→ 好，保留
  - 命名（"有个叫 XX 的类/函数"）→ 问自己：用户要求了这个名字吗？
2. 如果开发者用了不同的名字但功能完全一样，这个断言会失败吗？
  - 会 → 必须改为 L1/L2/L3 层的泛化检查
  - 不会 → 可以保留
3. 这个约束出现在用户的原始需求中吗？
  - 是（用户明确要求了某个框架、某个命名、某个版本）→ 可以精确检查
  - 否（出题者从参考实现中提取的）→ 必须泛化

### [修改类] 主验收路径与 verifier 要求

- 修改类至少要有 **1 条主验收路径**：
  - 优先复用项目已有命令、已有脚本、已有页面、已有产物检查
  - 评测员应能按 `ISSUE.md` 直接执行，不需要额外理解测试框架内部细节
  - 主验收路径要优先覆盖**用户最终能看到的结果**
- 当存在稳定自动化断言时，再补：
  - `$ISSUE_ROOT/tests/test_outputs.py`
  - `$ISSUE_ROOT/run-tests.sh`
- verifier 是**辅助验证**，不是唯一验收依据；不要为了"看起来完整"硬造一套脆弱测试
- 如果没有合适的稳定自动化断言，可以不提供 `run-tests.sh`；但必须把主验收路径写清，并实际跑通
- 每个 verifier 断言都必须对应一个最终验收点，建议按三类编号：
  - **问题复现测试**：`P1 / P2 / ...`，用于锁定 init 中真实存在的旧问题或缺失现象；这类测试的核心目标通常是 **init fail / final pass**
  - **功能验收测试**：`F1 / F2 / ...`，用于验证用户最终能看到的结果是否达标；通常也应满足 **init 不达标 / final 达标**
  - **非回归保护测试**：`G1 / G2 / ...`，用于验证修复问题时没有破坏原有关键能力；若该能力与旧 bug 无关，通常应在 **init / final 都通过**
- 若提供 `run-tests.sh`，它必须能一次性跑完所有 verifier；题包作者需要实际验证它在 Docker 环境中可执行，而不是只写文件不跑
- 建议控制在 **2-5 个高价值断言** 内，避免为了"看起来完整"堆很多低价值 case，反而降低题面清晰度
- 覆盖建议：
  - **Bug 修复题**：通常至少 1 个问题复现测试 + 1 个功能验收测试；若修复风险高，可再补 1 个非回归保护测试
  - **新功能题**：通常至少 1 个功能验收测试；若 init 中存在明确缺失现象，可再补 1 个问题复现测试
- verifier 至少覆盖以下一种强断言，最好多种组合：
  - **内容级**：输出中必须包含 / 不包含什么
  - **结构级**：JSON / CSV / HTML / 报告中的关键字段、列、DOM 片段
  - **数值级**：条数、比率、统计值、排序、去重结果
  - **交互级**：有限且可重复的页面行为或状态变化
- 禁止只有这些弱 verifier：
  - 只检查 exit code = 0
  - 只检查文件存在
  - 只 grep 一个很宽泛的关键词
  - 只验证页面能打开
- 禁止这些脆弱的硬编码 verifier（泛化性不合格）：
  - 检查具体文件路径是否存在
  - grep 具体函数名/变量名，除非用户需求中明确指定了该名称
  - 匹配具体 UI 文本，除非用户需求中明确指定了该文案
  - 检查具体类名，应改为检查注解/基类/接口等结构特征

### [理解类] 验收映射要求

- 理解类**不强制**提供 `run-tests.sh`、`tests/test_outputs.py` 或其他自动化测试
- 必须先整理一份最小验收映射：
  - 每个问题的标准结论是什么
  - 结论分别对应哪些文件 / 函数 / 调用链 / 配置
  - 若需要运行代码辅助理解，应该运行什么命令，以及运行后主要观察什么
- 只有在以下场景下，才建议额外提供 `run-checks.sh` 一类辅助脚本：
  - 需要确认代码快照是否完整
  - 需要 smoke 某个入口才能理解行为
  - 需要验证关键样例数据、配置、测试文件确实存在
- 不要把理解题硬做成"模型回答字符串完全匹配"；理解题的核心仍然是人工根据代码依据评分

### [修改类] verifier 文件约定

- `run-tests.sh` 作为统一入口，调用 `tests/test_outputs.py` 或其他测试文件
- 测试函数按 `P*` / `F*` / `G*` 前缀命名，与验收点映射清楚
- 重点不是文件怎么拆，而是每个 verifier 断言都对应一个最终验收点、且脚本真的跑过
- 如果主验收路径已经足够清晰，但某些点很难稳定自动化，不要为了凑 verifier 降低题面质量

**写完主验收路径 / verifier 后，先做一次最小验证：**

```bash
# 修改类：先按主验收路径验证 init / final
bash "$ISSUE_ROOT/reproduce.sh"
cd "$ISSUE_ROOT/workspace/<入口目录>"
<按 ISSUE.md 中的主验收路径执行 init 验证>

# 应用 final 后再按主验收路径验证
<应用 final 的方式>
<按 ISSUE.md 中的主验收路径执行 final 验证>

# 若提供了 run-tests.sh，再额外跑辅助 verifier
[ -f "$ISSUE_ROOT/run-tests.sh" ] && bash "$ISSUE_ROOT/run-tests.sh" || true
```

> **推荐流程：修改类先主验收路径，再补 verifier（若适合）；理解类先答案映射，再写 ISSUE。** 这样 `ISSUE.md`、checklist、人工评分单都会更清晰。

---

## Step 5：定义运行环境

> **核心原则**：环境配置必须根据 Step 2 的依赖分析结果动态生成，不要照搬模板。所有环境文件的内容（包名、版本号、工具链）完全取决于当前题目的项目。

**修改类**：必须提供完整可运行的环境定义。
**理解类**：如果题目只要求阅读与分析代码、并不依赖实际运行验证，环境定义文件可选，重点保证代码快照和上下文完整。

### eval 容器约束（所有语言通用）

eval 容器基于 `python:3.11-slim`（Debian），以**非 root 用户**运行。

**预装工具**：Python 3.11、conda、pip、Node.js、npm、git、curl、bash、tini
**没有预装**：JDK、Go、Rust、Android SDK、系统级 C 库、apt-get 权限等。

### 安装优先级（三级策略）

1. **优先 conda**：conda-forge 提供了很多非 Python 工具（`openjdk`、`nodejs`、`go`、`cmake`、`rust` 等），能用 `environment.yml` 统一声明的就放进去，一条 `conda env create` 全部装好，最简洁
2. **conda 版本不匹配或没有时，用 reproduce.sh 手动安装**：下载 tar 包到 `$HOME/.tools/`，设置环境变量并持久化到 `~/.bashrc`
3. **复杂工具链（Android SDK 等）**：只能走 reproduce.sh

> **判断标准**：先去 https://anaconda.org/conda-forge/ 搜一下目标工具和版本，有且版本匹配就用 conda，否则 reproduce.sh。

### 按技术栈选择环境定义方式

| 技术栈 | conda-forge 有吗 | 推荐方式 |
|--------|------------------|----------|
| **Python** | 有（本职工作） | `environment.yml` 声明 Python 版本 + conda/pip 包 |
| **Node.js / 前端** | 有（`nodejs`） | 优先 `environment.yml` 装 Node.js，`npm ci` 装项目依赖 |
| **Java / Kotlin** | 有（`openjdk`） | 优先 `environment.yml` 装 `openjdk=17`；若需要特定发行版（Oracle JDK）则 reproduce.sh 手动装 |
| **Go** | 有（`go`） | 优先 `environment.yml` 装 `go=1.22`；若版本不匹配则 reproduce.sh 手动装 |
| **Rust** | 有（`rust`） | 优先 `environment.yml`；若需要 nightly 或特定 toolchain 则 reproduce.sh 装 rustup |
| **C / C++** | 有（`gcc`、`cmake`） | 优先 `environment.yml` |
| **Android SDK** | 没有 | 只能 reproduce.sh 手动下载 |
| **多语言混合** | 部分有 | conda 能管的放 `environment.yml`，剩余的放 reproduce.sh |

### environment.yml 写法

不只是 Python 包——conda-forge 里能找到的非 Python 工具也可以一起声明。

```bash
# 查 Python 包版本
pip show <包名1> <包名2> ... | grep -E "^Name:|^Version:"
# 查 conda-forge 是否有某个工具
conda search -c conda-forge openjdk  # 或 nodejs / go / cmake 等
```

**纯 Python 项目**：
```yaml
name: <env-name>
channels:
  - defaults
  - conda-forge
dependencies:
  - python=<版本>
  - pip
  - <conda包>=<版本>       # pandas、numpy、matplotlib 等优先走 conda
  - pip:
    - <pip包>==<版本>      # litellm、openai 等第三方 LLM 库走 pip
```

**多语言项目（Python + Java + Node.js 等，按项目实际需要组合）**：
```yaml
name: <env-name>
channels:
  - conda-forge
dependencies:
  - python=<版本>
  - openjdk=17           # Java 运行时（按项目实际版本）
  - nodejs=20            # Node.js（按项目实际版本）
  - go=1.22              # Go（如果需要，按项目实际版本）
  - cmake=3.28           # C++ 构建（如果需要）
  - pip
  - pip:
    - <pip包>==<版本>
```

> conda 装不了的（版本不匹配、Android SDK、特定发行版等）再放到 reproduce.sh 里手动安装，详见 Step 6。

### 环境生成后，至少做以下验证（所有语言通用）

1. **入口可运行**
  - 项目入口命令至少能跑到预期的 init/final 分岔点，不得在启动阶段崩溃
  - Python 项目：`python <入口脚本>` 或 `<运行命令>`
  - Java 项目：`./gradlew build` 或 `mvn compile` 能通过
  - Node.js 项目：`npm start` 或 `node <入口>` 能启动
  - Go 项目：`go build ./...` 能通过
2. **依赖可解析**
  - Python：对核心模块执行 `python -c "import ..."`，重点验证跨目录 import、`sys.path.insert` 依赖
  - Java：classpath 完整，编译不报缺失依赖
  - Node.js：`require` / `import` 不报 MODULE_NOT_FOUND
  - Go：`go vet ./...` 不报缺失 package
3. **主验收路径可运行**
  - 必须按 `ISSUE.md` 中的主验收路径至少实跑一遍，确认评测员按已有入口命令 / 已有页面 / 已有产物检查就能判断结果
  - 优先复用项目已有能力，不要只靠临时新写的一组测试用例
4. **产物生成链完整**
  - 若 final 目标是生成 HTML/报告/图表/转换文件，必须实际运行生成命令，确认 package 内输入足以产出最终文件
5. **修改类 verifier 可运行（若提供）**
  - 若题型是修改类且提供了 `run-tests.sh`，它必须能在题包环境中真实执行
  - 修改类 verifier 的核心 `P`* / `F*` 测试必须至少能做到"init 不达标 / final 达标"
  - 若额外提供了 `G*` 非回归保护测试，也必须被实际跑过，并明确预期是"init/final 都通过"还是"仅 final 通过"
  - 若未提供 verifier，不扣分；但主验收路径必须足够清晰且已实跑

**理解类补充说明：**

- 如果题目只要求阅读代码，不强求提供 `environment.yml`
- 如果题目提供了 `run-checks.sh` 或其他辅助脚本，这些脚本也必须实际跑过；若未提供，则只要求 `reproduce.sh` 与代码快照完整性验证通过

### 不同语言的额外要求

- **Node.js / 前端题**：优先带 lockfile 和离线依赖缓存；若使用 `npm ci --offline`，必须把 cache 一并打包并实际验证
- **Java / Kotlin / Android 题**：优先用 conda 装 `openjdk`；若需要 Android SDK 等 conda 没有的工具，reproduce.sh 需自动下载到用户目录；优先带 Gradle/Maven wrapper
- **Go / Rust 题**：优先 vendor 依赖；不要假设 Docker 内可直接访问外网拉 module/crate
- **C / C++ 题**：优先用 conda 装 `gcc`/`cmake`；必须说明并验证编译器版本、构建命令和系统库依赖；若依赖系统级 apt 包，需提前确认评测镜像具备，或改题
- **数据库 / 全栈题**：迁移、seed、fixture、静态资源、前端构建产物链都要真实跑通；不能只验证某个页面能打开

> 只写出一个"看起来精简"的环境配置不算完成；**必须经实际运行验证**。

---

## Step 6：生成 reproduce.sh

**reproduce.sh 的硬性要求：**

- 只能使用 `ISSUE_DIR` 下的相对路径；禁止引用宿主机绝对路径、共享目录、个人目录
- 默认只能创建运行环境并生成 `workspace/`，不要偷偷修改 issue 包外目录
- 必须可重复执行：已有环境时可跳过安装，已有 `workspace/` 时可安全重建
- 若题目在 Docker 验证阶段仍需要额外手工准备数据，说明题包不完整，应先回到 Step 0 修题

**eval 容器运行约束（reproduce.sh 必须兼容）：**

eval 容器基于 `python:3.11-slim`（Debian），以**非 root 用户**运行。reproduce.sh 必须在此约束下正常工作：

- **无 root 权限**：`apt-get`、`mkdir /opt/...` 等需要 root 的操作会失败。conda 装不了的系统级工具必须安装到用户可写目录（如 `$HOME/.tools/` 或 `$ISSUE_DIR/.tools/`），通过下载 tar 包/zip 包解压的方式安装
- **预装工具有限**：只有 Python 3.11、conda、pip、Node.js、npm、git、curl、bash、tini。其他工具优先通过 `environment.yml`（conda-forge）安装，conda 没有的才在 reproduce.sh 中手动安装
- **workspace 创建必须始终成功**：即使某些可选工具安装失败（如模拟器、GUI 工具），`cp -r init/ workspace/` 这一步不能被阻断
- **环境变量需持久化**：安装的工具路径需写入一个 env 文件并追加到容器 bashrc，否则后续 shell 会话找不到

**宿主机依赖声明：**

如果题目的某些验证步骤（如 UI 交互、模拟器、浏览器测试）无法在容器内完成，必须在 ISSUE.md 中明确声明：
- 哪些验证在容器内完成（构建、自动化测试）
- 哪些验证需要宿主机（人工 UI 操作、模拟器）
- 宿主机需要预装什么（只列必要项，给出安装命令）

### [修改类] reproduce.sh 骨架

```bash
#!/bin/bash
set -e
ISSUE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
TOOLS_DIR="$HOME/.tools"
ENV_FILE="$HOME/.issue_env"

# ──────────────────────────────────────────
# 1. 安装运行环境
# ──────────────────────────────────────────

# [conda 环境] 大部分依赖（含非 Python 工具）优先通过 environment.yml 安装
# if ! conda env list | grep -q "^<env-name> "; then
#     conda env create -f "$ISSUE_DIR/environment.yml"
# fi

# [conda 装不了的工具] 手动下载到用户目录（按项目实际需要）
# mkdir -p "$TOOLS_DIR"
# if [ ! -d "$TOOLS_DIR/<tool>" ]; then
#     curl -fsSL "<下载地址>" | tar -xz -C "$TOOLS_DIR"
# fi
# export <TOOL>_HOME="$TOOLS_DIR/<tool>"
# export PATH="$<TOOL>_HOME/bin:$PATH"
# echo "export <TOOL>_HOME=$<TOOL>_HOME" >> "$ENV_FILE"
# echo "export PATH=$<TOOL>_HOME/bin:\$PATH" >> "$ENV_FILE"

# [Node.js 项目依赖] npm ci（若项目有 package.json）
# cd "$ISSUE_DIR/init" && npm ci

# 持久化环境变量（如有手动安装的工具）
# [ -f "$ENV_FILE" ] && grep -qF "source $ENV_FILE" "$HOME/.bashrc" || echo "source $ENV_FILE" >> "$HOME/.bashrc"

# ──────────────────────────────────────────
# 2. 将 init/ 复制为 workspace/（此步必须成功）
# ──────────────────────────────────────────
[ -d "$ISSUE_DIR/workspace" ] && rm -rf "$ISSUE_DIR/workspace"
cp -r "$ISSUE_DIR/init" "$ISSUE_DIR/workspace"
```

### [理解类] reproduce.sh 骨架

```bash
#!/bin/bash
set -e
ISSUE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# 可选：安装运行环境（仅当理解题需要运行代码辅助分析时）

# 将 code/ 复制为 workspace/
[ -d "$ISSUE_DIR/workspace" ] && rm -rf "$ISSUE_DIR/workspace"
cp -r "$ISSUE_DIR/code" "$ISSUE_DIR/workspace"
```

---

## Step 7：生成 ISSUE.md

### [修改类] ISSUE.md 结构

**出题质量要求：**

- 用户真实请求必须像真实工单，不要把根因、修复思路、实现提示提前泄露给解题模型
- 题目要有明确"改前失败 / 改后成功"的边界，避免模糊需求
- 新功能题必须说明输入、输出、格式、条数、边界条件，不能只说"加个功能"
- 若需求本身不是唯一解（如"选最难 300 题""挑最好看的布局""给一个更合理的聚类"），必须在 ISSUE.md 中补齐 reference output 或确定性判定规则
- 若最终交付物是生成文件，必须说明它由什么输入生成、如何验证不是手工写死
- **期望结果必须先写主验收路径**：优先复用已有命令、已有页面、已有脚本、已有产物检查，让评测员直接按"运行什么 / 检查哪里 / 应看到什么"执行
- **如提供 verifier，最好与 verifier 一一对应**：每条期望结果尽量能映射到一个 `P1 / P2 / ...`、`F1 / F2 / ...` 或可选的 `G1 / G2 / ...` 自动化检查点
- **不要把过程信息写成验收项**：比如"改了某函数""加了某配置""重构了某模块""日志里打印了调试信息"，默认都不是验收标准
- 题目应尽量避免依赖特定业务背景知识，重点考察代码定位、实现和验证能力
- **尽量保留真实用户对话演化过程**：如果用户是分几轮逐步追加需求，应在 `模拟的用户真实请求` 中体现这种演化，而不是强行压缩成一句平铺需求
- **但不要引入"对话过程中偶发的新 bug"**：如果某个追加 bug 只是某次实现过程中新产生的偶发现象、并非题目本身想考察的目标，不要一并塞进题面

#### 1. 模拟的用户真实请求

以用户自然口吻写一段求助 LLM 的 prompt：

- **口语化**：像在工单系统反馈问题，不堆砌技术术语
- **现象完整**：依次列出所有异常或新增诉求，每个点单独说清楚
- **引用真实日志**：把终端实际异常输出原样贴进来（用 `>` 格式）
- **不给结论**：只描述现象和困惑，不分析原因
- **不机械拆题**：强相关的多轮诉求尽量写在同一道题里，保持真实复杂度

**多轮对话模拟（重要）：**

如果真实场景中用户是分多轮逐步追加需求的，**必须在题面中还原这种多轮交互过程**，而不是把所有需求压缩成一段平铺的文字。这直接影响题目的真实感和难度——真实场景中模型需要处理需求演化、上下文追踪、前后一致性，压成一段话会把这些考察点全部抹掉。

**判断标准：**

- 用户一开始就把需求说清楚了 → 写成单段需求即可，不要硬拆
- 用户先给了模糊/部分需求，后续逐步澄清、追加约束、补充细节 → **必须用多轮格式**
- 用户在看到初步结果后追加了新要求（如"再加个暗色主题""这个验证不够，改成运行时检查"）→ **必须用多轮格式**

**多轮格式写法：**

用 `> **[用户]`** 引用块标记每轮用户输入，按时间顺序排列。**不写模型的回复**（模型回复是被考察对象）。若某轮追加需求依赖前一轮的结果或模型的输出，用 `*（上下文：……）`* 斜体简要交代衔接背景，不要变成对模型实现的详细描述。

**常见多轮模式（三种都应支持）：**

- **逐步追加功能**：用户先提核心需求，后续追加新能力
- **实现后发现 bug，追加修正**：用户跑了初步结果后发现问题，追加修复要求（用上下文行交代"用户跑了之后发现……"）
- **对实现质量不满意，要求改进**：用户看到结果后认为处理方式不对，要求调整策略

**禁止写法：**

- 不要把多轮压成一段平铺文字——这会丢失"用户跑了才发现问题""看到结果后追加要求"的真实过程
- 不要在用户请求中夹带模型的具体回复代码或实现方案
- 不要把对话过程中偶发的新 bug 塞进题面（除非那个 bug 本身就是题目要考察的）
- 上下文行只写一句话交代背景，不要变成对模型实现的详细描述

#### 2. 详细问题描述

- **影响文件**：`<文件路径>`，`<函数名>`（约第 N 行）
- **现象一**：具体描述（粘贴真实日志片段）
- **现象二**：……（按实际情况增减）
- **对用户的实际影响**：说明这些问题如何妨碍正常工作

#### 3. 根本原因分析

- 指出有问题的代码（带文件名和行号，附代码片段）
- 逐一解释每个现象的成因

#### 4. 期望结果

**注意：期望结果不是"解释你做了什么"，而是"告诉评测员应该怎么检查"。必须与用户真实请求中的每个要求严格对应。**

**期望结果分类组织（重要）：**

期望结果应按验证方式分类组织，方便后续转为评分 checklist 和自动化 verifier：

| 分类 | 含义 | 写法要求 |
|------|------|----------|
| **自动化验证** | 可被脚本客观检查的技术约束 | 写细，每条对应一个 verifier 断言；评测时由脚本一次性跑完 |
| **人工操作验证** | 需要人在 UI/页面/终端上实际操作才能判断的 | 写清操作步骤 + 预期结果；评测时由评测员逐条执行 |
| **关键里程碑** | 构建成功、服务能启动、数据能导入等前置条件 | 单独列出，作为后续验证的前提 |

**为什么要分类：**
- 自动化验证项可以很细（注解数量、SDK 版本、文件结构），因为脚本会处理，评测员不需要逐条手动核对
- 人工操作验证项必须是评测员看得懂、做得到的，不要写"检查是否存在某注解"这种应该交给脚本的事
- 分类清晰的期望结果，后续无论是生成 scorecard、写 verifier、还是评测员对照检查，都能直接按类别使用

**写法规则：**

- 每一行只写一个**最终验收点**，后续可以直接转成人工 checklist
- 每一行必须同时写清：
  - **评测动作**：运行什么命令 / 打开什么页面 / 触发什么交互
  - **检查位置**：看哪个终端输出、哪个文件、哪个页面区域、哪个 JSON 字段
  - **通过标准**：应该看到什么确定性结果
  - **失败表现**：常见失败现象是什么，方便评测员快速判断
- 优先使用**已有入口**来做验收动作：已有 CLI、已有脚本、已有页面、已有报表、已有导出文件，都比额外新造一套专用测试命令更好
- 只写**最终交付状态**，不要写根因、改动方案、内部实现步骤、过程日志、调试信息
- 不要只写"文件存在""命令返回 0""页面能打开"这种弱标准；至少要补**内容级、结构级、数值级、交互级**之一
- **通过标准必须泛化**：描述功能行为（"程序能启动且界面有核心交互控件"），不要描述实现细节（"某文件中有某个具体函数"）；参考 Step 4.5 的泛化分层
- 自动化测试可以作为辅助证据，但**不能替代用户可观察结果**；期望结果里必须保留评测员看得懂、做得到的主验收路径
- 如果是统计/转换/可视化题，必须明确输入样本、输出关键字段、数量、数值或关键文案
- 如果是 HTML/报告题，必须写清"由命令重新生成该文件并检查内容"，不能写成"仓库里有这个文件"
- 页面题要写清页面入口、要检查的模块/Tab/字段/按钮，以及刷新后仍应保持的结果

**禁止写法示例：**

- "支持了 xx 功能"
- "修复了图表问题"
- "逻辑更合理了"
- "加了一个新文件 / 新接口 / 新配置"
- "日志里不再打印某内部报错"（除非用户就是直接看这个日志验收）

```markdown
| 验收编号 | 辅助 verifier（可选） | 评测动作（运行什么） | 检查位置（检查哪里） | 通过标准（应得到什么） | 常见失败表现 |
|----------|------------------------|----------------------|----------------------|------------------------|--------------|
| A1 | `P1`（可选） | 在 `$ISSUE_ROOT/workspace/<入口目录>` 运行项目已有 `<命令>` | 终端输出 `/tmp/final_output.txt` | 原先的错误现象不再出现，且不应出现 `...` | 命令虽成功退出，但仍出现 `...` |
| A2 | `F1`（可选） | 删除旧产物后重新运行已有 `<生成命令>` | `$ISSUE_ROOT/workspace/<产物路径>` | 文件由本次运行重新生成，且包含 `<关键字段/数值/文案>` | 文件存在，但字段缺失/数值不对/内容是旧结果 |
| A3 | `G1`（可选） | 打开 `http://127.0.0.1:3001/<页面>`，切换到 `<Tab/区域>` | 页面 `<组件/表格/图表/按钮>` | 原有 `<区域>` 仍正常渲染，`<字段>` 保持正确，点击 `<按钮>` 后 `<交互结果>` 未被修复改坏 | 页面能打开，但原有区域空白、报错、字段错误或交互失效 |
```

如有必要，可以在表格后补一小段"改动后终端输出样例"，但它只能作为辅助证据，**不能替代验收表本身**。

#### 5. 改动方案

diff 格式对比改动前后代码，简要说明每处改动解决了哪个问题。

#### 6. 复现步骤（四步结构）

```markdown
### 第一步：安装初始环境
bash $ISSUE_ROOT/reproduce.sh

### 第二步：激活环境，运行，观察输出
conda activate <env-name>
cd $ISSUE_ROOT/workspace/<入口目录>
<运行命令> 2>&1 | tee /tmp/init_output.txt

### 第三步：验证初始现象
# 按主验收路径检查：Bug 修复题应看到旧问题存在，新功能题应看到目标能力缺失

### 第四步：验证改动后效果
<应用 final 的方式>
# 按期望结果中的 A1/A2/A3... 逐条检查，每条对应一个验收编号
```

> 若涉及生成文件或构建产物，第四步必须先删除旧产物再重新生成/构建后验证。
>
> **强制要求：第四步中的每条检查都要能和 `期望结果` 表里的验收编号一一对应。**

#### 7. 元信息

```markdown
- 仓库：<repo_url>
- Base commit：`<full_sha>`（<date>）
- 题型：新功能实现 / Bug 修复
- 难度：低 / 中 / 高
```

#### 8. 文件清单


| 文件/目录           | 用途说明                     |
| --------------- | ------------------------ |
| ISSUE.md        | Issue 描述、复现步骤、验证方法       |
| reproduce.sh    | 一键安装脚本                   |
| environment.yml | conda 环境定义               |
| changes.diff    | unified diff             |
| init/           | base commit 的代码库快照       |
| final/          | 实现功能后的代码库快照              |
| workspace/      | 由 reproduce.sh 自动生成，用于复现 |


---

### [理解类] ISSUE.md 结构

与修改类结构相同，区别仅在于：用户的提问是代码理解性问题（无需改代码），没有"改动方案"一节，"期望结果"变为"期望答案"。

#### 1. 模拟的用户真实请求

以用户自然口吻写一段求助 LLM 的 prompt，提出对代码的疑问：

- **口语化**：像在技术群里问问题，不堆砌技术术语
- **问题完整**：把所有想理解的点依次列出；如果这些问题是阅读过程中逐轮追加的，也尽量保留这种顺序
- **引用真实代码片段**：把看不懂的代码贴进来（用代码块格式）
- **不给结论**：只描述困惑，不自己回答
- **多轮追问**：若真实场景是先问宏观问题、看完初步解释后再追问细节，同样用 `> **[用户]`** 多轮格式还原追问过程，不要把所有问题一次性平铺

#### 2. 详细问题描述

- **涉及文件**：`<文件路径>`，`<函数名/类名>`（约第 N 行）
- **问题一**：具体描述（贴出相关代码片段）
- **问题二**：……（按实际情况增减）
- **背景说明**：这些代码在整个系统中的定位，帮助评测者建立上下文

#### 3. 代码分析

- 指出每个问题对应的代码位置（带文件名和行号，附代码片段）
- 逐一解释每个问题的答案和背后的设计意图

#### 4. 期望答案

**注意：期望答案与用户真实请求中的每个问题必须严格一一对应，要十分详细，给评测者充分指引。**

- 不仅写"答案是什么"，还要写"评测员应该去哪些文件/函数/调用链里找到依据"
- 每个问题的答案都应包含**结论**和**代码依据位置**
- 不要只给抽象总结；至少提供能支撑结论的文件路径、函数名、关键变量或调用关系

```markdown
| 问题编号 | 问题 | 期望答案 | 代码依据（文件/函数/调用链） |
|----------|------|----------|--------------------------------|
| Q1 | <问题一描述> | **<期望答案>** | `<文件路径>` / `<函数名>` / `<关键调用关系>` |
| Q2 | <问题二描述> | **<期望答案>** | `<文件路径>` / `<函数名>` / `<关键调用关系>` |
| QN（按实际情况增减） | …… | …… | …… |
```

#### 5. 复现步骤（三步结构）

```markdown
### 第一步：准备代码探索环境
bash $ISSUE_ROOT/reproduce.sh

### 第二步：在 workspace 中探索代码
cd $ISSUE_ROOT/workspace/<核心模块目录>
# 阅读 ISSUE.md 中的问题，在 workspace/ 中探索代码
# 若题目确实需要运行代码辅助理解，可选执行：
<可选运行命令>

### 第三步：验证代码快照完整性
# 手动或脚本确认关键问题涉及的文件都能在 workspace 中找到
# 若题目需要运行代码辅助理解，再额外执行可选命令
```

#### 6. 文件清单


| 文件/目录           | 用途说明                       |
| --------------- | -------------------------- |
| ISSUE.md        | 考察问题列表及各题期望答案              |
| reproduce.sh    | 一键安装脚本                     |
| environment.yml | 可选；仅当需要运行代码辅助理解时提供         |
| run-checks.sh   | 可选；仅当需要做完整性检查或运行 smoke 时提供 |
| code/           | 待理解的完整代码快照（含充足上下文）         |
| workspace/      | 由 reproduce.sh 自动生成，用于探索   |


---

## Step 8：端到端验证（必做，不能跳过）

在隔离的 Docker 容器中执行验证，确保复现步骤与外界环境完全无关。

**验证前先做静态扫描：**

```bash
echo "=== 扫描宿主机路径 / 外链 / CDN ==="
for p in "$ISSUE_ROOT/init" "$ISSUE_ROOT/final" "$ISSUE_ROOT/code" "$ISSUE_ROOT/reproduce.sh" "$ISSUE_ROOT/ISSUE.md"; do
    [ -e "$p" ] || continue
    grep -rn '/Users/\|https://\|http://' "$p" || true
done
```

**判定规则：**

- `ISSUE.md` / `reproduce.sh` 中出现 `$ISSUE_ROOT` 相关路径属于正常
- 代码、模板、HTML、脚本中若出现宿主机路径、原项目路径、CDN 外链，默认视为风险项，必须解释并尽量消除
- 若 final 产物包含硬编码统计值、硬编码生成日期、写死结论，需进一步确认这些内容是否真由运行生成

```bash
# 启动临时 Docker 容器
docker run -it --rm \
    -v "$ISSUE_ROOT":/issue:ro \
    -v /tmp/issue_verify:/verify \
    --name issue_verify \
    continuumio/miniconda3:latest bash

# ── 以下命令在容器内执行 ──
cp -r /issue /verify/issue
cd /verify/issue
bash reproduce.sh
```

### [修改类] 验证

```bash
conda activate <env-name>
cd /verify/issue/workspace/<入口目录>

# 按 ISSUE.md 主验收路径验证 init
timeout 120 <运行命令> 2>&1 | tee /verify/init_output.txt
<检查旧问题仍存在，或目标功能仍缺失>

# 应用 final，按 A1/A2/A3... 逐条验收
<应用 final 的方式>
<按期望结果逐条检查>

# 若提供了 run-tests.sh，作为辅助验证
[ -f /verify/issue/run-tests.sh ] && bash /verify/issue/run-tests.sh

exit
```

### [理解类] 验证

```bash
cd /verify/issue/workspace/<核心模块目录>

# 重点验证：关键文件、调用链、配置、测试、样例是否都在 workspace 中
# 手动对照 ISSUE.md 中的期望答案，确认每个答案在 code/ 中均可找到代码依据
# 若题目声明了"可选运行命令"，再额外执行该命令辅助验证理解
# 若题包额外提供了 run-checks.sh，可执行一次辅助检查；未提供则不要求
[ -f /verify/issue/run-checks.sh ] && bash /verify/issue/run-checks.sh

exit
```

**常见报错排查：**


| 错误类型                    | 原因                        | 解决方案                                             |
| ----------------------- | ------------------------- | ------------------------------------------------ |
| `ModuleNotFoundError`   | 跨目录依赖未复制                  | 检查 `sys.path.insert` 指向的路径，把对应目录加入 init/ 或 code/ |
| 数据文件不存在                 | 题包依赖 issue 包外数据           | 优先裁剪最小可复现样本并打包；不要默认保留原始路径                        |
| conda 缺包                | environment.yml 遗漏包       | `pip show <包名>` 查版本后补充                           |
| [修改类] init 是改动后版本       | 还原步骤失败                    | 场景A用 Python 字符串替换重新还原；场景B用 rm 删除新增文件             |
| [理解类] 答案无法在 code/ 中找到依据 | 上下文文件遗漏，或为了缩包删掉了关键目录      | 按 Step 3 补充调用方/被调用模块/测试/配置，不要为减小题包牺牲难度           |
| 页面/图表在 Docker 中不显示      | 使用了 CDN / 外部字体 / 外部 JS    | 改为本地 vendor 文件、内嵌资源或无外网依赖实现                      |
| final 文件存在但不是运行生成       | 把静态结果误当实现产物提交             | 删除产物后重新运行生成命令，验证内容确由代码产生                         |
| 题型与评测目标不匹配              | 误归类为代码编辑题/代码理解题，或一题混入多种目标 | 重新归类，或拆成更聚焦的单题                                   |
| Docker 镜像拉取失败           | 网络问题                      | 提前 `docker pull continuumio/miniconda3`          |


---

## Step 9：提交到 Git 仓库

**目标仓库**：`https://github.com/Qinyun718/external_manual_dataset.git`
**分支命名规则**：`<你的名字拼音>/<task-slug>`（如 `huangming/add-stepper-nuxtui`、`wangyanqing/add-rate-limiter-hono`）

```bash
# 1. 初始化一个干净的本地仓库（不继承 main 分支内容）
REPO_CLONE_DIR="$ISSUE_ROOT/../_external_manual_dataset_clone"
mkdir -p "$REPO_CLONE_DIR"
cd "$REPO_CLONE_DIR"
git init
git remote add origin https://github.com/Qinyun718/external_manual_dataset.git

# 2. 创建 orphan 分支（无父提交，分支内只有题目文件，不含 main 的内容）
BRANCH="<你的名字拼音>/<task-slug>"
git checkout --orphan "$BRANCH"

# 3. 将 $ISSUE_ROOT/ 内容复制进来（排除 workspace/ 和临时文件）
# 注意：题目文件直接放在仓库根目录下，不要嵌套多层
printf "workspace/\n.init_staging/\n.new_files_list\n" > "$REPO_CLONE_DIR/.gitignore"
rsync -a --exclude='workspace/' --exclude='.init_staging' --exclude='.new_files_list' "$ISSUE_ROOT/" "$REPO_CLONE_DIR/"
git add .
git status   # 检查暂存内容——应只有题目文件，不含 main 分支的任何内容

# 4. 提交（commit message 中类型三选一）
git commit -m "task: add <task-slug>

类型: <Bug修复 / 新功能实现 / 代码理解>
影响文件: <文件路径>
现象/考察点: <一句话描述>"

# 5. 推送到远程
git push -u origin "$BRANCH"
```

**提交后输出分支 URL**，供评测人员使用：

```
Issue 已提交至：
  分支：<你的名字拼音>/<task-slug>
  URL：https://github.com/Qinyun718/external_manual_dataset/tree/<你的名字拼音>/<task-slug>

评测人员复现步骤：
  git clone https://github.com/Qinyun718/external_manual_dataset.git
  git checkout <你的名字拼音>/<task-slug>
  bash reproduce.sh
```

**注意事项：**

- 使用 `--orphan` 创建分支，确保分支内**只有题目文件**，不继承 main 分支的任何内容
- `workspace/` 必须加入 `.gitignore`，不提交到仓库（由 reproduce.sh 生成）
- `session_history.jsonl` 如有导出，需要提交到仓库，作为还原需求上下文的依据
- 每道题独立一个分支，分支之间互不干扰
- 同一题目需要更新时，在同一分支追加 commit，不要新建分支

---

## 质量检查清单

### [修改类]

- 题型与用户评测目标一致，且不是多任务拼盘
- Issue 包不依赖 `/Users/...`、原项目 `/data/...`、共享磁盘、私有数据库等宿主机资源
- `diff -rq "$ISSUE_ROOT/init" "$ISSUE_ROOT/final"` 只输出被改动的文件
- 场景A：`grep "<改动前关键词>" "$ISSUE_ROOT/init/<目标文件>"` 找到改动前代码；场景B：init 中该文件不存在
- `grep "<改动后关键词>" "$ISSUE_ROOT/final/<目标文件>"` 找到改动后代码
- `bash "$ISSUE_ROOT/reproduce.sh"` 能在 eval 容器内（非 root、python:3.11-slim）成功执行
- 若依赖非预装工具（JDK、Go 等），reproduce.sh 能自动下载安装到用户目录，不依赖 `apt-get`
- 在 workspace 中运行入口命令能**完整复现**初始现象（不崩溃、无 `ModuleNotFoundError` / 缺文件错误）
- 对所有关键入口做过 import smoke test，确认依赖闭包完整
- 打包过程中未对用户原工作区做任何写操作；所有回退 / 删除 / 还原都只发生在 issue 副本或临时目录内
- 获取改动前版本时仅使用只读导出方式（如 `git show` / `git diff` / `git archive` / 在副本内反向 apply patch），没有切换原仓分支或 restore 原工作区
- 已先写清至少 1 条用户视角 / 评测员视角的主验收路径，并已实际跑通
- 修改类主验收路径至少能稳定判断"init 不达标，final 达标"
- 若提供 verifier，其中至少有 1 个高价值功能验收测试；Bug 修复题通常还应至少有 1 个问题复现测试
- 若修复风险较高，且适合自动化，已额外补充 1 个高价值 `G`* 非回归保护测试，验证未破坏原有关键能力
- 若提供 `run-tests.sh`，它已被实际执行；`P*` / `F*` 核心测试都能在 Docker 环境中跑通，若提供了 `G*` 也已跑过
- 若提供 verifier，它不是"只检查文件存在 / exit code / 宽泛 grep"的弱占位实现
- 若提供 verifier，已逐条检查泛化性：每个断言验证的是功能行为或结构模式，而非从参考实现中抄来的具体命名
- verifier 中的精确匹配（具体文件名/类名/函数名/UI 文本）仅限用户需求中明确指定的约束
- 已做过"换名测试"思想实验：若开发者用不同但等价的命名，verifier 仍能通过
- 若未提供 verifier，理由合理（例如更适合通过已有命令 / 页面 / 产物验收），且主验收路径足以稳定判断对错
- 若题目存在多个"看起来都能说得通"的输出，已额外提供 canonical 答案 / reference output / 确定性规则
- 若题目生成 HTML/报告/图表/转换文件，已经实际删除旧产物并重新生成验证，不是仓库里写死的静态文件
- 若题目包含前端构建 / 编译 / 打包流程，已经删除旧 build/dist/target 产物并从源码重建验证
- 若题目包含前端可视化，运行结果不依赖 CDN / 外部字体 / 外部 JS 才能成立
- ISSUE.md 的 `期望结果` 已写成"评测动作 / 检查位置 / 通过标准 / 常见失败表现"的验收表，而不是抽象描述
- `期望结果` 中没有混入根因、改动方案、过程日志、实现细节等非验收信息
- `期望结果` 优先复用了已有命令 / 已有页面 / 已有产物做主验收，不是被临时测试脚本牵着走
- `复现步骤` 第四步中的检查项能与 `期望结果` 的 A1/A2/A3... 一一对应
- ISSUE.md 的复现/验证步骤经过实际操作验证，可以一步步跟着做

### [理解类]

- 题型与用户评测目标一致，且确实是代码理解题
- `code/` 中包含核心模块、调用方代码、被调用模块、测试/样例文件
- 每个考察问题的答案都能在 `code/` 中找到对应的代码依据
- ISSUE.md 中每个问题都附有期望答案，且包含代码依据（文件/函数/调用链），核心数值/逻辑准确
- `bash "$ISSUE_ROOT/reproduce.sh"` 能成功执行（至少能生成 workspace）
- 不依赖自动化测试也能完成评测；评测员仅凭 `ISSUE.md` 和 `workspace/` 就能找到问题对应的代码依据
- 若提供了 `run-checks.sh` 或其他辅助脚本，这些脚本已实际验证；若未提供，不扣分
- 即使不运行代码，评测者也能仅凭 workspace 中的代码快照建立完整上下文
- 若题目声明了可选运行命令，相关命令已实际验证；若不需要运行，则不强求 import / 执行通过

### 两种类型通用

- 整个流程不修改 `$ISSUE_ROOT/` 以外的任何文件
- 所有 >5MB 的无关日志已剔除，issue 目录总大小合理（通常几十 MB 以内）
- 代码、模板、HTML、脚本中不存在未解释的宿主机绝对路径和外链引用
- 用户需求中引用的参考文件、对照文件、样例数据都已一并打包
- init / final / code 内的目录结构保持原项目相对关系，运行时相对路径不会失真
- `workspace/` 已加入 `.gitignore`
- 已成功推送到 `https://github.com/Qinyun718/external_manual_dataset.git` 的独立分支
- 输出了分支 URL 和评测人员复现步骤

---

*文档版本：v2.0（通用版） | 日期：2026-04-15*
