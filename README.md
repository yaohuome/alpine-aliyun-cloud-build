# Alpine for Alibaba Cloud (ECS)

这是一个面向阿里云 ECS 的 Alpine 自定义镜像项目。
由于官方未直接提供可用的阿里云 Alpine 镜像，这里提供可上传的 `qcow2` 镜像版本。

## 适用场景

- 需要在阿里云 ECS 使用 Alpine Linux
- 需要一个轻量、可通过 cloud-init 初始化的基础镜像
- 需要国内可用的软件源配置（已切换阿里源）

---

## 使用说明（阿里云）

1. 从 Releases 下载镜像，文件名是 `alpine-custom-<Alpine版本>-<引导标签>.qcow2`，例如 `alpine-custom-3.24.1-bios-uefi.qcow2`
2. 在阿里云导入自定义镜像。默认产出的镜像 **BIOS 和 UEFI 都能启动**，「启动模式」选哪个都行；如果构建时选了单一模式（`boot_mode=bios` 或 `uefi`），导入时「启动模式」必须与之一致
3. 创建 ECS 时可绑定 SSH 密钥对（镜像内已内置公钥，非必需）
4. 系统盘最小选择 **1G 即可**

---

## 构建镜像（GitHub Actions）

镜像由 `.github/workflows/build-alpine-image.yml` 构建，触发方式二选一：

- 推送 `v*` 形式的 tag：
  ```bash
  git tag v2.0.2 && git push origin v2.0.2
  ```
- 在仓库 Actions 页面选择 `Build Alpine Cloud Image` → **Run workflow** 手动触发，可以设置：
  - `boot_mode`：引导方式，`both`（默认，BIOS+UEFI 通用）/ `bios` / `uefi`。选 `both` 时导入镜像的「启动模式」选哪个都能启动；选单一模式时导入的「启动模式」必须与之一致，否则实例会卡在 `Booting from Hard Disk...`
  - `ssh_pubkey`：注入镜像的 SSH 公钥（`root` 与 `alpine` 用户都会写入），默认已填好仓库内置的那把，改成你自己的即可；填 `none` 则本次构建不注入任何公钥
  - `password`：同时为 `root` 与 `alpine` 设置的登录密码，留空则不设置任何密码
  - `release_tag`：要发布的 release tag，留空则用 `manual-<run number>`

构建完成后：

- Release tag 会带上引导方式后缀便于区分：`<tag>-bios-uefi`（both）、`<tag>-bios`、`<tag>-uefi`。例如推 `v2.0.2` 且 `boot_mode=both`，发布出来的就是 `v2.0.2-bios-uefi`
- 镜像文件名带版本号和引导标签：`alpine-custom-<Alpine版本>-<引导标签>.qcow2`，例如 `alpine-custom-3.24.1-bios-uefi.qcow2`
- 同时作为 workflow artifact 保留 14 天，可在对应 run 页面直接下载

发布 Release 使用 GitHub 内置的 `GITHUB_TOKEN`，不需要额外配置 secret。

> `boot_mode=both` 的做法：基础镜像用 UEFI（GPT）那个，构建时在磁盘尾部空闲空间新建一个 `bios_grub` 分区，把 GRUB 的 BIOS 引导（i386-pc）写进去，UEFI 引导完全不动，两边共用同一份 `/boot/grub/grub.cfg`。构建过程中会断言 bios_grub 分区确实写入了引导代码，避免静默产出一个起不来的镜像。

> `ssh_pubkey` 是手动触发的输入项，只在那一次运行生效。推 tag 触发时没有输入值，会使用 workflow 里写死的默认公钥；想永久更换默认公钥，需要改 `.github/workflows/build-alpine-image.yml` 里的两处默认值（`workflow_dispatch` 输入的 `default`，以及 `build` job 的 `env.SSH_PUBKEY` 兜底值）。
>
> 填 `none` 时镜像内不含任何公钥，此时只能靠创建 ECS 时绑定密钥对（cloud-init 注入）或控制台登录。镜像本身的密码登录策略不受影响（允许密码登录、禁止空密码）。

