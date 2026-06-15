---
author: HKL
categories:
- Default
date: "2026-06-14T23:12:00+08:00"
slug: enable-ip-cert-in-caddy
draft: false
tags:
- Operating
- Networking
title: "Caddy + Let's Encrypt: Automated Public IP TLS Certificates in 2026 (Pitfalls & Solutions)"
---

在 HTTPS 普及的今天，“为公网裸 IP 直接签发 TLS 证书” 一直是个令人头疼的痛点。以往我们不得不依赖 ZeroSSL 或昂贵的商业 CA。

好消息是，**Let's Encrypt 在 2026 年正式开放了公网 IP 地址的免费公开签发能力**。配合 Caddy 的自动化 ACME 集成，我们可以轻松实现裸 IP 的证书自动申请与续期。本文记录了我在实际配置中踩到的“SNI 握手失败”大坑以及最终的完美解决方案。

## Let's Encrypt IP 证书的核心规则

在开始配置前，必须了解 Let's Encrypt 对 IP 证书的硬性限制：
1. **有效期仅 6 天**：必须使用 `shortlived` profile（短寿命证书），不支持常规的 90 天证书。
2. **必须关闭 TLS-ALPN-01 挑战**：目前 IP 证书仅支持 `HTTP-01` 挑战（即必须开放公网 80 端口）。
3. **高频自动续期**：Caddy 会在证书还剩约 48 小时时自动续期，只要自动化靠谱，6 天有效期完全不是负担。


## 致命的大坑：裸 IP 访问引发的 ERR_SSL_PROTOCOL_ERROR

在初步配置好 Caddyfile 后，我发现了一个诡异的现象：**同台服务器下的域名访问完全正常，但直接通过 `https://您的IP` 访问时，浏览器会报错 `ERR_SSL_PROTOCOL_ERROR`，curl 提示 `tls alert internal error`**。

查看 Caddy 日志，发现 IP 证书其实已经成功下载到了本地缓存中。那为什么无法握手呢？

### 原因：SNI（服务器名称指示）缺失
当通过域名访问时，浏览器会在 TLS 握手时携带 **SNI** 扩展，明确告诉 Web 服务器“我要访问哪个域名”；而当直接通过裸 IP 访问时，许多客户端默认**不发送 SNI**。
Caddy 在收到空 SNI 请求时，不知道该返回手里的哪张证书，为了安全会直接中止握手。

### 解法：配置 `default_sni`
我们必须在 Caddy 的全局配置中指定 `default_sni`。当客户端不带 SNI 访问时，强行让 Caddy 默认使用 IP 证书进行响应。


## 最终完美工作的 Caddyfile 配置

以下是兼顾**公网裸 IP 证书**和**多域名虚拟主机**的完整生产环境配置：

```caddy
{
    # 核心修复：解决裸 IP 访问时客户端不带 SNI 导致 TLS 握手失败（internal error）的问题
    default_sni YOUR_IP_ADDRESS
}

# 1. 公网裸 IP 配置
YOUR_IP_ADDRESS {
    root * /var/www
    file_server

    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }

    tls {
        issuer acme {
            profile shortlived          # 必须显式指定 Let's Encrypt 的 6 天短证书策略
            disable_tlsalpn_challenge   # 必须禁用 ALPN 挑战，强制走 HTTP-01 验证
        }
    }
}

# 2. 业务域名 A（反向代理）
a.example.com {
    handle / {
        root * /var/www
        file_server
    }

    handle {
        reverse_proxy localhost:33333
    }
}

# 3. 业务域名 B（静态托管）
n.example.com {
    root * /var/www
    file_server
}

```

## 运维与验证

修改配置后，执行以下命令使配置生效：

```bash
# 格式化 Caddyfile
caddy fmt --overwrite /etc/caddy/Caddyfile

# 重载配置
caddy reload --config /etc/caddy/Caddyfile

```

通过 `curl -v https://YOUR_IP_ADDRESS` 进行验证，可以看到 TLS 握手完美通过：

* **Server certificate**: `subject: CN=YOUR_IP_ADDRESS`
* **Expire date**: 6 天后过期，Caddy 将在后台无感自动续期。

