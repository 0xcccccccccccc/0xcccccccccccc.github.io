# Server Security Assessment Report

**Target:** 111.231.94.26:8899  
**Date:** 2026-05-24  
**Type:** Non-destructive remote reconnaissance (Red Team)  
**Assessor:** Automated security assessment  

---

## Executive Summary

对目标服务器 `111.231.94.26` 进行了非破坏性的远程安全评估。**端口 8899 从外部完全不可达**（连接超时），仅发现端口 80 和 443 对外开放。端口 80 上运行的反向代理返回 503 错误，表明后端服务未正常运行或未正确配置。

---

## 1. Port Scan Results

### Scanned Ports (45+ ports)

| Port | Status | Service |
|------|--------|---------|
| **80** | OPEN | HTTP (反向代理，返回 503) |
| **443** | OPEN | TLS 握手成功，但上层应用无响应 |
| **8899** | FILTERED/CLOSED | 连接超时，完全不可达 |
| 22 (SSH) | FILTERED | 不可达 |
| 3306 (MySQL) | FILTERED | 不可达 |
| 6379 (Redis) | FILTERED | 不可达 |
| 27017 (MongoDB) | FILTERED | 不可达 |
| 8080, 8443, 9090 等 | FILTERED | 不可达 |

**Additional scanned ports (all FILTERED):** 21, 23, 25, 53, 110, 143, 389, 636, 1433, 1521, 2222, 3000, 4000, 5000, 5001, 5432, 7070, 7071, 7443, 8000, 8001, 8008, 8081, 8800, 8880, 8888, 9000, 9200, 9300, 9443, 10000, 10080, 10443, 18899

### Assessment

- **正面发现：** SSH (22)、数据库端口 (3306/6379/27017/5432) 等敏感端口均未对外暴露，防火墙配置合理。
- **目标端口 8899 不可达：** 该端口被防火墙过滤或服务未运行。

---

## 2. HTTP Service Analysis (Port 80)

### Response

```
HTTP/1.1 503 Service Unavailable
content-length: 190
content-type: text/plain

upstream connect error or disconnect/reset before headers. retried and the latest 
reset reason: remote connection failure, transport failure reason: delayed connect 
error: Connection refused
```

### Findings

| # | Severity | Finding |
|---|----------|---------|
| 2.1 | **INFO** | 反向代理（疑似 Envoy/Istio）在端口 80 运行 |
| 2.2 | **MEDIUM** | 后端服务 (upstream) 连接被拒绝，503 错误 |
| 2.3 | **LOW** | 错误信息泄露了内部架构细节（"upstream connect error"、"transport failure reason"） |

### Detail

- 错误信息格式与 **Envoy Proxy** 一致，暗示使用了服务网格或 Envoy 做反向代理。
- 所有路径（`/admin`, `/login`, `/api`, `/.env`, `/.git/config`, `/actuator`, `/phpmyadmin` 等 30+ 个路径）均返回相同的 503 错误，说明代理层统一处理，后端完全不可达。

---

## 3. TLS/SSL Analysis (Port 443)

### Findings

| # | Severity | Finding |
|---|----------|---------|
| 3.1 | **GOOD** | TLS 1.3 (最新协议版本) |
| 3.2 | **GOOD** | Cipher: TLS_AES_256_GCM_SHA384 (强加密套件) |
| 3.3 | **MEDIUM** | TCP 握手成功但 HTTPS 应用层无响应（超时） |

---

## 4. HTTP Security Headers

Port 80 返回的 503 错误页面缺少所有安全头：

| Header | Status | Risk |
|--------|--------|------|
| `Strict-Transport-Security` | MISSING | 无 HSTS，可能遭受降级攻击 |
| `X-Content-Type-Options` | MISSING | MIME 嗅探风险 |
| `X-Frame-Options` | MISSING | 点击劫持风险 |
| `Content-Security-Policy` | MISSING | XSS 风险增加 |
| `Referrer-Policy` | MISSING | Referrer 信息泄露 |
| `Permissions-Policy` | MISSING | 浏览器特性未限制 |
| `Server` | NOT DISCLOSED | 良好，未暴露服务器软件 |
| `X-Powered-By` | NOT DISCLOSED | 良好，未暴露技术栈 |

