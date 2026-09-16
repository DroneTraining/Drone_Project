# Unity MCP（KitWright）接入说明

本工程接入了 [KitWright MCP for Unity](https://github.com/kitwright/unity-mcp)：
它把 MCP 服务跑在 Unity 编辑器进程里，AI 客户端通过本机 HTTP 连进来操作编辑器
（建场景、生成脚本、进出播放模式、截图、看日志与性能等）。

## 工程里已经接好的部分

- `Packages/manifest.json` 依赖 `com.kitwright.unity.mcp`（git URL），
  Unity 打开工程时会自动拉取；`Packages/packages-lock.json` 锁定了版本。
- 编辑器启动后，MCP 服务监听 `http://127.0.0.1:8765/`。
- 在 MCP 客户端里，工具名形如 `mcp__unity__<工具名>`。

## 端口与 URL 形状

- 端口默认由工程路径派生（`8765 + offset`）。本工程用一个 settings 文件把它固定成 **8765**。
- 客户端连接的完整形状是 `http://127.0.0.1:8765/p/<pin>/`：
  - `<pin>` = `SHA256(工程绝对路径)` 的前 8 位十六进制。路径先规范化为全小写、正斜杠、去掉结尾斜杠。
  - pin 不匹配时服务端返回 **404**，而不是静默去操作另一个 Unity 工程。
  - 包里自己生成的 URL 还带一段 `/t/<token>/`；**缺 token 的请求仍会被受理**，只在编辑器控制台
    警告一次。要收紧的话，在 `Window > KitWright > MCP Window` 点 **Configure**，
    把窗口里显示的完整 URL 复制到客户端配置里。
- 若端口被别的 KitWright 工程占用，本工程会向后顺延，此时固定端口的 URL 会失效——
  以 `Window > KitWright > MCP Window` → **Server** 标签页显示的实时 URL 为准。

## UserSettings 不入库

`UserSettings/` 按 Unity 惯例被 `.gitignore` 忽略。让服务在编辑器启动时自动监听 8765 的
是 `UserSettings/KitWrightMcpSettings.json`，所以**新克隆的工程默认不会自动启动服务**。
需要的话手工创建该文件即可，缺的字段会自动取默认值：

```json
{
  "enabled": true,
  "port": 8765,
  "mcpAutostartEnabled": true
}
```

也可以用编辑器窗口手动起：`Window > KitWright > MCP Window` → **Server** → 启动。

## 启动编辑器的一个注意点

从别的 shell（脚本、CI、终端）里用 `start` 启动编辑器时，编辑器是那个 shell 的子进程，
shell 一关就被一起收掉。想让它独立存活，可以交给计划任务托管，例如：

```
schtasks /create /tn "<任务名>" /tr "\"<编辑器路径>\Unity.exe\" -projectPath \"<工程路径>\"" /sc once /st 00:00 /f
schtasks /run /tn "<任务名>"
```

直接在 Unity Hub 里打开本工程当然也可以。

## 排错

| 现象 | 原因 |
|---|---|
| 客户端返回 404 | URL 的 pin 不对；或本工程没抢到 8765（被别的 KitWright 工程占用会顺延） |
| 端口在监听但连不上 | 传输方式必须是 `streamable-http`；服务端**拒绝任何带 `Origin` 头的请求**（浏览器页面访问不了，原生客户端不受影响） |
| git 拉包失败 | 可改为内嵌包：把仓库 clone 到 `Packages/com.kitwright.unity.mcp/`，并从 `Packages/manifest.json` 删掉那行 git 依赖 |
| 编辑器起不来 | 看日志（见下），并确认编辑器进程没有被启动它的 shell 一起关掉 |

日志：Windows 下在 `%LOCALAPPDATA%\Unity\Editor\Editor.log`。
包内的详细调试日志可在 `Window > KitWright > MCP Window` → **Settings** 里打开。

## 安装时踩过的坑

1. **受限令牌下 Unity 根本起不来。** 在受限的执行环境里，`Unity.exe` 收不到命令行参数
   （日志里 `BatchMode: 0`、`COMMAND LINE ARGUMENTS` 只有 exe 路径）、不写 `-logFile`
   指定的日志，直接以 `140063` 退出；同一环境下 `git` 会报
   `schannel: AcquireCredentialsHandle failed: SEC_E_NO_CREDENTIALS`，是同一个根因
   （拿不到系统凭据/加密）。用正常权限运行即恢复正常，**与 Unity 授权无关**
   （日志里 `Access token is unavailable` 后面紧跟着 `Successfully updated license`，
   授权其实是成功的，容易被误导）。
2. **registry 的 DNS 可能间歇性失败。** 典型报错是
   `Cannot connect to 'download-packages.unity.cn' (error code: ENOTFOUND)`。
   如果每次失败的主机都不一样、而有的主机下一次就成功，说明是解析不稳定而不是域名不存在，
   直接重试即可。
3. **别沿用老模板的包版本。** 编辑器自带的 3D 模板可能来自更早的 Unity 版本，带着
   `com.unity.collab-proxy 1.13.5`、`com.unity.textmeshpro 3.0.6`、`com.unity.timeline 1.6.2`、
   `com.unity.visualscripting 1.7.6` 这类旧锁定版本，在新编辑器上容易编译报错。
   本工程改用编辑器自带的 `com.unity.feature.development`（它用 `"default"` 版本说明符，
   由编辑器自己挑兼容版本，含 IDE 集成、Test Framework、Code Coverage），并显式列出
   `com.unity.modules.*` 内置模块。
4. **`Get-NetTCPConnection` 在受限环境里会漏报正在监听的端口**，判断端口用
   `netstat -ano | Select-String ':8765\s'` 更可靠。
