# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在此代码仓库中工作时提供指导。

## 项目概述

雀阁（CardRoomPro）是一个实时在线纸牌房平台：后端使用 Node.js 18+、Express 和 Socket.IO，前端使用 Vue 2 UMD 版本和 jQuery，无构建步骤。当前支持两种游戏：**斗地主（3 人、1 副牌）**和**掼蛋（4 人、2 副牌）**，并提供房间匹配、AI 机器人、观战模式、“智囊”出牌建议、可选的 MySQL 积分持久化和 Discuz JWT 单点登录。服务器直接提供 `static/` 下的静态文件。

本项目由原始斗地主项目持续重构为多玩法房间平台。两个游戏状态机有意保持为两个独立、平行的模块，不要将它们强行抽象为共享状态机。

## 常用命令

```bash
npm install              # 安装依赖（express、socket.io、socket.io-client、jsonwebtoken、mysql2）
npm start                # 等同于 node server.js，监听 PORT，默认 8002
$env:PORT='8012'; node server.js   # Windows PowerShell：指定端口
```

项目没有构建步骤，`package.json` 也没有定义 lint 或 test 脚本。目前没有自动化测试，因此也没有运行单个测试的命令。验证方式是语法检查和手动规则回归：

```bash
node --check server.js
node --check game.js
node --check guandan-game.js
node --check db.js
node --check static/js/guandan-suggest.js
node --check static/js/effects.js
```

修改牌型规则后需手动回归：

- 掼蛋顺子：`A2345`、`23456`、`10JQKA`
- 掼蛋三连对（`223344`）、二连三（`333444`）、三带二（`55522`）
- 同花顺大于不超过 5 张的炸弹；6 张及以上炸弹大于同花顺；天王炸大于所有牌型
- 斗地主：叫分、底牌、出牌和结算

## 配置

启动配置优先级为：**环境变量 > 根目录 `config.json` > 代码默认值**。`server.js` 会将 `config.json` 的值载入 `process.env`，但不会覆盖已经设置的环境变量，使 `db.js` 等下游模块都能通过 `process.env.*` 读取配置。复制 `config.example.json` 为 `config.json`；该文件已被 Git 忽略。

主要配置项包括：`PORT`、`JWT_SECRET`、`DB_HOST`、`DB_PORT`、`DB_USER`、`DB_PASSWORD`、`DB_NAME`、`DB_TABLE_PREFIX`、`DB_DISABLE=1`（仅内存模式）、`SCORE_BASE` 和 `DISCUZ_AVATAR_BASE`。

`JWT_SECRET` 的解析顺序为：`process.env.JWT_SECRET` → `sso-secret.txt` 文件 → 占位值 `change_this_in_production`。`DISCUZ_AVATAR_BASE` 控制备用头像 URL 模板，支持 `{uid}` 占位符。`GET /api/sso/health` 只返回密钥的 SHA-256 指纹，不会返回密钥本身。

## 整体架构

### 服务端（`server.js`）

这是项目核心。文件创建 Express 和 Socket.IO 服务，然后定义 `GameServer` 对象；响应式事件处理函数集中在 `proto` 中，并通过 `Object.assign(GameServer.prototype, proto)` 挂载。每个 `GameServer` 维护以下关键状态：

- `clients`：已连接的 Socket 客户端，每项包含 `{userName, uid, avatarUrl, socket, deskId, posId}`；`posId === 'spec'` 表示观战者。
- `desks`：房间列表。每个 `desk` 包含 `positions[]`（座位状态：`0` 空位、`1` 已入座、`2` 已准备）、`gameType`、`roomCode`、`isPrivate`；掼蛋房间还包含 `guandanLevelRank` 和 `guandanLevelLabel`。
- `gameDatas[deskId]`：房间当前的 `Game` 或 `GuandanGame` 状态机实例。
- `botTimers[deskId]`：等待执行的 AI 操作 `setTimeout`。

HTTP `/api` 中间件只允许 GET/HEAD 请求，并包含限流和 JSON 404 兜底。API 设计为只读；积分写入只会在服务端游戏结算时发生。Socket.IO 事件由 `proto` 处理，处理器修改房间和座位状态，并向房间或大厅广播 `*_CHANGE` 等事件。

客户端断开连接或返回大厅后，`cleanupRoomIfEmpty` 会清理没有真人玩家的房间：移除 AI、清除计时器、调用 `game.init()` 重置游戏，并删除房间。

### 游戏状态机

