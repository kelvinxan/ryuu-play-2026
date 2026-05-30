# RyuuPlay Code Wiki

## 项目概述

**项目名称**: RyuuPlay  
**项目版本**: 0.2.1  
**项目描述**: Pokémon Trading Card Game (PTCG) 模拟器，使用 TypeScript 编写，采用 monorepo 结构。  
**许可证**: MIT

RyuuPlay 是一个用于教育目的的开源项目，主要用于测试各种 AI 机器人实现方法。项目支持真人玩家与机器人对战、机器人与机器人自动对战，并提供排行榜功能。

---

## 项目结构与技术栈

### 项目架构

RyuuPlay 采用 monorepo 架构，使用 npm workspaces 管理多个子包，主要包含以下五个核心包：

```
/workspace
├── packages/
│   ├── common/     # 共享核心库（客户端和服务端共用）
│   ├── sets/       # 卡牌实现集合
│   ├── server/     # 游戏服务器
│   ├── play/       # Angular Web 客户端
│   └── cordova/    # Cordova Android 应用包装
├── avatars/        # 用户头像目录
├── docker/         # Docker 相关配置
├── fastlane/       # Android 应用元数据
├── init.js         # 项目初始化配置
└── package.json    # 根项目配置
```

### 技术栈

| 技术/库 | 用途 |
|---------|------|
| TypeScript | 主要编程语言 |
| Express.js | 后端 API 框架 |
| Socket.IO | WebSocket 通信 |
| Angular 16 | 前端 UI 框架 |
| Angular Material | UI 组件库 |
| TypeORM | 数据库 ORM |
| SQLite/MySQL | 数据库支持 |
| Cordova | Android 应用包装 |

---

## 核心包详解

### 1. @ptcg/common

**路径**: `/workspace/packages/common`  
**职责**: 提供客户端和服务器共享的核心功能和数据类型。

**主要功能模块**:
- `src/game/`: 游戏核心逻辑
  - `cards/`: 卡牌管理和分析
  - `bot-arbiter.ts`: 机器人仲裁器
  - `game-settings.ts`: 游戏设置
  - `player-stats.ts`: 玩家统计
  - `replay.ts`: 游戏回放
  - `simulator.ts`: 游戏模拟器
- `src/interfaces/`: 共享接口定义
- `src/serializer/`: 状态序列化/反序列化
- `src/store/`: 游戏状态管理（核心）
  - `actions/`: 游戏动作定义
  - `card/`: 卡牌类型和实现
  - `effect-reducers/`: 效果处理
  - `effects/`: 游戏效果
  - `prompts/`: 用户提示
  - `reducers/`: 状态更新逻辑
  - `state/`: 游戏状态定义
- `src/utils/`: 工具函数

**关键类**:

#### `State` 类（游戏状态）
**文件**: `src/store/state/state.ts`
```typescript
export class State {
  public cardNames: string[] = [];
  public logs: StateLog[] = [];
  public rules: Rules = new Rules();
  public prompts: Prompt<any>[] = [];
  public phase: GamePhase = GamePhase.WAITING_FOR_PLAYERS;
  public turn = 0;
  public activePlayer: number = 0;
  public winner: GameWinner = GameWinner.NONE;
  public players: Player[] = [];
}
```
**职责**: 存储完整的游戏状态，包括回合数、活动玩家、游戏阶段、卡牌、规则、提示等。

#### `CardManager` 类（卡牌管理）
**文件**: `src/game/cards/card-manager.ts`
```typescript
export class CardManager {
  private static instance: CardManager;
  private cards: Card[] = [];
  private formats: Format[] = [];
  // ...
}
```
**职责**:
- 单例模式管理所有可用卡牌
- 定义卡牌格式（如 Standard、EX Sets、Base Sets）
- 提供卡牌查询和验证功能

