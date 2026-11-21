# 从零开始把 Codex 接入飞书

## Windows + PowerShell + Miniconda 实操教程

本教程按我们实际走通的 Windows 方案整理。完成后，你可以在飞书私聊自己的机器人，把任务交给本机 Codex，也可以让它读取、修改已授权的飞书文档。

整条链路如下。

```text
飞书中的用户消息
        ↓
本机 lark-channel-bridge
        ↓
已经登录的 Codex CLI
        ↓
飞书 CLI / 本地文件与工具
        ↓
处理结果返回飞书
```

飞书客户端负责聊天和扫码，桥接程序负责接收消息，Codex CLI 负责处理任务，飞书 CLI 负责调用文档等接口。桥接可以独立于 ChatGPT 桌面客户端运行，Codex 登录状态由本机 CLI 保存。

本教程使用以下已经在本机验证过的版本，方便复现。

| 组件                | 教程使用的版本 |
| ------------------- | -------------- |
| Node.js             | 24.x           |
| Codex CLI           | 0.169.0        |
| 飞书 CLI            | 1.0.100        |
| lark-channel-bridge | 0.7.1          |

---

## 1. 准备账号与安装位置

准备好以下条件。

- Windows 10/11 x64 电脑，能够访问 npm、OpenAI 和飞书服务。
- 能使用 Codex 的账号，用于浏览器登录 Codex CLI。
- 已登录本人账号的飞书手机客户端，用于扫码。
- 飞书应用创建或管理权限。组织内账号如受管理员限制，需要由该组织管理员协助完成应用创建、权限申请或审批。
- 对准备交给机器人修改的文档，拥有添加文档应用、分配编辑权限的能力。

本教程沿用已有 Miniconda。我们之前的安装根目录为 `D:\miniconda3`。其余文件集中放在它的子目录内。

```text
D:\miniconda3
├─ envs\feishu             Node.js、npm、Codex、桥接和飞书 CLI
├─ pkgs                    Conda 包缓存
└─ feishu-data
   ├─ bridge               飞书应用绑定及各 profile 的配置
   ├─ npm-cache            npm 缓存
   ├─ tmp                  运行时临时文件
   ├─ workspace            Codex 的初始工作目录
   ├─ logs                 后台进程的输出日志
   └─ start-codex.cmd       稍后生成的手动后台启动脚本
```

已有 Miniconda 时复用其安装位置。下文会从 `conda info --base` 自动取得路径。Codex 的账号登录资料由 Codex CLI 保存在当前 Windows 用户的配置目录中。

---

## 2. 安装 Miniconda，并让 PowerShell 识别 conda

已有 Miniconda、且 PowerShell 中运行 `conda --version` 正常时，直接进入第 3 节。

### 2.1 下载并安装

**需要本人操作。**

1. 下载官方 Windows x64 安装包。
2. 运行安装器，选择当前用户安装，也就是 `Just Me`。
3. 选择安装目录。本文示例使用 `D:\miniconda3`，已有安装则复用原目录。
4. 完成安装，保持安装器默认的 PATH 相关选项。

选择安装目录时遵循安装器的路径要求，优先使用不含空格和中文的路径。

