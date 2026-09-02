# 在路由器上安装 OAF（OpenWrt 24.10 / opkg）

前置检查（SSH 登录路由器后）：

```sh
uname -r                    # 应为 6.6.133（与编译所用内核源码一致）
opkg print-architecture     # 应有 aarch64_cortex-a53
opkg list-installed | grep -E '^kernel '   # 已装内核包版本
```

## 安装

把 4 个 ipk 传到 `/tmp` 后依次执行：

```sh
cd /tmp

# 1) 内核模块（最关键；固件是第三方分支、版本号与官方不同，务必用 --force-depends）
opkg install --force-depends kmod-oaf_*.ipk

# 1b) 若路由器上没装 kmod-ipt-conntrack（opkg list-installed | grep conntrack 为空），
#     用本次构建产物一并安装（同样加 --force-depends）
opkg install --force-depends kmod-ipt-conntrack_*.ipk

# 2) 用户态守护进程 + LuCI 界面 + 中文包
opkg install appfilter_*.ipk luci-app-oaf_*.ipk luci-i18n-oaf-zh-cn_*.ipk

# 3) 启动服务
/etc/init.d/appfilter enable
/etc/init.d/appfilter start

# 4) 验证
lsmod | grep oaf            # 应看到 oaf 模块
ps w | grep oafd            # 应看到 oafd 进程
```

如果 `opkg install kmod-oaf` 报依赖冲突，改用：

```sh
opkg install --force-depends /tmp/kmod-oaf_*.ipk
opkg install --force-depends /tmp/appfilter_*.ipk /tmp/luci-app-oaf_*.ipk /tmp/luci-i18n-oaf-zh-cn_*.ipk
```

然后浏览器访问 LuCI（注意清缓存或强制刷新），左侧菜单出现 **服务 → OAF（应用过滤）** 即成功。

## 模块加载失败的排查

```sh
insmod /lib/modules/$(uname -r)/oaf.ko   # 看具体报错
dmesg | tail -50
opkg info kernel                          # 核对与编译时 SDK 内核是否一致
```

常见原因与对策：

- `version magic '6.6.133 ...' should be '6.6.xxx ...'` → 编译用的内核与路由器内核不一致，
  把 `uname -r` 发给我，调整构建锚点重新编译。
- `Unknown symbol xxx` → 内核 config 差异（少见），发 `dmesg` 给我。
- `no symbol version for module_layout` → 同上，需要匹配的编译环境。

## 卸载

```sh
opkg remove --force-depends luci-app-oaf luci-i18n-oaf-zh-cn appfilter kmod-oaf
```
