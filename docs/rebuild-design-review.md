# TenonAdmin 重构计划审阅建议

> 审阅对象：`docs/rebuild-design.md`  
> 参考补充：`docs/补充.txt`、`docs/补充2.txt`  
> 审阅时间：2026-07-06  
> 结论摘要：方向正确，但 v1.0 范围偏大；建议收敛 MVP，修正文档冲突，并补充安全、会话、数据库、OpenAPI、错误码等落地细节。

---

## 1. 总体评价

整体判断：**方向是对的，定位也清楚，但 v1.0 范围偏大，文档里有几处自相矛盾/旧方案残留，需要收口。**

如果目标是“新开源项目第一版能打”，建议把 v1.0 改成：

> **后端 NuGet 核心 + Vue Naive UI 一套 + Docker demo + 覆写能力验证**

React、Soybean 二皮肤、MQTT、Excel、Minio、代码生成等全部后置。

### 1.1 方案里做得好的地方

这份方案最好的地方有 5 个：

1. **产品定位明确**  
   “引用一个 NuGet 包 + 三行 Program.cs + 直接跑起来”这个卖点很清楚，也比继续维护旧 SimpleAdmin 大而全框架更适合开源传播。

2. **核心原则对：少依赖、可替换、可覆写**  
   旧 SimpleAdmin 依赖 MoYu / Mapster / Captcha / Minio / BouncyCastle / NewLife 等包较多，现在改成核心只保留 SqlSugar，这个方向能明显降低使用门槛。

3. **可重写性写得比较好**  
   `TryAdd`、继承覆写、`DisabledModules`、扩展点接口，这些都是真正能形成“框架产品能力”的点，不是普通 CRUD 模板。

4. **多机构数据权限保留是对的**  
   这是旧版 SimpleAdmin 的核心价值，不能砍。很多开源 admin 都有 RBAC，但数据权限做得不好，你这个可以作为差异点。

5. **前端收敛到 Naive UI 一套是对的**  
   不要一开始就 Vue 两套 UI。Naive UI 做一套完整、好看、稳定的，比 Element Plus / Soybean / Naive 三套半成品更有价值。

### 1.2 当前主要问题

现在的问题也比较明显：

- v1.0 还想同时做后端、Vue、React、Docker、多语言、数据权限、可重写机制、上传分片、OpenAPI 生成、设计系统，**工作量已经接近重新做一个商业框架**。
- 文档里有几处**和原则冲突**：比如说不用 Mapster，但示例里还写了 `input.Adapt<Device>()`；说显式 DI，但用户模块又写“自动扫描”；说 MQTT v1.x，但配置里又把 MQTT 放到 v1。
- “除 SqlSugar 外不引第三方运行时依赖”这个原则要再精确定义：是**核心四包**不引，还是整个生态都不引？现在可选包里 SimpleRedis / SimpleMQTT / Scalar / BouncyCastle 其实都是第三方或外部依赖。

---

## 2. 建议删减 / 后移的内容

### 2.1 React 版不要进 v1.0

现在文档里 M3 是 React 版，M4 才 v1.0 发布。建议直接改：

> v1.0 只发布后端 NuGet + Vue Naive UI 模板；React 版列为 v1.1 / v1.x。

原因：

- 双前端维护成本太高。
- v1.0 最重要的是验证后端 NuGet 产品形态。
- Vue 版如果没打磨好，React 版做出来也只是多一套负担。
- 开源用户第一眼看的是“能不能跑、页面是否完整、文档是否清楚”，不是“有没有 React 版”。

建议文档改成：

```md
### M3 —— v1.0 发布准备
- [ ] 文档补全
- [ ] Docker demo
- [ ] NuGet v1.0.0 + Vue 前端 tag
- [ ] OpenAPI 契约产物归档

### v1.x
React 版模板、SoybeanUI 皮肤、Excel、代码生成、Minio、MQTT、任务调度、国密。
```

---

### 2.2 SoybeanUI 二皮肤继续后移，不要在 v1.0 做任何适配层

