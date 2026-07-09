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

## 已覆盖区域

- 鉴权后端链路（AuthController/AuthService/JwtHandler/事件订阅）— 已静态审查
- ⚠️ 登录/鉴权前端 UI 交互 — 未完成（web 服务 8848 未运行，浏览器取证被阻塞）

## 各轮小结

### 第 1 轮（鉴权）
- 后端链路静态审查完成，UI 交互因服务未启动被阻塞。
- 修复 1 个（A01 死代码）。
- 待你确认 2 个（A02 验证码占位、A03 密码可逆存储）；待确认 2 个（A04 越权待验、A05 空catch）。
