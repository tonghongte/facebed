# 使用 GHCR + Portainer Stack 部署 facebed

透過 GitHub Actions 自動建置的映像，在 Portainer 以 Stack 方式快速部署，無需自行 build。

---

## 前置需求

- 已安裝 Portainer（v2.x 以上）與 Docker 的伺服器
- GitHub 帳號（已 fork 本專案）

---

## 步驟一：確認 GHCR 映像已公開

每次推送至 `main` 分支，GitHub Actions 會自動建置並推送映像至：

```
ghcr.io/tonghongte/facebed:main
```

首次使用前，需將映像設為公開（否則 Portainer 拉取時需額外設定認證）：

1. 前往 GitHub → **Packages** → 找到 `facebed`
2. **Package settings** → **Change visibility** → **Public**

---

## 步驟二：在主機建立設定檔

> **如果不需要自訂設定**（只是想快速跑起來），可跳過此步驟，直接使用步驟三的「無設定檔版本」。

**Linux 伺服器（SSH 連入後）：**

```bash
mkdir -p /opt/facebed
cat > /opt/facebed/config.yaml << 'EOF'
host: 0.0.0.0
port: 9812
timezone: 8
banned_users: []
banned_notifier_webhook: ''
EOF
```

**Windows（Docker Desktop）— 以 PowerShell 執行：**

```powershell
New-Item -ItemType Directory -Force -Path C:\facebed
@"
host: 0.0.0.0
port: 9812
timezone: 8
banned_users: []
banned_notifier_webhook: ''
"@ | Set-Content -Encoding UTF8 C:\facebed\config.yaml
```

| 設定項 | 說明 | 預設值 |
|--------|------|--------|
| `host` | 監聽位址 | `0.0.0.0` |
| `port` | 監聽埠號 | `9812` |
| `timezone` | 顯示時區（UTC+N，例如 `8` 為 UTC+8） | `7` |
| `banned_users` | 封鎖的 Facebook 用戶 ID 列表 | `[]` |
| `banned_notifier_webhook` | 觸發封鎖時通知的 Discord Webhook URL | `''` |

---

## 步驟三：在 Portainer 建立 Stack

1. 登入 Portainer → 選擇 Environment → **Stacks** → **Add stack**
2. **Name**：填入 `facebed`
3. **Build method** 選 **Web editor**，依需求擇一貼入：

**（A）有自訂設定檔（完成步驟二後使用）**

Linux 伺服器：
```yaml
services:
  facebed:
    image: ghcr.io/tonghongte/facebed:main
    container_name: facebed
    restart: unless-stopped
    ports:
      - "9812:9812"
    volumes:
      - /opt/facebed:/facebed/conf:ro
    command: ["python3.14", "./facebed.py", "-c", "/facebed/conf/config.yaml"]
```

Windows（Docker Desktop），路徑改用 Windows 格式：
```yaml
services:
  facebed:
    image: ghcr.io/tonghongte/facebed:main
    container_name: facebed
    restart: unless-stopped
    ports:
      - "9812:9812"
    volumes:
      - C:\facebed:/facebed/conf:ro
    command: ["python3.14", "./facebed.py", "-c", "/facebed/conf/config.yaml"]
```

> **注意**：設定檔必須在部署前已存在於主機，否則容器會因找不到設定檔而不斷重啟。若出現 `config file … not found` 錯誤，請先完成步驟二，再至 Portainer → Container → **Restart**。

**（B）無設定檔（使用映像內建預設值，timezone UTC+7）**

```yaml
services:
  facebed:
    image: ghcr.io/tonghongte/facebed:main
    container_name: facebed
    restart: unless-stopped
    ports:
      - "9812:9812"
```

4. 點選 **Deploy the stack**

---

## 步驟四：反向代理（選填）

若要透過自訂網域（例如 `facebed.yourdomain.com`）存取，請設定反向代理指向 `http://127.0.0.1:9812`。

**Caddy 範例：**

```
facebed.yourdomain.com {
    reverse_proxy localhost:9812
}
```

**Nginx 範例：**

```nginx
server {
    listen 80;
    server_name facebed.yourdomain.com;
    location / {
        proxy_pass http://127.0.0.1:9812;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

設定完成後，你的 facebed 即可透過 `https://facebed.yourdomain.com` 存取，可取代 `facebed.com` 使用。

---

## 更新至最新版本

推送新 commit 到 `main` 後，GitHub Actions 自動更新映像。在 Portainer 拉取最新版：

**Portainer** → **Stacks** → 選擇 `facebed` → **Pull and redeploy**

---

## 以自訂網域取代 facebed.com

部署完成後，可在以下地方把 `facebed.com` 替換為自己的網域（假設你的網域為 `facebed.example.com`）。

### embed-fixer-bot

在 `src/cogs/embed_fixer.py` 的 `DOMAINS` 中找到 Facebook 設定，將 `new` 的值改為你的網域：

```python
"fix_methods": {
    "facebed": [
        {"old": "facebook.com", "new": "facebed.example.com"},
    ],
},
```

修改後重啟 bot，之後 Facebook 連結會自動轉換為：

```
https://www.facebook.com/share/p/AbCdEf/
→ https://facebed.example.com/share/p/AbCdEf/
```

### Vencord（手動貼上連結時自動替換）

使用 **Vencord** 的 **RegexReplace** 外掛（Settings → Plugins → RegexReplace），新增規則：

| 欄位 | 值 |
|------|----|
| Find | `https://(www\.)?facebook\.com/(.*)` |
| Replace | `https://facebed.example.com/$2` |

設定後，在 Discord 輸入框貼上 Facebook 連結時會自動替換為你的 facebed 網域。

**範例：**
```
貼入：https://www.facebook.com/share/p/AbCdEf/
自動改為：https://facebed.example.com/share/p/AbCdEf/
```

---

## 常見問題

**Q：部署時出現 `not a directory` / `mount … is not a directory` 錯誤**
→ 代表 `/opt/facebed/config.yaml` 在主機上不存在，Docker 把來源路徑建成了目錄。請先在主機執行步驟二建立設定檔，再重新 Deploy。

**Q：Container 啟動後立刻停止**
→ Portainer → Container → **Logs** 查看原因。最常見為 `config.yaml` 路徑不存在或格式錯誤。

**Q：映像拉取失敗（unauthorized）**
→ 確認 GHCR 套件已設為 Public，或在 Portainer → Registries 新增 GHCR 認證。

**Q：時間顯示不對**
→ 修改 `config.yaml` 的 `timezone` 值（整數，UTC+N），儲存後重啟 Container。

**Q：貼文顯示「Log in or sign up to view」**
→ facebed 目前不建議使用 cookies 功能，此為需登入才能瀏覽的貼文，無法處理。