文档里已经判断“不做适配层”是对的。建议更坚决一点：

- v1.0：只做 Naive UI。
- v1.x：如果用户真有需求，再基于相同 composables 补 Soybean 视图层。
- 不要设计 `UiAdapter` / `ComponentAdapter` / `TableAdapter` 这种东西。

薄适配层最后会变成：

- Naive 的特性用不了；
- Soybean 的特性也用不了；
- 两边都写得别扭；
- 后期维护成本爆炸。

建议把“支持两种 UI”的措辞改为：

> v1.0 不支持多 UI 运行时切换；只保证页面业务逻辑与 UI 视图分离，为未来补第二套视图层保留低成本路径。

---

### 2.3 MQTT 不要放进 v1.0

文档里有冲突：

- §2 包矩阵写了 `TenonAdmin.Mqtt` 可选包；
- §4 又把 MQTT 放进 v1.x；
- §12 里 `Notify:Mode = Polling | Mqtt` 又像 v1 配置。

建议收口为：

- v1.0：只有 HTTP 轮询。
- v1.x：`TenonAdmin.Mqtt` 可选包。
- v1.0 配置里不要出现 `Notify:Mode = Mqtt`，否则用户会以为第一版就支持。

可以改成：

```jsonc
"Notify": {
  "Mode": "Polling",
  "PollingIntervalSeconds": 30
}
```

然后文档注明：

> `Mqtt` 模式由 v1.x 的 `TenonAdmin.Mqtt` 可选包提供，v1.0 不暴露该配置值。

---

### 2.4 分片上传建议不要进 v1.0，除非旧代码可低成本搬迁

§4 里文件模块写了“本地上传含分片”。这个容易拖进度：

- 断点续传；
- 分片合并；
- 秒传；
- 临时分片清理；
- 文件 hash；
- 并发上传；
- 权限控制；
- 本地/OSS 两套存储一致性。

如果 v1.0 核心目标是 NuGet 后端和管理系统可跑，建议先做：

- 普通本地上传；
- 下载；
- 删除；
- 文件列表；
- 大小限制；
- 后缀白名单；
- 路径安全。

分片上传放 v1.x，或者单独列成：

> 若旧版 chunk upload 可直接复用且不引入额外依赖，可作为 v1.0 stretch goal；否则后移到 v1.x。

---

### 2.5 多布局切换可以弱化

前端 §7.4 里写了 Vertical / Classic / Columns / Transverse 多布局。这个对第一版不是关键能力。

建议 v1.0 只做：

- 默认左侧菜单布局；
- 顶栏；
- 面包屑；
- 用户菜单；
- 主题色；
- 暗黑模式；
- 可选 tabs。

Classic / Columns / Transverse 放 v1.x。  
第一版要避免“框架看起来很全，但每个布局细节都不够稳”。

---

### 2.6 `typeui MCP / Claude Design / Figma` 这部分建议从主计划里降级

§7.1 里把设计生产流水线写得有点“依赖外部工具”。建议主计划只保留最终产物：

- `DESIGN.md`
- `tokens.css`
- 核心页面规范图
- Vue/React 共享 token 规则

至于 typeui、Figma、Claude Design、pencil MCP 可以写到“可选参考工具”，不要让它看起来像项目必要前置条件。

否则别人读文档会觉得：这个项目启动还依赖一堆 MCP / AI 设计工具。

---

### 2.7 可选包矩阵先别铺太满

现在列了：

- Redis
- MQTT
- Scalar
- 国密
- Excel
- Minio
- IP2Region
- Observability
- Scheduling
- CodeGen
- Storage

建议 v1.0 文档只保留“已规划但不承诺”的 v1.x 列表，不要一开始就建所有包目录。否则仓库一建就是很多空包。

v1.0 后端实际建议目录：

```text
src/
├─ TenonAdmin.Core
├─ TenonAdmin.SqlSugar
├─ TenonAdmin.Services
├─ TenonAdmin.AspNetCore
└─ TenonAdmin
samples/
tests/
docs/
```

