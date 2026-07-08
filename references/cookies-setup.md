# Cookie 设置（一次性）

## 为什么需要 Cookies

B 站视频字幕 + 视频下载需要登录态。YouTube 下载无 cookie 时高概率返回 412/403（bot 检测）。
yt-dlp 的 `--cookies-from-browser` 在 Windows 上因 DPAPI 解密兼容性问题不可用，因此采用浏览器扩展导出 cookies 文件的方案。

---

## B站 Cookies

### 1. 安装扩展

Chrome Web Store 搜索 **"Get cookies.txt LOCALLY"** 并安装。

直达链接: https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc

### 2. 导出 Cookies

1. 打开 `https://www.bilibili.com/`，确保已登录
2. 点击浏览器工具栏中的扩展图标
3. 点击 **Export** 按钮
4. 保存为 `C:\Users\<用户名>\bilibili_cookies.txt`

### 3. 验证

```powershell
yt-dlp --no-update --cookies "$env:USERPROFILE\bilibili_cookies.txt" --list-subs "https://www.bilibili.com/video/BV1ksJ3zeE7s/"
```

### 4. 维护

Cookies 有效期约 30 天。过期后重复步骤 2 重新导出即可。

---

## YouTube Cookies

### 1. 导出 Cookies

1. 打开 `https://www.youtube.com/`，确保已登录
2. 点击扩展图标 → **Export**
3. 保存为 `C:\Users\<用户名>\youtube_cookies.txt`

> **注意**：YouTube 不强制登录态也能下载公开视频的字幕，但无 cookie 时 yt-dlp 高概率被限流返回 HTTP 412。
> 配置 cookies 后下载成功率显著提升。

### 2. 验证

```powershell
yt-dlp --no-update --cookies "$env:USERPROFILE\youtube_cookies.txt" --dump-json "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
```
