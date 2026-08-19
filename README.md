# 通用 Android GKI (KernelSU-Next + SuSFS) 自动化编译与维护指南

> **适用架构**：Google GKI (Generic Kernel Image) 通用内核设备（如一加、小米、OPPO、vivo、Pixel、三星等）  
> **支持系统**：Android 13 / 14 / 15 / 16 及未来任意版本  
> **核心组件**：KernelSU-Next (KSUN) + SuSFS 深度隐藏 + AnyKernel3 (AK3) 纯净通用包

---

## 目录
- [一、 核心原理](#一-核心原理)
- [二、 GitHub Actions 永久通用工作流 (build.yml)](#二-github-actions-永久通用工作流-buildyml)
- [三、 内核刷入方式](#三-内核刷入方式)
- [四、 系统 OTA 无缝保 Root 流程](#四-系统-ota-无缝保-root-流程)
- [五、 常见问题与日常维护 (FAQ)](#五-常见问题与日常维护-faq)

---

## 一、 核心原理

1. **Google GKI 标准与 KMI 通用性**：
   * GKI 是 Google 官方统一定义的通用 Linux 内核镜像。在同一个 Android 内核大版本下，所有 GKI 设备共享相同的内核接口（KMI）。
   * 编译生成的 GKI 内核包具有**全机型跨设备通用性**，无需为每款机型单独适配。
2. **纯净编译，无分区写拦截**：
   * 本方案从 Google AOSP 官方直接拉取源码，只集成 KSUN 与 SuSFS 补丁，**去除了所有第三方防格机/写保护补丁**，确保底层分区读写正常。
3. **自适应与永久通用架构**：
   * 工作流代码全动态解析，无论未来升级到哪个 Android 版本，直接输入对应分支名即可秒级构建，无需修改脚本代码。

---

## 二、 GitHub Actions 永久通用工作流 (build.yml)

在 GitHub 仓库中，创建 `.github/workflows/build.yml` 文件
---

## 三、 内核刷入方式

### 方式 1：手机端直接刷入（推荐）
* 打开 **KernelSU / KernelSU-Next 管理器** -> 进入「模块」 -> 选择生成的 AK3 `.zip` 刷入 -> 重启。
* 或使用 **KernelFlasher** App 直接刷入当前槽位。

### 方式 2：电脑 Fastboot 模式刷入
* 解压 AK3 `.zip` 提取其中的 `Image` 文件。
* 手机进入 Fastboot 模式执行：
  ```bash
  fastboot flash boot Image
  fastboot reboot
  ```

---

## 四、 系统 OTA 无缝保 Root 流程

1. **下载安装更新**：系统内正常下载 OTA 全量包并安装，等待出现 **“安装完成，请重启设备”**。
2. **切勿立即重启**：打开 **KernelFlasher** App。
3. **刷入新槽位**：选择 AK3 `.zip` 包，刷入目标勾选 **`Inactive Slot`（未激活槽位）** 或 **`Both Slots`（全部槽位）** 并确认刷写。
4. **重启设备**：刷写完成后重启，手机即可直接进入升级后的新系统并自动保留最新内核与 Root 权限。

---

## 五、 常见问题与日常维护 (FAQ)

### Q1: KernelSU 管理器提示内核需要更新，能在 App 内点“直接安装”吗？
> **答：绝对不要点！**  
> SuSFS 内核是内置在 `boot` 分区中的（In-Tree 模式）。在 App 里点“直接安装”会往 `init_boot` 写入 LKM 模块造成双重冲突。  
> **正确更新方式**：去 GitHub 点一次 `Run workflow`，下载新生成的 AK3 刷入即可。

### Q2: 未来跨安卓大版本（如 Android 16 / 17）怎么操作？
> **答**：打开 GitHub 页面点击 `Run workflow`，直接在输入框修改内核分支名（如 `android16-6.12`、`android17-6.xx` 等），脚本会自动匹配全套流程并完成编译。