`TenonAdmin.Caching.Redis` 可以作为 v1.0 或 v1.1，看你是否必须要 Redis 登录态。如果默认内存能跑，Redis 就先后移。

---

## 3. 必须补充的内容

### 3.1 补一节“v1.0 非目标 / 不做什么”

这个很重要。现在文档写了很多“做什么”，但没有一个清晰的“不做什么”。建议加在 §0 或 §4 后面。

建议内容：

```md
## v1.0 非目标

v1.0 明确不做：
- 不支持旧 SimpleAdmin 数据迁移；
- 不支持多租户；
- 不提供 React 版正式模板；
- 不提供 Vue 第二套 UI 皮肤；
- 不提供代码生成；
- 不提供 Excel 导入导出；
- 不提供 Minio / OSS 存储；
- 不提供 MQTT 推送；
- 不提供 SignalR；
- 不提供国密；
- 不提供任务调度；
- 不提供复杂工作流 / 表单引擎；
- 不承诺生产环境自动改表。
```

这样可以防止项目膨胀。

---

### 3.2 补“安全基线”

现在安全提到了登录锁定、验证码、限流，但还不够系统。建议新增一节：

```md
## 安全基线

- 密码哈希：PBKDF2，存储格式包含算法版本、迭代次数、盐、hash，便于未来升级。
- Refresh Token：数据库/缓存持久化，支持轮换、吊销、复用检测。
- JWT：生产必须配置 SecretKey；开发密钥只允许 Development 环境自动生成。
- 登录防护：验证码 + 登录失败锁定 + RateLimiter。
- 文件上传：后缀白名单、大小限制、文件名重写、路径穿越防护、Content-Type 不作为唯一依据。
- 权限：默认拒绝；未标记 `[IgnoreRolePermission]` 的后台业务接口默认需要认证。
- 敏感日志：密码、token、密钥、手机号等字段脱敏。
- CORS：默认只允许本地开发源；生产必须显式配置。
```

特别是 Refresh Token 需要写清楚。否则 JWT + 多端登录 + 强退 + 在线用户，最后很容易实现成“token 发出去就不可控”。

---

### 3.3 补“认证 / 会话模型”的具体设计

文档现在只写了：

- 多端并存；
- 单端踢旧；
- 限并发数；
- 在线用户；
- 强退。

但缺关键表/缓存模型。

建议补：

```md
### 会话与 Token 模型

- AccessToken：短期 JWT，不落库。
- RefreshToken：服务端保存 hash，不保存明文。
- SessionId：写入 JWT claim，作为强退和在线用户的稳定标识。
- 缓存 key：`tenon:session:{sessionId}`，保存用户、设备、过期时间、状态。
- 强退：删除/标记 session，权限过滤器每次请求校验 session 状态。
- Refresh 轮换：每次刷新吊销旧 RefreshToken，签发新 RefreshToken。
- 单端模式：新登录时吊销同用户其他 session。
- MaxConcurrent：超过数量时按最早登录时间吊销旧 session。
```

这块是后面能否支持在线用户、强退、权限变更立即生效的基础。

---

### 3.4 补“数据库迁移 / CodeFirst 策略”

你现在写“CodeFirst 自动建表，生产默认禁用”，方向对，但还需要具体一点。

建议补：

```md
### 数据库结构策略

- Development：默认允许 CodeFirst 建表和追加字段。
- Production：默认禁止自动改表；必须显式开启 `Database:EnableCodeFirstInProduction`。
- 框架内置表使用 `SchemaVersion` 记录版本。
- 种子数据必须幂等，按稳定主键/编码判重。
- 不做破坏性自动迁移：不自动删表、不自动删列、不自动改窄字段。
- 发布说明里列出每个版本的表结构变更。
```

SqlSugar CodeFirst 很方便，但生产环境自动改表要非常谨慎。开源框架更应该默认保守。