从 2026 年开始，单独为了 IP 证书去购买商业 CA 或折腾第三方脚本已经成为过去式。**Caddy + Let's Encrypt 短证书**就是目前最优雅的解法！

# English Version

In the modern web ecosystem, securing a public "naked IP address" with a trusted TLS certificate has always been a major headache. Historically, we were forced to rely on ZeroSSL or expensive commercial CAs.

Exciting news arrived in **January 2026 when Let's Encrypt officially opened public IP certificate issuance to the general public**. Combined with Caddy's native ACME integration, we can now fully automate the issuance and renewal of IP certificates. This post documents a critical "SNI handshake failure" pitfall I encountered during setup and the ultimate working solution.

## Core Rules for Let's Encrypt IP Certificates

Before diving into the configuration, you must understand the strict requirements imposed by Let's Encrypt for IP identifiers:
1. **6-Day Validity Period**: You *must* use the `shortlived` profile. The standard 90-day certificates are not supported for IP addresses.
2. **HTTP-01 Challenge Only**: The `TLS-ALPN-01` challenge must be disabled. Let's Encrypt only validates IP ownership via port 80.
3. **High-Frequency Auto-Renewal**: Caddy automatically renews the certificate when about 48 hours of validity remain. As long as automation works, the 6-day lifespan is non-issue.


## The Fatal Pitfall: Naked IP Access Triggers `ERR_SSL_PROTOCOL_ERROR`

After my initial Caddyfile setup, I ran into a bizarre issue: **Domain names hosted on the same server worked perfectly, but accessing `https://YOUR_IP` directly triggered `ERR_SSL_PROTOCOL_ERROR` in browsers and `tls alert internal error` in curl.**

Checking the Caddy logs showed that the IP certificate had been successfully issued and stored in the local cache. So why did the handshake fail?

### The Root Cause: Missing SNI (Server Name Indication)
When accessing via a domain name, the browser sends the **SNI** extension during the TLS handshake, explicitly telling Caddy which site it wants. However, when connecting via a raw IP address, many clients **do not send any SNI** by default.
When Caddy receives a request with an empty SNI, it gets confused about which certificate to present and aborts the handshake for security reasons.

### The Fix: Configuring `default_sni`
We must define a `default_sni` in Caddy's global options. If a client connects without an SNI, Caddy will fallback to this default value and serve the corresponding IP certificate.

## The Final, Perfectly Working Caddyfile

Here is the complete production configuration that handles both **public naked IP TLS** and **multi-domain virtual hosting**:

```caddy
{
    # CRITICAL FIX: Solves TLS handshake failures (internal error) caused by clients not sending SNI during raw IP access
    default_sni YOUR_IP_ADDRESS
}

# 1. Public Naked IP Configuration
YOUR_IP_ADDRESS {
    root * /var/www
    file_server

    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }

    tls {
        issuer acme {
            profile shortlived          # Required: Specifies Let's Encrypt's 6-day short-lived profile
            disable_tlsalpn_challenge   # Required: Disables ALPN challenge, forcing HTTP-01 validation
        }
    }
}

# 2. Domain A (Reverse Proxy)
a.example.com {
    handle / {
        root * /var/www
        file_server
    }

    handle {
        reverse_proxy localhost:33333
    }
}

# 3. Domain B (Static File Server)
n.example.com {
    root * /var/www
    file_server
}

```


## Operations & Verification

To apply the changes, run the following commands in your terminal:

```bash
# Format the Caddyfile
caddy fmt --overwrite /etc/caddy/Caddyfile

# Reload the configuration
caddy reload --config /etc/caddy/Caddyfile

```

Verify the setup using `curl -v https://YOUR_IP_ADDRESS`. You should see a successful TLS handshake:

* **Server certificate**: `subject: CN=YOUR_IP_ADDRESS`
* **Expire date**: Valid for 6 days, with silent background auto-renewals handled seamlessly by Caddy.

As of 2026, paying for commercial CAs or wrestling with third-party ACME scripts just for an IP certificate is a thing of the past. **Caddy + Let's Encrypt Short-lived Profile** is the most elegant solution available today!
