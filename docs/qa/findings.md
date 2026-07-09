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

## 已覆盖区域

- 鉴权后端链路（AuthController/AuthService/JwtHandler/事件订阅）— 已静态审查
- 登录/鉴权前端 UI 交互 — 已完成浏览器取证（登录/刷新持久化/退出/后退越权均正常；边界输入无 XSS 无崩溃）
- ⚠️ A04 非超管越权（BatchEditController）— 仅测了超管，非超管角色越权验证留待"角色/权限"轮次
- 动态路由与菜单 — 已完成浏览器取证(菜单遍历/深层刷新持久化/多标签/折叠/1280+1920布局均正常) + 后端菜单/资源链路静态审查
- ⚠️ 会话管理(sys/ops/monitor)菜单白屏 = 缺失视图组件，见 A09

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
