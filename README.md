# Other-MailServer

```
Docker Mailserver（Postfix/Dovecot）+ Roundcube Web 郵件客戶端 + Nginx 反向代理
```

## 目錄

- [Other-MailServer](#other-mailserver)
  - [目錄](#目錄)
  - [專案說明](#專案說明)
  - [架構概覽](#架構概覽)
  - [服務說明](#服務說明)
  - [連接埠對應](#連接埠對應)
  - [執行流程](#執行流程)
  - [用法](#用法)
  - [DNS 設定](#dns-設定)
  - [防火牆設定](#防火牆設定)
  - [建議注意事項](#建議注意事項)
  - [參考資料](#參考資料)

## 專案說明

本專案以 Docker Compose 建立完整的自架郵件伺服器環境，包含：

- **Docker Mailserver**：基於 Postfix（SMTP）與 Dovecot（IMAP）的後端郵件收發引擎，並啟用 Rspamd（垃圾信過濾）、ClamAV（病毒掃描）、Fail2Ban（暴力破解防護）。
- **Roundcube**：Web 郵件客戶端，提供瀏覽器介面收發信件。
- **MySQL**：Roundcube 所需的資料庫後端。
- **Nginx**：反向代理，負責對外的 HTTP/HTTPS 流量，並轉導至 Roundcube。

## 架構概覽

```
Internet
  └─ Nginx (80 / 443)        ← 反向代理（HTTP 轉導 HTTPS）
       └─ Roundcube           ← Web 郵件 UI
            └─ MySQL          ← Roundcube 資料庫
  └─ Mailserver (25/465/587/993)  ← 實際郵件收發（Postfix + Dovecot）
       ├─ Rspamd              ← 垃圾信過濾
       ├─ ClamAV              ← 病毒掃描
       └─ Fail2Ban            ← 暴力破解防護
```

## 服務說明

| 服務 | 說明 | 關鍵設定 |
|------|------|---------|
| mailserver | Docker Mailserver（Postfix + Dovecot） | ENABLE_RSPAMD=1, ENABLE_CLAMAV=1, ENABLE_FAIL2BAN=1 |
| mysql | Roundcube 資料庫 | - |
| roundcube | Web 郵件客戶端 UI | 連接 mailserver IMAP/SMTP |
| nginx | 反向代理 | HTTP(80) 轉導 HTTPS(443) |

## 連接埠對應

| 埠號 | 協定 | 用途 |
|------|------|------|
| 25   | TCP  | SMTP（接收外部郵件） |
| 465  | TCP  | SMTPS（SSL 加密發信） |
| 587  | TCP  | Submission（用戶端發信，STARTTLS） |
| 993  | TCP  | IMAPS（SSL 加密收信） |
| 80   | TCP  | HTTP（自動轉導至 HTTPS） |
| 443  | TCP  | HTTPS（Roundcube Web UI） |

## 執行流程

1. Clone 專案並進入目錄
2. 設定網域名稱變數
3. 複製並替換設定範本（`.env`、Nginx conf、Postfix 設定）
4. 複製 SSL 憑證至 `conf/ssl/`
5. 啟動 Docker Compose 服務
6. 設定 DNS 記錄指向伺服器 IP
7. 開放防火牆連接埠

## 用法

```bash
# 1. Clone 專案
git clone https://github.com/open222333/Other-MailServer.git mail_server
cd mail_server

# 2. 設定網域名稱（替換為實際網域）
domain_name="yourdomain.com"

# 3. 複製並套用設定範本
cp .env.sample .env
cp conf/admin.example.com.conf conf/nginx/admin.$domain_name.conf
cp conf/mail.example.com.conf conf/nginx/mail.$domain_name.conf
cp conf/postfix-main.cf.bak conf/docker-mailserver/postfix-main.cf

# 4. 替換範本中的 example.com 為實際網域
sed -i "s/example.com/$domain_name/g" .env
sed -i "s/example.com/$domain_name/g" conf/docker-mailserver/postfix-main.cf
sed -i "s/example.com/$domain_name/g" conf/nginx/admin.$domain_name.conf
sed -i "s/example.com/$domain_name/g" conf/nginx/mail.$domain_name.conf

# 5. 複製 Let's Encrypt SSL 憑證
cp -r /etc/letsencrypt/live/$domain_name conf/ssl

# 6. 啟動服務
docker compose up -d

# 查看服務狀態
docker compose ps

# 查看日誌
docker compose logs -f mailserver
```

## DNS 設定

在 DNS 管理介面新增以下記錄（將 IP 替換為實際伺服器 IP）：

**格式 1（BIND zone file）**

```
$ORIGIN example.com
@     IN  A      10.11.12.13
mail  IN  A      10.11.12.13

; 郵件伺服器 MX 記錄
@     IN  MX  10 mail.example.com.

; SPF 記錄（防止偽造發件人）
@     IN  TXT    "v=spf1 mx -all"
```

**格式 2（一般 DNS 管理介面）**

```
admin.example.com.  IN  A   xxx.xxx.xxx.xxx
mail.example.com.   IN  A   xxx.xxx.xxx.xxx
example.com.        IN  MX  10 mail.example.com.
```

> DNS 記錄生效可能需要數分鐘至數小時，請耐心等待。

## 防火牆設定

```bash
# 開放郵件相關連接埠
ufw allow 25
ufw allow 465
ufw allow 587
ufw allow 993

# 開放 Web UI 連接埠（如需對外）
ufw allow 80
ufw allow 443
```

## 建議注意事項

- **SSL 憑證**：建議使用 Let's Encrypt 取得有效憑證，自簽憑證可能導致郵件被標記為垃圾或被拒收。
- **PTR（反向 DNS）記錄**：向 ISP 或主機商申請設定 PTR 記錄（`mail.example.com` ↔ 伺服器 IP），有助於提高郵件送達率。
- **DKIM / DMARC**：建議另外設定 DKIM 簽名與 DMARC 策略，進一步降低信件被列為垃圾的機率。
- **Port 25 封鎖**：部分雲端主機商預設封鎖 Port 25，需向主機商申請開放後才能接收外部郵件。
- **ClamAV 啟動時間**：ClamAV 病毒資料庫更新需要一段時間，首次啟動後可能需要等待幾分鐘才完全就緒。
- **資料備份**：`conf/docker-mailserver/` 與 MySQL 資料庫為核心資料，請定期備份。
- **`.env` 檔案**：包含敏感資訊（密碼等），請勿提交至版本控制。

## 參考資料

- [Docker Mailserver（郵件伺服器）](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/MailServer(%E9%83%B5%E7%AE%B1%E4%BC%BA%E6%9C%8D%E5%99%A8)/Docker%20Mailserver(%E9%83%B5%E4%BB%B6%E4%BC%BA%E6%9C%8D%E5%99%A8).md)
- [RainLoop（Web 郵件客戶端）](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/MailServer(%E9%83%B5%E7%AE%B1%E4%BC%BA%E6%9C%8D%E5%99%A8)/RainLoop(Web%20%E9%83%B5%E4%BB%B6%E5%AE%A2%E6%88%B6%E7%AB%AF).md)
