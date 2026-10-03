# 联排提示板（Rehearsal CueBoard）

为剧场联排制作的 Web 提示板：舞监（Stage Manager）在草稿中录入提示项的计划开始时间、时长、负责岗位与前置关系，校验通过后发布**只读联排版本**；灯光 / 音响 / 道具等执行岗位在现场页面只登记**本岗位**提示项的实际开始、完成或预计结束时间，系统依据前置项的实际/预计结束自动重算未开始项，并展示顺延导致的岗位冲突。

- 后端：Node.js + Express + better-sqlite3（SQLite，同步事务）
- 前端：浏览器原生 HTML/CSS/JavaScript（无需构建）
- 本地账号体系：服务端 Session 登录，区分**舞监**与**执行岗位**
- 打包：Dockerfile + Docker Compose 一键启动（自动安装依赖、初始化数据）

## 一、快速启动（推荐，Docker 一键）

要求已安装 Docker 与 Docker Compose 插件。

```bash
docker compose up --build -d
```

启动过程会自动完成：

1. 构建镜像并安装 npm 依赖（含 better-sqlite3）；
2. 初始化 SQLite 数据库：建表、创建默认账号、写入一份可直接发布的示例草稿；
3. 在容器内启动 Web 服务。

浏览器访问：

> **http://localhost:3000**

如宿主机 3000 端口被占用，可指定其他端口：

```bash
PORT=8080 docker compose up --build -d
# 访问 http://localhost:8080
```

停止与查看日志：

```bash
docker compose logs -f
docker compose down            # 停止并移除容器（保留数据卷）
docker compose down -v         # 同时删除数据库数据卷（恢复出厂）
```

数据库文件持久化在 Docker 卷 `cueboard-data`（容器内 `/app/data/cueboard.db`）。

## 二、本地直接运行（无需 Docker）

要求 Node.js ≥ 18（推荐 20）。

```bash
npm install
npm run initdb     # 首次运行：建表 + 播种（重复执行安全、幂等）
npm start
```

浏览器访问 **http://localhost:3000**。

## 三、默认账号

首次启动自动创建（请在生产环境及时更改代码/重新播种密码）：

| 角色 | 用户名 | 密码 | 说明 |
| --- | --- | --- | --- |
| 舞监 | `sm` | `sm12345` | 草稿增删改、发布版本、查看全部 |
| 灯光执行 | `light` | `light123` | 仅可登记「灯光」岗位提示项 |
| 音响执行 | `sound` | `sound123` | 仅可登记「音响」岗位提示项 |
| 道具执行 | `props` | `props123` | 仅可登记「道具」岗位提示项 |

## 四、使用说明

### 舞监

1. **草稿编辑**页新增提示项：编号、岗位、计划开始（HH:MM）、时长（分钟）、说明；
2. 在「前置关系」区为提示项指定「某提示项完成后 → 本提示项」，可构成多前置链；
3. 页面实时显示校验结果。满足以下全部条件时才可发布：
   - 每个前置项的**计划结束 ≤ 后项计划开始**；
   - **同一岗位**的计划时段不重叠（首尾相接允许）；
   - 前置依赖**无环**；
   - 失败时明确列出每一项冲突（冲突双方、岗位、时间、环路径）；
4. 点击「发布联排版本」生成只读版本，现场随即以该版本为准；草稿可继续修改并发布新版本。

#### 发布新版前：旧执行版已完成登记的交接（预览 + 选择）

发布新版本前，舞监可在发布区点击「**预览交接资格**」（打开草稿页时也会自动预览），查看**旧执行版**每个提示项能否把实际登记延续到新版：

- **可选（可延续）**条件（须同时满足）：
  - 草稿与旧版是**同一提示项引用**（`cue_ref`，即草稿中未被删除重建的同一条）；
  - **编号、说明、岗位、计划开始、时长、前置集合**六项与旧版**完全一致**；
  - 旧版该登记**同时有实际开始和实际完成**（即旧版中已完成）。
