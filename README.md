# PikPak 挂载群晖 Synology NAS 完整指南：WebDAV 怎么配置？rclone 怎么用？免费用户有没有替代方案？（附 PikPak 套餐对比与邀请码优惠）

你有没有遇到过这种情况：PikPak 里存了一堆视频，结果每次看都要打开网页或者 App 手动找——能不能直接挂到群晖上，让 Plex、Infuse 或者 Emby 自动扫到？

答案是可以的，但要分情况说，因为 PikPak 跟群晖的联动方式不止一种，每种各有限制。这篇文章就把 **PikPak Synology** 这个组合能玩的几种方式都理清楚，从最省心的 WebDAV 到更灵活的 rclone 方案，逐一讲清楚步骤和坑点。

---

## 先搞清楚：PikPak 能跟群晖怎么连？

目前 PikPak 跟群晖 NAS 的集成方式主要有以下三条路：

1. **PikPak 官方 WebDAV → 群晖 Cloud Sync / File Station**（需要 Premium 会员，只读）
2. **rclone + Docker 在群晖上跑**（免会员可用，功能最完整）
3. **第三方工具 CloudsLinker / Alist** 作为中转，让群晖识别 PikPak 内容

这三种方式覆盖了从"我只是想把文件中转下来"到"我想直接挂载当网盘用"的不同需求，下面逐个讲解。

---

## 方案一：PikPak 官方 WebDAV 接入群晖（会员专属）

### PikPak WebDAV 是什么？

PikPak 在 2023 年底上线了 WebDAV 功能，入口在 **设置 → 实验室功能 → 启用 WebDAV**。开启后你会看到一个统一的连接地址（`http://dav.mypikpak.com`）以及专用的用户名和密码（注意：这不是你的账号登录密码，是单独生成的 WebDAV 凭证）。

有几点要先说清楚：
- **仅限 Premium 会员使用**，免费账户无法启用
- **目前仅支持读取操作**，不能通过 WebDAV 上传或修改文件
- Windows 系统偶尔会出现文件夹被识别为文件的兼容性问题

### 在群晖 DSM 上接入 PikPak WebDAV

**准备工作：**

在开始之前，先在 PikPak 网页端（`mypikpak.com/drive/all`）里打开设置，找到「实验室功能」，启用 WebDAV，记下以下三个信息：
- 统一连接点地址（通常是 `http://dav.mypikpak.com`）
- WebDAV 用户名
- WebDAV 密码

**在群晖 DSM 中操作：**

1. 打开群晖的 **套件中心**，搜索并安装 **Cloud Sync**
2. 打开 Cloud Sync，点击左下角「+」添加新同步任务
3. 在云服务商列表里选择 **WebDAV**
4. 填入以下信息：
   - 服务器 URL：`http://dav.mypikpak.com`
   - 用户名：PikPak WebDAV 用户名
   - 密码：PikPak WebDAV 密码
5. 选择 PikPak 中要同步的文件夹，以及群晖上的目标路径
6. 同步模式建议选「下载远端变更」（单向同步，从 PikPak 拉到群晖）

设置完成后，Cloud Sync 就会自动把 PikPak 里的内容同步到群晖本地，之后 Plex 或 Emby 扫描本地文件夹就能识别了。

> 💡 **提示**：如果你不需要长期同步，只是偶尔想把某个文件夹搬到群晖，也可以直接用群晖的 **File Station** 连接 WebDAV，手动拷贝即可。

---

## 方案二：rclone 在群晖上挂载 PikPak（免会员可用，功能更强）

这是目前最灵活的方案，也是很多 NAS 玩家的首选。rclone 是一个开源的云存储命令行工具，原生支持 PikPak 协议，不需要走 WebDAV，功能比官方 WebDAV 更完整，而且**不要求 PikPak Premium 会员**。

### 核心原理

rclone 直接用 PikPak 的 API 协议连接账号，然后可以作为一个本地 WebDAV 服务运行起来，让群晖的 File Station 或 Emby 等应用通过 HTTP 访问 PikPak 内容——就像访问本地文件夹一样。

### 在群晖上安装 rclone（通过 SSH）