**主要方法**:
- `defineFormat(name, sets, rules)`: 定义新的游戏格式
- `defineSet(set)`: 定义卡牌组
- `getCardByName(name)`: 根据名称获取卡牌
- `getAllCards()`: 获取所有卡牌
- `getAllFormats()`: 获取所有可用格式

### 2. @ptcg/sets

**路径**: `/workspace/packages/sets`  
**职责**: 包含所有可用卡牌的实现。

**结构**:
- `src/base-sets/`: 基础卡牌组
- `src/ex-sets/`: EX 系列卡牌组
- `src/standard/`: 标准系列卡牌组
- `src/common/`: 通用攻击、特性、训练师卡牌效果

**已实现的卡牌组**:
- 基础组: Base, Jungle, Fossil
- EX 组: Ruby & Sapphire, Sandstorm, Firered & Leafgreen
- 标准组: Diamond & Pearl, Black & White 系列, Sword & Shield

### 3. @ptcg/server

**路径**: `/workspace/packages/server`  
**职责**: 提供游戏服务器功能，处理客户端连接、数据库交互和游戏状态管理。

**主要模块**:
- `src/backend/`: 后端 API 和控制器
  - `controllers/`: 处理 HTTP 请求
  - `services/`: 认证、CORS、验证等服务
  - `socket/`: WebSocket 通信处理
- `src/email/`: 邮件发送功能
- `src/game/`: 游戏核心
  - `bots/`: 机器人管理
  - `client/`: 客户端接口
  - `core/`: 核心游戏逻辑
  - `tasks/`: 定时任务
- `src/simple-bot/`: SimpleBot AI 实现
- `src/storage/`: 数据库模型和存储层

**关键类**:

#### `Core` 类（核心游戏系统）
**文件**: `src/game/core/core.ts`
```typescript
export class Core {
  public clients: Client[] = [];
  public games: Game[] = [];
  public messager: Messager;
  public db: Storage;
  
  // 主要方法
  connect(client: Client): Client
  disconnect(client: Client): void
  createGame(client: Client, deck: string[], ...): Game
  joinGame(client: Client, game: Game): void
  leaveGame(client: Client, game: Game): void
  deleteGame(game: Game): void
}
```
**职责**:
- 管理所有连接的客户端
- 管理游戏实例
- 处理游戏的创建、加入、离开
- 管理排行榜定期更新

#### `SimpleBot` 类（AI 机器人）
**路径**: `src/simple-bot/`
**职责**:
- 通用 AI 实现，支持任意牌组
- 通过模拟所有可能的行动并评估状态分数来选择最优行动
- 支持各种提示解决器（选择卡牌、能量、宝可梦等）

**核心组件**:
- `simple-tactics/`: 各种战术实现
- `state-score/`: 状态评分系统
- `prompt-resolver/`: 各种提示的解决器

#### `Storage` 类（数据存储）
**文件**: `src/storage/`
**职责**: 使用 TypeORM 管理数据库实体

**数据库模型**（`src/storage/model/`）:
- `User`: 用户账户
- `Deck`: 牌组
- `Match`: 比赛记录
- `Avatar`: 用户头像
- `Replay`: 游戏回放
- `Conversation`: 对话
- `Message`: 消息

### 4. @ptcg/play

**路径**: `/workspace/packages/play`  
**职责**: Angular Web 客户端应用，提供用户界面。

**主要模块** (`src/app/`):
- `api/`: API 服务和 Socket 通信
- `deck/`: 牌组管理
- `games/`: 游戏列表和游戏界面
- `login/`: 登录功能
- `profile/`: 用户个人资料
- `ranking/`: 排行榜
- `replays/`: 游戏回放
- `table/`: 游戏对战桌面
- `shared/`: 共享组件和模块

**关键配置** (`src/environments/`):
- `environment.ts`: 开发环境配置
- `environment.prod.ts`: 生产环境配置
- `environment.cordova.ts`: Cordova 环境配置

### 5. @ptcg/cordova