- **不可延续**的项会逐条列出不合格原因，如：`TITLE_CHANGED`（说明已改）、`POSITION_CHANGED`（岗位变化）、`START_CHANGED`（计划开始变化）、`DURATION_CHANGED`（时长变化）、`DEPS_CHANGED`（前置集合变化）、`CODE_CHANGED`（编号变化）、`NOT_IN_DRAFT`（提示项已从草稿删除）、`NO_ACTUAL_START` / `NO_ACTUAL_END`（旧登记缺实际开始/完成，未完成）；
- 首次发布（无旧版）时预览列表为空，按原流程发布。

勾选要延续的项后点「发布联排版本」：

- 请求会携带**草稿版本号**、**旧执行版 id**与每个所选登记行的**行版本号**；确认发布时服务端在同一事务内**重新核对**草稿版本、旧版仍为当前执行版、每项资格仍成立、登记行版本未变——任一变化或资格失效则**整体拒绝**（`409 HANDOVER_STALE` / `409 VERSION_CONFLICT`），不写入任何数据，并提示重新预览；
- 成功后**仅把所选项的实际开始、实际完成**带入新版（不带入预计结束），未选项保持空登记；
- 带入的行在现场页与回看页标注「**交接自第 N 版**」（新版标明来源版本与提示项；登记人为「交接自第N版」，行版本从 1 递增）；
- **旧版登记与更正历史保持可回看**、已发布计划仍冻结；交接本身不写更正历史；
- 计划冲突（同岗位重叠 / 前置顺序 / 依赖环）仍以 `422 PUBLISH_CONFLICT` 阻止发布。

兼容性行为：发布请求**不传 `handover` 字段**时完全维持原发布行为（全部空登记）；草稿修改后未重新预览时，页面会提示预览已过期、需重新预览（或取消所有勾选按原行为发布）。

#### 舞监现场更正（仅当前执行版本）

舞监在「现场联排」页可对任一行登记进行**更正**（点击该行「更正」按钮）：

- 一次只能更正或清除**实际开始 / 实际完成 / 预计结束**中的**任意一项**，并必须填写更正原因（≤200 字）；
- 更正**只改变现场登记**，不修改已发布计划（计划开始/时长/前置关系保持冻结）；
- 提交必须携带所见登记行的**行版本号**，且与岗位登记**共用同一版本校验**：岗位或舞监任一方先提交后，另一方持旧版本提交会收到 **409** 并提示刷新，不能覆盖新登记；
- **归档版本（已有更新版本发布）禁止更正**，现场页对应行显示「归档不可更正」；
- 联动约束（不满足则整体拒绝，不写入任何部分更正与历史）：
  - 更正后的实际完成必须有实际开始，且不得早于实际开始；
  - 清除实际开始时系统**同时清除实际完成**（级联清除，历史中标注「级联清除」）；
  - 保留的预计结束不得早于实际开始；若无实际开始则不得早于原计划开始；
  - 无变化的提交（前后值相同）视为无效。
- 更正成功后按现有规则**重算后续未开始项与同岗位冲突**（顺延不早于原计划，冲突在红条展示）；
- 每次更正都记录**更正项、更正前/更正后值、操作者、时间、原因**；点击行内「更正记录(N)」可回看，历史版本回看页中也可查看（只读）。

### 执行岗位

- 「现场联排」页只能看到当前执行版本，本岗位的行高亮并出现操作按钮：
  - **开始**：登记实际开始（默认取当前时间，可手填 HH:MM）；
  - **完成**：登记实际完成（须先开始，完成时刻不得早于开始）；
  - **预计结束**：未开始或已开始均可登记，不得早于（实际或原计划）开始；
