# Anthropic Partner Basecamp · 本地工作区速查

> 这份文档是给本机使用者的中文导航，配合官方 `README.md` 一起看。
> 工作区根目录：`C:\Basecamp-Exercises-main`

---

## 1. 一次性准备

### 1.1 工作区结构

```
C:\Basecamp-Exercises-main\
├── README.md                          官方仓库说明（英文）
├── SETUP.md                           本文档·中文导航
├── .env.example                       API key 模板
├── .gitignore                         覆盖所有 session 的忽略规则
├── Basecamp-Exercises.code-workspace  VS Code 多根工作区文件
├── .venv\                             共享 Python venv（给所有 notebook 用）
├── day1\
│   ├── 01_inventory-management\       Claude Code 主练习（FastAPI + Vue）
│   │   ├── server\.venv\              后端独立 venv
│   │   └── client\node_modules\       前端依赖
│   ├── 02_developer-platform\         Notebook
│   ├── 03_prompt-rescue\              Notebook
│   └── 04_diagnosing-ai-problems\     纯文档诊断，无代码
└── day2\
    ├── 01_evals\                      Notebook
    ├── 02_inference-optimization\     Notebook
    ├── 03_context-engineering\        Notebook
    └── 04_agent-build-hackathon\      Notebook
```

### 1.2 配置 API key

把 `.env.example` 复制为 `.env`，填入真 key：

```powershell
Copy-Item .env.example .env
notepad .env
```

或者直接在当前 PowerShell 窗口里临时设：

```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-..."
```

> 多数 notebook 顶部都有一个 "API Key Configuration" 单元格，可以直接把 key 粘在那里——但**不要把粘了 key 的 notebook 提交到任何仓库**。

### 1.3 打开工作区

```powershell
code C:\Basecamp-Exercises-main\Basecamp-Exercises.code-workspace
```

打开后 VS Code 会建议安装 Python、Jupyter、Vue (Volar)、Claude Code 扩展。

---

## 2. 各 session 怎么跑

### Day 1 · 01 Inventory Management（Claude Code 主练习）

需要开两个终端：

**后端（PowerShell A）**：
```powershell
cd C:\Basecamp-Exercises-main\day1\01_inventory-management\server
.\.venv\Scripts\Activate.ps1
python main.py
# 监听 http://localhost:8001  ·  API 文档 http://localhost:8001/docs
```

**前端（PowerShell B）**：
```powershell
cd C:\Basecamp-Exercises-main\day1\01_inventory-management\client
npm run dev
# 打开 http://localhost:3000
```

停止：每个窗口 `Ctrl+C`。

> 仓库自带的 `scripts/start.sh` / `stop.sh` 是 macOS/Linux 用的，Windows 上**不用**它们，手动跑两个终端即可。
> `.claude/` 目录下已经配好了 agents（code-reviewer、security-auditor、vue-expert）、自定义 slash commands（/start、/stop、/test、/optimize 等）、hooks 和 skill，进到这个目录用 Claude Code 时会自动生效。

### Day 1 · 02 Developer Platform / 03 Prompt Rescue & Day 2 · 全部

这些是 Jupyter notebook。**所有都用根目录的共享 `.venv` 当 kernel**。

VS Code 操作：
1. 用 `Basecamp-Exercises.code-workspace` 打开工作区
2. 点开任一 `*.ipynb`
3. 右上角 "Select Kernel" → "Python Environments" → 选 `.venv (Python 3.14.x)`（路径里带 `Basecamp-Exercises-main\.venv\Scripts\python.exe`）
4. 顶部的 API key 单元格里粘上 key（或先 `$env:ANTHROPIC_API_KEY = "..."` 再启动 VS Code）
5. Shift+Enter 一格一格跑，或 "Run All"

命令行 Jupyter 方式（备用）：
```powershell
cd C:\Basecamp-Exercises-main
.\.venv\Scripts\Activate.ps1
jupyter notebook
```

> Notebook 顶部经常有 `%pip install -q anthropic ...` 这种 cell，在 `.venv` 已预装的情况下它会快速跳过；如果 notebook 引入了新依赖，会在 `.venv` 里追加安装，不影响其他 session。

### Day 1 · 04 Diagnosing AI Problems

**没有代码可运行**——这是个读 artifact 做根因分析的练习。
1. 先单独读 `Priya_Email.pdf`，写 3 个猜想
2. 再去翻 system prompts、tool schemas、execution traces 验证
3. `diagnostic-framework.md` 是一页 cheat sheet

---

## 3. 共享 `.venv` 里都装了什么

为了让所有 notebook 直接能跑，根目录 `.venv` 里预装了：

| 包 | 用在哪 |
|---|---|
| `anthropic` | 所有 notebook |
| `jupyter`, `ipykernel` | 全部 notebook（VS Code Jupyter kernel） |
| `python-dotenv` | 部分 notebook 用它读 `.env` |
| `matplotlib`, `seaborn` | Prompt Rescue、Context Engineering 等画图 |
| `pandas`, `numpy` | Context Engineering、Evals |
| `tabulate` | Inference Optimization（结果表格） |
| `tqdm` | Context Engineering（进度条） |
| `python-Levenshtein` | Context Engineering（重复词忠实度评分） |
| `claude-agent-sdk` | Inference Optimization、可能用于 Hackathon |

后端 `server\.venv` 单独装：`fastapi`、`uvicorn`、`pydantic`、`pytest`、`pytest-asyncio`、`httpx`、`pytest-cov`。

---

## 4. 常见坑

- **PowerShell 激活 venv 报"无法加载，因为运行脚本已被禁用"**：
  以管理员开一次 PowerShell 跑 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`，再回到正常窗口。
- **API key 没生效**：
  notebook 是在它启动那一刻读环境变量的——改了 `.env` 或 `$env:ANTHROPIC_API_KEY` 后，**重启 kernel** 才会读到。
- **notebook 里 `%pip install` 装包后还报 ImportError**：
  装完包后**重启 kernel**（VS Code: 笔记本工具栏 ⟳ 图标 → Restart）。
- **端口 3000 / 8001 被占**：
  `Get-NetTCPConnection -LocalPort 8001` 找到占用进程后 `Stop-Process -Id <PID>`。
- **Python 3.14 上某些 wheel 装不上**：
  目前装好的依赖都没问题；如果某个 notebook `%pip install` 新包失败，回退方案是用 Python 3.12 重建 `.venv`。

---

## 5. 关于安全

`.env` 已经被 `.gitignore` 排除，但仍然**不要**把 key 写进任何 notebook cell 后 commit。Notebook 顶部的 `api_key = ""` 这种 cell，结束练习前清空再保存。
