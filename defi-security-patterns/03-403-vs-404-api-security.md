# 03 · 权限接口信息泄露（403 vs 404）

**类别**：信息安全 / API 设计  
**影响场景**：后台接口、管理 API、敏感路由  
**危险等级**：🟡 中（辅助攻击，帮助攻击者侦察系统结构）  
**发现来源**：DCS 团队代码审查讨论（Ben Zhao 提出）

---

## 问题描述

当用户访问无权限的接口时，返回不同的 HTTP 状态码会泄露不同程度的信息：

```
攻击者试探：GET /admin/users

→ 返回 403 Forbidden：
  攻击者获知：「/admin/users 路由存在，只是没权限访问」
  攻击者下一步：尝试权限绕过（JWT 伪造、参数注入、水平越权等）

→ 返回 404 Not Found：
  攻击者获知：「不确定这个路径是否存在」
  攻击者无从判断：侦察成本大幅提高
```

这是**安全通过模糊性（Security through Obscurity）**的应用，作为纵深防御的一层。

---

## 实际危害场景

```
攻击者通过 403 发现了以下接口存在：
  /admin/users
  /admin/config
  /internal/metrics
  /debug/logs

获得这张「地图」后：
  1. 针对 /admin/users 尝试 IDOR（水平越权）：/admin/users/1 → /admin/users/2
  2. 针对 /internal/metrics 尝试未鉴权访问（换个 IP、换个 header）
  3. 针对 /debug/logs 尝试路径遍历：/debug/logs/../../../etc/passwd

如果全部返回 404，攻击者的侦察几乎无法进行。
```

---

## 防御实现

### Spring Boot 示例

```java
// ❌ 危险：暴露路由存在性
@RestController
public class AdminController {

    @GetMapping("/admin/users")
    public ResponseEntity<?> getAdminUsers(Authentication auth) {
        if (!hasAdminRole(auth)) {
            // 403 告诉攻击者：这个接口存在！
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                                 .body("Access denied");
        }
        return ResponseEntity.ok(userService.getAllUsers());
    }
}

// ✅ 安全：统一返回 404，不暴露接口存在性
@RestController
public class AdminController {

    @GetMapping("/admin/users")
    public ResponseEntity<?> getAdminUsers(Authentication auth) {
        if (!hasAdminRole(auth)) {
            // 对非管理员，这个接口「不存在」
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(userService.getAllUsers());
    }
}
```

### 全局统一处理（推荐）

```java
// 在 Spring Security 或全局异常处理中统一配置
@Component
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            // 未认证时返回 404 而不是 401/403
            .exceptionHandling()
                .authenticationEntryPoint((request, response, ex) -> {
                    response.sendError(HttpServletResponse.SC_NOT_FOUND);
                })
                .accessDeniedHandler((request, response, ex) -> {
                    // 区分：普通接口返回 403，敏感接口返回 404
                    if (isSensitiveEndpoint(request.getRequestURI())) {
                        response.sendError(HttpServletResponse.SC_NOT_FOUND);
                    } else {
                        response.sendError(HttpServletResponse.SC_FORBIDDEN);
                    }
                });
    }
    
    private boolean isSensitiveEndpoint(String uri) {
        return uri.startsWith("/admin/") || 
               uri.startsWith("/internal/") || 
               uri.startsWith("/debug/");
    }
}
```

---

## 什么时候用 403，什么时候用 404

```
判断标准：
  「如果攻击者知道这个接口存在，会不会增加攻击面？」
  
  → 是 → 返回 404
    案例：/admin/*、/internal/*、/debug/*、/actuator/*
  
  → 否 → 403 也可以（甚至更好，帮用户理解要登录）
    案例：/api/profile（用户知道这个接口存在是正常的）
```

| 接口类型 | 推荐返回码 | 原因 |
|---------|---------|------|
| 后台管理接口 `/admin/*` | 404 | 不应让外部知道存在 |
| 内部服务接口 `/internal/*` | 404 | 仅内网访问 |
| 调试接口 `/debug/*`, `/actuator/*` | 404 | 生产环境不应暴露 |
| 普通用户接口（需登录） | 401/403 | 告知用户需要登录 |
| 公开接口 | 200/400 | 正常响应 |

---

## 配合其他防御措施

单靠 403→404 还不够，需要配合：

1. **接口路径不可猜测**
   ```
   ❌ /admin/users
   ✅ /a8f3k2/users（随机前缀）
   ```

2. **访问日志 + 告警**
   ```
   连续多次 404 访问 /admin/* → 触发告警
   这是典型的目录扫描行为
   ```

3. **WAF 规则**
   ```
   对敏感路径的 404 请求进行频率限制
   ```

4. **MAS TRM 合规要求**
   ```
   敏感接口需要：
   - 访问控制审计日志
   - 异常访问告警
   - 定期权限审查
   ```

---

## 参考
- [OWASP API Security Top 10: API3 - Excessive Data Exposure](https://owasp.org/API-Security/)
- [OWASP Testing Guide: Testing for Sensitive Information in HTTP Headers](https://owasp.org/www-project-web-security-testing-guide/)
- MAS Technology Risk Management Guidelines 2021