- 非本岗位提示项只读；对非最新（已归档）版本不能登记；
- 已完成项保留实际时间不再变化；未开始项按前置链重算，**重算开始不早于原计划**；
- 顺延产生的同岗位时段冲突在页面顶部红色横幅逐条展示，同时相关行标注「已顺延」；
- 岗位登记可能被舞监按规定更正（见上节），更正与岗位登记共用行版本号，持旧版本提交会被 409 拒绝。

### 已发布版本回看

「发布版本」页列出全部历史版本（版本号、说明、发布时间、登记数量），点击「回看」查看该版本冻结的提示项、前置关系与当时/最新的现场重算结果；舞监还可在回看页查看每行的**更正记录**（更正前后值、操作者、时间、原因），归档版本为只读、不能更正。

### 版本号（乐观锁）规则

- 草稿维护全局**草稿版本号**，页面显示 `vN`。任何编辑、增删依赖、发布都必须携带页面加载时的版本号；他人已修改导致版本号过期时返回 **409** 并明确提示当前版本号，前端自动刷新，需要重新编辑后提交；
- 单条提示项的修改同时校验**行版本号**；
- 现场登记按「提示项行版本号」校验：两位岗位同时操作同一提示项时，先提交者成功，后提交者收到 409 明确报错并刷新；
- **舞监更正与岗位登记共用同一条登记行版本号**：任一方更新后行版本 +1，另一方（或持旧页面重复提交）会收到 409 拒绝并被要求刷新，确保更正不能覆盖新登记、新登记也不能覆盖更正。

## 五、配置项（环境变量）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `3000` | Web 服务监听端口（容器内） |
| `DATA_DIR` | `/app/data`（镜像）/ `./data`（本地） | SQLite 数据库所在目录 |
| `DB_PATH` | `$DATA_DIR/cueboard.db` | 数据库文件完整路径 |
| `SESSION_SECRET` | 开发用默认值 | Session  Cookie 签名密钥，**生产必须修改** |
| `TZ` | `Asia/Shanghai` | 容器时区，影响「取当前时间」登记 |

通过 Compose 修改可在项目根目录创建 `.env`，例如：

```env
PORT=8080
SESSION_SECRET=your-long-random-secret
```

## 六、目录结构

```
.
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh          # 容器入口：先 initdb 再启动服务
├── package.json
├── src
│   ├── server.js          # Express 路由、权限与版本号校验、发布事务
│   ├── db.js              # SQLite 连接与建表
│   ├── initdb.js          # 幂等初始化：账号 + 示例草稿
│   ├── auth.js            # scrypt 密码哈希
│   └── scheduling.js      # 校验/拓扑环检测/现场重算核心算法
└── public
    ├── index.html
    ├── styles.css
    └── app.js             # 登录、草稿、现场、版本回看三个页面
```

## 七、主要 HTTP API