---

### 3.5 补“错误码目录和命名规则”

§13 写了错误码，但还没形成规范。建议加：

```md
### 错误码分段

- 0：成功
- 40000-40999：认证与登录
- 41000-41999：权限与数据范围
- 42000-42999：用户/组织/角色/菜单
- 43000-43999：字典/配置
- 44000-44999：文件上传
- 50000-50999：系统内部错误

错误返回统一结构：
{
  "code": 40001,
  "msgKey": "error.auth.passwordWrong",
  "args": {},
  "message": "密码错误",
  "data": null
}
```

注意：不要只用 `error.${code}`，最好也有语义 key。纯数字 key 可维护性差。

---

### 3.6 补“OpenAPI 到前端代码生成”的具体工具链

现在写的是 `openapi-ts`，不够明确。建议二选一写死。

方案 A：

```md
- 使用 `openapi-typescript` 生成类型；
- 使用 `openapi-fetch` 生成轻量请求客户端；
- 前端不手写接口类型；
- 人工封装只允许写业务 hooks/composables，不复制 DTO。
```

方案 B：

```md
- 使用 `orval` 从 `openapi.json` 生成 TS client + types；
- 每个前端仓提供 `npm run gen:api`。
```

更建议 **openapi-typescript + openapi-fetch**，依赖更轻，和“少依赖”理念一致。

同时要补一个规则：

> 后端 CI 生成的 `openapi.json` 必须作为发布产物归档；前端仓固定拉取某个后端 tag 的 openapi.json，不能直接追 main。

否则前端容易和后端 main 分支漂移。

---

### 3.7 补“可重写能力的测试契约”

§8 里已经写了测试要覆盖 Replace、继承覆写、DisabledModules，这个很好。建议再具体一点，直接写验收测试名：

```text
- ReplaceService_ShouldUseUserImplementation
- OverrideAuthStep_ShouldAffectLoginFlow
- DisabledModule_ShouldRemoveBuiltInController
- CustomController_ShouldOwnSameRouteAfterModuleDisabled
- CustomSeedData_ShouldRunOnceAndBeIdempotent
- DataScope_ShouldFilterByCurrentUserOrg
```

这是产品承诺，必须有测试锁住。

---

### 3.8 补“用户自定义模块扫描机制”

§5.7 写“用户项目什么都不用注册，框架启动时扫描到上述类型即接管”。但前面 §2.2 又写“DI 自动扫描 → 显式 TryAdd 注册”。这两者冲突。

建议设计成：

- 框架内部服务：显式 `TryAdd` 注册，不靠扫描。
- 用户应用扩展：默认扫描入口程序集和其引用程序集里的 `ITransient` / `IScoped` / `ISingleton` / `ISeedData`。
- 用户可以关闭扫描或手动指定程序集。

示例：

```csharp
builder.Services.AddTenonAdmin(builder.Configuration, options =>
{
    options.ScanApplicationAssemblies = true;
    options.ApplicationAssemblies.Add(typeof(DeviceService).Assembly);
});
```

默认三行启动时，扫描 `Assembly.GetEntryAssembly()` 及直接引用程序集即可。

---

### 3.9 补“权限码生成策略”，不要只靠手写路由字符串

你现在设计“权限码 = 路由地址本身”，这个方向可以，但会带来一个问题：前端 `v-auth="'/biz/device/add'"` 还是硬编码字符串。

建议补：

```md
- 后端权限码以规范化路由为准：`METHOD:/api/v1/biz/device/add` 或 `/api/v1/biz/device/add`。
- CI 从 OpenAPI / 菜单种子生成前端权限常量。
- 前端页面优先使用生成常量：`v-auth="Permission.BizDeviceAdd"`。
- 菜单管理里仍展示实际路由权限码，便于排错。
```

另外建议考虑是否要包含 HTTP Method。  
如果不包含，会出现：

- `GET /biz/device`
- `POST /biz/device`

同路径不同操作的权限难区分。

