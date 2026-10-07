
# 七彩虹蓝天模具英特尔 8-14代 BIOS 解锁工具包
# BIOS 解锁工具包 · 七彩虹蓝天模具

## ⚠️ 适用声明
本工具**仅适用于七彩虹蓝天模具**，且搭载 **Intel（英特尔）8-14代处理器** 的笔记本电脑。
请务必确认机型匹配后再使用，因机型不符导致的任何问题，概不负责。

## 🤖 制作声明
本工具包**全程由 AI 生成，零手工代码**。
仅供学习、研究与交流使用，请勿用于商业用途。

[![Platform](https://img.shields.io/badge/platform-Windows-blue)](#环境要求)
[![Python](https://img.shields.io/badge/python-3.8%2B-blue)](#环境要求)
[![依赖](https://img.shields.io/badge/%E4%BE%9D%E8%B5%96-%E4%BB%85%E6%A0%87%E5%87%86%E5%BA%93-success)](#环境要求)
[![License](https://img.shields.io/badge/license-MIT-green)](#许可)

---

## 这是什么

七彩虹 / 蓝天模具笔记本（Insyde BIOS）的 **SREP 运行时解锁**工具。

SREP（SmokelessRuntimeEFIPatcher）是一个 U 盘引导的 UEFI 应用：它在**内存里**
改写 SetupUtility 模块，把被 `Suppress If` / `Gray Out If` 隐藏或置灰的菜单项
解锁出来。

**全程只读固件，不刷写 SPI，掉电即恢复，没有变砖风险。**

传统做法要人工做完这些事：

1. UEFITool 打开备份的 BIOS，检索模块，`Extract as is` 导出 `.sct`
2. IRFExtractor 把 `.sct` 转成 IFR 文本（4 万行）
3. 在文本里肉眼找出目标菜单的守卫条件地址
4. 到 `.sct` 里按地址找那一段十六进制
5. 手工把 `12` 改成 `47`
6. 把改过的整段抄进 `SREP_Config.cfg`

本项目把 1-6 步全部自动化，并且**不含任何机型/固件相关常量** ——
所有地址、锚点、QuestionId 都由脚本从 `backup.fd` 内容现场推导。

---

## 特点

| | |
|---|---|
| **零手工** | 一条命令跑完，自动打开成品文件夹 |
| **零依赖** | 只用 Python 标准库，不需要 `pip install` |
| **不写死地址** | 换固件/换机型无需改代码，地址全部自动推导 |
| **上机前验证** | 先在 host 侧模拟 SREP 的写入行为，确认改对了字节 |
| **失败即停** | 任一环节不通过就中止，**不产出错误成品** |
| **自带提取工具** | 内置 8-9 / 10 / 11 / 12-14 代 FPTW64 备份工具 |

---

## 环境要求

- **Windows**
- **Python 3.8+**（只用标准库）

> Windows 商店版 Python 也能用。启动器会**实际执行测试**
> （跑一次 `import lzma,struct,json,re`）来判断解释器是否可用，


---

## 快速开始

### 1. 备份你自己的 BIOS

进入 `BIOS提取工具\`，按 CPU 代数选目录：

### 📁 BIOS提取工具目录

| 适用代次 | 包含文件 |
| :--- | :--- |
| **8-9代** | `FPTW64.exe` + `开始提取本机BIOS.bat` |
| **10代** | `FPTW64.exe` + `开始提取本机BIOS.bat` |
| **11代** | `FPTW64.exe` + `开始提取本机BIOS.bat` |
| **12-14代** | `FPTW64.exe` + `1.备份.bat` |

<!--EndFragment-->
</body>
</html>
**右键 `.bat` →「以管理员身份运行」**（FPTW64 需要提权）。
备份好会在同目录生成 `backup.fd`。

### 2. 跑解锁

双击根目录的 `run.cmd`，把 `backup.fd` 拖进窗口，回车。

### 3. 取成品

跑完程序会**自动打开** `SREP包\`：
| 目录/文件名 | 说明 |
| :--- | :--- |
| `SREP_Config.cfg` | SREP 配置文件 |
| `EFI\BOOT\BOOTX64.efi` | EFI 引导文件 |
```
SREP包\
├── SREP_Config.cfg
└── EFI\BOOT\BOOTX64.efi
```

把这两项拷到 **FAT32** U 盘（或硬盘分区）根目录，
开机按启动菜单键选 U 盘「UEFI」引导。

引导后根目录会生成 `SREP.log`，出现 `Patched` 即成功。

---

## ⚠️ 务必核对固件

在错误固件上生成解锁包**不会报错**，但对你机器无效。

程序每次开始都会打印：

```
======================================================================
 将要处理的固件 —— 请核对这就是你自己备份的那一份
======================================================================
  路径   : D:\mybackup\backup.fd
  大小   : 33,554,432 B
  sha256 : <64 位十六进制>
======================================================================
```

程序**不会**递归整个目录树去找固件，也**不会**在多个候选中替你选 ——
就是为了避免误用示例固件。

---

## 工作原理

### 为什么必须先解压

SetupUtility 在固件卷里是 **LZMA 压缩**存储的：

| 搜索目标 | 未解压的 `backup.fd` | 解压后镜像 |
|---|---|---|
| SREP Pattern 片段 | **0 命中** | 命中 |

所以「在裸固件里按地址找十六进制」这条路走不通。

### 地址映射

`.sct` 是 SetupUtility 的 PE 镜像（形态：3 字节前缀 + 完整 PE）。
各节 `VA == RawPtr`、`ImageBase == 0`，所以 **IFR 地址 == PE 文件内偏移**。

脚本用 IFR 文本里 `Form: Advanced` 那一行的**完整 6 字节 opcode**
（含该固件自己的 StringId）去镜像精确匹配，反推地址映射：

```
镜像偏移 = IFR 地址 + <自动推导的 delta>
```

## 0x12 → 0x47 的语义

IFR 用条件块表达「灰 / 隐藏」。以下是三种常见的守卫条件及其 Opcode：

| 条件类型 | Opcode | 作用 |
| :--- | :--- | :--- |
| **Suppress If** | `{0A 82}` | 隐藏项 |
| **Gray Out If** | `{19 82}` | 置灰项（不可选） |
| **Disable If** | `{1E 82}` | 禁用项 |

**通用结构如下：**

```text
<条件类型>   {Opcode}
  <条件 opcode>
  <被守卫项>
End If       {29 02}
```

把条件 opcode 首字节修改为 `0x47` 的效果：

| 改前 | 改后 | 效果 |
| :--- | :--- | :--- |
| `0x12` `EFI_IFR_EQ_ID_VAL` | `0x47` `EFI_IFR_FALSE` | 条件恒假 → 永不隐藏 / 永不置灰 |
| `0x14` `EFI_IFR_EQ_ID_VAL_LIST` | `0x47` `EFI_IFR_FALSE` | 同上 |

长度不变，符合 SREP `CopyMem` 的等长要求。

---

## SREP 的两个硬约束

1. **只替换第一次匹配**（源码首个命中就 `break`）
2. **不做等长校验** —— 长度不等会静默破坏镜像

本项目对每条 Pattern 都校验这两点，不满足就报错中止，绝不带病工作。

---

## 自动化处理流程（四个阶段）


| 阶段 | 执行脚本 | 输出/成果 |
| :--- | :--- | :--- |
| **起点** | - | `backup.fd` |
| **阶段1** | `全自动提取IFR.py` | `out/auto/extract1.sct`（复现 UEFITool extract as is）<br>`out/auto/extract1 IFR.txt`（调 CLI 版 ifrextract） |
| **阶段2** | `解锁模块检索.py` | `out/patches.json`（教程 4 步规则检索结果） |
| **阶段3** | `验证解锁效果.py` | 🛡️ 安全网：模拟 SREP 打补丁 |
| **阶段4** | `组装SREP解锁包.py` | `SREP包/SREP_Config.cfg`<br>`SREP包/EFI/BOOT/BOOTX64.efi` |

### 阶段 2 实现的教程 4 步规则

| 步骤 | 模块 | 锚点 | 方向 | 检索串 | 十六进制要求 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Advanced | `Form: Advanced, FormId` | 向下 50 行 | `equals value 0x0` | `12` |
| 2 | OverClocking Performance Menu | `Ref: OverClocking Performance Menu` | 向上 | `equals value 0x0` | `12` |
| 3 | Memory Configuration | `Form: Memory Configuration` | 向下 50 行 | `equals value 0x1` | `12` |
| 4 | OverClocking Feature | `OverClocking Feature` | 向上 | `equals value in list (0x0)` | `14` |



### 阶段 3 是安全网

它在解压镜像上**完全复现 SREP 的写入行为**，然后检查每个目标条件的首字节是否真的变成 `0x47`。**不通过就不生成成品** —— 否则一个静默改错位置的 Pattern 会被直接打进 `SREP_Config.cfg` 上机。

---

## 📁 目录结构

| 目录/文件名 | 说明 |
| :--- | :--- |
| `使用说明.txt` | 用户先看这个（纯文本） |
| `run.cmd` | 启动器（双击） |
| `双击运行.ps1` | 启动器本体 |
| `_runner.py` | 输出捕获（绕开 PowerShell 管道的编码问题） |
| `一键运行.py` | 主流程 |
| `全自动提取IFR.py` | 阶段1：提取 IFR |
| `解锁模块检索.py` | 阶段2：检索解锁模块 |
| `验证解锁效果.py` | 阶段3：验证解锁效果 |
| `组装SREP解锁包.py` | 阶段4：组装 SREP 解锁包 |
| `提取BIOSLock.py` | 可选：单独提取 BIOS Lock 项 |
| `BIOS提取工具/` | 8-9 / 10 / 11 / 12-14 代 FPTW64 备份工具 |
| `tools/ifrextract.exe` | 官方 CLI 版 IFR 提取器（LongSoft v0.3.6） |
| `template/EFI/BOOT/BOOTX64.efi` | 引导器（与固件无关） |

> 工具包内**不含**任何固件；`out/`、`SREP包/`、`运行日志.txt` 都是运行时生成。

---

## 适用范围与限制

### 前提条件

- **BIOS 必须是 Insyde**。换 AMI BIOS 则整套方法不适用。
- **依赖教程约定的 4 个模块名**（见上表）。名字不一致的 BIOS 会明确报「未找到锚点」并中止。

### 已验证

已在 **4 份互不相同**的固件上验证通过（`.sct` 原点、`Form: Advanced` 地址、QuestionId 均不同），包括：

- 13 代 13650HX / Insyde
- 七彩虹 蓝天模具 隐星P15TA（两个不同 BIOS 版本）
- 多份同版本的不同 dump

### 反例（明确失败，而非静默出错）

| 现象 | 含义 |
| :--- | :--- |
| 「未找到合法 PE 候选」 | `.sct` 结构不符，或不是这套 BIOS |
| 自检有模块 ✗ | 该 BIOS 没有这几个菜单 |
| 「未找到锚点」 | 模块名与教程不一致 |
| 「存在未改写 ✗」 | Pattern 改错位置，**不生成成品** |

---

## ❓ 常见问题

**Q：跑成功了但解锁没生效？**
先核对程序打印的固件指纹。最常见的原因是固件选错了（用了示例固件）。

**Q：`ifrextract.exe` 被杀软拦截？**
它来自 LongSoft 官方 release，无代码签名；原版 GUI 程序同样会被误报。可加白名单，或用 `提取BIOSLock.py` 里的纯 Python 路径做单项提取。

**Q：换固件版本要重跑吗？**
要。锚点唯一性、命中次数、地址都会变。直接重跑即可。

**Q：能解锁别的模块吗？**
`解锁模块检索.py` 顶部的 `RULES` 表就是规则表，照着教程格式加一行即可。新增项建议先用 `验证解锁效果.py` 确认。

---

## 免责声明

本工具面向**你拥有或已获得明确授权**的设备，用于合法的硬件调试、性能调优与安全研究。

- 修改 BIOS 设置可能影响系统稳定性、保修状态与安全策略，**风险自负**。
- 请遵守当地法律法规与设备厂商的服务条款。
- 本工具**只读**固件并生成运行时补丁，**不刷写 SPI**、不修改固件内容。

---

## 许可

MIT

`tools/ifrextract.exe` 来自 [LongSoft/Universal-IFR-Extractor](https://github.com/LongSoft/Universal-IFR-Extractor)（GPL-3.0）

`template/EFI/BOOT/BOOTX64.efi` 为 GRUB 引导器，版权归各自作者所有。
