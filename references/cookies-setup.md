# B站 Cookie 设置（一次性）

## 为什么需要 Cookies

B 站视频字幕下载需要登录态。yt-dlp 的 `--cookies-from-browser` 在 Windows 上因 DPAPI 解密兼容性问题不可用，因此采用浏览器扩展导出 cookies 文件的方案。

## 设置步骤

### 1. 安装扩展

Chrome Web Store 搜索 **"Get cookies.txt LOCALLY"** 并安装。

直达链接: https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc

### 2. 导出 Cookies

1. 打开 `https://www.bilibili.com/`，确保已登录
2. 点击浏览器工具栏中的扩展图标
3. 点击 **Export** 按钮
4. 将文件保存为：

   - **Windows**: `C:\Users\<用户名>\bilibili_cookies.txt`
   - **macOS/Linux**: `~/bilibili_cookies.txt`

### 3. 验证

```powershell
# Windows
yt-dlp --no-update --cookies "$env:USERPROFILE\bilibili_cookies.txt" --list-subs "https://www.bilibili.com/video/BV1ksJ3zeE7s/"
```

```bash
# macOS/Linux
yt-dlp --cookies ~/bilibili_cookies.txt --list-subs "https://www.bilibili.com/video/BV1ksJ3zeE7s/"
```

应该看到 `ai-zh`、`ai-en` 等字幕轨。

### 4. 维护

Cookies 有效期约 30 天。过期后重复步骤 2 重新导出即可。

---

## 备选方案（如果不想装扩展）

### 方案 A: Firefox 浏览器（仅 Linux/macOS）

Firefox 的 cookie 存储不使用 DPAPI 加密，yt-dlp 可以直接读取：

```bash
yt-dlp --cookies-from-browser firefox --cookies ~/bilibili_cookies.txt \
  --skip-download -i "https://www.bilibili.com/"
```

### 方案 B: macOS Keychain

macOS 上 Chrome cookies 可通过 Keychain 解密：

```bash
yt-dlp --cookies-from-browser chrome --cookies ~/bilibili_cookies.txt \
  --skip-download -i "https://www.bilibili.com/"
```

然后后续调用使用 `--cookies ~/bilibili_cookies.txt`。