如果你的路由一直是 `/add`、`/edit`、`/delete`，不含 Method 也能用，但要作为规范写死。

---

### 3.10 补“包发布与版本策略”

现在写了 `.NET 主版本跟随包主版本`，但还要写 NuGet 包之间版本一致规则。

建议：

```md
- 所有 TenonAdmin.* 包同版本发布，不做独立版本号。
- 后端仓使用 Directory.Packages.props / Directory.Build.props 统一版本。
- TFM 跟随当前 .NET LTS；包版本独立语义化。
- breaking change 只在主版本升级发生。
- 可选包必须声明兼容的核心包版本范围。
```

如果你要做 `TenonAdmin 1.0.0` 而 TFM 是 `net10.0`，那“包主版本跟随 .NET 主版本”就冲突。要么：

- 包版本 `1.0.0`，TFM `net10.0`；
- 要么包版本 `10.0.0`。

建议第一版开源项目用 `1.0.0`，不要一开始就 `10.0.0`，否则用户会误会已经迭代了十个大版本。  
`.NET 主版本跟随包主版本` 这条建议删掉或改成：

> TFM 跟随当前 .NET LTS；包版本独立语义化。

---

## 4. 文档里必须修正的冲突 / 旧残留

### 4.1 示例还在用 Mapster 的 `Adapt`

文档 §5.7：

```csharp
await _repo.InsertAsync(input.Adapt<Device>());
```

但决策记录里已经说用 Mapperly，不用 Mapster。

建议改成：

```csharp
await _repo.InsertAsync(DeviceMapper.ToEntity(input));
```

或者：

```csharp
var entity = input.ToEntity(); // Mapperly source generator 生成
await _repo.InsertAsync(entity);
```

同时补一个 Mapperly 示例：

```csharp
[Mapper]
public static partial class DeviceMapper
{
    public static partial Device ToEntity(DeviceAddInput input);
}
```

---

### 4.2 “显式 DI 注册”和“用户模块自动扫描”冲突

§2.2 替代表写：

> DI 自动扫描 → 各模块扩展方法内显式 `TryAdd` 注册

§5.7 又写：

> 继承标记接口即被自动注册，无需手动 DI

建议改成双层模型：

- 框架内置模块显式注册；
- 用户外部模块可选扫描，默认扫描入口程序集。

否则实现时会左右摇摆。

---

### 4.3 i18n 章节引用写错

§6 里写：

> 错误码与文案见 §12 i18n/错误码目录

但 i18n 实际是 §13。  
§7.4 里也写：

> locales # i18n 资源(见 §12 i18n 设计)

都要改成 §13。

---

### 4.4 API 路由前缀冲突

§3.2 配置：

```jsonc
"RoutePrefix": "api"
```

§12 又写：

```text
/api/v1/...
```

建议统一成：

```jsonc
"Api": {
  "RoutePrefix": "api",
  "Version": "v1"
}
```

实际路由拼成：

```text
/api/v1/...
```

或者更简单：

```jsonc
"RoutePrefix": "api/v1"
```

不要两处说法不一致。

---

### 4.5 MQTT 的 v1 / v1.x 冲突

§2、§4、§12、§9 对 MQTT 的归属不一致。建议统一：

- v1.0：Polling only。
- v1.x：`TenonAdmin.Mqtt`。

---

### 4.6 Dockerfile 里的 `curl` 可能不可用

§11 Dockerfile：

```dockerfile
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1
```

`mcr.microsoft.com/dotnet/aspnet` 镜像不一定带 `curl`。这会导致容器健康检查直接失败。

建议三选一：

1. 安装 curl，但会增加镜像体积；
2. 去掉 Dockerfile 里的 `HEALTHCHECK`，交给 compose / k8s 配置；
3. 做一个极小自包含 healthcheck 工具，不推荐。

建议 v1 模板里不要强塞 `HEALTHCHECK CMD curl`，改成文档给 compose/k8s 示例。

---

