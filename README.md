# kharon-manifest（模型 C 的清单仓）

这个目录是**工作区的单一入口**，承担三件事：

| 文件 | 谁读 | 作用 |
|---|---|---|
| `west.yml` 的 `manifest:` 块 | `west` | `west init -m <本仓>` 之后的清单；zephyr 用 `import: true`，模块 revision 不抄第二遍 |
| `west.yml` 的 `zephyr:` / `projects:` 块 | `kharon bootstrap` | pinned 通道：6 个模块闭包 + 钉住 revision + zephyr commit |
| `products/kharon/*.yaml` | `kharon`（驾驶舱/`kharon ws`） | 产品登记：哪个 APP 配哪块板、产物与判据 |

> 一个文件两个角色是**实测可行**的（west 1.5.0 只校验 `manifest` 键，其余顶层键忽略），
> 好处是"别人 clone 用的清单"和"我们钉版本用的快照"永远描述同一棵树。

## 别人拿到这台机器的等价环境（不依赖 VS Code 扩展）

```powershell
# ① 清单仓直接克隆成 west topdir
git clone <kharon-manifest 地址> zephyr-workspace
cd zephyr-workspace

# ② IDE 侧（另一个仓）
git clone <kharon-ide 地址> ..\kharon-ide
cd ..\kharon-ide
python -m venv .deps\venv
.deps\venv\Scripts\python.exe -m pip install -r requirements-dev.txt
copy config\host.local.yaml.example host.local.yaml   # 只填 SDK / gdb / openocd

# ③ 拉齐工作区（含 zephyr + import 的模块 + 本地补丁重放）
.\kharon.cmd bootstrap --pinned     # CI / 复现
#   或  .\kharon.cmd bootstrap        # learning：跟随上游全量

# ④ 自检 + 构建 + 驾驶舱
.\kharon.cmd bootstrap --check
.\kharon.cmd deps --check
.\kharon.cmd ws -p f411_breath
.\kharon.cmd run -p f411_breath --steps doctor,build
.\kharon.cmd cockpit
```

`kharon-boards/`、`elab-app-common/` 目前**还没有远端仓**，靠
`providers/official/zephyr/west/provider.yaml` 的 `EXTRA_ZEPHYR_MODULES` 注入；
建好仓之后在 `west.yml` 里取消那两段注释，`west update` 就会连它们一起拉。

## 升级 Zephyr（这是唯一该动这份文件的时候）

```powershell
# 在已有工作区里先试升级，确认能编过
git -C ..\zephyr-workspace\zephyr checkout <新 commit>
.\kharon.cmd bootstrap --learning
.\kharon.cmd run -p f411_breath --steps doctor,build

# 编过之后，把新 commit 写进两处（必须一致）
#   manifest.projects[zephyr].revision
#   zephyr.commit
# 然后把 .kharon\bootstrap.json 里的 zephyr_commit 同步（bootstrap 自动写）
# 最后跑：.\kharon.cmd bootstrap --check
```

模块 revision **不要**手工抄进本文件——它们来自 zephyr 自己的 `west.yml`（import）。
只有必须偏离上游的（例如 `hal_espressif` 的 fork）才写进顶层 `projects:`，
由 `bootstrap._pin_modules` 在 `west update` 之后把 checkout 拉回来。

## 目录约定

- `products/kharon/<app>.yaml`：一个 APP × 一块板 × `targets[]`；文件名 = APP 名。
- 板级真相不在这里，在 `../kharon-boards/boards/<vendor>/<board>/`。
- 这里**不放**构建产物、不放本机路径、不放 `.vscode` 状态。