**路径**: `/workspace/packages/cordova`  
**职责**: Cordova 包装，将 Web 应用打包为 Android 原生应用。

**主要功能**:
- 提供 Android 应用支持
- 图片缓存功能（离线存储卡牌图像）
- 避免 CORS 问题
- 电源管理、文件操作等原生插件

---

## 依赖关系

### 包之间的依赖关系

```
@ptcg/cordova
  └── @ptcg/play
        └── @ptcg/common

@ptcg/play
  └── @ptcg/common

@ptcg/sets
  └── @ptcg/common

@ptcg/server
  ├── @ptcg/common
  └── @ptcg/sets

根项目
  ├── @ptcg/server
  └── @ptcg/sets
```

### 主要第三方依赖

| 包 | 用途 | 版本 |
|----|------|------|
| Express | 后端服务器 | ^4.17.1 |
| Socket.IO | WebSocket 通信 | ^4.8.1 |
| TypeORM | 数据库 ORM | ^0.3.22 |
| SQLite3 | 数据库 | ^5.0.2 |
| MySQL | 数据库 | ^2.18.1 |
| Angular | 前端框架 | ^16.2.12 |
| Angular Material | UI 组件库 | ^16.2.14 |
| Nodemailer | 邮件发送 | ^6.6.5 |
| Jimp | 图像处理 | ^0.16.1 |
| Cordova | Android 包装 | ^11.0.0 |

---

## 游戏流程

### 1. 游戏初始化流程

```
1. 服务器启动
   ├── 加载 init.js 配置
   ├── 初始化数据库
   ├── 加载卡牌组和格式
   ├── 注册机器人
   └── 启动定时任务

2. 用户登录
   ├── 验证用户名和密码
   ├── 获取用户牌组和头像
   └── 获取可用卡牌列表

3. 创建游戏
   ├── 用户选择牌组和格式
   ├── 选择对手（可以是玩家或机器人）
   └── 服务器创建游戏实例
```

### 2. 游戏回合流程

```
游戏阶段:
  ├── WAITING_FOR_PLAYERS (等待玩家)
  ├── SETUP (设置阶段: 抽初始牌、放基础宝可梦等)
  ├── PLAYER_TURN (玩家回合)
      ├── 抽牌阶段
      ├── 行动阶段 (放宝可梦、能量、训练师卡、撤退、攻击等)
      └── 检查胜利条件
  ├── ATTACK (攻击阶段)
  ├── BETWEEN_TURNS (回合间)
  └── FINISHED (游戏结束)
```

### 3. 状态管理架构

项目采用类似 Redux 的状态管理模式：

```
Action (动作)
  ↓
Dispatcher (分发器)
  ↓
Reducer (状态更新器)
  ↓
State (新状态)
  ↓
Effect (副作用)
  ↓
Prompt (提示用户)
```

**主要动作类型** (`src/store/actions/`):
- `AddPlayerAction`: 添加玩家
- `PlayCardAction`: 使用卡牌
- `ResolvePromptAction`: 解决提示
- `AbortGameAction`: 中止游戏
- 等等...

---

## 配置说明

### 服务器配置 (config.ts)

**主要配置项**:

```typescript
// 后端配置
config.backend.address = 'localhost';      // 监听地址
config.backend.port = 12021;              // 端口
config.backend.serverPassword = '';       // 注册所需密码
config.backend.registrationEnabled = true;// 是否启用注册

// 存储配置
config.storage.type = 'sqlite';           // 数据库类型 (sqlite/mysql)
config.storage.database = './database.sq3';// 数据库路径/名称

// 核心配置
config.core.schedulerInterval = 15 * 60 * 1000; // 任务调度间隔
config.core.rankingDecraseRate = 0.95;    // 排名降低率

// 机器人配置
config.bots.defaultPassword = 'bot';      // 机器人默认密码
config.bots.actionDelay = 250;            // 机器人行动延迟

// 卡牌图像配置
config.sets.scansDir = './scans';         // 卡牌图像存储目录
config.sets.scansDownloadUrl = 'https://ptcg.ryuu.eu/scans'; // 图像下载源
```