[Miniconda 官方安装说明](https://docs.conda.io/projects/conda/en/stable/user-guide/install/windows.html) · [Windows x64 官方安装包](https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe)

### 2.2 初始化 PowerShell

从开始菜单打开安装器提供的 **Anaconda Prompt / Miniconda Prompt**，执行。

```shell
conda init powershell
```

关闭这个窗口，再打开一个新的 PowerShell，验证。

```powershell
conda --version
conda info --base
```

预期能看到 Conda 版本和安装目录。PowerShell 的提示符可能显示 `(base)`。

如果 PowerShell 提示禁止加载配置脚本，在允许自行管理执行策略的个人电脑上，可以运行以下命令，再关闭并重新打开 PowerShell。

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

该设置作用于当前 Windows 用户。由组织策略管理的电脑，按管理员要求配置 PowerShell。

[PowerShell 执行策略说明](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_execution_policies)

---

## 3. 创建专用环境，并设置缓存和配置目录

以下操作在 PowerShell 中进行。整段复制执行，变量路径会从你的 Miniconda 安装位置取得。

```powershell
$CondaRoot = (conda info --base).Trim()
$FeishuEnv = Join-Path $CondaRoot 'envs\feishu'
$FeishuData = Join-Path $CondaRoot 'feishu-data'

$env:CONDA_PKGS_DIRS = Join-Path $CondaRoot 'pkgs'

conda create --prefix "$FeishuEnv" --channel conda-forge --override-channels --yes nodejs=24
conda activate "$FeishuEnv"

$RequiredFolders = @(
    $FeishuData
    (Join-Path $FeishuData 'npm-cache')
    (Join-Path $FeishuData 'tmp')
    (Join-Path $FeishuData 'workspace')
    (Join-Path $FeishuData 'logs')
)

foreach ($Folder in $RequiredFolders) {
    New-Item -ItemType Directory -Path $Folder -Force | Out-Null
}

conda env config vars set --prefix "$FeishuEnv" "NPM_CONFIG_PREFIX=$FeishuEnv" "NPM_CONFIG_CACHE=$FeishuData\npm-cache" "LARK_CHANNEL_HOME=$FeishuData\bridge" "TEMP=$FeishuData\tmp" "TMP=$FeishuData\tmp"

conda deactivate
conda activate "$FeishuEnv"
```

已有 `feishu` 环境时，先用 `conda env list` 确认位置，跳过上面的 `conda create`，再激活已有环境完成后续目录和环境变量配置。

接着验证。

```powershell
node --version
npm.cmd --version
npx.cmd --version
npm.cmd config get prefix
npm.cmd config get cache
Get-Command node
```

预期结果为 Node.js 24.x，npm 和 npx 均有版本号，npm 的安装位置指向 `envs\feishu`，缓存指向 `feishu-data\npm-cache`。

这里使用 `npm.cmd` 和 `npx.cmd`，便于在 Windows PowerShell 中直接调用。激活环境时如果显示将覆盖 `TEMP`、`TMP`，对应的是本节为该环境设置的临时目录。

---

## 4. 安装 Codex、飞书 CLI 和桥接工具

确认提示符已处于 `feishu` 环境，然后执行。

```powershell
npm.cmd install --global "@openai/codex@0.169.0" "@larksuite/cli@1.0.100" "lark-channel-bridge@0.7.1"
```

检查三个程序。

```powershell
codex.cmd --version
lark-cli.cmd --version
lark-channel-bridge.cmd --version
```

三条命令都能输出版本号，再继续后面的登录步骤。

[Codex CLI 官方说明](https://developers.openai.com/codex/cli) · [飞书 CLI 官方仓库](https://github.com/larksuite/cli) · [桥接项目说明](https://github.com/zarazhangrui/lark-coding-agent-bridge)

---

## 5. 登录 Codex CLI

执行。

```powershell
codex.cmd login
```

**需要本人操作。**

1. 在弹出的浏览器页面选择自己的账号并登录。
2. 阅读并完成授权。
3. 回到 PowerShell，等待登录完成。

浏览器没有自动打开时，复制终端提供的登录地址，在浏览器中完成。也可以使用当前版本支持的设备授权方式。

```powershell
codex.cmd login --device-auth
```

按终端实际显示的地址和验证码完成操作。随后验证。

```powershell
codex.cmd login status
```

使用 ChatGPT 账号登录时，预期出现 `Logged in using ChatGPT` 或同等含义的成功提示。

---

## 6. 首次启动桥接，用飞书扫码绑定机器人

仍在 `feishu` 环境中执行。

```powershell
$FeishuData = Split-Path -Parent $env:LARK_CHANNEL_HOME
$Workspace = Join-Path $FeishuData 'workspace'

lark-channel-bridge.cmd run --agent codex --profile codex --workspace "$Workspace"
```

本教程将桥接 profile 命名为 `codex`。它用于区分应用绑定和会话配置。后文命令始终使用同一个 profile。

### 6.1 扫码与创建应用

**需要本人操作。**

1. 保持终端窗口打开，等待出现飞书二维码。
2. 打开已登录本人账号的飞书手机客户端，扫描二维码。
3. 按页面提示选择或创建 PersonalAgent 应用。
4. 给应用起一个容易搜索的名字，例如“我的 Codex 助手”。
5. 阅读申请内容并完成绑定或授权。
6. 回到终端，等待初始化及飞书连接完成。

终端同时提供浏览器地址时，也可以在浏览器打开该地址，按页面提示完成飞书确认。二维码过期后重新运行本节命令，以新生成的信息完成绑定。

应用名称、后续开发者后台选择的应用，以及文档中添加的文档应用，都应对应这次绑定的同一个应用。组织限制导致创建或授权无法完成时，由组织管理员处理页面显示的具体限制，再重新运行向导。

### 6.2 测试机器人收发消息

**需要本人操作。**

1. 在飞书中找到刚创建的机器人，打开私聊。
2. 发送一条简单消息，例如“连接测试，请只回复连接成功”。
3. 等待机器人返回回复。
4. 再发送 `/status`，查看 agent、profile、工作目录和 CLI 身份摘要。

默认情况下，创建者可以使用自己的机器人。当前版本支持在群中 `@机器人` 发送任务，群内使用还需要完成相应的机器人加入与访问配置。

首次配置后，可通过 `/config` 核对访问模式。桥接 0.7.1 的新 profile 默认使用 `full`，对应 Codex 的 `danger-full-access`。这会让本机 Codex 按当前账户和该模式执行本地工具任务，工作目录只是任务的起点。

### 6.3 认识当前运行状态

本节的 `run` 是前台运行，关闭该终端会结束这次桥接进程。完成测试后，可以按 `Ctrl+C` 停止前台进程，再按第 9 节改用隐藏后台启动。[桥接项目中文说明](https://github.com/zarazhangrui/lark-coding-agent-bridge/blob/main/README.zh.md)

---

## 7. 给智能体开通应用权限和目标文档权限

接下来验证机器人能否处理文档。需要同时检查应用能否调用相关 API，以及它对某篇文档具有什么访问权限。

| 授权内容             | 本人在哪里操作                   | 影响范围                                         |
| -------------------- | -------------------------------- | ------------------------------------------------ |
| 飞书应用 API 权限    | 飞书开放平台中该应用的权限管理   | 文档读取、编辑、图片上传等接口能力               |
| 某篇文档的协作者权限 | 文档右上角添加文档应用并分配权限 | 对这篇文档阅读或编辑                             |
| 个人身份 OAuth 授权  | CLI 输出的个人授权页面           | 以本人身份访问日历、云盘等个人资源，详见第 12 节 |

### 7.1 在开放平台检查应用 API 权限

**需要本人操作。**

1. 打开飞书开放平台，使用创建机器人时的同一个飞书账号登录。
2. 找到第 6 节创建的应用，核对应用名称或 App ID。
3. 进入该应用的“权限管理”。
4. 按计划使用的功能检查文档读取、编辑等接口能力。
5. 如果执行 CLI 返回 `missing_scopes`，按错误中的权限代码搜索并申请相应权限。
6. 若页面要求发布新版本或管理员审批，完成对应流程，直到权限显示已开通或已生效。

扫码向导会完成其支持的初始化配置。实际缺少哪些权限，以当前应用的状态和调用接口返回的信息为准。

[飞书开放平台应用管理](https://open.feishu.cn/app)

### 7.2 图片和附件上传需要单独的接口能力

如果需要让机器人自动上传原图或附件，在应用权限管理中搜索并开通。

```text
docs:document.media:upload
```

**需要本人操作。** 如果应用处于企业管理下，权限可能需要管理员审批或发布后生效。完成后重新尝试一次图片插入，确认实际结果。

[飞书上传素材接口说明](https://open.feishu.cn/document/server-docs/docs/drive-v1/media/upload_all)

如果当前工作方式是由人类补图，可以先完成正文读写测试，并在交付时列出需要手动补充的原图、原表编号和放置位置。

### 7.3 把机器人添加为目标文档的可编辑应用

**需要本人操作。**

1. 打开准备让机器人修改的飞书文档。
2. 点击右上角的“…”。
3. 找到“更多 → 添加文档应用”。部分界面的菜单会直接显示“添加文档应用”。
4. 搜索第 6 节创建的机器人名称。
5. 选择该应用，权限设为“可编辑”，再确认添加。

同样的方法可用于知识库中的具体文档页面。父级或知识库的权限继承，以该文档实际显示的权限为准。

如果当前用户没有添加应用或分配权限的能力，需要让文档所有者或有管理权限的协作者完成。

[飞书云文档权限说明](https://www.feishu.cn/hc/zh-CN/articles/360044508113-%E4%BA%91%E6%96%87%E6%A1%A3%E6%9D%83%E9%99%90%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)

---

## 8. 完成一次真实文档读写测试

**需要本人操作。**

在飞书创建一篇用于测试的空白文档，并按第 7.3 节添加机器人，授予可编辑权限。

保持桥接运行，在机器人私聊中发送以下内容，并把最后一行替换为自己的文档链接。

```text
请通过飞书 CLI 读取这篇文档，在文末追加一段“Codex 接入测试成功”，写完后重新读取并确认保存结果。

文档链接粘贴在这里
```

打开文档，确认出现了这段文字。此时已经验证了飞书收发消息、Codex 处理任务、CLI 调用接口和文档写入这条完整链路。

后续可以直接在飞书私聊中发送文档链接、修改位置和具体要求。每篇新文档都需要具备对应的访问权限。

---

## 9. 设置手动后台启动，启动后关闭终端

本节使用当前用户的隐藏后台进程，适合我们之前的 Windows 安装情况。请先停止第 6 节的前台进程。如果已经有一个该 profile 的桥接在运行，先按第 10 节查看并停止，再切换启动方式。

### 9.1 生成自己的后台启动脚本

重新打开 PowerShell 时，先执行 `conda activate feishu`。然后完整复制下面的代码。它会把真实路径写入 `feishu-data\start-codex.cmd`。

```powershell
conda activate feishu

$CondaRoot = (conda info --base).Trim()
$FeishuEnv = $env:CONDA_PREFIX
$FeishuData = Split-Path -Parent $env:LARK_CHANNEL_HOME
$Workspace = Join-Path $FeishuData 'workspace'
$LogDir = Join-Path $FeishuData 'logs'
$Launcher = Join-Path $FeishuData 'start-codex.cmd'
$NodeExe = Join-Path $FeishuEnv 'node.exe'
$BridgeEntry = Join-Path $FeishuEnv 'node_modules\lark-channel-bridge\bin\lark-channel-bridge.mjs'
$CondaBin = Join-Path $CondaRoot 'condabin'
$NpmCache = Join-Path $FeishuData 'npm-cache'
$TempDir = Join-Path $FeishuData 'tmp'
$BridgeHome = $env:LARK_CHANNEL_HOME
$StdoutLog = Join-Path $LogDir 'bridge.stdout.log'
$StderrLog = Join-Path $LogDir 'bridge.stderr.log'

New-Item -ItemType Directory -Path $Workspace, $LogDir, $NpmCache, $TempDir -Force | Out-Null

if (-not (Test-Path -LiteralPath $NodeExe)) {
    throw '当前 feishu 环境中找不到 node.exe，请先检查第 3、4 节。'
}

if (-not (Test-Path -LiteralPath $BridgeEntry)) {
    throw '当前环境中找不到桥接入口，请先完成第 4 节安装。'
}

$LauncherLines = @(
    '@echo off'
    'chcp 65001 >nul'
    ('set "LARK_CHANNEL_HOME={0}"' -f $BridgeHome)
    ('set "NPM_CONFIG_PREFIX={0}"' -f $FeishuEnv)
    ('set "NPM_CONFIG_CACHE={0}"' -f $NpmCache)
    ('set "TEMP={0}"' -f $TempDir)
    ('set "TMP={0}"' -f $TempDir)
    ('set "PATH={0};{1};%PATH%"' -f $FeishuEnv, $CondaBin)
    ('cd /d "{0}"' -f $Workspace)
    ('"{0}" "{1}" run --profile codex >> "{2}" 2>> "{3}"' -f $NodeExe, $BridgeEntry, $StdoutLog, $StderrLog)
)

[System.IO.File]::WriteAllLines($Launcher, $LauncherLines, [System.Text.UTF8Encoding]::new($false))
Write-Host "后台启动脚本已生成：$Launcher"
```

脚本保存程序路径、运行目录和日志位置。启动时桥接会读取对应 profile 的飞书绑定和凭证配置。

### 9.2 启动隐藏后台进程

执行。

```powershell
conda activate feishu

$Launcher = Join-Path (Split-Path -Parent $env:LARK_CHANNEL_HOME) 'start-codex.cmd'
Start-Process -FilePath $env:ComSpec -ArgumentList ('/d /c ""{0}""' -f $Launcher) -WindowStyle Hidden
```

再验证。

```powershell
lark-channel-bridge.cmd ps
```

列表中应出现你的机器人和对应 PID。然后在飞书发一条测试消息，确认它能回复。通过这两项检查后，可以关闭启动用的 PowerShell 窗口。

### 9.3 电脑重启后的操作

本节采用手动后台启动。每次电脑重启并登录 Windows 后，打开 PowerShell，执行。

```powershell
conda activate feishu
$Launcher = Join-Path (Split-Path -Parent $env:LARK_CHANNEL_HOME) 'start-codex.cmd'
Start-Process -FilePath $env:ComSpec -ArgumentList ('/d /c ""{0}""' -f $Launcher) -WindowStyle Hidden
```

随后用 `lark-channel-bridge.cmd ps` 和飞书测试消息确认连接，再关闭终端。

首次扫码的配置会保存在本机，启动脚本会读取它。若后续提示账号或授权失效，再按对应登录提示处理。

电脑关机或休眠期间，机器人连接会中断。ChatGPT 桌面客户端可以按个人需要打开，桥接任务由本机 Node.js 和 Codex CLI 运行。