| 方法 | 路径 | 权限 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/login` `/api/logout` | 公开 | 登录 / 退出 |
| GET | `/api/me` | 公开 | 当前会话与岗位列表 |
| GET | `/api/draft` | 舞监 | 草稿（含实时校验结果） |
| POST/PUT/DELETE | `/api/draft/cues[/:id]` | 舞监 | 提示项增改删（校验草稿版本号/行版本号） |
| POST/DELETE | `/api/draft/deps[/:id]` | 舞监 | 前置关系增删（校验草稿版本号） |
| POST | `/api/publish` | 舞监 | 发布只读版本；冲突返回 422 并列出冲突项；可携带 `handover` 交接旧版已完成登记（见下） |
| GET | `/api/publish/handover-preview` | 舞监 | 预览旧执行版各提示项的交接资格与不合格原因（无旧版时 `oldVersion` 为 `null`、`items` 为空） |
| GET | `/api/live` | 已登录 | 当前最新版本的现场重算结果（含每行更正历史 `corrections`） |
| POST | `/api/registrations` | 执行岗位 | start / complete / estimate（校验行版本号、岗位归属、版本最新） |
| POST | `/api/registrations/corrections` | 舞监 | 更正/清除单个登记字段（校验行版本号、仅当前执行版本；级联清除与一致性校验；原子写入更正历史） |
| GET | `/api/versions` `/api/versions/:id` | 已登录 | 版本列表与历史回看（回看含每行更正历史） |

## 八、交接预览与选择：HTTP 调用示例

页面（草稿编辑页发布区「预览交接资格」按钮 + 勾选框）与 HTTP API 均支持。先登录并保存 Cookie：

```bash
curl -c cookie.txt -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"sm","password":"sm12345"}'
```

### 1) 预览交接资格

```bash
curl -b cookie.txt http://localhost:3000/api/publish/handover-preview
```

返回（无旧版时 `oldVersion` 为 `null`、`items` 为 `[]`）：

```json
{
  "draftVersion": 7,
  "oldVersion": { "id": 1, "versionNo": 1, "note": "第一版", "createdAt": "2026-10-03 19:30:12" },
  "items": [
    {
      "cueRef": 2,
      "code": "S1", "title": "开场音乐", "position": "音响",
      "oldPlannedStartText": "19:05", "oldDuration": 3,
      "actualStartText": "19:06", "actualEndText": "19:09",
      "rowVersion": 3,
      "eligible": true,
      "reasons": []
    },
    {
      "cueRef": 4,
      "code": "L2", "title": "追光主角上场", "position": "灯光",
      "oldPlannedStartText": "19:10", "oldDuration": 8,
      "actualStartText": "19:11", "actualEndText": null,
      "rowVersion": 2,
      "eligible": false,
      "reasons": [
        { "code": "NO_ACTUAL_END", "message": "旧登记缺少实际完成（尚未完成）" }
      ]
    }
  ]
}
```

### 2) 携带所选项发布（`handover`）

`items` 中的 `cueRef` 与 `rowVersion` 均取自预览响应；草稿版本号仍沿用现有发布校验。

```bash
curl -b cookie.txt -X POST http://localhost:3000/api/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 7,
    "note": "10 月 4 日联排",
    "handover": {
      "oldVersionId": 1,
      "items": [
        { "cueRef": 2, "rowVersion": 3 }
      ]
    }
  }'
```

成功返回与原发布响应相同的字段（`id/versionNo/cues/conflicts/...`），并多出：

- 顶层 `carriedCount`：本次带入登记的行数；
- `cues[].carrySourceVersionNo`：该行实际时间交接自的旧版版本号（未交接为 `null`）；
- 带入行的 `actualStartText/actualEndText` 已为旧版实际值，`estimatedEndText` 为 `null`，`updatedBy` 为「交接自第1版」。

发布时重新核对失败均为整体拒绝：

| 场景 | 状态码 / error |
| --- | --- |
| 草稿版本号已变化 | `409 VERSION_CONFLICT`（含 `currentVersion`） |
| 旧版已不是当前执行版 / `oldVersionId` 不存在 | `409 HANDOVER_STALE`（含 `previewRequired: true`） |
| 所选项资格失效（草稿被改、旧登记被清空等） | `409 HANDOVER_STALE`（含 `cueRef`、`previewRequired: true`） |
| 登记行版本过期（旧版登记/更正后行版本 +1） | `409 HANDOVER_STALE`（含 `cueRef`、`currentRowVersion`） |
| 计划仍有冲突 | `422 PUBLISH_CONFLICT`（含 `errors`） |
| `handover` 结构/字段非法、`cueRef` 重复 | `400 BAD_INPUT` |

收到 `HANDOVER_STALE` 后应重新调用预览接口，按最新资格与行版本重新选择再发布。

### 3) 不交接（原行为）

不传 `handover` 字段，或预览后取消全部勾选（前端此时不发送该字段）：

```bash
curl -b cookie.txt -X POST http://localhost:3000/api/publish \
  -H 'Content-Type: application/json' \
  -d '{"version": 7, "note": "10 月 4 日联排"}'
```

新版本所有登记行为空，`carriedCount` 为 0。
