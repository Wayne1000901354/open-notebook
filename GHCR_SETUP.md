# GitHub Container Registry (GHCR) Deployment Guide

這份文件說明如何使用我們剛剛建立的 GitHub Actions Workflow 來自動化構建 Docker Image，並在您的 NAS 上使用。

## 1. Image 命名建議

我們採用 GitHub 官方建議的標準命名方式：

- **Registry Domain**: `ghcr.io`
- **Namespace**: 您的 GitHub 用戶名 (e.g., `wayne1000901354`)
- **Image Name**: 倉庫名稱 (e.g., `open-notebook`)

**完整 Image URL**:
```
ghcr.io/wayne1000901354/open-notebook:latest
```

## 2. 適用於 Private Repository 的設定

如果您的 GitHub 倉庫是私有的 (Private)，或者您希望在未公開的情況下讓 NAS 下載 Image，您需要建立一個 **Personal Access Token (PAT)**。

### 步驟 A: 建立 Token
1. 前往 GitHub [Developer Settings > Personal access tokens > Tokens (classic)](https://github.com/settings/tokens).
2. 點擊 **Generate new token (classic)**.
3. **Scopes (權限)**: 務必勾選 `read:packages`. (如果需要上傳則勾選 `write:packages`, 但 NAS 只需要 read)
4. 產生並複製這個 Token (以 `ghp_` 開頭)。

### 步驟 B: 在 NAS (或任何 Docker Client) 上登入
在您的 NAS 終端機 (SSH) 或 Docker 管理介面中執行登入：

```bash
# 使用您的 GitHub 用戶名和剛才產生的 PAT
docker login ghcr.io -u Wayne1000901354 -p ghp_YourPersonalAccessToken...
```

登入成功後，您就有權限拉取 (Pull) 私有倉庫的 Image 了。

## 3. NAS 更新流程

### 方法 A: Docker Compose (推薦)
在 NAS 上建立 `docker-compose.yml`：

```yaml
version: '3.8'
services:
  open_notebook:
    image: ghcr.io/wayne1000901354/open-notebook:latest
    pull_policy: always  # 強制每次啟動檢查更新
    restart: unless-stopped
    ports:
      - "8502:8502"
    environment:
      - BATCH_UPLOAD_LIMIT=50
    # ... 其他設定 ...
```

更新時只需執行：
```bash
docker-compose pull
docker-compose up -d
```

### 方法 B: Watchtower (自動更新)
您也可以在 NAS 上運行 [Watchtower](https://github.com/containrrr/watchtower) 容器，它會定時檢查 GHCR 是否有新版本並自動更新您的容器。

**注意**: 對於 GHCR 私有倉庫，Watchtower 也需要配置上述的登入憑證 (通常透過掛載 `config.json` 或環境變數)。

## 4. Workflow 運作方式
- **觸發時機**: 每次 Push 到 `main` 分支時。
- **Tag 策略**:
  - `latest`: 對應 main 分支最新版。
  - `sha-xxxxxxx`: 對應特定的 Commit hash (適合回退版本)。
  - `v1.0.0`: 如果您在 Git 打上 Tag，會自動構建對應的版本號 Image。