---

## 5. Path Enumeration

测试了 30+ 常见敏感路径，所有路径均返回 503：

- `/admin`, `/login`, `/dashboard`, `/console` — 503
- `/api`, `/api/v1`, `/graphql` — 503
- `/.env`, `/.git/config` — 503
- `/actuator`, `/actuator/health`, `/metrics` — 503
- `/phpmyadmin`, `/wp-admin` — 503
- `/robots.txt`, `/.well-known/security.txt` — 503
- `/server-status`, `/server-info` — 503

**Assessment:** 代理层在后端不可达时统一返回 503，不存在路径信息泄露。

---

## 6. Risk Summary

### Critical / High

无严重或高风险漏洞。

### Medium

| # | Issue | Description | Recommendation |
|---|-------|-------------|----------------|
| M1 | **端口 8899 服务不可达** | 目标端口完全不可达，如果这是期望提供的服务，则存在可用性问题 | 确认防火墙规则是否正确，检查服务是否正在运行 |
| M2 | **后端服务宕机** | 端口 80 的反向代理无法连接到后端服务 | 检查后端应用是否启动并正确绑定端口 |
| M3 | **缺少安全头** | 缺少 HSTS、CSP 等安全头 | 在反向代理层统一添加安全头 |

### Low

| # | Issue | Description | Recommendation |
|---|-------|-------------|----------------|
| L1 | **错误信息泄露** | 503 错误页面暴露了内部代理架构信息 | 自定义错误页面，隐藏内部细节 |
| L2 | **443 端口配置异常** | TCP 层可达但应用层无响应 | 检查 443 端口的 TLS 终止配置 |

---

## 7. Recommendations

### Immediate Actions (立即执行)

1. **检查端口 8899 服务状态**
   ```bash
   # 在服务器上执行
   ss -tlnp | grep 8899
   systemctl status <your-service>
   ```

2. **检查后端应用健康状态**
   ```bash
   # 检查后端应用是否运行
   ps aux | grep <your-app>
   journalctl -u <your-service> --since "1 hour ago"
   ```

3. **检查防火墙规则**
   ```bash
   iptables -L -n | grep 8899
   # 或使用云厂商控制台检查安全组规则
   ```

### Security Hardening (安全加固)

4. **添加安全响应头**（在反向代理/Envoy 配置中）
   ```yaml
   # Envoy 配置示例
   response_headers_to_add:
     - header:
         key: "Strict-Transport-Security"
         value: "max-age=31536000; includeSubDomains"
     - header:
         key: "X-Content-Type-Options"
         value: "nosniff"
     - header:
         key: "X-Frame-Options"
         value: "DENY"
     - header:
         key: "Content-Security-Policy"
         value: "default-src 'self'"
     - header:
         key: "Referrer-Policy"
         value: "strict-origin-when-cross-origin"
   ```

5. **自定义错误页面**，避免泄露 "upstream connect error" 等内部信息。

6. **确保 SSH (22) 使用密钥认证**，禁用密码登录（虽然端口已被防火墙过滤，仍建议加固）。

### Recommended Follow-up (建议后续检查)

7. **从服务器本地运行内部安全扫描**（在可以直接访问的环境下）：
   ```bash
   # 安装并运行 nmap 做本地扫描
   nmap -sV -sC -p- 127.0.0.1
   
   # 检查 SSL/TLS 配置
   testssl.sh 111.231.94.26:443
   
   # Web 应用扫描（服务恢复后）
   nikto -h http://127.0.0.1:8899
   ```

8. **检查服务器安全基线**：
   - 系统补丁是否最新
   - 是否启用自动安全更新
   - 是否配置了入侵检测 (fail2ban 等)
   - 日志审计是否开启

---

## 8. Limitations

本次评估存在以下限制：

- 评估从云端沙箱环境发起，非标准端口的出站连接可能受限
- 端口 8899 完全不可达，无法对目标服务进行应用层安全测试
- 未进行认证测试、漏洞利用或压力测试
- 建议在服务恢复后，从可直接访问的网络环境重新进行评估

---

*Report generated: 2026-05-24*