1. 在群晖控制面板中启用 SSH（控制面板 → 终端机和 SNMP → 启用 SSH）
2. 用 SSH 工具（如 MobaXterm 或终端）连接群晖
3. 切换到 root：`sudo -i`
4. 安装 rclone：
   bash
   curl https://rclone.org/install.sh | sudo bash
   

### 配置 PikPak 远程源

bash
rclone config


交互步骤：
- 选 `n` 创建新远程
- 名称输入 `my-pikpak`（任意，建议含 pikpak 便于区分）
- 存储类型列表中找到 **PikPak**，输入对应编号
- 输入 PikPak 的登录邮箱（或手机号加区号，如 `+86138xxxx`）
- 输入密码

配置完成后，测试一下：
bash
rclone ls my-pikpak:'My Pack'


如果看到文件列表，说明连接成功。

### 启动 WebDAV 服务让群晖挂载

bash
rclone serve webdav \
  --addr 0.0.0.0:8000 \
  --cache-dir /volume1/.cache/rclone \
  --vfs-cache-mode full \
  my-pikpak:'My Pack'


这条命令会在群晖本机的 8000 端口启动一个 WebDAV 服务，所有来自 PikPak 的文件通过这个端口暴露出来。

之后在群晖的 File Station 或者 Emby 的媒体库设置里，添加 WebDAV 源：
- 地址：`http://127.0.0.1:8000`（群晖内部访问）
- 无需用户名密码（默认无认证，如需加认证自行加 `--user` 和 `--pass` 参数）

### 让 rclone 随群晖开机自启

最简单的方式是写一个启动脚本，放到 `/usr/local/etc/rc.d/` 目录下，或者通过群晖的「任务计划」设置一个触发式任务，在群晖启动时自动执行上面的挂载命令。

---

## 方案三：通过 Alist 或 CloudsLinker 中转（图形化，小白友好）

如果你不喜欢命令行操作，可以用 **Alist**（一个支持多云存储的文件管理器）配合 Docker 在群晖上跑，Alist 支持 PikPak 直接添加，配置完成后会生成一个本地 WebDAV / HTTP 访问地址，群晖的 Cloud Sync 或 Emby 直接接入即可。

**群晖上部署 Alist 的大致步骤：**
1. 打开群晖的 **Container Manager**（DSM 7 的 Docker 管理器）
2. 搜索 `alist` 镜像，拉取并运行
3. 映射端口（默认 5244）和存储路径
4. 在浏览器里访问 `http://群晖IP:5244`，进入 Alist 管理界面
5. 添加存储 → 选 PikPak → 填入账号密码
6. 完成后 Alist 会生成 WebDAV 地址，直接在群晖里使用

这个方案的好处是图形界面，不用敲命令，坏处是多了一层中间件，性能略有损耗。

---

## PikPak 适合 NAS 用户吗？对比分析

很多群晖用户问：我都有 NAS 了，还要 PikPak 干嘛？

其实两者的使用场景互补，而不是替代关系：

| 对比维度 | 群晖 NAS | PikPak |
|---------|---------|--------|
| 存储位置 | 本地物理硬盘 | 云端服务器 |
| 离线下载 | 需要下载到本地再保存 | 直接在云端完成下载，秒存 |
| 带宽需求 | 播放需要稳定上行 | 云端直播，无需上行带宽 |
| 冷门资源 | 本地做种少，速度慢 | 服务器集群加速，秒存 |
| 初始成本 | 硬件 + 硬盘，几千起步 | 按年订阅，几百元内 |
| 隐私 | 完全自有 | 加密传输，但文件存在云端 |

很多 NAS 用户的实际用法是：**用 PikPak 先把磁力/BT 资源云端下好，再同步到群晖本地归档**。这样既解决了冷门资源下载慢的问题，又能把文件永久留在自己的硬盘上。

