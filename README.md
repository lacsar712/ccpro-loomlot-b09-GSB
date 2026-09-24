# LoomLot-01 · 染坊缸染与色牢度抽检

靛蓝染坊台：按 **染坊 → 染缸 → 染程 → 色牢度** 工序推进，聚焦缸染调度与抽检，不是库存出入库系统。

## 技术栈

| 层 | 技术 |
| --- | --- |
| Backend | FastAPI + SQLAlchemy 2 + Pydantic v2 + Postgres + JWT |
| Frontend | Svelte 4 + Vite + svelte-spa-router |
| 部署 | docker-compose（db + backend + frontend/nginx） |

## 端口

| 服务 | 端口 |
| --- | --- |
| 前端 | **3600** |
| 后端 API | **8600** |
| PostgreSQL | **5439** |

数据库账号：`loomlot` / `loomlot` / 库名 `loomlot`。

## 演示账号

| 用户名 | 密码 | 角色 |
| --- | --- | --- |
| `admin` | `123456` | 染坊主管 |
| `dyer` | `123456` | 染程操作员 |

容器启动时 entrypoint 自动建表并 seed。

## 快速启动

```bash
cd D:\work\document\bytecode\claudeCodePro\LoomLot\LoomLot-01
docker compose up -d --build
```

浏览器：http://localhost:3600  
API：http://localhost:8600/api/health

停止：

```bash
docker compose down
```

## 业务实体

1. **DyeHouse** — `name`, `waterNote`, `notes`
2. **Vat** — `dyeHouseId`, `vatCode`, `fiberType`, `capacityL`, `status` ∈ `ready|dyeing|drain`
3. **DyeLot** — `vatId`, `recipeName`, `fabricKg`, `startedAt`, `operatorName`, `closed`, `closedAt`
4. **FastnessCheck** — `dyeLotId`, `checkedAt`, `washFastness`(1–5), `rubFastness`(>0), `tempC`, `notes`

### 角色矩阵

| 动作 | 染程操作员 `dyer` | 染坊主管 `admin` |
| --- | --- | --- |
| 新建染程 | ✅（操作人必须且恒为本人登录显示名，异名请求体 400） | ✅（同规则） |
| 修改染程（配方/缸/时间等） | ✅ | ✅ |
| 修改操作人 | ❌ 接口不接收 `operatorName`，任何角色均不可改 | ❌ 同左 |
| 关闭染程 `POST /api/dye-lots/{id}/close` | ❌ 403 | ✅ |
| 重复关闭染程 | — | ❌ 409 |
| 未关闭染程追加色牢度抽检 | ✅ | ✅ |
| 已关闭染程追加色牢度抽检 | ❌ 409「该染程已关闭，禁止再追加色牢度抽检」 | ❌ 同左（任何人都被拦） |
| 染程列表按关闭状态筛选 | ✅ `GET /api/dye-lots?closed=false|true` | ✅ |

> 前端仅按角色隐藏「关闭」按钮、并把已关闭染程移出抽检下拉；真正的 403/409 强制均在后端，绕过前端直调 API 同样被拒。被拒绝的抽检不产生记录，因此不计入看板统计。

### 规则

- 仅当染缸状态为 `ready` 或 `dyeing` 时可新建染程，否则 409
- 新建染程后，染缸状态自动设为 `dyeing`
- 染程创建时 `operatorName` 必须等于登录用户的 `displayName`，否则 400；落库值恒取登录显示名
- 染程只能由主管关闭；关闭后不可再追加色牢度抽检（409 中文错误），关闭不可逆
- 看板 `GET /api/dashboard/stats` 增加 `openLotCount`（未关闭染程数，与 `GET /api/dye-lots?closed=false` 行数一致）
- 可选接口：`POST /api/vats/{id}/drain` 将染缸置为 `drain`

## 主要 API

- `POST /api/auth/login`（OAuth2 表单）
- `GET /api/auth/me`
- `GET/POST/PUT/DELETE /api/dye-houses`
- `GET/POST/PUT/DELETE /api/vats` · `POST /api/vats/{id}/drain`
- `GET/POST/PUT/DELETE /api/dye-lots`（`GET` 支持 `?closed=true|false`）· `POST /api/dye-lots/{id}/close`（仅主管）
- `GET/POST/PUT/DELETE /api/fastness-checks`（向已关闭染程追加返回 409）
- `GET /api/dashboard/stats`

除登录外需 `Authorization: Bearer <token>`。字段对外为 camelCase。

## 目录

```
LoomLot-01/
├── docker-compose.yml
├── backend/          # FastAPI
├── frontend/         # Svelte 4 + Vite + nginx
└── README.md
```

## 本地开发

### 数据库

```bash
docker compose up -d db
```

### 后端

```bash
cd backend
python -m venv .venv
# Windows: .\.venv\Scripts\activate
pip install -r requirements.txt
$env:DATABASE_URL="postgresql+psycopg2://loomlot:loomlot@127.0.0.1:5439/loomlot"
python -c "from app.database import Base, engine; from app import models; Base.metadata.create_all(bind=engine)"
python -c "from app.seed import seed; seed()"
uvicorn app.main:app --reload --port 8600
```

### 前端

```bash
cd frontend
npm install
npm run dev
```

开发态 Vite 将 `/api` 代理到 `http://127.0.0.1:8600`。
