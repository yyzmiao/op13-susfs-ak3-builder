# 一加 13 (OnePlus 13 / PJZ110) KernelSU-Next + SuSFS 自动化编译与维护指南

> **适用机型**：一加 13 (PJZ110 / Snapdragon 8 Elite)  
> **系统版本**：ColorOS 16 (Android 15) 及后续大版本更新  
> **架构特性**：Google GKI 6.6 (Generic Kernel Image) 通用内核架构  
> **核心组件**：KernelSU-Next (KSUN) + SuSFS (v1.5.5 / v2.x) 深度隐藏 + AnyKernel3 (AK3) 刷机包

---

## 目录
- [一、 核心原理与优势](#一-核心原理与优势)
- [二、 GitHub Actions 云端全自动编译](#二-github-actions-云端全自动编译)
- [三、 内核刷入方式](#三-内核刷入方式)
- [四、 ColorOS OTA 系统更新与无缝保 Root 流程](#四-coloros-ota-系统更新与无缝保-root-流程)
- [五、 常见问题与避坑指南 (FAQ)](#五-常见问题与避坑指南-faq)

---

## 一、 核心原理与优势

1. **Google GKI 2.0 (KMI 严格兼容)**：
   * 在 Android 15 (Kernel 6.6) 体系下，Google 强制推行通用的 Linux 内核接口（KMI）。
   * 只要处于同个安卓大版本（Android 15 6.6），无论官方系统小版本如何演进（例如从 `16.0.9.402` 升级到后续版本），编译出的 GKI 6.6 内核均可完全通用，WiFi、蓝牙、触屏、相机等硬件驱动 100% 正常工作。
2. **纯净构建，解除底层写保护**：
   * 本方案从 Google AOSP 官方仓库拉取纯净 GKI 源码，只打入纯净的 KSUN 与 SuSFS 补丁。
   * 去除了第三方内核的“防格机/块设备写保护”拦截，保证底层分区读写正常，避免系统工具或分区操作报错。
3. **SuSFS 深度隐藏**：
   * 将 SuSFS 深度编入内核源码（In-Tree 模式），配合配套的 KSU 模块，实现针对银行、游戏以及严格环境检测的底层级隐藏。

---

## 二、 GitHub Actions 云端全自动编译

### 1. 仓库配置
在你的 GitHub 仓库中，创建路径为 `.github/workflows/build.yml` 的文件，并写入工作流代码（支持自动探测内核具体版本、智能匹配 SuSFS 补丁并自动输出打包好的 AK3 zip）。

### 2. 触发编译与下载
1. 打开 GitHub 仓库，进入 `Actions` 页面。
2. 在左侧选择工作流，点击右侧 `Run workflow` 按钮。
3. 确认分支参数（一加 13 保持默认 `android15-6.6`，KSUN 保持 `next`），点击绿色按钮启动编译。
4. 等待 6 ~ 10 分钟，编译完成后在页面下方的 `Artifacts` 区域直接点击下载打包好的 `.zip` 文件。

---

## 三、 内核刷入方式

### 方式 1：手机端一键刷入（推荐）
* **已有 Root 环境**：
  * 打开 KernelSU / KernelSU-Next 管理器 -> 进入「模块」 -> 选择下载的 AK3 `.zip` 包刷入 -> 重启手机。
  * 或者使用 KernelFlasher App 选择该 zip 刷入当前槽位。

### 方式 2：电脑 Fastboot 模式刷入
* 如果当前处于未 Root 状态或开机异常：
  1. 将 AK3 `.zip` 解压出里面的 `Image` 文件。
  2. 手机进入 Fastboot 模式（`adb reboot bootloader`）。
  3. 执行刷入命令：
     ```bash
     fastboot flash boot Image
     fastboot reboot
     ```

---

## 四、 ColorOS OTA 系统更新与无缝保 Root 流程

一加 13 采用 A/B 双槽位无缝升级机制。系统更新时，新系统会安装在闲置的另一个槽位。

### 标准无痛升级步骤（免电脑）
1. **下载安装更新**：
   * 打开系统「设置」->「关于本机」->「系统更新」，下载全量更新包并点击安装。
   * 等待安装进度完成，系统提示 “安装完成，请重启设备”。
2. **⚠️ 切勿立即点击系统重启**：
   * 停留在该界面，切到后台打开 KernelFlasher App。
3. **刷入新槽位**：
   * 在 KernelFlasher 中选择编译好的 AK3 刷机包（.zip）。
   * 刷入目标槽位选择 `Inactive Slot`（未激活槽位 / 另一个槽位） 或 `Both Slots`（全部槽位） 并确认刷写。
4. **重启设备**：
   * 刷写完成后，在工具内或回到系统更新页面点击重启。
   * 手机开机后即直接进入升级后的新系统，并自动保留最新的 AK3 内核与 Root 权限！

---

## 五、 常见问题与避坑指南 (FAQ)

### Q1: KernelSU 管理器提示内核需要更新，能直接在 App 里点“直接安装”吗？
> **答：绝对不要在 App 里点“直接安装”！**  
> SuSFS 内核是直接编译在 `boot` 分区中的（In-Tree 模式）。如果在 App 里点“直接安装”，App 会向 `init_boot` 分区写入一套 LKM 模块，造成双重驱动冲突甚至卡开机。  
> **正确更新方式**：去 GitHub 点一次 `Run workflow`，下载新生成的 AK3 刷入即可。

### Q2: 刷了新内核后，SuSFS 如何启用深度隐藏？
> **答**：在 KernelSU 管理器中安装配套的 `ksu_module_susfs` 模块，重启后在终端执行 `ksu_susfs --status` 即可查看 SuSFS 运行状态。

### Q3: 未来如果升级到了 Android 16 (ColorOS 17)，怎么维护？
> **答**：无需改动脚本代码。在 GitHub 点击 `Run workflow` 时，在下拉菜单中将内核分支切换到 `android16-6.12`（或对应的新版本分支），点击编译即可生成专属 Android 16 的全新 AK3 刷机包。