### 4.7 `TenonAdmin.Core` 说“纯 BCL”，但 Mapperly 是第三方

你写的是：

> Core 零依赖，Mapperly 仅编译期，不进运行时

这个可以，但要措辞精确：

- Runtime dependency：无；
- PrivateAssets compile-time dependency：允许 Mapperly。

例如：

```xml
<PackageReference Include="Riok.Mapperly" Version="..." PrivateAssets="all" OutputItemType="Analyzer" />
```

并在文档里写清楚“零运行时依赖，不代表零开发期/analyzer 依赖”。

---

### 4.8 `HybridCache` 的定位要谨慎

你写默认缓存是 `HybridCache`。如果目标是最小核心，建议考虑默认先用你自己的 `ICacheProvider` + `IMemoryCache` 实现，Redis 可选包再换。

因为 HybridCache 概念是“混合缓存”，默认只做进程内缓存时，名字会让用户以为有二级缓存。

可以这样写：

- 对外接口：`ICacheProvider`
- 默认实现：`MemoryCacheProvider`
- Redis 可选包：`RedisCacheProvider`
- 如果要用 Microsoft HybridCache，放到内部实现细节，不在用户文档里作为核心概念强调。

---

## 5. 依赖处置表建议更新

当前后端 `.csproj` 里的真实依赖大概是这些：

- `SqlSugarCore`
- `SimpleRedis`
- `Lazy.Captcha.Core`
- `Minio`
- `MoYu.Extras.Authentication.JwtBearer`
- `MoYu.Extras.ObjectMapper.Mapster`
- `MoYu.Pure`
- `NewLife.Core`
- `Portable.BouncyCastle`
- `SimpleTool`
- `System.Drawing.Common`
- `Yitter.IdGenerator`
- `Masuit.Tools.Core`
- `SharpZipLib`
- `SimpleMQTT`
- `IP2Region.Net`
- `UAParser`
- `Magicodes.IE.Excel`
- `Microsoft.Extensions.Hosting`
- `Microsoft.Extensions.Hosting.WindowsServices`

你的 §2.3 基本覆盖了，但建议把处置结论更果断一些：

| 依赖 | 建议处置 | 原因 |
|---|---|---|
| `SqlSugarCore` | 核心保留 | 唯一 ORM，项目特色之一 |
| `MoYu.Pure` | 移除 | 新框架应基于 ASP.NET Core 标准管道 |
| `MoYu.Extras.Authentication.JwtBearer` | 移除 | 用 Microsoft JWT |
| `MoYu.Extras.ObjectMapper.Mapster` / `Mapster` | 移除 | 已决定 Mapperly |
| `Lazy.Captcha.Core` | 移除 | SVG captcha 自写 |
| `System.Drawing.Common` | 移除 | 跨平台风险 |
| `Yitter.IdGenerator` | 移除 | 自写雪花或 GUID v7 |
| `SharpZipLib` | 移除 | BCL compression |
| `NewLife.Core` | 移除 | 用到再按需自写 |
| `Masuit.Tools.Core` | 移除 | 工具包太大，容易拖依赖 |
| `SimpleTool` | 拷必要源码 | 你的自有工具可内化 |
| `Portable.BouncyCastle` | v1.x 可选包 | 国密/特殊加密不进核心 |
| `Minio` | v1.x 可选包 | 本地上传先跑通 |
| `Magicodes.IE.Excel` | v1.x 可选包 | 导入导出后置 |
| `IP2Region.Net` | 建议 v1.x 可选包 | 登录日志可以先只存 IP/UA 原文 |
| `UAParser` | 建议移除或极简解析 | v1 不必精准解析设备 |
| `SimpleRedis` | v1.0 可选或 v1.1 | 默认内存可跑；Redis 增强分布式 |
| `SimpleMQTT` | v1.x 可选包 | 不进 v1 |
| `Microsoft.Extensions.Hosting.WindowsServices` | 后移/独立 Worker 模板 | 管理系统后端 v1 不需要 Windows Service |

