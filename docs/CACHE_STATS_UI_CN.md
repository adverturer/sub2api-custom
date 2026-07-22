# 缓存统计 UI 自定义说明

本文记录本仓库在 Sub2API Usage 统计卡片上的自定义 UI 改动，方便后续跟随上游更新时检查、迁移和验证该功能。

## 功能概览

在管理端 Usage 页面的“总 Token”统计卡片中，保留原有输入、输出、总缓存和悬停明细，并额外常驻显示：

```text
缓存命中: 22 · 缓存创建: 12
缓存命中率: 22/134 16.4%
```

对应英文界面：

```text
Cache hit: 22 · Cache create: 12
Cache hit rate: 22/134 16.4%
```

该展示让缓存读取量、缓存创建量和命中率无需悬停即可查看。

## 数据映射

| UI 字段 | 数据字段 | 含义 |
| --- | --- | --- |
| 缓存命中 | `total_cache_read_tokens` | 从上游缓存读取的 Token 数量 |
| 缓存创建 | `total_cache_creation_tokens` | 写入或创建上游缓存的 Token 数量 |
| 普通输入 | `total_input_tokens` | 不含缓存统计项的输入 Token 数量 |

这些字段已经存在于现有 Usage API 类型中，本改动不新增后端接口，也不修改数据库结构。

## 命中率计算

命中率与 `TokenUsageTrend.vue` 使用相同口径：

```text
分母 = total_input_tokens
     + total_cache_read_tokens
     + total_cache_creation_tokens

命中率 = total_cache_read_tokens / 分母 * 100%
```

示例：

```text
input          = 100
cache_read     = 22
cache_creation = 12
分母           = 100 + 22 + 12 = 134
命中率         = 22 / 134 = 16.4%
```

当分母为 `0` 时，命中率显示为 `0.0%`，避免出现 `NaN` 或除零错误。

## 展示规则

- 缓存命中使用蓝色 `#0ea5e9`。
- 缓存创建使用橙色 `#f59e0b`。
- 两项之间使用中点 `·` 分隔。
- 命中率标签使用辅助灰色，数值使用缓存命中的蓝色。
- Token 数值继续复用卡片原有的 `formatTokens` 格式化逻辑。
- 百分比固定保留一位小数。
- 使用 `flex-wrap`，避免窄屏下文本挤压或溢出。
- 原有总缓存数值和悬停明细保持不变。

## 改动文件

### `frontend/src/components/admin/usage/UsageStatsCards.vue`

新增内容：

1. 缓存命中与缓存创建常驻展示。
2. 缓存命中率常驻展示。
3. `cacheReadTokens` 和 `cacheCreationTokens` 计算属性。
4. `cacheHitRateDenom` 分母计算属性。
5. `cacheHitRatePct` 百分比计算属性及零分母保护。

### `frontend/src/components/admin/usage/__tests__/UsageStatsCards.spec.ts`

测试覆盖：

- 原有总缓存和悬停明细仍然显示。
- 缓存命中和缓存创建文案正常渲染。
- 缓存命中率文案正常渲染。
- 示例数据按照 `22 / 134` 显示 `16.4%`。

## 国际化

本功能复用项目已有的中英文文案：

```text
usage.cacheHit
usage.cacheCreate
usage.cacheHitRate
```

对应文件：

```text
frontend/src/i18n/locales/zh/dashboard.ts
frontend/src/i18n/locales/en/dashboard.ts
```

目前无需新增或修改翻译文件。如果上游后续删除或重命名这些键，需要同步调整组件与测试。

## 验证方式

进入前端目录：

```bash
cd frontend
```

运行组件测试：

```bash
pnpm test:run src/components/admin/usage/__tests__/UsageStatsCards.spec.ts
```

运行类型检查：

```bash
pnpm typecheck
```

构建前端：

```bash
pnpm build
```

也可以在仓库根目录构建完整镜像：

```bash
docker build -t sub2api:local .
```

部署后进入管理端 Usage 页面，确认以下场景：

| 场景 | 预期结果 |
| --- | --- |
| 所有 Token 均为 0 | 显示 `0/0 0.0%` |
| 有输入但没有缓存读取 | 命中率显示 `0.0%` |
| 有缓存读取和缓存创建 | 分母包含 input、read 和 creation |
| Token 数量超过 1K/1M/1B | 继续使用原有 K/M/B 格式化 |
| 中文界面 | 显示“缓存命中 / 缓存创建 / 缓存命中率” |
| 英文界面 | 显示“Cache hit / Cache create / Cache hit rate” |
| 窄屏 | 两项允许换行，不应溢出卡片 |

## 跟随上游更新

上游更新后，重点检查以下内容：

1. `UsageStatsCards.vue` 是否仍接收 `AdminUsageStatsResponse | UsageStatsResponse`。
2. API 类型中是否仍存在 `total_cache_read_tokens` 和 `total_cache_creation_tokens`。
3. `TokenUsageTrend.vue` 的命中率口径是否发生变化。
4. `usage.cacheHit`、`usage.cacheCreate`、`usage.cacheHitRate` 是否仍存在。
5. 上游是否已经提供同类常驻展示，避免重复 UI。

若 `UsageStatsCards.vue` 发生冲突，优先保留上游的新布局和组件结构，再重新加入以下最小逻辑：

```ts
const cacheReadTokens = computed(() => props.stats?.total_cache_read_tokens || 0)
const cacheCreationTokens = computed(() => props.stats?.total_cache_creation_tokens || 0)

const cacheHitRateDenom = computed(() => {
  const input = props.stats?.total_input_tokens || 0
  return input + cacheReadTokens.value + cacheCreationTokens.value
})

const cacheHitRatePct = computed(() => {
  const denom = cacheHitRateDenom.value
  if (denom <= 0) return 0
  return (cacheReadTokens.value / denom) * 100
})
```

迁移完成后重新运行组件测试、类型检查和前端构建。

## Git 记录

初始 UI 提交：

```text
94bc082 feat(ui): show cache hit/create stats and hit rate
```

该提交仅修改组件和对应测试，不包含 Docker、部署环境或个人构建配置。
