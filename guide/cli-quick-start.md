---
title: Zenith CLI 快速上手
description: 在 CPU 开发机上查看 GPU 和配额，进入临时 GPU 终端，提交训练任务并跟踪日志。
---

# Zenith CLI 快速上手

登录平台已初始化的独占 CPU 开发机，进入放代码的项目盘目录，就可以申请 GPU。CLI 自动使用机器所有者的身份，无需填写个人 Token。支持火山资源区 5 和百度资源区 10。

## 从项目目录开始

运行 `zenith auth status` 查看当前账号和权限。`credential_source` 为 `development_machine` 表示正在使用机器身份。如果已经配置了个人 profile，可以用 `zenith --profile @cpu auth status` 显式选择机器身份。

每次通过 SSH 交互登录时，「掠星」动画结束后会提示 `zenith guide`。这条命令可以离线阅读本指南，也可以运行 `cat /etc/zenith/cpu-guide.md` 查看随开发机安装的版本。

## 查看能用的 GPU 和剩余配额

```sh
zenith resources
zenith quota
```

`resources` 按资源区列出已分配的 GPU 型号和每个节点的卡数。`Development` 和 `Training` 为 `true` 时，分别表示该规格支持开发机和训练。实际分配取决于提交时的容量。

`quota` 显示运行中 GPU 的已用数、上限和剩余数，以及个人和项目的剩余卡时。要查看指定项目，两条命令都支持 `--project-id "$PROJECT_ID"`。

## 从 CPU 进入临时 GPU 开发机

先在控制台登记 SSH 公钥，并确认 CPU 开发机上的 SSH 客户端可以使用对应私钥。下面假设 `zenith resources` 列出了每节点 2 卡的 A100，请按实际结果替换型号和卡数。

```sh
zenith worker launch --type a100 --gpu 2 --dry-run
zenith worker launch --type a100 --gpu 2
```

第一条命令预览规格、项目和工作目录。运行第二条并确认申请后，CLI 会打开 GPU 上的 SSH 终端。运行 `pwd` 就能看到与 CPU 上相同的项目目录。

**在 GPU 终端输入 `exit`，CLI 会删除这台临时机和系统盘，项目盘上的代码和结果会保留。** 如果原终端意外关闭，在发起申请的 CPU 开发机上用原账号运行 `zenith worker list` 找到实例 ID，再用 `zenith worker kill "$INSTANCE_ID"` 确认回收。使用 `zenith worker login "$INSTANCE_ID"` 连接已有开发机时，退出会话后该机继续运行。

## 提交训练并跟踪日志

在 CPU 项目目录中准备好训练脚本（下例为 `train.sh`）。先预览，再提交；`--` 后面是要在项目目录中执行的命令。

```sh
zenith job submit --type a100 --gpu 2 --dry-run -- bash train.sh
SUBMISSION_ID=$(zenith job submit --type a100 --gpu 2 --parsable -- bash train.sh)
zenith job status "$SUBMISSION_ID"
zenith logs get --job "$SUBMISSION_ID" -f
```

提交后可以关闭终端，训练会继续排队或运行。跟踪日志时按 Ctrl-C 只会结束查看；要停止训练，运行 `zenith job stop --job "$SUBMISSION_ID"`。也可以用 `zenith job list` 找到平台任务 ID，查看状态、日志或停止任务时加上 `--by-id`。如果提交时断网，用 CLI 打印的 `zenith operations recover` 命令找回原提交。

多节点训练用 `--nodes`，例如 `--nodes 2 --gpu 2` 共申请 4 张卡。要额外挂载逻辑盘或存储桶，在 `worker launch` 或 `job submit` 后添加 `--mount-disk "$DISK_ID"` 或 `--mount-bucket "$BUCKET_GRANT_ID"`，可以重复使用。已初始化的 CPU 开发机支持在 Bash 和 Zsh 中按 Tab 补全挂载 ID。

## 在个人电脑上使用 CLI

先按客户端 Release 随附的安装说明安装 CLI。在个人电脑上，或在尚未配置机器身份的开发机上，需要用个人 Token 登录。到对应站点的「账号设置 → API Token」创建 Token，然后在本机终端运行以下命令，将示例域名换成该站点地址：

```sh
zenith profiles create work --origin https://zenith.example.com/pt/api2
zenith --profile work auth login
zenith --profile work auth status
```

在隐藏输入提示里粘贴 Token。查询 `resources` 需要 `catalog:read`，查询 `quota` 需要 `profile:read`。首个 profile 会成为默认值，之后可以直接运行这两条命令。

个人电脑也可以运行 `zenith job submit --body-file train.json --dry-run` 预览训练配置。文件包含 `offering_id` 和 `request`，后者填写 `name`、`entrypoint` 等创建参数。提交前请确认训练环境能访问代码和数据。

使用个人 Token 从 CPU 项目目录申请临时机或提交训练时，需要 operator 权限，以及 `instances:read`、`projects:read`、`storage:read`、`catalog:read` 和 `operations:read`。临时机还需要 `instances:create`、`instances:delete`、`ssh:read` 和 `ssh-keys:read`。提交训练需要 `jobs:create`，查看状态和日志需要 `jobs:read`，停止需要 `jobs:cancel`。各命令的完整参数可用 `--help` 查看。
