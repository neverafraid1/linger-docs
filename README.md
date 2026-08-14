# Linger

Linger 是一个使用 Rust 编写、兼容 Emby 客户端的视频媒体服务器，支持电影、剧集、本地视频和 `.strm` 媒体库。镜像内置 Web 管理端、FFmpeg/FFprobe，并使用 PostgreSQL 保存媒体、用户和播放状态；Redis 用作可丢弃的共享缓存。

当前不支持 Live TV、IPTV、DVR、音乐媒体库和 DLNA。云盘挂载、STRM 生成及下载整理应由外部工具完成。

## 隐私与联网

Linger 不会向项目方上传媒体文件、文件名、目录路径、媒体库名称、用户名、密码、API Key、播放记录、设备信息或客户端信息。媒体、图片、数据库、日志和备份均保留在你自己的存储与 Docker 卷中。

联网仅发生在你主动启用的功能：配置 TMDB、TheTVDB、fanart.tv 或 OpenSubtitles 后，Linger 会向相应第三方查询你请求刮削的标题或 Provider ID；这些请求受对应服务的隐私政策约束。Free 和 Pro 均会向授权服务登记服务器 ID、安装公钥和版本，用于验证许可证，不包含用户或媒体业务数据。

只有 Pro 会额外上报一份可覆盖的静态聚合快照，用于统计安装规模：用户数、媒体库数、电影数、剧集数、单集数和 Linger 版本。它不包含任何媒体名称、路径、账户、播放或设备信息；只用于总量统计，不用于广告、画像、销售、内容分析或向第三方共享。介意此统计时可继续使用 Free，功能不受影响。

## 社区交流

