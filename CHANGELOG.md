# OpenCut 折腾记录

从想搞个 exe 开始，到 docker 被墙，最后用 bun 跑起来。全是坑。

---

## 桌面端编译（放弃了）

写了 `build-opencut.bat` 和 `build-opencut.ps1`，用 Rust 编译 opencut-classic 的 `apps/desktop`。

装完 VS Build Tools、Rust，确实编出了 `opencut.exe`，打开一看就一个黑底白字写着 "OpenCut"，什么都没有。

README 里写了 desktop 是 "(in progress)"，只是个 GPUI 窗口框架，编辑器功能还没移植过来。

**踩的坑：**
- 编译出来叫 `opencut.exe` 不叫 `opencut-desktop.exe`，脚本找错了文件名
- VS Build Tools 要勾选 "Desktop development with C++"

---

## Docker 部署（也放弃了）

写了 `deploy-opencut.ps1`，用 docker compose 一键拉 PostgreSQL、Redis、Next.js。

代码是能拉下来，但 upstream 的 main 分支根本编不过去，有几个 TypeScript 类型错误：

- `keybinding.ts` 少了 `isShortcutKey` 函数
- `definitions.ts` 少了 `isActionWithOptionalArgs` 函数
- `runner.ts` 里 `IndexedDBAdapter` 构造函数和 set 方法的参数格式变了，但调用处没改

这些都是 [Issue #809](https://github.com/OpenCut-app/OpenCut/issues/809)。

所以脚本里加了自动打补丁的逻辑，克隆完代码自动修这三个问题。

**更大问题是网络。** docker 镜像和 npm 包全在境外，动不动超时。`bun install` 快 2000 个包，断了就得重来。试了好几次都没跑通。

---

## 现在用的方案：bun 跑开发模式

不折腾生产编译了，`next dev` （开发模式）跳过严格类型检查，补丁打完就能跑。

**需要的东西：**
- Git
- Bun

**补丁：** 和上面 docker 那三个一样，自动打。

**为什么不用 Turbo：** 从旧电脑拷项目到新电脑后，Turbo 缓存里带了旧电脑的用户路径，Windows 报 code 53（找不到网络路径），直接 `next dev --turbopack` 就行，不需要 Turbo。

---

## 脚本里踩过的坑

**bat 里 `!` 字符被吞掉**
PowerShell 在 bat 的 `()` 块里，`!` 会被延迟变量展开吃掉，生成的 TypeScript 代码 `!actionNames.includes(value)` 变成了 `actionNames.includes(value)`，语法乱套。后来补丁逻辑用 Here-String 写在 ps1 里，不再走 bat 中转。

**git clone 输出走 stderr 被当成错误**
`$ErrorActionPreference = "Stop"` 时，git 的进度输出（Cloning into...）走 stderr，PowerShell 当成异常直接炸了。改成 `"Continue"` 解决。

**端口被占用**
新电脑 3000 端口可能被别的程序占了，加了自动检测并切换到空闲端口。

**npm 被墙**
加了 `bunfig.toml` 自动设置淘宝镜像 `registry.npmmirror.com`。

**GitHub 被墙**
加了 gitclone.com 作为备用克隆地址。

---

## 俩脚本

**`start-opencut.bat`** — 丢在项目文件夹里，双击就启动。第一次运行会自动调 run-opencut.ps1，装完以后直接启动 + 打开 Chrome。

**`run-opencut.ps1`** — 在没装过的电脑上一条龙搞定。自动装 Git、装 Bun、克隆代码、打补丁、写环境变量、装依赖、启动。支持 `-Port` 指定端口，`-Update` 拉最新代码。

---

## 搬到新电脑

两个文件拷过去，双击 start-opencut.bat 就行。第一次会自动跑 run-opencut.ps1 装一切。

如果有 GitHub 访问问题，最稳的办法是从旧电脑直接把 `.bun` 文件夹和 `opencut-local` 文件夹拷过去，完全不走网络。

---

## 还没解决的问题

- 开发模式没有 PostgreSQL 和 Redis，项目管理、登录这些功能用不了
- 编辑器本身（时间线、预览）可以正常用
- 真要脱离浏览器跑，只能等官方 Rust 重写完成以后编译 exe
- 主仓库 `opencut-app/opencut` 还在重写中，现在能用的就是 `opencut-classic` 这个存档仓库

2026-08-10
