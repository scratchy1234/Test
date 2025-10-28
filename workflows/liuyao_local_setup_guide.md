# 六爻多智能体本地部署新手指南

> 目标：按照本指南，你可以在本地 Mac/Windows/Linux 电脑上搭建最小可用的六爻解读产品原型，完成“输入六爻盘面 → 获得结构化中文报告”的闭环。全程无需对云端环境或复杂 DevOps 有深入经验。

## 0. 准备工作

### 0.1 必备账号
- **OpenAI 账户与 API Key**：用于调用推理模型。进入 [OpenAI 平台](https://platform.openai.com/) 生成密钥。
- **GitHub 账户**（可选）：便于拉取开源数据项目并管理代码。

### 0.2 推荐软件
| 工具 | 作用 | 下载地址 |
| ---- | ---- | ---- |
| Python 3.11+ | 运行后端、脚本 | [python.org](https://www.python.org/downloads/) |
| Node.js 18+ | 运行前端（可选） | [nodejs.org](https://nodejs.org/) |
| Docker Desktop | 启动 Postgres、向量库 | [docker.com](https://www.docker.com/products/docker-desktop) |
| Git | 管理代码、同步仓库 | [git-scm.com](https://git-scm.com/downloads) |
| VS Code / PyCharm | 编写代码 | [code.visualstudio.com](https://code.visualstudio.com/) |

安装完成后，在终端中确认版本：
```bash
python --version
node --version
npm --version
docker --version
```

### 0.3 创建工作目录
```bash
mkdir liuyao-agent && cd liuyao-agent
git init
```

> **小贴士**：所有命令均在终端执行。Windows 用户可使用 PowerShell；如果命令以 `./` 开头，需要在 PowerShell 中改为 `python` 或 `bash` 的形式。

---

## 1. 克隆资料 & 建立知识库

### 1.1 下载开源卦象数据
我们使用已确认 MIT 许可证的仓库，确保版权安全。
```bash
git submodule add https://github.com/kentang2017/ichingshifa external/ichingshifa
git submodule add https://github.com/chengjun/iching external/iching
git submodule add https://github.com/adamblvck/iching-wilhelm-dataset external/iching-wilhelm
```

### 1.2 安装 Python 虚拟环境
```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install --upgrade pip
```

### 1.3 安装依赖
创建 `requirements.txt`：
```bash
cat <<'REQ' > requirements.txt
fastapi
uvicorn
sqlalchemy
psycopg[binary]
pydantic
httpx
openai
python-dotenv
pandas
polars
pgvector
REQ
pip install -r requirements.txt
```

### 1.4 运行知识库整理脚本
创建 `scripts/bootstrap_knowledge.py`：
```bash
mkdir -p scripts data/raw data/processed
cat <<'PY' > scripts/bootstrap_knowledge.py
import json
import os
from pathlib import Path

RAW_DIR = Path("data/raw")
PROCESSED = Path("data/processed")
PROCESSED.mkdir(parents=True, exist_ok=True)

# 示例：从 ichingshifa 中抽取卦辞
source_file = RAW_DIR / "ichingshifa_hexagrams.json"
if not source_file.exists():
    raise SystemExit("请先将 external/ 仓库中的数据文件复制到 data/raw 目录")

data = json.loads(source_file.read_text(encoding="utf-8"))
out = []
for item in data:
    out.append({
        "hexagram_id": item["id"],
        "name": item["name"],
        "upper": item["upper"],
        "lower": item["lower"],
        "judgement": item.get("judgement"),
        "image": item.get("image"),
        "license": "MIT",
        "source": "kentang2017/ichingshifa"
    })

(Path("data/processed/hexagrams.json")).write_text(
    json.dumps(out, ensure_ascii=False, indent=2), encoding="utf-8"
)
print("已生成 data/processed/hexagrams.json")
PY
python scripts/bootstrap_knowledge.py
```
> 脚本仅为示例，真实项目中请继续为六亲、六神、爻辞等建立相似抽取脚本。

---

## 2. 配置数据库与向量检索

### 2.1 创建 `docker-compose.yml`
```bash
cat <<'YML' > docker-compose.yml
version: "3.9"
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: liuyao
      POSTGRES_PASSWORD: liuyao
      POSTGRES_DB: liuyao
    ports:
      - "5432:5432"
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
  vectordb:
    image: ankane/pgvector
    depends_on:
      - db
    environment:
      POSTGRES_USER: liuyao
      POSTGRES_PASSWORD: liuyao
      POSTGRES_DB: liuyao_vector
    ports:
      - "7432:5432"
    volumes:
      - ./pgvector-data:/var/lib/postgresql/data
YML
```

### 2.2 启动容器
```bash
docker compose up -d
```

### 2.3 初始化数据库结构
创建 `alembic.sql`（简化版）：
```bash
cat <<'SQL' > alembic.sql
CREATE TABLE IF NOT EXISTS hexagrams (
    hexagram_id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    upper TEXT,
    lower TEXT,
    judgement TEXT,
    image TEXT,
    source TEXT,
    license TEXT
);
SQL
psql postgresql://liuyao:liuyao@localhost:5432/liuyao -f alembic.sql
```

### 2.4 导入处理后的数据
```bash
python - <<'PY'
import json
import psycopg
from pathlib import Path

conn = psycopg.connect("postgresql://liuyao:liuyao@localhost:5432/liuyao")
cur = conn.cursor()
with open(Path("data/processed/hexagrams.json"), encoding="utf-8") as f:
    data = json.load(f)
for item in data:
    cur.execute(
        """
        INSERT INTO hexagrams (hexagram_id, name, upper, lower, judgement, image, source, license)
        VALUES (%(hexagram_id)s, %(name)s, %(upper)s, %(lower)s, %(judgement)s, %(image)s, %(source)s, %(license)s)
        ON CONFLICT (hexagram_id) DO UPDATE
        SET name = EXCLUDED.name,
            upper = EXCLUDED.upper,
            lower = EXCLUDED.lower,
            judgement = EXCLUDED.judgement,
            image = EXCLUDED.image,
            source = EXCLUDED.source,
            license = EXCLUDED.license;
        """,
        item,
    )
conn.commit()
print("导入完成")
PY
```

---

## 3. 构建后端服务（FastAPI）

### 3.1 创建 `.env`
```bash
cat <<'ENV' > .env
OPENAI_API_KEY=在此填入你的密钥
DATABASE_URL=postgresql+psycopg://liuyao:liuyao@localhost:5432/liuyao
ENV
```

### 3.2 编写 `app/main.py`
```bash
mkdir -p app
cat <<'PY' > app/main.py
from fastapi import FastAPI
from pydantic import BaseModel
from sqlalchemy import create_engine, text
from dotenv import load_dotenv
import os

app = FastAPI(title="Liuyao Agent Backend")
load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"], future=True)

class HexagramRequest(BaseModel):
    hexagram_id: str

@app.get("/health")
def health_check():
    return {"status": "ok"}

@app.post("/hexagram")
def hexagram_lookup(req: HexagramRequest):
    with engine.connect() as conn:
        row = conn.execute(
            text("SELECT * FROM hexagrams WHERE hexagram_id = :hid"),
            {"hid": req.hexagram_id}
        ).mappings().first()
    if not row:
        return {"found": False}
    return {"found": True, "data": dict(row)}
PY
```

### 3.3 启动后端
```bash
uvicorn app.main:app --reload --env-file .env
```
访问 `http://127.0.0.1:8000/docs` 检查接口是否正常。

---

## 4. 准备多智能体流程

### 4.1 本地编排（简单模式）
创建 `orchestrator/run_local.py`：
```bash
mkdir -p orchestrator
cat <<'PY' > orchestrator/run_local.py
import json
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

SYSTEM_PROMPT = """
你是資深的六爻解析助手，會根據提供的盤面輸出結構化報告。若資料缺失，請在對應區塊標註“資料不足”。全程使用繁體或簡體中文均可。
"""

TEMPLATE = """
請依照以下格式生成：
1. 一頁摘要（TL;DR）
2. 盤面複述（核對區）
3. 卦象結構分析
4. 用神與爻位
5. 核心判斷鏈（列 3-5 條）
6. 逐爻 × 時間段解讀
7. 風險與邊界（What NOT to do）
8. 機遇（列 2-4 條）
9. 風險（列 2-4 條）
10. 一句總結（≤20 字）
11. 建議（實操，僅限當前主題）
12. 一句話總結（20-35 字）
最後附註：“以上解讀僅供參考，請結合自身情況決策。”
"""

def run(payload_path: str):
    payload = json.loads(open(payload_path, encoding="utf-8").read())
    completion = client.responses.create(
        model="gpt-4.1-mini",
        input=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": TEMPLATE + "\n" + json.dumps(payload, ensure_ascii=False)}
        ]
    )
    print(completion.output[0].content[0].text)

if __name__ == "__main__":
    run("sample_payload.json")
PY
```

### 4.2 准备示例输入
```bash
cat <<'JSON' > sample_payload.json
{
  "本卦": {"卦名": "乾", "上卦": "乾", "下卦": "乾", "宫": "乾宫", "特定格局": "八纯"},
  "互卦": {"卦名": "屯"},
  "变卦": {"卦名": "姤"},
  "世": "五爻",
  "应": "二爻",
  "六亲": ["兄弟", "父母", "官鬼", "妻财", "子孙", "父母"],
  "六神": ["青龙", "朱雀", "勾陈", "腾蛇", "白虎", "玄武"],
  "日月建": {"月建": "壬午", "日建": "乙酉"},
  "动静爻": ["静", "动", "静", "静", "动", "静"],
  "空亡": "子丑空",
  "特定格局": "六合",
  "主题": "投资",
  "背景": "计划对新技术项目进行资金投入，时间紧张。",
  "求问事项": "是否应在本季度启动投资？",
  "起卦时间": {"原始输入": "2024-06-16 09:30 GMT+8", "干支": "甲辰年壬午月乙酉日己巳时"},
  "六爻盘面": [
    {"爻位": 1, "本卦六亲": "兄弟", "本卦阴阳": "阳", "六神": "青龙", "动静": "静", "伏神": "子孙", "变卦六亲": "兄弟"},
    {"爻位": 2, "本卦六亲": "父母", "本卦阴阳": "阴", "六神": "朱雀", "动静": "动", "伏神": "妻财", "变卦六亲": "子孙"}
  ]
}
JSON
```

### 4.3 运行测试
```bash
python orchestrator/run_local.py
```
如果一切顺利，终端会打印完整的结构化中文报告。初版只依赖单个 Agent；未来可以将 `SYSTEM_PROMPT` 拆分为多个角色，并根据 `liuyao_multiagent_workflow.md` 中的流程接入校验、审校逻辑。

---

## 5. 升级为多 Agent 编排

1. **输入校验 Agent**：将 `sample_payload.json` 的校验逻辑写成独立函数，先运行确认字段齐全，再将标准化结果传入主 Agent。
2. **卦理解析 Agent**：调用后端 `/hexagram` 接口，获取数据库中的卦辞、动静规则，再汇总给生成 Agent。
3. **报告生成 Agent**：沿用 `SYSTEM_PROMPT`，但输入改为 `校验结果 + 卦理要点 + 用户背景`。
4. **合规 Agent**（可选）：使用敏感词列表或自定义 Prompt 检查输出，必要时要求重写。

> OpenAI Agent Builder 中可按照“工作流 → 添加节点 → 选择模型/工具”方式搭建。将后端 API 注册为工具，使 Agent 能查询卦象资料。

---

## 6. 前端最小原型（可选）

1. 初始化前端项目：
   ```bash
   npm create vite@latest web -- --template react-ts
   cd web
   npm install
   npm install axios tailwindcss
   npx tailwindcss init -p
   ```
2. 配置 `.env.local`：
   ```env
   VITE_API_BASE=http://127.0.0.1:8000
   ```
3. 在 `src/App.tsx` 中构建一个表单，提交数据到后端，并显示 AI 返回的 Markdown（可以使用 `react-markdown` 渲染）。
4. 启动前端：
   ```bash
   npm run dev
   ```

---

## 7. 本地运维小贴士
- 每次开发前执行 `source .venv/bin/activate` 激活虚拟环境。
- 若 Docker 占用内存，可在完成测试后 `docker compose down`。
- 重要文件建议提交到 Git：
  ```bash
  git add .
  git commit -m "chore: bootstrap local liuyao agent"
  ```
- 将 `.env`、`.venv/`、`postgres-data/`、`pgvector-data/` 加入 `.gitignore`，避免泄露密钥。

---

## 8. 下一步
- 参照 `workflows/liuyao_multiagent_workflow.md` 细化每个 Agent 的 Prompt 与输入输出格式。
- 结合 `workflows/liuyao_product_implementation.md` 中的测试、监控方案，逐步扩展为可上线版本。
- 持续补充知识库字段（六亲、六神、特定格局等），并记录来源与许可证。
- 为不同主题编写禁止词/提示词表，保证建议与主题一致。

祝贺你完成第一版本地原型！有了以上骨架，就可以在机器上反复调试、逐步扩展功能，再考虑上线或提供给真实用户。