👉 [用邀请码注册 PikPak，有机会获得免费 Premium 会员体验](https://mypikpak.com?invitation-code=74098243)

---

## PikPak 套餐一览（全部方案对比）

PikPak 的会员体系分为**全球会员**和**区域会员**两类。系统会根据你的 IP、设备语言、SIM 卡信息自动判断并提供对应套餐。

| 套餐类型 | 存储空间 | 下载速度 | 适用地区 | 计费周期 | 参考价格 | 购买链接 |
|---------|---------|---------|---------|---------|---------|---------|
| 免费版 | 6 GB | 基础速度 | 全球 | 永久免费 | 免费 |  [注册体验](https://mypikpak.com?invitation-code=74098243) |
| 区域会员（月付） | 10 TB | 8 MB/s | 部分区域 | 按月 | 约 ¥10（首月）/ ¥30（续费） |  [立即订阅](https://mypikpak.com?invitation-code=74098243) |
| 区域会员（年付） | 10 TB | 8 MB/s | 部分区域 | 按年 | 约 ¥211（首年） |  [立即订阅](https://mypikpak.com?invitation-code=74098243) |
| 全球会员（月付） | 10 TB | 最高 20 MB/s | 全球不限 | 按月 | 约 $5.79–$10 |  [立即订阅](https://mypikpak.com?invitation-code=74098243) |
| 全球会员（年付） | 10 TB | 最高 20 MB/s | 全球不限 | 按年 | 约 $57.59–$110 |  [立即订阅](https://mypikpak.com?invitation-code=74098243) |

**几个注意事项：**
- 全球会员适合美国、日本、德国、英国等 33 个发达国家的用户，旅行或跨国使用更稳定
- 区域会员价格更低，但只在特定地区有效，不能在发达国家激活
- 购买兑换码时注意区分是「全球会员码」还是「区域会员码」，两者不通用
- 免费试用体验码使用后不会自动扣费，到期前 PikPak 会主动提醒

---

## 关于邀请码：新用户怎么获得优惠

目前 PikPak 面向新用户的主要优惠机制是**邀请码注册**，通过邀请码注册的用户有机会获得免费 Premium 会员体验资格，相当于可以白嫖一段时间的会员权限，顺便测试一下 WebDAV 功能到底好不好用。

邀请码：**74098243**

👉 [点击这里用邀请码注册 PikPak](https://mypikpak.com?invitation-code=74098243)

注册后登录网页端，进入「设置 → 实验室功能」就可以尝试开启 WebDAV，然后按本文的方案一直接在群晖 Cloud Sync 里接入。

---

## 常见问题 FAQ

**Q：rclone 方案需要 PikPak 会员吗？**

不需要。rclone 直接通过 PikPak 的 API 连接，绕开了官方 WebDAV 的会员限制，免费用户也可以用 rclone 访问和下载 PikPak 的文件。

**Q：挂载后能直接用 Plex / Emby 扫描 PikPak 里的视频吗？**

可以，但要注意缓存设置。rclone 用 `--vfs-cache-mode full` 参数时，播放前会先缓存到本地，对群晖的存储有一定占用。如果只是想偶尔浏览，可以用 `--vfs-cache-mode minimal` 减少缓存。

**Q：PikPak WebDAV 为什么只读？**

目前 PikPak 官方 WebDAV 实现只支持读取，写入、移动、复制功能在开发中，尚未开放。如果需要完整读写功能，用 rclone 方案（rclone 支持向 PikPak 上传文件）。

**Q：群晖 Cloud Sync 和 rclone 方案哪个适合我？**

- 想定期把 PikPak 的内容自动同步到群晖本地 → **Cloud Sync 方案**（配合官方 WebDAV，需要 Premium）
- 想直接挂载 PikPak 当网盘用，不占本地空间 → **rclone 方案**（免费即可，功能更强）

**Q：PikPak 挂载会影响群晖性能吗？**

rclone 本身占用资源较少，主要消耗网络带宽。挂载后播放 4K 视频建议至少有稳定的 5 MB/s 以上网速，否则容易缓冲。

---

把 PikPak 和群晖 Synology NAS 结合起来用，其实是一个蛮实用的组合——PikPak 负责搞资源（磁力、BT、Telegram），群晖负责长期归档和局域网播放，两者分工明确，互补性很强。无论你选 WebDAV 直连还是 rclone 挂载，上面的步骤走下来基本都能跑通，遇到问题多半是端口或路径配置的细节问题，对照一遍就能解决。

👉 [还没注册 PikPak？用邀请码 74098243 注册，获得免费会员体验机会](https://mypikpak.com?invitation-code=74098243)