特别建议：**IP2Region 和 UAParser 都不要进 v1 核心。**

登录日志先记录：

- IP；
- UserAgent 原文；
- 登录时间；
- 登录结果；
- 用户 ID；
- 失败原因。

地理位置和浏览器解析可以后置，否则又多两个依赖/资源文件。

---

## 6. v1.0 推荐范围

建议把 v1.0 收敛成下面这样。

### 6.1 v1.0 后端必须有

1. 三行启动：
   - `AddTenonAdmin`
   - `MapTenonAdmin`
   - SQLite 默认库
   - 首启种子

2. 认证：
   - 登录；
   - 刷新 token；
   - 登出；
   - 强退；
   - 登录锁定；
   - SVG 验证码。

3. RBAC：
   - 用户；
   - 角色；
   - 菜单；
   - 按钮权限；
   - 角色授权。

4. 数据权限：
   - 全部；
   - 本机构；
   - 本机构及以下；
   - 仅本人；
   - 自定义机构。

5. 基础系统：
   - 组织；
   - 职位；
   - 字典；
   - 系统配置；
   - 登录日志；
   - 操作日志；
   - 普通本地上传；
   - 个人中心。

6. 可重写性：
   - 服务替换；
   - 继承覆写；
   - 禁用模块；
   - 自定义模块扫描；
   - 自定义种子。

7. OpenAPI：
   - 生成 `openapi.json`；
   - 前端按此生成类型。

8. 测试：
   - 登录流程；
   - 数据权限；
   - 覆写机制三件套；
   - SQLite + MySQL。

---

### 6.2 v1.0 前端必须有

只做 Vue Naive UI：

- 登录页；
- 主布局；
- 动态菜单；
- 路由守卫；
- 用户管理；
- 机构管理；
- 职位管理；
- 角色管理；
- 菜单管理；
- 字典管理；
- 系统配置；
- 登录日志；
- 操作日志；
- 在线用户；
- 个人中心；
- 暗黑模式；
- i18n zh-CN / en-US；
- `v-auth`；
- OpenAPI 生成 API 类型。

不要做：

- React；
- Soybean 二皮肤；
- 多布局全套；
- 复杂 dashboard；
- 代码生成页面；
- Excel；
- MQTT 消息中心。

---

## 7. 建议补一个“优先级表”

文档现在里程碑是按模块写的，但缺优先级。建议加一个 P0/P1/P2。

### 7.1 P0：没有它项目不能成立

- 三行启动；
- 默认 SQLite；
- 默认超管；
- 登录；
- RBAC；
- 数据权限；
- Vue 登录到菜单闭环；
- OpenAPI；
- 核心测试；
- NuGet 打包。

### 7.2 P1：v1.0 应该有

- 字典；
- 系统配置；
- 操作日志；
- 登录日志；
- 本地上传；
- Docker demo；
- i18n；
- 限流；
- 健康检查。

### 7.3 P2：后置

- React；
- MQTT；
- Excel；
- Minio；
- 国密；
- 代码生成；
- 任务调度；
- OpenTelemetry；
- Soybean 皮肤；
- IP 地理库；
- UA 精准解析。

---

## 8. 建议新增“最小验收闭环”

这个可以放在 §3 后面。不要只写模块，要写验收场景。

```md
## v1.0 最小验收闭环

1. 新建空 ASP.NET Core 项目；
2. 安装 `TenonAdmin` NuGet 包；
3. Program.cs 写三行；
4. `dotnet run`；
5. 控制台打印首次超管账号/密码；
6. 打开 Vue 前端；
7. 登录成功；
8. 进入用户管理；
9. 新增机构、角色、用户；
10. 给角色授权菜单和数据范围；
11. 用新用户登录；
12. 验证菜单权限和数据权限生效；
13. 修改系统配置/字典；
14. 查看操作日志和登录日志；
15. 强退在线用户；
16. 刷新页面后动态路由仍正常。
```

这比单纯模块列表更能指导开发。

---

