# OpenSysprepDriver

`OpenSysprepTool` 的系统部署测试用驱动集合。

- 远端：https://github.com/misakano7545/OpenSysprepDriver
- 主项目：https://github.com/misakano7545/OpenSysprepTool

## 这是什么

从 **系统总裁 SC 3.0.0.178**（`FUNK.exe` 内嵌的口令加密 7z 包）中提取的存储控制器驱动，
用于系统封装（sysprep）场景下给目标镜像预置 SRS（SATA / RAID / SCSI）驱动。

**本仓库不是 SC 的一部分，也不含 SC 的任何代码。** 这里的驱动全部是第三方厂商或
微软随 Windows 分发的成品文件，SC 只是把它们收集起来打了个包。

## 来源与提取方式

| 项 | 值 |
|---|---|
| 来源产物 | `FUNK.exe`（SC 3.0.0.178 的驱动包载体） |
| 容器 | 4 个口令加密的 7z |
| 原始规模 | 596 文件 / 53.2 MB |
| 本仓库 | **93 文件 / 12.7 MB** |

提取过程见主仓库 `docs/` 下的驱动清单说明。所有文件带 SHA-256 校验值，见 `manifest.json`。

## 结构

```
srs/     厂商 SRS 驱动包（注册表绑定式，见下）
  ALI(ULI) AMD ITE Intel JMicron MS Marvell SIS VIA nVIDIA
  <厂商>/<序号>/drivers/*.sys    驱动二进制
  <厂商>/<序号>/Drv.reg          绑定的注册表项（当前系统用）
  <厂商>/<序号>/Drv.pesrs        同一内容，离线挂载 hive 用
vmd/     Intel RST VMD（标准驱动包）
  intel/rstvmd/<版本>/iaStorVD.{inf,sys,cat}
```

## 两种驱动，两种用法 —— 这点很重要

### `srs/` —— 注册表绑定式，**不能用 `dism /add-driver`**

`srs/` 下**没有 `.inf`，也没有 `.cat`**。每个包里是裸 `.sys` + 两份注册表内容：

```
Drv.reg    [HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Control\CriticalDeviceDatabase\...]
Drv.pesrs  [HKEY_USERS\DCSRS\ControlSet001\Control\CriticalDeviceDatabase\...]
```

两份是**同一内容、只差 hive 前缀**：
- `Drv.reg` 写当前系统（`HKLM\SYSTEM`）
- `Drv.pesrs` 写离线挂载的目标系统（`HK_USERS\DCSRS`，`DCSRS` 是 SC 用的挂载点名字）

注入方式因此是**离线注册表注入**，不是驱动安装：

1. `RegLoadKey` 挂载目标镜像的 `SYSTEM` hive 到临时键
2. 把 `<name>.sys` 复制到 `<目标>\Windows\System32\drivers\`
3. 写 `ControlSet00x\Services\<name>`（Type / Start / ImagePath）
4. 写 `ControlSet00x\Control\CriticalDeviceDatabase\<硬件ID>`（`ClassGUID` + `Service`）
5. `RegUnLoadKey` 卸载

一共 **1555 条 CriticalDeviceDatabase 条目 + 201 条 Services 条目，覆盖 81 个存储控制器服务**
（`pciide` `IntelIDE` `aliide` `viaide` `iaStor` `iaStor47` `iaStor78` `ahci8086` `ahcix86`
`mvSata` `mv61xx` `mv64xx` `mv91xx` `nvgts` `nvraid` `nvatabus` `JRAID` `SiSRaid2` `SiSRaid4`
`viamraid` `symc8xx` `mraid35x` `adpu160m` `aic78xx` `dac960nt` `perc2` `vmscsi` 等）。

> 想用这批驱动，`OpenSysprepTool` 的 `--inject-drivers` 必须实现上面这套离线 hive 注入，
> 光做 `dism /add-driver` 是用不上的。

### `vmd/` —— 标准驱动包，可以用 `dism /add-driver`

`vmd/` 下是正规的 Intel RST VMD 驱动包（`.inf` + `.sys` + `.cat`），面向新平台 NVMe 硬盘
识别（12 代酷睿起的 VMD 控制器）。用法就是标准的：

```
dism /image:<挂载点> /add-driver:<inf 路径> /recurse
```

## 本仓库含什么、不含什么

**含**：`srs/` 下第三方厂商的 SRS 包 + `vmd/` 下的 Intel VMD 包。

**不含**（已剔除，共 503 文件 / 40.6 MB）：

| 剔除内容 | 文件数 | 大小 | 理由 |
|---|---|---|---|
| `FUND/` `FUNE/` —— Windows 自带 `System32\{drivers, DriverStore\FileRepository, catroot}` 的 x64/x86 副本 | 466 | 39.6 MB | 目标 Windows 上本来就有，无必要；且属微软 in-box 驱动 |
| `FUNB/MS/` 里 37 个版本信息标为 Microsoft 的 `.sys` | 37 | 1.0 MB | 同上，微软 in-box |

## 版权声明

**本仓库不主张对这些文件的任何权利。**

- 所有驱动的著作权归各自权利人所有（Intel、AMD、NVIDIA、Marvell、VIA、SiS、JMicron、
  ITE、ALi/ULi、Microsoft、LSI Logic、Adaptec、QLogic、Symbios、Mylex、Promise、
  American Megatrends、VMware 等）
- 文件未经修改，按原样收录，`manifest.json` 里带 SHA-256 供校验
- 收录目的是让 `OpenSysprepTool` 的部署流程可复现测试，**不是再分发**
- **权利人若要求删除，立即移除对应文件**
- 本仓库的 `LICENSE` 只覆盖本 README 与 `manifest.json` 这类原创文本，**不覆盖任何驱动文件**

如果你是要给自己的机器做封装，**建议从硬件厂商官网下载对应型号的驱动**，
本仓库只作为测试与兼容性验证的参照。