如需交流使用经验或获取版本动态，请加入 [Linger Telegram 群](https://t.me/+P5yINqUERM04Mzk9)。遇到 Bug 时，欢迎发送邮件至 [linger_good_luck@proton.me](mailto:linger_good_luck@proton.me)，或在 [文档仓库提交 Issue](https://github.com/neverafraid1/linger-docs/issues)。请附上 Linger 版本、复现步骤、预期与实际结果；日志和截图可帮助定位，但不要提交密码、API Key、服务器地址或媒体路径。

## 按步骤部署

### 1. 准备部署目录与 `.env`

准备一台安装了 Docker Engine 与 Docker Compose 的主机，然后创建部署目录：

```sh
mkdir -p linger/media
cd linger
```

先新建 `.env`，数据库密码请只使用字母和数字：

```dotenv
POSTGRES_PASSWORD=
# Linger 进程监听端口；仅 host 网络模式或直接运行时通常需要修改
SERVER_PORT=8096
# 默认桥接模式发布到宿主机的端口；通常只需修改这一项
HOST_PORT=8096
# Redis 缓存上限；大媒体库或高并发可按可用内存提高到 2gb 或 4gb
REDIS_MAXMEMORY=1gb
MEDIA_PATH=./media
MEDIA_GID=1000
# 容器日志和定时任务的时区；请使用 IANA 时区名称
TZ=Asia/Shanghai
```

普通 Docker Compose 部署通常只需修改 `HOST_PORT`；host 网络模式则修改 `SERVER_PORT`。两者互不影响，也不会暴露 PostgreSQL 或 Redis 的容器内端口。`TZ` 可改为 `Asia/Tokyo`、`Europe/Berlin` 等 [IANA 时区名称](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)。`REDIS_MAXMEMORY` 设置偏小不会损坏媒体数据，但会更频繁淘汰缓存并增加 PostgreSQL 与磁盘读取。

`PROBE_SEMAPHORE_LIMIT` 控制读取视频编码、码率、时长和音视频流信息时的并发。默认**不需要配置**：Linger 启动时会读取容器可用的逻辑核数，并自动使用其中一半（至少 1 路），适合普通 NAS。若希望手动调节，在 `.env` 添加例如 `PROBE_SEMAPHORE_LIMIT=6`；显式配置时实际并发为 `min(设置值, 可用逻辑核数 - 1)`，单核机器保底为 1。媒体探测会短暂占满一个 CPU 核：普通 NAS 可设为 `1` 或 `2`，专用服务器可调高，但系统始终会保留一个逻辑核给数据库、请求处理和播放。

`MEDIA_PATH` 是宿主机媒体目录，`MEDIA_GID` 是该目录的数字组 ID。普通 Linux 目录通常可以保留默认值；绿联、群晖等 NAS 的共享目录如果仅允许所属组读取，请执行 `ls -ldn <媒体目录>`，把输出中的组 ID 填入 `MEDIA_GID`。例如目录显示 `1002 10`，应配置：

```dotenv
MEDIA_PATH=/volume1/docker/my_emby/media
MEDIA_GID=10
```

这种方式只让 Linger 加入媒体目录所属组，不修改宿主机权限，也不会让容器以 root 运行。

### 2. 创建 `docker-compose.yml`

新建 `docker-compose.yml`：

```yaml
name: linger

services:
  app:
    container_name: linger
    image: ghcr.io/neverafraid1/linger:latest
    ports:
      - "${HOST_PORT:-8096}:${SERVER_PORT:-8096}"
    group_add:
      - "${MEDIA_GID:-1000}"
    environment:
      SERVER_PORT: ${SERVER_PORT:-8096}
      DATABASE_URL: postgres://linger:${POSTGRES_PASSWORD:?请在 .env 中设置 POSTGRES_PASSWORD}@db:5432/linger
      REDIS_URL: redis://redis:6379/0
      TZ: ${TZ:-Asia/Shanghai}
      # 0 为自动值；在 .env 设置正整数即可手动限制
      PROBE_SEMAPHORE_LIMIT: ${PROBE_SEMAPHORE_LIMIT:-0}
    volumes:
      - ${MEDIA_PATH:-./media}:/media:ro
      - linger-data:/data
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  db:
    container_name: linger-db
    image: pgvector/pgvector:pg18-trixie
    environment:
      POSTGRES_USER: linger
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?请在 .env 中设置 POSTGRES_PASSWORD}
      POSTGRES_DB: linger
    volumes:
      - linger-postgres18-data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U linger -d linger"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    restart: unless-stopped

  redis:
    container_name: linger-redis
    image: redis:8-alpine
    command: redis-server --save "" --appendonly no --maxmemory ${REDIS_MAXMEMORY:-1gb} --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

volumes:
  linger-data:
  linger-postgres18-data:
```

媒体目录只读挂载到容器内的 `/media`。默认桥接模式下需要避开宿主机端口冲突时只修改 `HOST_PORT`；只有 host 网络模式或直接运行 Linger 时才需要修改 `SERVER_PORT`。

`.env` 中的 `POSTGRES_PASSWORD` 是 PostgreSQL 用户 `linger` 的密码，应用和数据库会自动使用同一个值。PostgreSQL 数据卷创建后，再修改该值不会自动更新数据库中的密码，反而会导致 Linger 无法连接数据库；已有部署请不要直接修改。

### 3. 启动并创建管理员

启动服务：

```sh
docker compose pull
docker compose up -d
docker compose ps
```

NAS 用户启动后可确认权限是否正确：

```sh
docker compose exec app id
docker compose exec app ls /media
```

`id` 应包含 `.env` 中配置的 `MEDIA_GID`，并且 `ls /media` 不应出现 `Permission denied`。如果仍无权读取，请确认 `MEDIA_PATH` 的每一级父目录都允许该组进入。

服务健康后，桥接模式访问 `http://服务器地址:<HOST_PORT>/web/`，host 网络模式访问 `http://服务器地址:<SERVER_PORT>/web/`；使用默认配置时均为 `http://服务器地址:8096/web/`。首次访问会要求创建管理员；Linger 不提供默认用户名或密码。PostgreSQL 和 Redis 只在 Compose 内部网络中使用，不会占用宿主机端口。

Compose 会创建两个持久化卷：

- `linger-data`：配置、图片、日志、缓存目录和备份文件。
- `linger-postgres18-data`：PostgreSQL 18 数据库。

Redis 只保存可重建缓存，不需要持久化。媒体目录默认以只读方式挂载到容器的 `/media`，创建媒体库时应选择 `/media` 下的目录。

### 4. 配置元数据与字幕服务

以管理员登录后，打开“服务器设置 → 元数据与字幕”。先填写 TMDB Key；它是电影、剧集、人物和大部分图片的首选来源。保存后无需重启。

| 服务 | 用途 | 申请方式 |
| --- | --- | --- |
| [TMDB](https://www.themoviedb.org/) | 电影、剧集、人物与图片，建议必配 | 注册并在 [账户 API 设置](https://www.themoviedb.org/settings/api) 创建 API Key |
| [TheTVDB](https://thetvdb.com/) | 剧集、季、分集与剧集图片补充 | 查看 [API 与许可说明](https://thetvdb.com/api-information)，按其当前流程申请 Key |
| [fanart.tv](https://fanart.tv/) | 海报、背景、Logo 等补充图片 | 注册后在 fanart.tv 账户页面申请个人 API Key |
| [OpenSubtitles](https://www.opensubtitles.com/) | 在线字幕搜索与下载 | 注册后在开发者/API 页面创建 API Key；账号密码可选，用于服务端支持的额度提升 |

不要把 Key 写入截图、Issue、日志或媒体库文件名。Key 保存在 Linger 的持久化数据库中，管理端仅显示脱敏状态；清空输入并保存会移除该服务的本地覆盖配置。

### 5. 创建媒体库并首次扫描

打开“媒体库 → 新建媒体库”，选择电影或电视，路径填写容器路径（通常为 `/media` 或其子目录），再选择简体中文和下载器顺序。建议元数据优先顺序为 TheMovieDb、TheTVDB；图片优先 TheMovieDb、fanart.tv、TheTVDB。首次扫描会识别文件名、下载元数据与图片；对远程 STRM 不会为首次扫描读取真实视频内容。

## 媒体文件与 STRM

支持常见视频、外挂字幕、NFO、`poster.jpg` 等本地伴随文件。剧集按“剧集 → 季 → 单集”组织；同一影片的多个媒体版本会合并为一个逻辑项目。

`.strm` 文件可以包含 HTTP(S) 地址或本地绝对路径。若内容是本地路径，该路径必须在 Linger 容器内可见；必要时为目标目录增加额外的只读卷挂载。只有确实需要从管理端删除源文件时，才应将媒体卷改为可写并启用媒体库的深度删除选项。

## 元数据与图片

每个媒体库都可以单独调整首选语言、元数据提供方和图片提供方顺序；设置会影响之后的扫描和手动刷新。中文媒体建议首选简体中文。

未配置某个服务的 Key 时，Linger 会跳过它，不会阻塞其他已配置来源，也不会影响播放、用户、扫描文件或已有元数据。若媒体库只选择了未配置的 TVDB，则该次元数据搜索/刮削没有可用远程来源，通常不会匹配到结果；远程图片搜索会返回“未配置 API Key”的明确错误。TVDB 缺失还意味着无法补全其独有的剧集、季和分集信息，fanart.tv 的电视剧图片也可能因缺少 TVDB ID 而减少。已有的 TMDB、NFO 或导入元数据不受影响。

NFO 和本地图片可在媒体库设置中配置为优先来源。修改提供方顺序或新增 Key 后，对已有媒体使用“刷新元数据”或“重新刮削”才会重新获取数据。

## 已有 Emby 用户：先并行尝试

建议不要一开始替换 Emby。先为 Linger 使用一个未冲突的 `HOST_PORT`，以只读方式挂载同一份媒体目录；然后在“媒体库 → 从 Emby 导入”中导入要尝试的媒体库。Linger 会复制受支持的媒体库设置、已刮削元数据、图片、人物和媒体信息，原 Emby 的数据库、文件和配置不会被修改。

导入完成后，可用同一 Emby 客户端分别连接两个服务，对比媒体显示、播放、进度和刮削结果。确认满足需求后再决定是否迁移用户或切换日常使用；在此之前保留 Emby 作为回退服务即可。

## 从 Emby 迁移

“媒体库”和“用户”页面均提供“从 Emby 导入”。填写 Linger 容器能够访问的 Emby 地址及管理员 API Key 后，可以全选或逐项选择。媒体库导入会同步创建或复用相同路径的库，先建立本地索引，再复制 Linger 支持的库设置、已刮削元数据、媒体流、章节、人物和图片；源路径与容器挂载不同时，应在界面修改路径映射。导入前应通过 `docker compose exec app ls /media` 确认容器可以读取媒体目录，否则媒体库会因目标路径不可用而失败。导入期间请保持页面打开。

Emby 不允许通过 API 导出密码哈希。导入的用户会保留受支持的权限、媒体库范围、播放偏好和头像，但默认处于禁用状态；管理员必须在 Linger 中设置新密码后再启用。API Key 只存在于本次请求内，不会保存到数据库。

## Free 与 Pro

Linger Free 最多支持 3 个用户（包括管理员）；Pro 永久解除用户数限制。管理员可在“Pro”页面查看服务器 ID、安装指纹和 Pro 申请码。需要 Pro 时，请发送邮件到 [linger_good_luck@proton.me](mailto:linger_good_luck@proton.me)，主题注明“Linger Pro 申请”，并附上管理端显示的完整 Pro 申请码。

Pro 不收费，也不是购买服务。申请邮件可以是一则真的好笑的笑话、一本值得反复读的好书、一部冷门好电影、一张精心整理的片单、一个 Emby 服务器白名单名额、一段有趣的使用故事、一条很有价值的改进建议、一次认真验证过的 Bug 报告，或者任何你觉得值得分享的小惊喜。内容由维护者凭心情判断是否授予，发送申请不保证获得 Pro；不要发送密码、API Key、私人数据、媒体文件或其他无权分享的内容。

如果 Linger 对你有帮助，也可以[通过爱发电自愿支持项目](https://ifdian.net/a/linger_good_luck)。赞助不会提高 Pro 申请的通过概率，也不附带授权或优先支持。

同一发件邮箱或同一服务器 ID 每 7 天只能申请一次；一周内重复发送会被加入拒收名单，后续申请不再处理。授权后在“Pro”页面点击“刷新证书”即可生效，无需重启服务。

为了只统计 Linger 的实际安装与使用规模，Pro 会在许可证刷新时上报一份静态聚合快照：用户数、媒体库数、电影数、剧集数、单集数和 Linger 版本。每台安装只覆盖保存最新一份，不记录历史行为；这份快照不包含用户名、邮箱、IP、媒体名称、文件路径、播放记录、设备或客户端信息。收集的唯一用途是统计使用量，不做其他用途：不用于广告、用户画像、销售或内容分析，也不向第三方共享。Free 完全不上报这份 usage；若介意任何用量统计，可以继续使用 Free，不影响正常功能。

Free 和 Pro 都需要发送服务器 ID、安装公钥、Linger 版本及签名挑战，用于登记安装、绑定证书和协议兼容；这些不是用量快照，也不包含用户或媒体业务数据。

Pro 需要定期联网刷新授权状态，签名凭证会在本地验证，修改系统时间不能延长有效期。正常联网重启会很快恢复 Pro；离线重启时暂按 Free 限制新增用户，但不会删除已有用户或影响播放，联网刷新后自动恢复。完整备份包含安装身份，恢复到同一实例后无需重新申请；不要单独删除 `/data/config`，否则会生成新的安装指纹。

## 更新与回滚

更新前先创建完整备份。跟随最新正式版时直接拉取；固定版本时将 Compose 中镜像的 `:latest` 改为具体版本，例如 `:0.0.8`：

```sh
docker compose pull app
docker compose up -d --force-recreate --no-deps app
docker compose logs --tail=100 app
```

固定版本标签不会被覆盖。需要回滚时将 Compose 中的镜像标签改回旧版本，再重复以上命令。不要执行 `docker compose down -v`，否则会删除数据库和 Linger 数据卷。

## 备份、恢复与密码找回

完整备份和恢复在 Web 管理端的备份页面操作，备份包含 PostgreSQL 数据以及持久化配置和图片。恢复期间服务会暂时不可用，并在完成后自动重启。恢复前请确认卷有足够空间。

忘记管理员密码时，在登录页发起密码找回，然后查看容器内的 PIN 文件：

```sh
docker compose exec app cat /data/passwordreset.txt
```

PIN 十分钟内有效，连续失败五次后失效。使用成功后密码会被清空，请立即登录并设置新密码。

## 常用运维命令

```sh
# 查看状态和日志
docker compose ps
docker compose logs -f --tail=200 app

# 重启 Linger，不影响数据库
docker compose restart app

# 检查服务和 FFmpeg 版本；桥接模式请将 8096 换成 HOST_PORT 的值
curl -fsS http://127.0.0.1:8096/System/Info/Public
docker compose exec app ffmpeg -version

# 停止服务但保留数据卷
docker compose down
```

管理端支持动态调整日志级别。大媒体库建议保留 Redis，并使用本地 SSD 存放 PostgreSQL 与 `linger-data`。默认数据库连接池为 20，通常无需调整。

## 许可证

本项目使用 [Elastic License 2.0](LICENSE)（SPDX：`Elastic-2.0`）。它允许使用、修改和再分发，但禁止绕过许可证功能、移除受许可证保护的功能或许可声明，也禁止将本软件的主要功能作为第三方托管服务提供。该许可证属于源码可用许可证，并非 OSI 认可的开源许可证。
