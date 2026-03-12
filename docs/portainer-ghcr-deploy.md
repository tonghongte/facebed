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

在伺服器上建立設定目錄並編寫設定：

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
3. **Build method** 選 **Web editor**，貼入以下內容：

```yaml
services:
  facebed:
    image: ghcr.io/tonghongte/facebed:main
    container_name: facebed
    restart: unless-stopped
    ports:
      - "9812:9812"
    volumes:
      - /opt/facebed/config.yaml:/facebed/config.yaml:ro
```

4. 點選 **Deploy the stack**

---

## 步驟四：反向代理（選填）

若要透過網域名稱存取（例如 `facebed.yourdomain.com`），請在 Nginx / Caddy / Traefik 設定反向代理指向 `http://伺服器IP:9812`。

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
        proxy_pass http://localhost:9812;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 更新至最新版本

推送新 commit 到 `main` 後，GitHub Actions 自動更新映像。在 Portainer 拉取最新版：

**Portainer** → **Stacks** → 選擇 `facebed` → **Pull and redeploy**

---

## 搭配 embed-fixer-bot 使用

若你同時部署了 [embed-fixer-bot](https://github.com/tonghongte/embed-fixer-bot)，可將 Facebook 修復服務改指向自己的 facebed 實例。

在 `embed_fixer.py` 的 `DOMAINS` 中找到 Facebook 設定：

```python
"fix_methods": {
    "facebed": [
        {"old": "facebook.com", "new": "facebed.yourdomain.com"},
    ],
},
```

將 `facebed.yourdomain.com` 替換為你的 facebed 網域。

---

## 常見問題

**Q：Container 啟動後立刻停止**
→ Portainer → Container → **Logs** 查看原因。最常見為 `config.yaml` 路徑不存在或格式錯誤。

**Q：映像拉取失敗（unauthorized）**
→ 確認 GHCR 套件已設為 Public，或在 Portainer → Registries 新增 GHCR 認證。

**Q：時間顯示不對**
→ 修改 `config.yaml` 的 `timezone` 值（整數，UTC+N），儲存後重啟 Container。

**Q：貼文顯示「Log in or sign up to view」**
→ facebed 目前不建議使用 cookies 功能，此為需登入才能瀏覽的貼文，無法處理。