### 客户端配置 (environment.ts)

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:12021',        // 服务器地址
  allowServerChange: true,                 // 是否允许更改服务器地址
  enableImageCache: false                  // 是否启用图像缓存
};
```

---

## 项目运行方式

### 前置要求

- Node.js 18.19 或更高版本
- npm 7 或更高版本（支持 workspaces）
- SQLite 3 或 MySQL 5

### 安装与构建

```bash
# 1. 克隆项目并安装依赖
npm install

# 2. 构建所有子包
npm run build --workspaces

# 3. 启动应用
npm start
```

### 开发模式

#### 服务器开发

```bash
cd packages/server
npm run start:dev  # 使用 nodemon 自动重启
```

#### 客户端开发

```bash
# 首先构建 common 包
npm run build -w @ptcg/common

# 启动 Angular 开发服务器
npm run start -w @ptcg/play
# 访问 http://localhost:4200
```

### 构建 Android 应用

```bash
# 1. 构建客户端到 cordova 的 www 目录
npm run build -w @ptcg/cordova

# 2. 添加 Android 平台
npx -w @ptcg/cordova cordova platform add android

# 3. 应用补丁
npm run patch-cordova -w @ptcg/cordova

# 4. 构建 Android 应用
npx -w @ptcg/cordova cordova build android --release
```

### 使用 Docker

项目包含 Docker 相关配置，可用于容器化部署。

---

## 开发指南

### 添加新卡牌

1. 在 `packages/sets/src/` 找到合适的卡牌组目录
2. 创建新的卡牌文件（遵循现有模式）
3. 在该卡牌组的 `index.ts` 中导出新卡牌
4. 在 `init.js` 中确保该卡牌组已添加到 CardManager
5. 无需重新构建客户端！

### 添加新机器人

1. 在 `packages/server/src/simple-bot/` 创建新的机器人类
2. 实现必要的战术和提示解决器
3. 在 `init.js` 中注册新机器人
4. 登录机器人账户，配置牌组和头像

### 自定义游戏格式

在 `init.js` 中添加新格式定义：

```javascript
const cardManager = CardManager.getInstance();
cardManager.defineFormat('MyFormat', [
  sets.setMySet1,
  sets.setMySet2
], new Rules({
  firstTurnDrawCard: true,
  firstTurnUseSupporter: false
}));
```

---

## 测试

项目使用 Jasmine 进行测试：

```bash
# 运行 common 包测试
npm run test -w @ptcg/common

# 运行 server 包测试
npm run test -w @ptcg/server

# 运行 sets 包测试
npm run test -w @ptcg/sets

# 运行客户端测试
npm run test -w @ptcg/play
```

---

## 关键文件索引

| 文件 | 功能 |
|------|------|
| `/workspace/init.js` | 服务器初始化配置 |
| `/workspace/packages/common/src/store/state/state.ts` | 游戏状态定义 |
| `/workspace/packages/common/src/game/cards/card-manager.ts` | 卡牌管理 |
| `/workspace/packages/server/src/config.ts` | 服务器默认配置 |
| `/workspace/packages/server/src/game/core/core.ts` | 核心游戏系统 |
| `/workspace/packages/server/src/simple-bot/simple-bot.ts` | AI 机器人 |
| `/workspace/packages/play/src/environments/environment.ts` | 客户端配置 |
| `/workspace/packages/play/src/app/table/table.component.ts` | 游戏桌面对战界面 |

---

## 联系方式与资源

- **演示服务器**: https://ptcg.ryuu.eu/
- **Matrix 聊天**: https://matrix.to/#/#ryuu-play:matrix.org
- **许可证**: MIT

---

*本 Code Wiki 文档最后更新: 2026-05-30*
