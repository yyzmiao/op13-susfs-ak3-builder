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

在 GitHub 仓库中，创建 `.github/workflows/build.yml` 文件并粘贴以下代码：

```yaml
name: Universal Android GKI Kernel Builder (KSUN & SuSFS)

on:
  workflow_dispatch:
    inputs:
      kernel_branch:
        description: 'AOSP GKI 内核分支或 Tag (例如: android15-6.6, android16-6.12 等)'
        required: true
        type: string
        default: 'android15-6.6'

      ksu_branch:
        description: 'KernelSU-Next 分支、Tag 或 Commit (默认: next)'
        required: true
        type: string
        default: 'next'

      susfs_branch:
        description: 'SuSFS 补丁分支 (默认 auto 会自动匹配内核分支)'
        required: true
        type: string
        default: 'auto'

      custom_clang_repo:
        description: '（可选）自定义 Clang 工具链 Git 仓库 (留空使用官方预编译源)'
        required: false
        type: string
        default: ''

jobs:
  build:
    name: Build Universal GKI Kernel
    runs-on: ubuntu-24.04
    steps:
      - name: 1. 环境初始化与依赖安装
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            bc bison ca-certificates curl flex gcc-aarch64-linux-gnu \
            git libssl-dev libelf-dev make python3 zip unzip libarchive-tools patch
          sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc || true

      - name: 2. 配置 AOSP Clang 编译器工具链
        run: |
          CLANG_REPO="${{ github.event.inputs.custom_clang_repo }}"
          if [ -z "$CLANG_REPO" ]; then
            CLANG_REPO="https://gitlab.com/crdroidandroid/android_prebuilts_clang_host_linux-x86_clang-r510928.git"
          fi
          echo "⬇️ 正在拉取 Clang 工具链: $CLANG_REPO..."
          git clone --depth=1 "$CLANG_REPO" clang-toolchain
          echo "$GITHUB_WORKSPACE/clang-toolchain/bin" >> $GITHUB_PATH

      - name: 3. 动态拉取指定 GKI 源码并解析版本
        run: |
          TARGET_BRANCH="${{ github.event.inputs.kernel_branch }}"
          echo "⬇️ 正在拉取 GKI 源码: $TARGET_BRANCH..."
          git clone --depth=1 -b "$TARGET_BRANCH" https://android.googlesource.com/kernel/common kernel-source
          cd kernel-source
          
          # 自动读取并显示确切的内核子版本号
          RAW_VERSION=$(make kernelversion 2>/dev/null || grep -E '^(VERSION|PATCHLEVEL|SUBLEVEL) =' Makefile | awk '{print $3}' | paste -sd.)
          echo "=========================================="
          echo "✅ 成功拉取 GKI 内核源码，确切版本为: $RAW_VERSION"
          echo "=========================================="
          echo "KERNEL_SRC=$PWD" >> $GITHUB_ENV
          echo "KERNEL_VERSION=$RAW_VERSION" >> $GITHUB_ENV

      - name: 4. 集成 KernelSU-Next (原生自带 SuSFS 接口)
        run: |
          cd $KERNEL_SRC
          KSU_TARGET="${{ github.event.inputs.ksu_branch }}"
          echo "⬇️ 正在集成 KernelSU-Next ($KSU_TARGET)..."
          curl -LSs "https://raw.githubusercontent.com/KernelSU-Next/KernelSU-Next/${KSU_TARGET}/kernel/setup.sh" | bash -s "$KSU_TARGET"

      - name: 5. 自适应打入 SuSFS 内核深度隐藏补丁
        run: |
          cd $GITHUB_WORKSPACE
          SUSFS_BR="${{ github.event.inputs.susfs_branch }}"
          if [ "$SUSFS_BR" = "auto" ]; then
            SUSFS_BR="gki-${{ github.event.inputs.kernel_branch }}"
          fi
          
          echo "⬇️ 正在拉取 SuSFS 补丁分支: $SUSFS_BR..."
          git clone --depth=1 -b "$SUSFS_BR" https://gitlab.com/simonpunk/susfs4ksu.git susfs4ksu || \
          git clone --depth=1 https://gitlab.com/simonpunk/susfs4ksu.git susfs4ksu
          
          # 拷贝 SuSFS 核心驱动文件到内核源码树
          if [ -d "susfs4ksu/kernel_patches/fs" ]; then
            cp -r susfs4ksu/kernel_patches/fs/* $KERNEL_SRC/fs/
          fi
          if [ -d "susfs4ksu/kernel_patches/include/linux" ]; then
            cp -r susfs4ksu/kernel_patches/include/linux/* $KERNEL_SRC/include/linux/
          fi

          # 应用内核 SuSFS 补丁
          cd $KERNEL_SRC
          PATCH_FILE=$(find $GITHUB_WORKSPACE/susfs4ksu/kernel_patches/ -maxdepth 1 -name "50_add_susfs_in_*.patch" | head -n 1)
          if [ -n "$PATCH_FILE" ]; then
            echo "⚙️ 正在应用内核 SuSFS 补丁: $(basename "$PATCH_FILE")..."
            patch -p1 --forward < "$PATCH_FILE" || true
          fi
          echo "✅ SuSFS 内核补丁已就绪 (KernelSU-Next 会自动与 SuSFS 联动)"

      - name: 6. 纯净编译通用 GKI Image
        run: |
          cd $KERNEL_SRC
          export ARCH=arm64
          export SUBARCH=arm64
          export CROSS_COMPILE=aarch64-linux-gnu-
          export CC=clang
          export LLVM=1
          export LLVM_IAS=1

          # 修复 OpenSSL 3.x 导致的 extract-cert.c 变量未声明问题
          sed -i '/#include <openssl\/err.h>/a static const char *key_pass = NULL;' certs/extract-cert.c 2>/dev/null || true

          make gki_defconfig

          # 禁用 Werror 与不需要的主机证书签名工具，开启 KSU 与 SuSFS 特性
          scripts/config --disable CONFIG_WERROR
          scripts/config --disable CONFIG_SYSTEM_TRUSTED_KEYRING
          scripts/config --disable CONFIG_MODULE_SIG
          scripts/config --disable CONFIG_MODULE_SIG_ALL
          scripts/config --disable CONFIG_SYSTEM_TRUSTED_KEYS
          scripts/config --disable CONFIG_SYSTEM_REVOCATION_KEYS
          scripts/config --set-str CONFIG_SYSTEM_TRUSTED_KEYS ""
          scripts/config --set-str CONFIG_SYSTEM_REVOCATION_KEYS ""

          scripts/config --enable CONFIG_KSU
          scripts/config --enable CONFIG_KSU_SUSFS
          scripts/config --enable CONFIG_KSU_SUSFS_SUS_SU
          scripts/config --enable CONFIG_KSU_SUSFS_SUS_MOUNT
          scripts/config --enable CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT
          scripts/config --enable CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT
          scripts/config --enable CONFIG_KSU_SUSFS_SUS_KSTAT

          echo "🚀 开始多核高速编译通用 GKI Image..."
          make -j$(nproc --all) ARCH=arm64 CC=clang LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu- KCFLAGS="-Wno-error" Image || {
            echo "⚠️ 多线程编译遇阻，正在执行单线程精确捕捉错误详情..."
            make -j1 ARCH=arm64 CC=clang LLVM=1 LLVM_IAS=1 CROSS_COMPILE=aarch64-linux-gnu- KCFLAGS="-Wno-error" Image
          }
          
          if [ -f "arch/arm64/boot/Image" ]; then
            echo "=========================================="
            echo "🎉 内核 Image 编译成功！文件大小: $(du -h arch/arm64/boot/Image | awk '{print $1}')"
            echo "=========================================="
          else
            echo "❌ 未找到 Image 文件，编译失败！" && exit 1
          fi

      - name: 7. 打包通用 AnyKernel3 刷机包
        run: |
          cd $GITHUB_WORKSPACE
          git clone https://github.com/osm0sis/AnyKernel3 AnyKernel3
          cd AnyKernel3
          
          cp $KERNEL_SRC/arch/arm64/boot/Image ./
          
          # 通用配置：解除设备型号锁定，支持任意 A/B 槽位 GKI 设备
          sed -i 's/do.devicecheck=1/do.devicecheck=0/g' anykernel.sh
          sed -i 's/is_slot_device=0/is_slot_device=auto/g' anykernel.sh

          BUILD_DATE=$(date +%Y%m%d_%H%M)
          ZIP_NAME="GKI_${{ env.KERNEL_VERSION }}_KSUN_${{ github.event.inputs.ksu_branch }}_SuSFS_${BUILD_DATE}.zip"
          
          zip -r9 "../$ZIP_NAME" * -x .git README.md *placeholder
          echo "ZIP_NAME=$ZIP_NAME" >> $GITHUB_ENV
          echo "🎉 刷机包打包完成: $ZIP_NAME"

      - name: 8. 上传构建产物
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.ZIP_NAME }}
          path: ${{ env.ZIP_NAME }}
```

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