## 9. 建议新增“内置表模型清单”

现在文档没有列核心表。重构前最好列一下，哪怕只是草案：

- `sys_user`
- `sys_role`
- `sys_menu`
- `sys_org`
- `sys_position`
- `sys_user_role`
- `sys_role_menu`
- `sys_role_data_scope`
- `sys_dict_type`
- `sys_dict_item`
- `sys_config`
- `sys_login_log`
- `sys_operation_log`
- `sys_file`
- `sys_session`
- `sys_refresh_token`
- `sys_seed_history` 或 `sys_schema_version`

为什么要列？因为你决定“不支持旧版迁移”，那新表结构就是第一版的长期契约，要早一点稳定。

---

## 10. 建议的文档结构调整

现在文档内容很全，但顺序可以更产品化一点。我建议改成：

```md
# TenonAdmin 重构设计方案

## 0. 产品定位
## 1. v1.0 目标与非目标
## 2. 核心原则
## 3. 用户侧最小体验
## 4. 仓库与包结构
## 5. 依赖处置策略
## 6. 后端核心设计
   - 启动模型
   - Options
   - 数据访问
   - 认证会话
   - RBAC
   - 数据权限
   - 缓存
   - 日志
   - 上传
## 7. 可扩展 / 可重写设计
## 8. 前端设计
## 9. OpenAPI 契约单源
## 10. i18n
## 11. 安全基线
## 12. Docker / 发布
## 13. 测试与验收
## 14. 里程碑
## 15. 风险与开放问题
```

你现在 §12、§13 有点像后补内容，建议吸收到主结构里。

---

## 11. 最关键的 10 条修改建议

如果只改最关键的，建议按这个顺序：

1. **v1.0 移除 React 版**，改成 v1.x。
2. **v1.0 移除 MQTT**，只保留 HTTP 轮询。
3. **v1.0 移除 Soybean 二皮肤**，只保留 Naive UI。
4. **分片上传改成可选/stretch goal**，普通本地上传先跑通。
5. **修正 Mapperly 示例**，删除 `Adapt`。
6. **修正 DI 扫描策略冲突**：框架显式注册，用户模块默认扫描。
7. **补认证会话/RefreshToken/强退模型**。
8. **补安全基线**。
9. **补 v1.0 非目标**。
10. **统一 API 路由版本、i18n 章节引用、MQTT 归属。**

---

## 12. 建议最终 v1.0 口号

现在一句话目标已经不错，但可以更开源传播化一点：

> TenonAdmin 是一个基于 ASP.NET Core + SqlSugar 的轻量企业管理系统内核：安装一个 NuGet 包，三行代码启动，默认提供登录、RBAC、多机构数据权限和 Vue 管理端模板；核心运行时除 SqlSugar 外不绑定第三方框架，所有关键能力均可替换或覆写。

这个比“开源企业级小型管理系统”更准确。

---

## 13. 最终结论

**这份计划可以作为新项目起点，但必须收 MVP。**

建议：

- **后端作为主产品**：NuGet 一键集成是核心卖点。
- **Vue Naive UI 是第一官方模板**：做完整、做漂亮、做稳定。
- **React / Soybean / MQTT / Excel / Minio / 代码生成全部后置**。
- **第一版一定要把可重写机制和数据权限做扎实**，这是和其他 admin 模板拉开差距的地方。
- **文档要补“非目标、安全基线、会话模型、OpenAPI 工具链、数据库策略、错误码规范”**，这些会直接影响后续实现质量。
- **马上修正文档里的几处冲突**，尤其是 Mapster 残留、DI 扫描冲突、MQTT v1/v1.x、`/api` vs `/api/v1`、i18n 章节引用。

评分：

| 项目 | 评分 |
|---|---:|
| 方向 | 8.5 / 10 |
| 范围控制 | 6 / 10 |
| 落地可执行性 | 7 / 10 |

把 v1.0 范围砍掉 30%-40% 后，可执行性会到 **8.5 / 10**。
