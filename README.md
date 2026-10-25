# worker-jobs

为 Worker 项目组织后台任务。当前只包含 IPTV，不提供通用任务框架，也不部署 Worker。

## 当前状态

这是待接入的最小版本：三个 workflow 只支持手动触发，且 `IPTV_JOBS_ENABLED` 默认为关闭。Cron 配置仅保留为注释，生产凭据正在接入，完成首次验证后再切换线上调度。原仓库仍保持暂停，避免同时运行。

公开仓库标准 runner 的计费规则不等于用途许可。GitHub 对 Actions 有软件项目相关用途限制；这里的 EPG 抓取、订阅更新和清理属于线上数据维护，正式启用前应确认这种用途适用，不将本仓库当作通用免费后台计算服务。见 [GitHub Actions 用途规则](https://docs.github.com/en/site-policy/github-terms/github-terms-for-additional-products-and-features#actions)。

## 任务

| Workflow | 任务 | 单次最长时间 |
| --- | --- | --- |
| IPTV EPG | 更新节目表并保存可恢复进度 | 6 分钟，执行器内部预算约 4 分钟 |
| IPTV playlists | 同步订阅到草稿，不发布到电视 | 3 分钟 |
| IPTV EPG cleanup | 清理未引用且超过保留期的节目缓存对象 | 5 分钟 |

三个任务共享 `iptv-maintenance` 并发组，避免在本仓库同时修改存储。GitHub 仅保留一个等待中的运行，新请求可能替换等待任务；这里不提供可靠队列语义。原执行器的 D1 租约、检查点及每日写入预算继续生效。

## 源码

`vendor/iptv` 是私有 IPTV 仓库的固定 Git submodule 指针。只在任务开始时读取该版本，不复制解析器或维护另一份业务代码，不初始化 IPTV 内的 WorkerKit 子模块。

CI 只安装 `@iptv/runner` 工作区的生产依赖。前端、WorkerKit、视频播放协议验证和 FFmpeg 不参与执行；节目表解析仍使用 IPTV 原有实现。

有私有仓库读取权限时，可在本机初始化用于审阅：

```sh
git clone git@github.com:pengzhendong/worker-jobs.git
cd worker-jobs
git submodule update --init vendor/iptv
```

通常只需 `git submodule update --init vendor/iptv`，不要递归初始化 WorkerKit。升级执行器时审核目标提交，更新 `vendor/iptv` 指针并验证三个脚本的依赖，再提交 gitlink；不自动跟随 IPTV 默认分支。

## 接入配置

下列内容应在本仓库的 Actions 设置中配置，不能提交到 Git。

| 类型 | 名称 | 用途 |
| --- | --- | --- |
| Secret | `IPTV_READ_SSH_KEY` | 专用、只读的 IPTV 部署密钥，不能复用只能读取 WorkerKit 的旧密钥 |
| Secret | `IPTV_D1_TOKEN` | 仅相应 Cloudflare 账号的 D1 写入权限 |
| Secret | `IPTV_R2_ACCESS_KEY_ID` | 仅节目缓存桶的 S3 密钥 ID |
| Secret | `IPTV_R2_SECRET_ACCESS_KEY` | 相应 S3 密钥，节目更新与清理需要读写及删除权限 |
| Variable | `IPTV_CLOUDFLARE_ACCOUNT_ID` | Cloudflare 账号 ID |
| Variable | `IPTV_D1_DATABASE_ID` | IPTV D1 数据库 ID |
| Variable | `IPTV_EPG_BUCKET` | 私有节目缓存桶名称 |
| Variable | `IPTV_PAUSED_UNTIL` | 可选的暂停截止时间，留空不暂停；无效日期拒绝执行 |
| Variable | `IPTV_JOBS_ENABLED` | 明确设为 `true` 后才允许任务执行 |

公开仓库的日志可被他人查看。执行器的原始输出在此丢弃，只公布固定的任务成功 / 失败状态；不上传业务日志、源码、缓存数据或其他 artifact。失败时在 IPTV 私有后台检查状态，必要时在私有环境运行同一固定版本诊断。

## 正式切换顺序

1. 确认用途，配置专用只读源码权限及限定范围的 Cloudflare 凭据。
2. 确认生产权限后，分别手动验证三个任务；成功后再启用 Cron。
3. 停用 IPTV 原仓库中对应的三个定时 workflow；保留原 CI、部署与台标同步。
4. 修改 IPTV Worker 的 EPG watchdog，使唤醒与状态查询指向这里，避免旧入口继续调度；不能只开启新 Cron。
5. 最后启用本仓库的 Cron，并更新此前的恢复跟进，防止新旧任务同时恢复。

本次仅准备入口，没有执行以上生产切换步骤。
