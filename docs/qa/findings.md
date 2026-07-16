# SimpleAdmin QA 台账

范围：`web/` + `api/`。不涉及 `uniapp/`。

## 缺陷表

| ID | 轮次 | 模块 | 页面/文件 | 严重度 | 现象 | 根因 | 状态 | 备注 |
|----|------|------|-----------|--------|------|------|------|------|
| A01 | 1 | 鉴权 | api/SimpleAdmin.Core/Utils/Cryptogram/CryptogramUtil.cs:30-40 | 低 | Sm2Decrypt 内 try/catch 块体是被注释掉的死代码，catch 永不触发 | 复制粘贴残留 | 已修复 | 已折叠为单一 return，本轮修复 |
| A02 | 1 | 鉴权 | api/SimpleAdmin.System/Services/Auth/Auth/AuthService.cs:50-68 | 中 | GetPhoneValidCode 随机6位验证码被 DateTime.Now.ToString("yyMMdd") 覆盖，全天验证码相同 | 短信未接入的开发占位 | 待你确认 | 疑似有意为之的占位，移除会破坏 dev 手机登录 |
| A03 | 1 | 鉴权 | api/SimpleAdmin.Core/Utils/Cryptogram/SM4Util.cs:20-26；Core.Development.json:5-8 | 高 | 用户密码以 SM4(可逆对称)加密存库而非哈希，SM4 密钥/IV 硬编码在源码、SM2 密钥硬编码在配置 | 设计选择 | 待你确认 | 改为加盐哈希会动 SysUser 实体+登录+改密+SeedData，跨模块 |
| A04 | 1 | 鉴权 | api/SimpleAdmin.Web.Core/Controllers/Dev/BatchEditController.cs:16-121 | 待定 | 类级与方法级均无 [SuperAdmin]/[RolePermission]，Add/Config/Delete/Sync 可对任意配置表批量改列，仅靠全局 JWT | 待运行态验证 | 待确认 | 需服务起来后验证非超管能否越权调用；对比同层 Controller 多数标了 [SuperAdmin] |
| A05 | 1 | 鉴权 | api/SimpleAdmin.System/EventSubscriber/AuthEventSubscriber.cs:157-168 | 低 | GetLoginAddress 空 catch 吞异常仅返回"未知"，未记录 | IP 定位失败的兜底 | 待确认 | 可接受的降级，暂不改 |
| A06 | 1 | 鉴权 | web/src/views/login/components/pwd-login/index.vue:117 | 低 | 登录失败时无条件请求 getPicCaptcha，但验证码 UI 被 v-if=captchaOpen 门控，策略关闭时恒不渲染，请求作废 | catch 回调未按 captchaOpen 门控 | 已修复 | 加 if(captchaOpen.value) 门控；npm type:check+lint 均 0 error；phone-login 验证码常显无此问题 |
| A07 | 1 | 鉴权 | web/src/views/login/index.vue:82-105；stores/modules/config.ts:54-89 | 低 | 登录页加载时 GET tenantList 发2次、GET sysInfo 发3次 | updateConfig 同时挂在 getSysBaseInfo+setSysBaseInfo 两条链各调一次；config store 缓存判断 SYS_NAME!="" 有并发竞态；Footer 也调 getSysBaseInfo | 待你确认 | 干净修法(store 加 in-flight 去重)会动公共 config store，跨 login/footer/dynamicRouter 三处，不擅动 |
| A08 | 1 | 鉴权 | node_modules/element-plus (debugWarn) | 低 | 空账号空密码提交时控制台出现 warn [object Object] | el-form 的 :rules deep-watch(validateOnRuleChange 默认true)触发库内 debugWarn(err)，非项目代码 | 已驳回 | element-plus 库内噪音，不为此改业务代码 |
| A09 | 2 | 菜单/路由 | api SeedData/Json/seed_sys_resource.json:1246-1254；web/src/views/sys/ops/(无monitor) | 中 | 「系统运维→会话管理」进入后主区白屏，2/2稳定复现，控制台 Vue Router warn 路由缺组件，页面零请求 | 菜单 Component 指向 sys/ops/monitor/index，但 web/src/views/sys/ops 下无 monitor 视图文件，动态路由绑不到组件 | 待你确认 | 修法=新建该会话管理页(功能开发)或从SeedData删该菜单(改种子数据)，均超"只修bug"范围 |
| A10 | 2 | 菜单/路由 | api MenuController.cs:99;MobileMenuController.cs:51,133;MenuService.cs:68-70;UserCenterService.cs:242-245,294;SpaService.cs:107 | 低 | 死代码：3处return后孤立分号空语句 + 4处注释掉的死代码块 | 复制粘贴/遗留 | 已修复 | 已删7处；三改动项目单独dotnet build 均0错误0警告；全量build仅DLL文件锁(运行中进程)非编译错误 |
| A11 | 2 | 菜单/路由 | web 前端(仅dev模式) | 低 | 首次访问未访问过的页面时整屏闪白、需点两次才进 | UnoCSS按需生成CSS触发Vite HMR热重载导致应用重挂载，仅开发模式 | 已驳回 | 生产 build:pro 无HMR不存在此现象 |
| A12 | 3 | 权限/越权 | api Web.Core/Controllers/System/BatchEdit/BatchEditController.cs:18；JwtHandler.cs:109-141 | 高(P0) | 非超管用户 qauser 可调 BatchEdit 全部8端点含写(delete/sync/config 均200进业务)，/tables 可读 sys_user 表结构含Password列 | 类级漏标[SuperAdmin]，JwtHandler对无[SuperAdmin]/[RolePermission]标注端点直接return true；Columns上孤立的[IgnoreSuperAdmin]佐证类级本应有[SuperAdmin] | 已修复(运行态已验证) | 第4轮销账：新构建下 qauser 调 /sys/batch/tables 和 /sys/batch/config 均 code:403 被拒，超管 200 正常放行；columns 非超管可调是设计(显式[IgnoreSuperAdmin])。★挂账清除 |
| A13 | 3 | 角色/岗位 | web sys/limit/role/index.vue:108；sys/organization/position/index.vue:76 | 中 | 角色名称搜索无效，返回全部未过滤且所搜项不在首页 | 前端搜索列 prop:"name" 发 name 参数，后端 RolePageInput 只有继承的 SearchKey，模型绑定丢弃 name → 过滤跳过 | 已修复 | 加 search.key:"searchKey" 对齐全局约定；type:check+lint 0错误；岗位页同bug一并修 |
| A14 | 3 | 用户 | api SysUserService.cs:816 | 低 | Template() 内一行注释掉的旧实现死代码 | 遗留 | 已修复 | 已删；System build 0错误0警告 |
| A15 | 3 | 用户 | api SysUserService.cs:744 | 低 | 删除用户处 TODO"踢下线/永久注销未实现"，仅清token缓存哈希项 | 功能未实现 | 待确认 | 清token缓存可能已足够使其失效，需运行态验证删掉用户的现有token是否真失效 |
| A16 | 3 | 角色 | api SysRoleService.cs:492-538 Delete | 中 | 删角色不校验"仍被用户持有"直接删，静默删除用户-角色关系；而机构/岗位删除前都拦截引用 | 三者删除校验不一致 | 待你确认 | 是有意级联清理还是应像机构/岗位拦截，行为改动请你定 |
| A17 | 3 | 鉴权 | api AuthService.cs:211 BeforeLogin | 中 | 已坐实：不带 TenantId/缺 Origin 头登录时 AuthService.cs:211 抛 IndexOutOfRangeException → 500(日志 Web.Entry/logs/Error/2026-07-10.log:4-17)。第4轮 p0-verify 登录时意外复现 | origin.Split("//")[1] 数组越界无防护 | 待你确认 | 已有确切行号+运行态证据。修法：Split 前判空/判长度,或用 Uri.TryCreate。小改但涉登录链路,请你确认后修 |
| A18 | 3 | 用户 | web sys/organization/user 新增表单 | 低 | 新增用户账号字段无长度上限，200+超长账号原样入库(name含<script>渲染时Vue转义安全非XSS) | 缺前后端长度校验 | 待确认 | 低危;加maxlength属小改但非明确bug |
| A19 | 3 | 用户 | web sys/organization/user/index.vue 1280宽 | 低 | 1280宽下用户表格右侧列被推出需横向滚动 | 列多宽度不足 | 待确认 | 非破坏性,窄屏体验,暂不改 |
| A20 | 4 | 角色/岗位 | web sys/limit/role/index.vue:115；sys/organization/position/index.vue:83；biz 下同名两页 | 中 | 「角色编码/岗位编码」搜索框静默空操作：填了点搜索无反应，返回全部数据 | 前端发 `code` 参数，但 RolePageInput/PositionPageInput 全继承链无 `Code` 字段，模型绑定丢弃；后端只有 `.WhereIF(SearchKey→Name.Contains)` | 待你确认 | 不能照 A13 映射成 searchKey(那会变成按名称搜)。修法二选一：后端加 Code 字段+WhereIF(倾向) / 删掉该搜索框。属行为改动 |
| A21 | 4 | 角色 | api Services/Limit/Role/Dto/RoleInput.cs:16 | 低 | `RolePageInput : PositionPageInput` 空类，角色分页入参继承岗位分页入参，白捡 OrgId/OrgIds/Category/Status 等无关字段 | 语义错误的继承，图省事复用 | 待你确认 | 改继承会牵动模型绑定与既有调用方,不擅动 |
| A22 | 4 | 权限/越权 | api Web.Core/Controllers/System/Dev/MessageController.cs:10；System/Services/Dev/Message/MessageService.cs:78-80,130-135,189-208 | 高(P0) | 运行态坐实：qauser(非超管)可 GET message/page 拿全表消息(含发给超管/他人的)、可 POST message/add ReceiverType=ALL 全站广播成功(createUser=qauser)。Edit/Delete 无所有权校验 | 类级漏标[SuperAdmin]，JwtHandler.cs:140 对无权限特性端点直接 return true(与 A12 同源) | 已修复 | 类级加[SuperAdmin]，commit b2881c3。修法安全已确证：UserCenter 有独立"我的消息"接口(MyMessagePage 按 UserId 过滤)，普通用户不依赖 sys/dev/message/*。★运行态复验待第5轮(API 已按新构建重启) |
| A23 | 4 | 权限/越权 | api Web.Core/Controllers/System/Dev/FileController.cs;System/Services/Dev/File/FileService.cs:26-34,54-58,100-112 | 高(P0) | 运行态坐实：qauser(非超管)可 GET file/page 拿全系统13个文件(含StoragePath)。Delete/Download 按 ID 直操作无所有权校验(按红线未实测删,静态确认) | 同 A22，JwtHandler.cs:140 | 已修复 | 类级加[SuperAdmin]，commit b2881c3。修法安全已确证：download 仅管理页 sys/dev/file/index.vue:88 调用,uniapp 完全不调 file 端点。★运行态复验待第5轮 |
| A24 | 4 | 鉴权 | api AuthService.cs:116-132,447-468 LoginOut | — | 静态证据推翻假设 | RemoveTokenFromRedis 作用域始终被 UserManager.UserId(调用者自身JWT claims)限定,input.Token 只在调用者自己的 token 列表内做相等匹配。传他人 token 最多自己列表找不到匹配(无效果),无法登出他人 | 已驳回 | 非漏洞,误报。LoginOut 也无[AllowAnonymous],需合法JWT才可达 |
| A25 | 4 | 机构 | api Application/Services/Organization/Org/OrgService.cs:32-40 | 中 | biz 机构管理页「分类」筛选静默失效：DTO SysOrgPageInput 有 Category 字段、前端已发 category 参数，但 biz 侧 Page 只有 ParentId/Name/Code/Status 四个 WhereIF，漏了 Category 过滤 | Service 层实现遗漏(sys 侧 SysOrgService.cs:96 有该过滤，biz 复用同 DTO 但漏抄一行) | 已修复 | 修法安全,照 sys 侧补一行 WhereIF(Category);已复核两侧源码 |
| A26 | 4 | 消息 | web sys/dev/message/index.vue:97；api Services/Dev/Message/Dto/MessageInput.cs:13-19 | 低 | 站内信管理页「状态」搜索列静默空操作：发 status 参数，MessagePageInput 只有 Category+基类字段无 Status，Service 也无 status 过滤 | DTO 缺字段(同 A20 模式) | 待你确认 | 后端加 Status 字段+WhereIF / 删该列,二选一,行为改动 |
| A27 | 4 | 菜单 | web sys/limit/menu/index.vue:101,111；sys/mobile/menu/index.vue:110-111；api MenuInput.cs:16-27 | 低 | 菜单管理页「名称 title/路径 path/类型 menuType」搜索列静默空操作：MenuTreeInput 只有 Module+SearchKey，无对应字段(Tree 接口,非分页) | DTO 缺字段(同 A20 模式) | 待你确认 | 后端加字段 / 改列参数,行为改动 |
| A28 | 4 | 模块 | web sys/limit/module/index.vue:53；sys/mobile/module/index.vue:61 | 中 | 模块管理页「名称 title」搜索列静默失效：发 title 参数，ModulePageInput 只有 SearchKey，但 Service(ModuleService.cs:38) 已有 SearchKey→Title.Contains 过滤 | 前端列缺 key:"searchKey"(与已修复 name 列完全同型) | 已修复 | 修法安全,前端加 key;后端过滤逻辑现成 |
| A29 | 4 | 代码生成 | web biz/ops/test/index.vue:49-51；api 无对应 Controller | 低 | biz/ops/test 页调 biz/ops/test/page，但仓库无该 Controller/Service,仅有实体 GenTest.cs 和代码生成模板 | 代码生成 demo 残留,后端未生成 | 待你确认 | 是删该 demo 页还是补生成后端,请你定;非静默失效类 bug |
| A30 | 4 | 消息 | api System/Services/Dev/Message/MessageService.cs Add；Entity SysMessage.ReceiverInfo | 低 | message/add 不传 ReceiverInfo 时,DB 列 NOT NULL 直接抛 500"操作失败"(超管也复现,日志 2026-07-10.log:28)。缺业务层必填校验,把 DB 约束异常暴露成 500 | 入参未校验 ReceiverInfo 必填,依赖 DB 约束兜底 | 待确认 | 健壮性 bug,非越权。应在 Add 里校验必填并返回友好提示,或给 ReceiverInfo 默认空集合。运行态实证时发现 |

| A31 | 5 | 权限/数据范围 | api Application/Services/Organization/Role/RoleService.cs:147-150(原) | 高(P0) | biz 角色管理 GrantResource 纯透传，一次 CheckApiDataScope 都没调；机构管理员可对数据范围外的角色授权，并授予自己都没有的系统管理菜单(越权提权) | 该调用的数据范围校验完全不存在；同文件 Delete(168-175)/Detail(82-85) 是"先查真实记录再校验"的正确写法，属遗漏非设计 | 已修复(仅编译验证) | 照 UserService 样板补校验，commit 9ab64e3；dotnet build Application --framework net8.0 → 0警告0错误。★运行态越权复验未做(本会话 curl/node 执行通道被安全检查封锁) |
| A32 | 5 | 权限/数据范围 | api RoleService.cs:153-156(原)；SysRoleService.cs:453-486 | 高(P0) | biz 角色管理 GrantUser 同样零校验：角色 Id 和被授权用户 Id 均来自请求体，未校验任一方在调用者数据范围内 → 可把任意机构用户绑到任意角色 | 同 A31 | 已修复(仅编译验证) | commit 9ab64e3：先校验角色，再取被授权用户真实记录走 CheckApiDataScope 列表重载。查不到的用户 Id 静默跳过(与 Delete 同风格，不影响越权面) |
| A33 | 5 | 权限/数据范围 | api RoleService.cs:140-144,186-192(原) | 高(P0) | biz 角色 Edit 的数据范围校验形同虚设：复用 Add 的 CheckInput，校验的是请求体里可篡改的 OrgId/CreateUserId，而非 input.Id 在库中真实角色的字段 | Edit 误用了 Add 的校验(Add 用请求体字段是正确的，新增时无真实记录)；对照 UserService.cs:274-280 是先按 Id 查库再校验 | 已修复(仅编译验证) | commit 9ab64e3：拆出 CheckBusinessRule 保留业务规则，Edit 改用库中真实角色校验，Add 语义不变 |
| A34 | 5 | 安全/凭证 | api Web.Core/Controllers/System/Mqtt/MqttController.cs:22；System/Services/Mqtt/MqttService.cs:20-59；SeedData/Json/seed_sys_config.json:277,294 | 高 | GET /mqtt/getParameter 无任何权限标注(JwtHandler:140 兜底放行)，直接返回 sys_config 里的 MQTT 用户名/密码，种子值为明文 admin/admin → 任意已登录用户(含最低权限账号)可拿到 broker 共享凭证 | 端点漏标 + 服务端原样下发共享凭证 | 待你确认 | 修法涉及占位符语义(见A35)与 SeedData，属行为改动。可选：改用 $token 方案下发当前 JWT 作 mqtt 密码 / 给端点加权限标注 |
| A35 | 5 | 消息/mqtt | api System/Services/Mqtt/MqttService.cs:44 | 中 | 密码占位符判断误写成 `password.ToLower() == "$username"`，注释却是"当前token作为mqtt密码"。把配置改成 $token 永远命不中，这条避免共享凭证的逃生通道从未生效 | 复制粘贴上方用户名分支时漏改字面量 | 待你确认 | A34 的成因。修法要先定占位符字面量语义($token? $password?)，一行改动但影响 mqtt 连接行为 |
| A36 | 5 | 安全/上传 | api Web.Core/Controllers/System/Upload/UploadController.cs:14-38；System/Services/Dev/File/FileService.cs:121-161 | 中 | POST /sys/upload/uploadImg 无任何权限标注 + [DisableRequestSizeLimit]，链路中无文件类型/大小校验(IsPic 仅决定是否生成缩略图，不拦截) → 任意登录用户可上传任意类型、任意大小文件落本地磁盘 | 端点漏标 + 业务层缺校验 | 待你确认 | 需先定"谁该能传、允许什么类型、多大上限"。FileController(Dev版)已加[SuperAdmin]，但这个通用上传口看起来是给业务用户用的 【轮次7 复核】证据已细化并升级为 A44，webshell/RCE 假设经查不成立（上传目录不在 wwwroot、静态映射段已注释、下载端点为 [SuperAdmin]），维持 P2 存储滥用定性。 |
| A37 | 5 | 岗位/路由 | api Web.Core/Controllers/Application/Organization/BizPositionController.cs:19 | 中 | BizPositionController 是裸类，既不继承 BaseController 也不实现 IDynamicApiController → 未被 MoYu 动态控制器扫描，biz 岗位管理整页后端路由应全部 404 | 漏写接口标记。同目录 BizOrgController.cs:19/BizRoleController.cs:19/BizUserController.cs:19 均显式 `: IDynamicApiController`；全项目搜该接口命中15个文件，本文件不在其中 | 待你确认 | 类级的[RolePermission]挂在一个根本没被扫描的类上=形同虚设。修法(加 : IDynamicApiController)一行，但会凭空启用一整套此前不存在的端点，属行为改动，请你定 |
| A38 | 5 | 角色/授权 | web sys/limit/role/components/grantResource.vue:64-65,368-375；grantPermission.vue:34-35 | — | 取证 agent 报告「授权资源」「授权权限」弹窗的确定/取消按钮点击后零请求、零 console、弹窗不关闭，仅右上角×可关 | 静态复核不支持：两个按钮均正常绑定 @click(onClose/handleSubmit)，handleSubmit 无早退分支，FormContainer 确实透传 #footer 插槽。三个观察特征(零请求+零console+×可关)符合"CDP 点击落在视口外元素"的取证工具伪影 | 已驳回 | 复测坐实为取证工具伪影：JS 求值显示两个 footer 按钮 rect.top=698、inViewport=false(弹窗高于视口，footer 被裁在可视区外)，坐标点击落空；改用 DOM .click() 程序化触发后，POST /api/sys/limit/role/grantResource 返回 200 且弹窗正常关闭(.el-dialog 计数 1→0)。按钮与 @click 绑定均正常，非产品 bug |
| A39 | 5 | 前端/公共组件 | web src/components/Form/FormContainer/index.vue:31,34,58 | 低 | FormContainer 内部自建 `const visible = ref(false)` 却从未声明 modelValue prop；父组件的 v-model 是靠 `v-bind="$attrs"`(第34行，位置在 v-model 之后覆盖了它)阴差阳错透传给 el-dialog 才生效 | 组件契约与实现不符，能跑但脆(依赖属性绑定顺序) | 待你确认 | 全站弹窗都走这个公共组件，改动面大。不改行为的前提下应显式声明 modelValue prop，属公共组件改动 |
| A40 | 6 | api-配置 | api System/Services/Ops/Config/ConfigService.cs:120-133 | 高 | `Edit` 原先直接信任请求体：`CheckInput`(:199-210) 第 209 行无条件 `sysConfig.Category = CateGoryConst.CONFIG_BIZ_DEFINE`，导致内置配置(SYS_BASE/MQTT_BASE 等)被编辑时会被静默改写成 BIZ_DEFINE——既绕过 `Delete`(:167-176) 只允许删 BIZ_DEFINE 的内置保护，又使原分类的缓存 key 不被刷新 | 校验请求体字段而非按 Id 查真实记录（invariant 层，同 A31-A33 根因家族） | 已修复 | 提交 `fecc904`。修法：按 Id 查出真实记录后校验 `config.Category != CONFIG_BIZ_DEFINE` 则拒绝。**仅编译验证**(`dotnet build SimpleAdmin.System --framework net8.0` 0 error)，运行态复验未执行 |
| A41 | 7 | web-上传 | web/src/api/modules/sys/upload/index.ts:26-28 | 低 | uploadImg 用 `http.get` 发 FormData；`instance.ts:174-176` 把它塞进 axios 的 params(query string)，`instance.ts:93-98` 拦截器又对 FormData 实例做对象展开 `{...config.params, _t}` —— FormData 无自有可枚举属性，展开结果为 `{}`，文件被静默丢弃，实发请求是 `GET uploadImg?_t=时间戳`；后端 `UploadController.cs:28` 是 `[HttpPost]`+`[Consumes(multipart/form-data)]` | 前端 get/post 错配 + FormData 不可对象展开；同类接口 `biz/document.ts:29-54` 三个都用 `http.post` 且显式设 Content-Type，唯独 uploadImg 例外 | 待你确认 | 仅静态证据(运行态未验证)。与 A42/A43 合看：该接口无任何活调用点，坏了也没人发现 |
| A42 | 7 | web-上传 | web/src/components/Upload/UploadImgs.vue | 低 | 组件全仓无任何模板标签引用（`<UploadImgs` 只匹配到自身 name 声明行 :46），无人 import 使用 | 死组件 | 待你确认 | 仅静态证据。它是 uploadImg 唯二调用点之一 |
| A43 | 7 | web-上传 | web/src/components/Upload/UploadImg.vue:106-115 | 低 | 唯二使用处 `sysBaseConfig.vue:12,22` 均传 `:auto-upload="false"`，走 106-109 行 base64 分支，永不执行 `api(formData)` | uploadImg 的另一个调用点也是死的 | 待你确认 | 仅静态证据 |
| A44 | 7 | api-上传 | api Web.Core/Controllers/System/Upload/UploadController.cs:28 | 中 | 端点无 `[SuperAdmin]`/`[RolePermission]` → 撞 `JwtHandler.cs:140` 默认放行，任意已登录用户可调；叠加 `[DisableRequestSizeLimit]` 且 FileService 全链路无大小/MIME/后缀白名单校验 → 磁盘耗尽与存储滥用。**不构成 webshell/RCE**：落盘根目录种子默认值 `C:/defaultUploadFolder`(绝对路径，不在 wwwroot 内)，全仓唯一生效静态中间件是 `Web.Core/Startup.cs:71` 裸 `app.UseStaticFiles()`(只服务 wwwroot)，带 `RequestPath="/files"`+`ServeUnknownFileTypes=true` 的那段已整体注释掉；下载走 `FileController`(类级 `[SuperAdmin]`，按 Id 查库取 StoragePath) | 漏注解=默认放行，同 A12/A22/A23 根因线；且读端点上了 `[SuperAdmin]` 锁、写端点被忘 | 待你确认 | 仅静态证据。A36 的细化版。结合 A41-A43，该端点前端零活调用点，是无人使用却敞开的孤儿写端点——**建议删端点而非给它补校验，需你拍板** |
| A45 | 7 | api-上传 | api Application/Services/Document/Common/DocumentStorageService.cs:132-137 | 中 | SaveChunk 的分片 sha256 校验整个裹在 `if (!string.IsNullOrWhiteSpace(chunkHash))` 里，chunkHash 传不传由前端决定；不传则完全跳过校验直接落盘。另 :125-131：目标分片已存在且未传 hash 时直接 `return`，静默丢弃本次内容 | 完整性校验由被校验方自己决定是否开启 | 待你确认 | 仅静态证据。属"校验靠调用方记得"的同一根因家族 |
| A46 | 7 | api-上传 | api Application/Services/Document/Common/DocumentStorageService.cs:143-155 | 中 | `tempPath = $"{targetPath}.uploading"` 是确定性路径，不含请求/线程标识；同一 session 同一 chunkIndex 并发（重试未取消旧请求、用户双击）会同时 `File.Create` 同一个 `.uploading` 文件，随后 `File.Delete(targetPath)`+`File.Move` 互踩 | 无并发锁 + 共享确定性临时文件名 | 待确认 | **需运行态复现，当前 API 起不来(Redis 连接超时)，仅静态推演** |
| A47 | 7 | api-启动 | api Web.Core/Components/WebSettingsComponent.cs:36-40 | 低 | `WebSettingsApplicationComponent.Load` 方法体为空，而 `Web.Core/Startup.cs:54` 仍在 `app.UseComponent<WebSettingsApplicationComponent>(env)` 调用它 | 死代码 | 待你确认 | 仅静态证据 |
| A48 | 7 | api-上传 | api Application/Services/Document/DocumentService.cs:310-315；DocumentStorageService.cs:201 | — | 【本轮主控假设，经查不成立】疑分片合并存在路径穿越：`NormalizeRelativePath`(`DocumentService.cs:730-733`) 只做 `Replace('\\','/')`+`Trim('/')`，不过滤 `..` 段 | 但 `DocumentService.cs:313` 在赋给 `session.FileName` 前经过 `Path.GetFileName()`，已剥离所有目录分隔符，故 `mergedPath = Path.Combine(mergedDir, 纯文件名)`；落盘侧 `BuildLocalStoragePath` 也只取 `Path.GetExtension()`。Windows/Linux 两种分隔符语义下均不成立 | 已驳回 | 静态证据充分，**勿在后续轮次重复调查** |
| A49 | 6 | api-字典 | api System/Services/Ops/Dict/DictService.cs 的 `Edit` | 中 | `Edit` 缺少内置分类(FRM)守卫，与 A40 修复前的 ConfigService.Edit 是同一形状：信任请求体分类而非按 Id 查真实记录 | 同 A40 根因(invariant 层) | 待你确认 | 仅静态证据，行号待补。A40 已按"查真实记录再校验"修好，此处可照搬同一模式 |
| A50 | 6 | api-配置 | api Web.Core/Controllers/System/Ops/ConfigController.cs 的 `List()` | 中 | `List()` 返回配置时未对敏感值做脱敏，MQTT 密码(`MQTT_PARAM_PASSWORD`)以明文出现在响应中 | 无出参脱敏 | 待你确认 | 仅静态证据，行号待补。关联 A34/A35(MqttService 密码占位符分支) |
| A51 | 6 | api-公共 | api Web.Core/Controllers/System/CommonController.cs 的 `SysInfo()` | 中 | 匿名可访问，返回全部 SYS_BASE 配置，仅靠一份**硬编码的 4 个 key 排除清单**做过滤——新增任何敏感 SYS_BASE 配置项都会默认泄露 | 排除清单是黑名单而非白名单（"默认放行"的第四种形态：接口层漏注解 / 数据层忘调用 / 不变量层信请求体 / 出参层黑名单） | 待你确认 | 仅静态证据，行号待补 |
| A52 | 8 | api-日志 | api Web.Core/Logging/DatabaseLoggingWriter.cs:64,136-153；System/Services/Auth/Auth/Dto/AuthInput.cs:36-62 | 中 | 登录**失败**时整个 `LoginInput`(含 `Password`)被序列化存入 `sys_log_operate_*.ParamJson`：`AuthService.cs:89` 密码不符 → `throw Oops.Bah` → writer:64 走异常分支 → `CreateOperationLog` → `ParamJson = Parameters[0].Value.ToJsonString()`，无字段级过滤。存的是 SM2 密文非明文，但服务端登录接口收的就是该密文(`Sm2Decrypt` 后比对)且无 nonce/时间戳 → **该密文可原样重放登录**。前端 `oplog/components/detail.vue:73-81` 原样 `JSON.stringify` 渲染 | 全仓搜 `脱敏\|mask\|desensit\|JsonIgnore` 零命中，无任何脱敏机制；`LoginInput.Password` 无序列化排除特性 | 待你确认 | 仅静态证据。登录**成功**路径安全(走 `CreateVisitLog`，`SysLogVisit` 无 ParamJson 字段)。触发不需攻击者：密码正确但租户/验证码错，同样抛异常、同样落库。读取面受 `[SuperAdmin]` 限制故定中；但与 A03(SM2 私钥硬编码在配置)叠加=拿库+拿配置即得明文 |
| A53 | 8 | api-日志 | api System/Services/LogAudit/OperateLog/OperateLogService.cs:31,107-111；VisitLog/VisitLogService.cs | 中 | "清空日志"清不干净：`Delete` 用 `.SplitTable(tabs => tabs.Take(_maxTabs))`，`_maxTabs = 100`(:31)。操作日志按 `{year}{month}{day}` 按天分表 → 100 张表 ≈ 100 天。超过 100 天的分表 `Page`/`Detail`/`Delete` 一律够不着：查不到、删不掉、永久滞留磁盘。用户以为清空了，实际只清了近 100 天 | `Take(_maxTabs)` 同时作用于查询与删除，成了数据可达性的硬天花板 | 待你确认 | 仅静态证据。且全仓(含 SimpleAdmin.Background)无任何定时清理/TTL/归档代码，唯一入口就是这个清不干净的按钮 |
| A54 | 8 | api-日志 | api System/Services/LogAudit/OperateLog/OperateLogService.cs:34-44；VisitLogService.cs:24-33；Entity/SysLogVisit.cs;SysLogOperate.cs | 中 | 日志分页无时间范围约束(`OperateLogPageInput`/`VisitLogPageInput` 全继承链只有 `Category`+`Account`+基类，无时间字段)；`SearchKey` 过滤走 `it.Name.Contains(k) \|\| it.OpIp.Contains(k)` 即 SQL `LIKE '%k%'`(前后模糊，索引不可用)，且对最多 100 张分表逐张执行；两个日志实体的 `[SugarColumn]` 参数只有 ColumnName/Description/Length/DataType/IsNullable，**无任何索引标注** | 无时间范围强制 + 前后模糊 LIKE + 无索引 + 跨百表 | 待你确认 | 仅静态证据，未实测。日志表是全站增长最快的表，这三者叠加 |
| A55 | 8 | api-日志 | api Web.Core/Controllers/System/LogAudit/LogOperateController.cs:64-68；LogVisitController.cs:64-68 | 中 | 两个"清空日志"端点均无 `[DisplayName]`，而 `DatabaseLoggingWriter.cs:46,70` 要求 `DisplayTitle != null` 且 `method=="POST"` 才记操作日志 → **清空日志这个动作本身不留任何痕迹**。超管清空后，日志没了，"谁清的/何时清的"也没了 | 漏标 `[DisplayName]` | 待你确认 | 仅静态证据。与 A53 叠加：该按钮既清不干净、清的动作又不留痕 |
| A56 | 8 | api-鉴权 | api Web.Core/Controllers/System/Auth/AuthBController.cs:69 | 中 | `LoginByPhone` 无 `[DisplayName]` → 手机验证码登录**整条路径零日志**(访问日志、操作日志都没有)。同 Controller 内 `Login`(:56)/`LoginOut`(:79) 均有 `[DisplayName(EventSubscriberConst.LOGIN_B/LOGIN_OUT_B)]`，唯独它没有 | 漏标 `[DisplayName]` | 待你确认 | 仅静态证据。密码登录可审计、手机登录不可审计，两条登录路径审计能力不对等。注：即便补标，也需用 `LOGIN_B` 常量才会进访问日志(writer:54 按常量值分支)，用字面量会落到操作日志 |
| A57 | 8 | api-代码生成 | api Web.Core/Controllers/System/Gen/GenBasicController.cs:113-118 | 低 | `ExecGenZip` 挂了 `[DisplayName("执行代码生成(压缩包)")]` 但方法是 `[HttpGet]`，而 `DatabaseLoggingWriter.cs:70` 判据是 `!operation.Contains("/") && method == "POST"` → **标了也不记录**。同 Controller 同功能的 `ExecGenPro`(:101，`[HttpPost]`)正常记录 | writer 以 HTTP 动词而非操作语义判定是否审计 | 待你确认 | 仅静态证据。同一操作走压缩包不留痕、走本地留痕 |
| A58 | 8 | api-日志 | api Web.Core/Logging/DatabaseLoggingWriter.cs:46,70 | — | 【本轮主控假设，经普查证伪】疑"审计覆盖依赖每个方法记得加 `[DisplayName]`"构成"默认放行"的第五种形态(审计层) | 全量普查不支持：`Controllers/` 下约 100 个 POST 方法中仅 8 个缺 `[DisplayName]`，覆盖率约 92%；且存在显式退出机制 `[SuppressMonitor]`(`UserController.cs:107` 实际使用)，说明作者知道如何正经排除方法。8 处缺失中 3 处有实质影响(已单列 A55/A56)、3 处为个人偏好类小操作(UserCenter 的 setDefaultModule/setRead/setDelete)、1 处显式 `[SuppressMonitor]`、1 处 MQTT 匿名钩子。属个别遗漏，非系统性形态 | 已驳回 | 主控预判模式、数据不支持，如实驳回。**勿在后续轮次重复该假设** |

## 已覆盖区域

- 鉴权后端链路（AuthController/AuthService/JwtHandler/事件订阅）— 已静态审查
- 登录/鉴权前端 UI 交互 — 已完成浏览器取证（登录/刷新持久化/退出/后退越权均正常；边界输入无 XSS 无崩溃）
- ⚠️ A04 非超管越权（BatchEditController）— 仅测了超管，非超管角色越权验证留待"角色/权限"轮次
- 动态路由与菜单 — 已完成浏览器取证(菜单遍历/深层刷新持久化/多标签/折叠/1280+1920布局均正常) + 后端菜单/资源链路静态审查
- ⚠️ 会话管理(sys/ops/monitor)菜单白屏 = 缺失视图组件，见 A09
- 用户/角色/机构/岗位 — 已完成浏览器CRUD取证(用户/角色增改删搜/授权/建受限用户qauser) + 后端四链路静态审查
- ✅ A04 越权已运行态验证并修复(见A12)；biz/organization 下 role+position 搜索副本同 A13 模式待验
- 消息中心/文件管理 越权面 — 第4轮运行态坐实并修复(A22/A23)，页面 UI 本身未系统性走查
- 文件上传（含分片上传、UploadCleanup 清理）— 第7轮完成静态审查
- 全站搜索列参数对齐 — 第4轮横扫(A20/A26/A27/A28)，静默失效模式已识别
- Controller 权限标注全量清点 — 第5轮完成(31个)。系统管理面全部有[SuperAdmin]或[RolePermission]；类级无标注的 UserCenter/AuthB/Common 属自助面或匿名面，设计如此。**除已修的外无新增漏标 P0**
- 资源与权限(数据范围) — 第5轮完成：biz 角色授权链路静态审查(查出A31-A33三个P0)、菜单管理+角色管理+授权资源/数据范围弹窗浏览器取证
- 日志/监控 — 第8轮完成静态审查(操作日志/访问日志/写入链路/脱敏/清理/性能)。**权限面是干净的**：两个日志 Controller 均有类级[SuperAdmin]；`LogExceptionHandler.cs:29-40` 堆栈不外泄；`Page` 用 IgnoreColumns 排除大字段
- ⚠️ "系统监控/在线用户"功能**在本仓库不存在**：全仓无 monitor/OnlineUser/SysMonitor 相关 Controller 与视图。佐证 A09(菜单指向 sys/ops/monitor/index 但无该视图)
- ⚠️ `[DisplayName]` 审计覆盖率经全量普查约92%(100个POST中8个缺)，非系统性问题，见已驳回的 A58
- ⚠️ 数据范围的运行态越权验证始终未做(curl/node 执行通道被安全检查封锁)，A12/A22/A23/A31-A33 均只有静态或编译级证据

## 各轮小结

### 第 1 轮（鉴权）
- 后端链路静态审查 + 前端 UI 浏览器取证均完成（服务原未启动，已授权拉起 web+api）。
- 修复 2 个：A01 死代码、A06 验证码废请求。
- 待你确认 3 个：A02 验证码占位、A03 密码可逆存储、A07 登录页重复请求(动公共store)。
- 待确认 1 个：A04 非超管越权待验。已驳回 1 个：A08 element-plus 库噪音。
- 正常项：XSS 转义、超长输入、路由守卫/越权残留、刷新持久化。

### 第 2 轮（动态路由与菜单）
- 浏览器取证 + 后端链路静态审查完成。
- 修复 1 个：A10 死代码(7处)。
- 待你确认 1 个：A09 会话管理菜单白屏(缺视图组件，涉及新建页面或改SeedData)。
- 已驳回 1 个：A11 首次访问闪白(dev-only HMR假象)。
- 正常项：菜单树遍历、深层路由F5持久化、多标签开关、菜单折叠、1280/1920布局、菜单类Controller权限标注齐全、UserCenter无标注属正确设计。
- 注：浏览器取证 agent 本轮耗时过长(全站逐页截图)，后续已收窄范围。

### 第 3 轮（用户/角色/机构/岗位）
- 浏览器CRUD取证 + 后端四链路静态审查 + A04越权运行态验证 完成。
- 修复 3 个：A12 BatchEdit越权(P0)、A13 角色/岗位搜索失效、A14 死代码。
- 待你确认 1 个：A16 删角色不拦截引用。
- 待确认 4 个：A15 删用户踢下线、A17 BeforeLogin 500、A18 账号超长、A19 1280布局。
- 正常项：四Controller权限标注齐全、授权界面非超管禁系统菜单(招牌特性生效)、System层不套数据范围属正确(超管专属)、删机构/岗位有引用校验、无N+1/吞异常/async void。
- 累积待你确认：A02 A03 A07 A09 A16。

### 第 4 轮（越权面复验 + 搜索列横扫 + biz 侧机构）
- A12 运行态销账(qauser 调 batch/tables、batch/config 均 403)；顺带发现 A22/A23 两个同源 P0 并修复。
- 修复 4 个：A22 消息越权(P0)、A23 文件越权(P0)、A25 biz机构分类筛选、A28 模块名称搜索。
  提交：b2881c3(A22/A23)、d31bebe(A25)、9ecfb5d(A28)。
- 已驳回 1 个：A24 LoginOut 登出他人(静态证据推翻)。
- 待你确认 4 个：A20 编码搜索缺后端字段、A21 RolePageInput 错误继承、A26 消息状态搜索、A27 菜单搜索列、A29 biz/ops/test 无后端。
- 待确认 1 个：A30 message/add 缺 ReceiverInfo 校验 → 500。
- 根因归纳：JwtHandler.cs:140 对无 [SuperAdmin]/[RolePermission] 标注的端点直接 return true —— 漏标即默认放行，
  这是 A12/A22/A23 的同一个源头。第5轮起把"Controller 权限标注全量清点"作为固定动作。
- 累积待你确认：A02 A03 A07 A09 A16 A17 A20 A21 A26 A27 A29。

### 第 5 轮（资源与权限 / 数据范围）
- 四路并发取证：Controller 权限标注全量清点、数据范围后端链路静态审查、运行态越权复验(被封锁)、资源页浏览器取证。
- 修复 3 个 P0：A31 GrantResource / A32 GrantUser 无数据范围校验、A33 Edit 校验攻击者可控字段。commit 9ab64e3，仅编译级验证。
- 待你确认 6 个：A34 mqtt 明文凭证泄露、A35 mqtt 占位符误写、A36 通用上传口无标注无校验、A37 BizPosition 路由未注册、A39 FormContainer v-model 契约不符。
- 已驳回 1 个：A38 授权弹窗按钮无响应 —— 复测确认是 agent-browser 坐标点击落在视口外(inViewport=false)，
  改用 DOM .click() 后请求正常发出、弹窗正常关闭。静态复核(按钮 @click 绑定完好、handleSubmit 无早退)先于复测给出了正确判断。
  ★方法论沉淀：子 agent 报"点了没反应"时，先静态复核绑定，再让它用 JS 求值查 inViewport + DOM .click() 对照，别直接入账。
- 已排除：资源树父子级联全选属标准行为；数据范围弹窗 1280/1920 布局均正常；角色名称搜索正常(A13 修复有效)。
- 菜单名称搜索失效获得运行态证据(tree?title=... 发出但结果未过滤)，归入既有 A27，不重复登记。
- ★阻塞：本会话 Bash/PowerShell 执行通道被安全检查整体封锁(curl/node/甚至本地 find 均被拒)，
  运行态越权验证无法进行。A12/A22/A23 的销账与 A31-A33 的复验均欠着，需用户放行权限或手动跑 curl。
- 根因复盘：A31-A33 与 A12/A22/A23 是同一类病 —— 权限/数据范围校验靠"每个方法自觉调一次"，
  漏调即默认放行。JwtHandler:140 的"无标注即放行"是接口级的表现，RoleService 的"忘了调 CheckApiDataScope"
  是数据级的表现。默认值反了。彻底修法是默认拒绝+显式放行，属架构改动，待用户拍板。
- 累积待你确认：A02 A03 A07 A09 A16 A17 A20 A21 A26 A27 A29 A34 A35 A36 A37 A39。

### 第 7 轮（文件上传）
- 本轮主题=文件上传含分片；**纯静态代码审查，零运行态验证**（API 因 `RedisCacheService.cs:82` Redis 连接超时起不来，浏览器侦察全程停摆），证据等级如实降档为"仅静态证据"。
- 新增 A41–A47 七条（A48 为主控自身假设，已自行驳回）。
- 最大收获：uploadImg 这条链是"死代码 + 坏实现 + 敞开端点"三合一——前端零活调用点(A41/A42/A43)，实现本身因 FormData 被对象展开而必然丢文件，后端端点却对全体登录用户敞开(A44)。
- 分片主链路（DocumentController）**权限是好的**：类级 `[RolePermission]` + `IDynamicApiController`，路径穿越也已被 `Path.GetFileName()` 挡住(A48 已驳回)；问题集中在完整性校验可选(A45)与并发临时文件互踩(A46)。
- 遗留：A46 需运行态复现；A44 的处置（删端点 vs 补校验）待用户拍板。
- **台账维护**：本轮发现 A40 号被提交 `fecc904` 占用但行未落盘（第 6 轮上下文截断所致），已回填 A40(ConfigService.Edit 内置分类，已修复)，第 7 轮整体顺延为 A41-A48；另补录第 6 轮判定遗留的 A49-A51。
- 累积待你确认新增：A41 A42 A43 A44 A45 A47 A49 A50 A51。

### 第 8 轮（日志/监控）
- 本轮主题=日志与监控；**同样是纯静态审查，零运行态验证**（API 仍因 Redis 连接超时起不来），每条均标"仅静态证据"。
- 新增 A52–A57 六条（A58 为主控自身假设，经全量普查证伪、已自行驳回）。
- **难得的反例：这一轮权限面是干净的**。两个日志 Controller 都规矩挂了类级 `[SuperAdmin]`；`LogExceptionHandler.cs:29-40` 把非友好异常统一替换成固定文案，堆栈/SQL 不外泄；`Page` 用 `IgnoreColumns` 排除 ParamJson/ResultJson 大字段；登录**成功**走 `CreateVisitLog`，而 `SysLogVisit` 表根本没有 ParamJson 字段。这些都是有人认真想过的痕迹。
- 最大收获：**问题全部集中在"日志自身的治理"，而不是权限**。三条串成一条线——日志里存着可重放的凭据(A52)、清空按钮清不干净(A53)、而清空这个动作还不留痕(A55)。
- 审计不对等：手机验证码登录整条路径零日志、密码登录有(A56)；同一个"执行代码生成"，走 POST 留痕、走 GET 不留痕(A57)。
- **方法论自省**：本轮主控预判 `[DisplayName]` 缺失是"默认放行"的第五种形态，全量普查后覆盖率约92%且存在显式退出机制 `[SuppressMonitor]`，数据不支持该叙事，已驳回(A58)。前四种形态的归纳成立不代表第五种存在，不为凑模式而定罪。
- 遗留：全部 6 条均为静态证据，无一条运行态复验。
