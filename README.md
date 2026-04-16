# gitmemo-release

GitMemo 的签名与分发仓库。负责对开发仓库构建产出的未签名 DMG 进行 Apple 代码签名、公证（Notarization）、装订（Stapling），并最终发布 GitHub Release。

开发者无需接触本仓库，所有 Apple 凭证仅在此处管理。

---

## 工作流程

```
sahadev/GitMemo (开发仓库)
  push to main → 构建未签名 DMG
      ↓ repository_dispatch
sahadev/gitmemo-release (本仓库)
  下载 artifact → 签名 → 公证 → staple → 发布 Release 到开发仓库
```

---

## 所需凭证一览

本仓库共需配置 **8 个 GitHub Secrets**，分为三类。

### 一、Apple 代码签名（.p12 证书）

用于对 `.app` 和 `.dmg` 文件进行代码签名，使 macOS Gatekeeper 信任该应用。

#### `APPLE_CERTIFICATE`

**是什么**：Developer ID Application 证书，Base64 编码的 `.p12` 文件内容。

**如何获取**：
1. 打开 **Keychain Access**
2. 在左侧选 **login** keychain，分类选 **My Certificates**
3. 找到 `Developer ID Application: <你的名字> (<Team ID>)`
4. 右键 → **Export** → 保存为 `.p12`，设置一个导出密码
5. 终端执行：
   ```bash
   base64 -i certificate.p12 | pbcopy
   ```
6. 将复制的内容粘贴为 Secret 值

> 如果还没有证书：登录 [Apple Developer](https://developer.apple.com/account/resources/certificates/list) → Certificates → 点击 **+** → 选 **Developer ID Application** → 按指引生成并下载 → 双击导入 Keychain。

---

#### `APPLE_CERTIFICATE_PASSWORD`

**是什么**：导出 `.p12` 时设置的密码。

**如何获取**：就是上一步你自己设置的那个密码，直接填入。

---

#### `APPLE_SIGNING_IDENTITY`

**是什么**：证书的完整 Common Name，`codesign` 命令用来查找 Keychain 中对应证书。

**如何获取**：终端执行：
```bash
security find-identity -v -p codesigning | grep "Developer ID Application"
```
输出示例：
```
1) ABC123DEF456... "Developer ID Application: Zhang San (A1B2C3D4E5)"
```
将引号内的完整字符串（如 `Developer ID Application: Zhang San (A1B2C3D4E5)`）填入。

---

### 二、Apple 公证认证（.p8 API 密钥）

用于向 Apple Notary Service 提交应用进行安全扫描，获得公证票据。  
与 `.p12` 职责完全不同：`.p12` 是"签谁的名"，`.p8` 是"证明你有权提交公证"。

#### `APPLE_API_KEY`

**是什么**：App Store Connect API 密钥文件（`.p8`）的完整文本内容。

**如何获取**：
1. 打开 [App Store Connect → Users and Access → Integrations → API Keys](https://appstoreconnect.apple.com/access/integrations/api)
2. 点击 **Generate API Key**（或 **+**）
3. Name 随意填，Role 选 **Developer**
4. 下载 `.p8` 文件（**只能下载一次**，请妥善保存）
5. 终端执行：
   ```bash
   cat AuthKey_XXXXXXXXXX.p8 | pbcopy
   ```
6. 将复制的完整内容（包含 `-----BEGIN PRIVATE KEY-----` 和 `-----END PRIVATE KEY-----`）填入

---

#### `APPLE_API_KEY_ID`

**是什么**：API 密钥的 Key ID，10 位字母数字字符串。

**如何获取**：在上一步 App Store Connect 的 API Keys 页面，生成密钥后可以看到 **Key ID** 列，例如 `ABC123DEF4`。

---

#### `APPLE_API_ISSUER_ID`

**是什么**：你的 App Store Connect 团队的 Issuer ID，UUID 格式。

**如何获取**：在同一页面（API Keys）的顶部，有一行灰色小字 **Issuer ID**，例如 `12345678-1234-1234-1234-123456789abc`。

---

### 三、GitHub PAT（跨仓库操作令牌）

用于从开发仓库下载构建产物，以及在开发仓库发布 Release。

创建入口：**GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**

#### `ARTIFACTS_TOKEN`

**是什么**：用于从开发仓库（`sahadev/GitMemo`）下载 GitHub Actions artifact。

**如何创建**：

| 字段 | 值 |
|------|----|
| Token name | `gitmemo-artifacts-read`（随意） |
| Resource owner | 开发仓库所在账号 |
| Repository access | Only selected repositories → 选 `GitMemo` |
| Permissions → Actions | **Read-only** |

---

#### `RELEASE_TOKEN`

**是什么**：用于在开发仓库（`sahadev/GitMemo`）创建 GitHub Release 并上传文件。

**如何创建**：

| 字段 | 值 |
|------|----|
| Token name | `gitmemo-release-write`（随意） |
| Resource owner | 开发仓库所在账号 |
| Repository access | Only selected repositories → 选 `GitMemo` |
| Permissions → Contents | **Read and write** |

---

## 开发仓库需配置的 Secrets

以下 2 个 Secret 需要配置在**开发仓库 `sahadev/GitMemo`** 中（不在本仓库）：

### `RELEASE_DISPATCH_TOKEN`

**是什么**：用于开发仓库 CI 向本仓库发送 `repository_dispatch` 触发信号。

**如何创建**：

| 字段 | 值 |
|------|----|
| Token name | `gitmemo-release-dispatch`（随意） |
| Resource owner | 本仓库所在账号（你自己） |
| Repository access | Only selected repositories → 选 `gitmemo-release` |
| Permissions → Contents | **Read and write** |

### `RELEASE_REPO`

**是什么**：本仓库的完整名称，供开发仓库 CI 知道向哪里发送 dispatch。

**值**：`sahadev/gitmemo-release`（替换为实际仓库路径）

---

## Secrets 配置汇总

| Secret | 配置在哪个仓库 | 类型 |
|--------|--------------|------|
| `APPLE_CERTIFICATE` | 本仓库 `gitmemo-release` | .p12 Base64 |
| `APPLE_CERTIFICATE_PASSWORD` | 本仓库 `gitmemo-release` | 文本 |
| `APPLE_SIGNING_IDENTITY` | 本仓库 `gitmemo-release` | 文本 |
| `APPLE_API_KEY` | 本仓库 `gitmemo-release` | .p8 文件内容 |
| `APPLE_API_KEY_ID` | 本仓库 `gitmemo-release` | 文本 |
| `APPLE_API_ISSUER_ID` | 本仓库 `gitmemo-release` | 文本 |
| `ARTIFACTS_TOKEN` | 本仓库 `gitmemo-release` | GitHub PAT |
| `RELEASE_TOKEN` | 本仓库 `gitmemo-release` | GitHub PAT |
| `RELEASE_DISPATCH_TOKEN` | 开发仓库 `GitMemo` | GitHub PAT |
| `RELEASE_REPO` | 开发仓库 `GitMemo` | 文本 |
