# hackathon

这个目录是本地工作区。下面 7 个项目用 Git 子模块挂在这里，各自仍推到原来的远程仓库。

| 路径 | 远程 | 分支 |
| --- | --- | --- |
| `OctoSense` | https://github.com/OctoSense-org/OctoSense.git | `main` |
| `OctoSense-App-Hub` | https://github.com/OctoSense-org/OctoSense-App-Hub.git | `main` |
| `OctoSense-App-Flow` | https://github.com/OctoSense-org/OctoSense-App-Flow.git | `main` |
| `OctoScript` | https://github.com/OctoSense-org/OctoScript.git | `main` |
| `OctoScript-Makepad` | https://github.com/OctoSense-org/OctoScript-Makepad.git | `main` |
| `makepad` | https://github.com/OctoSense-org/makepad.git | `main` |
| `AgentMail` | https://github.com/Jasonsu22/AgentMail.git | `master` |

`AgentMail` 的子模块路径是根目录下的 `AgentMail`。Git 不能把一个子模块记在另一个子模块目录里面，所以它不能直接记成 `OctoSense/apps/AgentMail`。本机用符号链接把应用留在原来的位置：

```bash
ln -s ../../AgentMail OctoSense/apps/AgentMail
```

打开 `OctoSense/apps/AgentMail` 时，进入的就是根目录的 `AgentMail`。

## 克隆

```bash
git clone --recurse-submodules https://github.com/Jasonsu22/OctoSense-Hackathon.git
cd OctoSense-Hackathon
ln -s ../../AgentMail OctoSense/apps/AgentMail
```

仓库已经克隆、子模块目录还是空的时候：

```bash
git submodule update --init
ln -s ../../AgentMail OctoSense/apps/AgentMail
```

需要稳定版 Rust（`cargo`）、Git、Python 3.9 以上。macOS 上还需要 Xcode Command Line Tools。

## 构建 OctoSense

在 `OctoSense` 目录里先准备它锁定的框架源码，再编内核和桌面。第一次构建要联网，时间比较长。

```bash
cd OctoSense
python3 tools/setup.py
python3 tools/kernel-artifact.py --host --stage target/release
cargo build --release -p octosense
```

`tools/setup.py` 把 Makepad、OctoScript 放到 `.sources/`。`kernel-artifact.py` 把与这次锁定版本一致的 `octos-kernel` 放到 `target/release/`，桌面靠它运行助手。没有这份内核时，桌面可以开，但没有助手。

桌面可执行文件是 `OctoSense/target/release/octosense`。

## 构建 App Hub

发布 AgentMail 到本地商店要用 `hub`。在 `OctoSense-App-Hub` 目录里构建：

```bash
cd OctoSense-App-Hub
cargo build --release -p octosense-app-hub
```

命令行工具是 `OctoSense-App-Hub/target/release/hub`。

## 第一次准备本地商店

脚本把状态放在工作区里的 `.octosense-run/`（密钥、商店镜像、桌面数据）。这个目录已忽略，不会提交。

还没有演练密钥时，生成一对，并用锚密钥认证工作密钥：

```bash
mkdir -p .octosense-run/keys
OctoSense-App-Hub/target/release/hub keygen .octosense-run/keys/anchor.key
OctoSense-App-Hub/target/release/hub keygen .octosense-run/keys/working.key
OctoSense-App-Hub/target/release/hub certify \
  --anchor .octosense-run/keys/anchor.key \
  --working .octosense-run/keys/working.key
```

`certify` 打出的那一行证书，脚本在发布时会自己再算一次，不必手抄。

## 启动 OctoSense 和 AgentMail

改完 AgentMail 之后，在任意目录执行：

```bash
bash AgentMail/scripts/update-and-open.sh
```

脚本会做这些事：

1. 把 `AgentMail/bundle` 复制到临时目录。源码清单保持未签名，版本号仍是清单里的版本。
2. 如果本地商店里已经有这个版本，只把临时包的补丁号加一。
3. 用 `hub` 盖戳、用工作密钥签名、检查，并发布到 `.octosense-run/mirror`。
4. 装进 `.octosense-run/desktop-apps`。
5. 关掉已经在跑的桌面，再打开。macOS 上它会放进 `.octosense-run/OctoSense.app` 再启动，这样系统日历授权才能弹出。

启动时用的环境是本地旧目录商店：`OCTOSENSE_HUB_CATALOG=legacy`，数据目录是 `.octosense-run/desktop-home` 和 `.octosense-run/desktop-apps`。邮箱密码存在宿主的文件保险库里（`OCTOSENSE_MAIL_VAULT=file`），不进系统钥匙串。

桌面打开后，在启动器里点 AgentMail。日志在 `.octosense-run/octosense.log`。

只想再打开已经装好的桌面、不重新发布时，再执行一次该脚本即可；它会按当前 bundle 再发布一个本地版本并重启。

## 不提交的内容

`.octosense-run/` 是本机运行数据，里面有密钥、商店镜像和桌面状态。`.aider*` 是本地辅助记录。这两类都在 `.gitignore` 里。
