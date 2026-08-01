# 项目开发规范（通用版）

> 这份文档给 AI 使用。每次开始写新功能前，先读一遍这份文档，确保风格和已有代码保持一致。
> 完成新功能后，如果引入了新的约定，回来更新这份文档。

---

## 1. 技术栈

- 前端框架：Next.js（React + TypeScript）
- 状态管理：Zustand
- 数据库 ORM：Prisma
- 数据库/后端服务：Supabase（PostgreSQL）

---

## 2. 项目文件结构

```
project/
├── prisma/
│   └── schema.prisma          # 数据库表结构，唯一真相来源
├── app/
│   ├── page.tsx                # 首页
│   └── api/                    # 后端接口（涉及密钥、敏感操作都放这里）
├── components/
│   ├── shared/                 # 公共基础组件（按钮、卡片外壳等）
│   │   └── PanelWrapper.tsx    # 所有"看板/卡片"类组件的统一外壳
│   └── panels/                 # 具体业务面板/看板，每个功能一个文件
│       ├── ExamplePanelA.tsx
│       └── ExamplePanelB.tsx
├── store/
│   └── useAppStore.ts          # Zustand 全局状态仓库
├── lib/
│   └── prisma.ts               # Prisma 客户端初始化
└── .env                        # 密钥配置，不上传 GitHub
```

**规则：**
- 新增一个"看板/卡片/面板"类功能 → 放进 `components/panels/`，文件名用 `功能名 + Panel.tsx`
- 会被多个面板复用的小组件（按钮、弹窗外壳、标签）→ 放进 `components/shared/`
- 任何操作数据库、调用有密钥的外部服务 → 必须走 `app/api/` 下的接口，不能直接在前端组件里调用

---

## 3. 组件开发规则

### 3.1 每个面板必须是独立组件，互不依赖内部实现

- 一个面板的内部状态（比如"是否展开"），只能自己管理，不能依赖另一个面板的内部变量
- 面板之间如果需要共享数据（比如"当前选中的日期"），必须通过 Zustand store 中转，不能互相直接引用

### 3.2 所有面板共用同一个外壳组件

新建面板时，不要重新写"展开/收起""标题栏""卡片样式"这些重复逻辑，统一使用 `components/shared/PanelWrapper.tsx`：

```tsx
<PanelWrapper title="面板标题" defaultExpanded={true}>
  {/* 面板具体内容 */}
</PanelWrapper>
```

如果发现 `PanelWrapper` 现有功能不够用（比如需要一个新的样式变体），修改 `PanelWrapper` 本身，而不是绕开它单独写一套。

### 3.3 命名规范

- 组件文件名：大驼峰，如 `AuctionPanel.tsx`
- 普通变量/函数：小驼峰，如 `currentDate`、`fetchPanelData`
- Zustand store 里的字段：全局共享的数据要有清晰、不冲突的命名，避免用 `data`、`value` 这种模糊名字

---

## 4. 状态管理规则（Zustand）

### 4.1 什么数据该放进 Zustand，什么不该

| 放进 Zustand（全局共享） | 留在组件内部（useState） |
|---|---|
| 当前选中日期 | 某个弹窗是否打开 |
| 登录用户信息 | 输入框当前打字的临时值（提交前） |
| 多个面板都要用的筛选条件 | 单个面板自己的展开/收起状态 |

判断标准：**这份数据如果变了，超过一个组件需要知道 → 放 Zustand；只有这个组件自己关心 → 留在组件内部。**

### 4.2 新增面板时，如何使用共享状态

```tsx
// 不要自己维护一份日期状态
// ✅ 正确：直接从共享仓库读取
const { currentDate } = useAppStore();

// ❌ 错误：自己重新定义一份，会和其他面板不同步
const [currentDate, setCurrentDate] = useState('2026-08-01');
```

---

## 5. 数据库规则（Prisma + Supabase）

### 5.1 表结构变更流程

1. 修改 `prisma/schema.prisma`
2. 本地执行 `npx prisma migrate dev` 同步到 Supabase
3. 确认 Supabase 后台 Table Editor 里表结构符合预期
4. 再开始写用到这张表的前端代码

**不允许**在没有更新 schema.prisma 的情况下，直接去 Supabase 后台手动改表——这会导致 schema 文件和真实数据库不一致，AI 后续生成代码时会依据错误的字段信息。

### 5.2 查询规则

- 禁止无条件 `SELECT *` / `findMany()` 不加任何筛选，必须带时间范围、分页或明确的 `where` 条件
- 只查询实际需要的字段，不需要的字段不要 `select`
- 新增/修改数据时，只提交真正变化的字段（字段级更新），不要把整条记录重新提交一遍

### 5.3 密钥管理

- `DATABASE_URL`、`SUPABASE_SERVICE_ROLE_KEY` 等高权限密钥，只能出现在 `.env` 文件和服务端代码（`app/api/`）里，绝不能出现在前端组件代码中
- 确认 `.gitignore` 已排除 `.env`

---

## 6. 给 AI 的任务边界规则

每次给 AI 分配新任务时，遵循以下原则：

1. **一次只做一个面板/一个功能**，不要一次性要求"把所有面板都写了"
2. **明确声明范围**：告诉 AI "只修改/新增 XX 相关文件，不要动其他已完成的面板文件，除非我明确要求"
3. **每完成一个独立功能，测试通过后立刻提交 Git**，作为可以回退的安全点
4. **如果 AI 判断需要修改公共组件（如 PanelWrapper）**，要先说明原因、影响范围，确认后再动手，因为公共组件的改动会影响所有面板

---

## 7. 常见提醒（针对本项目容易出现的问题）

- 新增数据字段时，检查是否同时需要更新：`schema.prisma` → 迁移 → 对应的 TypeScript 类型 → 用到该字段的所有面板
- 涉及"金额、数量"等数值字段，注意统一单位（如全部存"元"或全部存"分"，不要混用）
- 时间相关字段，注意统一时区处理方式，避免出现"看板A显示的时间和看板B对不上"的情况
