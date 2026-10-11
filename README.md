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

## 克隆

```bash
git clone --recurse-submodules <本仓库地址>
```

仓库已经克隆、子模块目录还是空的时候：

```bash
git submodule update --init
```

## 不提交的内容

`.octosense-run/` 是本机运行数据，里面有密钥、商店镜像和桌面状态。`.aider*` 是本地辅助记录。这两类都在 `.gitignore` 里。