- `game.js`：斗地主状态机。`Game()` 维护 `status`（`0` 未开始、`1` 叫分、`2` 游戏中、`3` 结束、`4` 需要重发、`5` 错误）、`contextCards`、`contextScore`、`lastCardInfo`、`userScore` 和用于春天/反春判定的 `sumCount`。牌值范围是 3–17，其中 16 为小王、17 为大王。校验逻辑使用 `core-validator.js`。
- `guandan-game.js`：掼蛋状态机。使用 108 张牌，每张牌包含 `deck` 字段（0/1）以区分重复牌。`SEQUENCE_ORDER = [14,15,3,...,14]` 使 A 可低位连接；`HEART_TYPE = 2` 表示级牌花色中的“逢人配”，通配逻辑是 `validate` 的核心。座位 0/2 为一队，1/3 为另一队。级别晋升由服务端的 `advanceRank` 处理。

两个状态机都向服务端提供相同的调用接口：`init()`、`start().getCards()`、`validate(posId, cards)`（返回 `{status, key, type, len, bomb}`）、`next(posId, cards)`、`getStatus()`、`getContextPosId()`、`getResult()` 等。

### AI 机器人

机器人通过服务端计时器 `scheduleBotAction` 执行动作，并且只在当前出牌座位是机器人时运行。斗地主机器人使用 `shouldCallDoudizhuScore` 手牌强度启发式算法，以及 `static/js/ai-suggest.js` 中的 `AISuggest.suggest`。掼蛋机器人使用 `static/js/guandan-suggest.js` 中的 `GuandanSuggest.suggest`；前端“智囊”按钮也使用同一模块，因此服务端机器人和前端建议功能共享一套规则引擎。每局结束后，`rePrepareBots` 会让机器人自动重新准备。

### 数据库（`db.js`）

MySQL 积分持久化是可选功能。服务启动时先执行 `db.init()`，完成后才调用 `server.js` 的 `init()`；缺少积分表时会自动创建 `<prefix>doudizhu_score` 和 `<prefix>guandan_score`。未安装 `mysql2`、未设置 `DB_USER`/`DB_NAME` 或设置 `DB_DISABLE=1` 时，系统会降级为内存模式。只有具有 `uid` 的 JWT 登录真人玩家会记录积分；游客、机器人和观战者不计分。`recordResultToDb` 在结算时计算积分变化，斗地主地主按 2 倍计分。

### 前端（`static/`）

前端是定义在 `index.html` 中的单页 Vue 2 应用，所有模板均为内联形式，没有编译组件。页面通过 `where` 管理状态：`0` 登录、`1` 大厅、`2` 房间。当前提交的界面默认直接使用访客登录（`authMode: 'guest'`、`guestMode: true`），同一页面中仍保留 Discuz 登录分支。SSO Token 通过 `#token=<JWT>` 传入，随后保存到 `localStorage`、从可见 URL 中移除，并作为 Socket.IO 的 `auth.token` 发送。

主要前端模块：

- `js/ai-suggest.js`：斗地主出牌建议，服务端也会使用。
- `js/guandan-suggest.js`：掼蛋出牌建议和 AI 共用规则引擎。
- `js/parser.js`：斗地主牌型解析，用于客户端校验预览。
- `js/effects.js`：音效和特效，包括静音开关。
- `css/theme.css`：国风主题变量。
- `css/style.css`：主要布局和组件样式。
- `js/layer/`：项目内置弹窗组件。
- `jquery.min.js`、`vue.min.js`：项目内置依赖，不使用 npm/ESM 打包。

## 扑克牌显示约定

`value` 为 3–14 时对应 3 到 A，15 对应 2，16 对应小王，17 对应大王。牌面文字通过 `rankLabel`/`labelValue` 转换：11=J、12=Q、13=K、14=A、15=2。掼蛋的顺序更特殊：`SEQUENCE_ORDER` 允许 A 同时作为低位和高位，因此 `A2345` 和 `10JQKA` 都是合法顺子。

修改牌型规则时，必须保持客户端的 `parser.js`/`ai-suggest.js` 与服务端的 `game.js`/`guandan-game.js` 一致；这些实现是手动同步的，不是自动生成的。

## Socket 协议

事件名称快照记录在 `README.md` 的“常用 Socket 事件”章节。服务端向大厅广播 `REFRESH_LIST`，向房间广播 `*_CHANGE` 事件，并在结算时发送 `GAME_OVER` 和 `MY_SCORE`。

如果存在 JWT，Socket.IO 中间件会在握手阶段校验 `socket.handshake.auth.token`。Token 校验失败时会设置 `socket.tokenError`，并立即发送 `LOGIN_FAIL`。同名用户重复连接时，旧连接会被强制断开，避免用户名被锁死。

## 部署注意事项

- 启用 Discuz SSO 时使用 HTTPS，并在生产环境设置强随机 `JWT_SECRET`。
- Socket.IO 的 `pingInterval`/`pingTimeout` 已针对 Cloudflare 橙云代理调整，以避免约 100 秒的空闲连接中断。
- `.codex-ui-audit/` 保存 UI 审计截图和记录，只是诊断输出，不属于运行时文件。