> `password` 留空时镜像内不预设任何密码；填了则两个账号用同一个密码登录。密码在构建日志里会用 `::add-mask::` 隐藏为 `***`。但要注意 `workflow_dispatch` 的输入值不属于 GitHub 的 secret 机制，运行记录中可能可见，不要用它传长期使用的敏感密码。
>
> 另外，Alpine 的默认用户 `alpine` 在 cloud.cfg 里默认带 `lock_passwd: true`，cloud-init 首启时会对它执行 `passwd -l` 把密码锁掉。因此镜像内额外写入 `/etc/cloud/cloud.cfg.d/99-custom.cfg` 把 `system_info.default_user.lock_passwd` 改成 `false`：有密码时保留可用（cloud-init 反而会解锁），没密码时保持无密码。

---

## 生成 SSH 密钥对

构建时可以把一把公钥烤进镜像（见构建里的 `ssh_pubkey` 输入），需要的话用下面的命令生成密钥对。没有特殊需求用 `ed25519` 即可，比 RSA 更短也更安全。

**Linux / macOS**

```bash
ssh-keygen -t ed25519 -C "alpine-image" -f ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
cat ~/.ssh/id_ed25519.pub
```

**Windows（PowerShell，Win10 1809+ 自带 OpenSSH）**

```powershell
ssh-keygen -t ed25519 -C "alpine-image" -f $env:USERPROFILE\.ssh\id_ed25519
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

**Windows（Git Bash / WSL）**

```bash
ssh-keygen -t ed25519 -C "alpine-image" -f ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

生成后得到两个文件：`id_ed25519` 是**私钥**，留在本地不要外传；`id_ed25519.pub` 是**公钥**，把它整行内容填进构建时的 `ssh_pubkey` 输入，或者交给阿里云做密钥对。

> 已经有密钥就不要重复生成，直接 `cat ~/.ssh/id_ed25519.pub` 取公钥即可。老系统若不支持 ed25519，可以换成 `ssh-keygen -t rsa -b 4096`。
>
> 如果这把密钥还要用来向 GitHub 推送代码，把公钥内容加到 GitHub 的 Settings → SSH and GPG keys 即可，生成命令是一样的。

---

## 登录与权限（重要）

- 镜像内已预置 SSH 公钥，`root` 与 `alpine` 两个用户均可直接用对应私钥登录
- 直接用 root 登录：
  ```bash
  ssh -i <你的私钥> root@<ECS公网IP>
  ```
- 也可用普通用户登录后提权：
  ```bash
  ssh -i <你的私钥> alpine@<ECS公网IP>
  sudo -i
  ```
- 已启用密码登录，且禁止空密码登录（`PermitEmptyPasswords no`）；镜像默认不预设密码，只有在构建时填了 `password` 才有密码，否则请登录后自行 `passwd` 设置

> 创建实例时额外绑定密钥对同样受支持，cloud-init 会把绑定密钥追加进去。

---

## 镜像已做的定制

- 启用 cloud-init（支持云端初始化）
- 配置 growpart / resize_rootfs（首启可自动扩容到系统盘大小）
- 预装常用基础组件：`cloud-init`、`chrony`、`sudo` 等
- SSH 配置（支持密钥登录；允许密码登录但不预设密码，禁止空密码）
- 内置 SSH 公钥（`root` 与 `alpine` 用户），无需绑定密钥对即可登录
- 时区设置
- 优化部分内核/网络参数
- 默认软件源已切换为阿里云镜像源（便于国内使用）

---

## 镜像未包含内容

- 不包含阿里云官方 Agent（如云助手等）
- 不包含额外业务软件栈（Docker/K8s/监控等）

如有需要，请在实例初始化后自行安装。

---

## 常见问题

### 实例启动卡在 `Booting from Hard Disk...`

引导方式不匹配。这句提示是传统 BIOS（SeaBIOS）输出的，说明实例按 Legacy BIOS 启动，但磁盘上没有 BIOS 引导程序。

默认的 `boot_mode=both` 镜像两种模式都支持，不会出现这个问题。看到这个提示说明用的是单一模式的镜像、且导入时「启动模式」选错了——阿里云 `ImportImage` 的 `BootMode` 参数**默认是 `BIOS`**，所以导入 `boot_mode=uefi` 的镜像时如果不手动改成 UEFI，就会被卡住。

解决办法：

- 用默认的 `both` 重新构建，导入时选哪个模式都能启动
- 或者重新导入，把「启动模式」改成与镜像一致的值（选 UEFI 需要实例规格族支持 UEFI 启动）

---

## 免责声明

本镜像为社区用途的自定义构建版本，请先在测试环境验证后再用于生产环境。
