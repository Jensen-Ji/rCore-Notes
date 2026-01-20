# 实验环境

## 1. 安装Ubuntu

### 1. 在本地环境（win10）上使用WSL2来安装Ubuntu

```powershell
# 1. 启用WSL子系统功能
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

# 2. 启用虚拟机平台组件
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform

# 3. 更新WSL内核
wsl --update

# 4. 设置默认版本为WSL2
wsl --set-default-version 2

# 5. 列出云端可用的发行版
wsl --list --online

# 6. 下载指定的Linux发行版
wsl --install -d Ubuntu-24.04
```

其中 5、6两步可替换为 在Microsoft Store中下载Linux发行版

### 2. 注：导出并在其他盘恢复
因为默认安装在C盘，可以创建快照导出，并在其他盘恢复快照，
```powershell
# 0. 查看当前的Ubuntu版本
wsl --list

# 1. 强制关闭Ubuntu
wsl --terminate Ubuntu-24.04

# 2. 将Ubuntu导出为一个压缩包
wsl --export Ubuntu-24.04 D:\WSL\Ubuntu.tar

# 3. 注销并和删除旧的Ubuntu
wsl --unregister Ubuntu-24.04

# 4. 从压缩包创建一个新的WSL系统
wsl --import Ubuntu D:\WSL\Ubuntu D:\WSL\Ubuntu.tar

# 注：可能在迁移之后，会出现默认用户是root的情况，此时需要修改配置文件
nano /etc/wsl.conf
# 添加以下内容，然后保存退出，重启WSL
[user]
default = 用户名
```

如此，安装的Ubuntu就从C盘迁移到了D:\WSL\Ubuntu。
然后启动，设置用户名和密码即可。

### 3. 附：注销和彻底删除Ubuntu后再重新安装
```powershell
wsl --list --verbose
wsl --terminate Ubuntu-24.04
wsl --unregister Ubuntu-24.04
wsl --install -d Ubuntu-24.04
```

## 2. 在Ubuntu中安装并配置实验环境

### 1. 换源
```bash
# 1. 备份原来的源配置文件
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak 

# 2. 使用 sed 命令一键替换网址 (将官网替换为阿里云镜像) 
# 替换 archive.ubuntu.com -> mirrors.aliyun.com 
sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list.d/ubuntu.sources 

# 替换 security.ubuntu.com -> mirrors.aliyun.com 
sudo sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list.d/ubuntu.sources

# 3. 更新索引
sudo apt update
```

### 2. 基础环境准备（Linux）
```bash
# 更新软件包列表并升级已安装软件
sudo apt update && sudo apt upgrade -y

# 安装基础开发工具（git、编译器、python等）
sudo apt install -y git build-essential ninja-build tmux curl python3 python3-pip

# 安装自动化构建与解析工具（编译QEMU源码的工具）
sudo apt install -y autoconf automake autotools-dev libtool pkg-config patchutils
sudo apt install -y gawk bison flex texinfo gperf bc

# 安装核心依赖库（QEMU运行需要）
sudo apt install -y libmpc-dev libmpfr-dev libgmp-dev libglib2.0-dev libpixman-1-dev zlib1g-dev libexpat-dev
```

### 3. 安装Rust工具链
```bash
# 安装rustup，按回车默认即可
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh

# 上命令失败或卡住，可尝试如下
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
curl https://sh.rustup.rs -sSf | sh

# 安装完成后，刷新环境变量
source $HOME/.cargo/env

# 查看Rust版本
rustc --version

# 安装并切换到Nightly版本
rustup install nightly
rustup default nightly

# 安装组件
rustup target add riscv64gc-unknown-none-elf 
rustup component add llvm-tools-preview 
rustup component add rust-src

# 安装cargo工具
cargo install cargo-binutils --vers =0.3.3
#或者 cargo install cargo-binutils --version 0.3.3

# 附加：更新和卸载Rust命令
rustup update
rustup self uninstall
```

### 4. 国内Rust下载慢的解决方法
```bash
# 参考rsproxy

# 1. 编辑 bash 配置文件
nano ~/.bashrc

# 2. 在文件末尾添加以下两行（指定 Rustup 更新源）
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"

# 3. 保存退出，并让配置生效
source ~/.bashrc

# 4. 创建 Cargo 配置目录
mkdir -p ~/.cargo

# 5. 新建/编辑配置文件
nano ~/.cargo/config.toml

# 6. 粘贴以下内容（这是 rsproxy 的优化配置）：
[source.crates-io]
replace-with = 'rsproxy-sparse'

[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"

[net]
git-fetch-with-cli = true

# 7. 刷新环境
source $HOME/.cargo/env


# 如果是windows环境 处理方法类似
# 1. 添加环境变量
变量名 RUSTUP_DIST_SERVER
变量值 https://rsproxy.cn

变量名 RUSTUP_UPDATE_ROOT
变量值 https://rsproxy.cn/rustup

# 2.安装官网下载的rustup-init.exe

# 3.配置Cargo源
# 在C:\Users\你的用户名\.cargo 下新建config.toml文件，内容如下
[source.crates-io]
replace-with = 'rsproxy-sparse'

[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"

[net]
git-fetch-with-cli = true
```

### 5. 安装QEMU
```bash
# 下载并解压QEMU，可能会出现SSL问题，重复尝试即可
wget https://download.qemu.org/qemu-7.0.0.tar.xz
tar xvJf qemu-7.0.0.tar.xz

# 附加：安装x86架构的QEMU
sudo apt install qemu-system-x86

# 编译，只编译RISC-V架构
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)

# 配置环境变量
# 先编辑.bashrc文件
nano ~/.bashrc
# 然后在文档最后，添加如下三行，然后ctrl+x，按Y，再按enter保存退出
export PATH="$HOME/qemu-7.0.0/build:$PATH"
export PATH="$HOME/qemu-7.0.0/build/riscv64-softmmu:$PATH"
export PATH="$HOME/qemu-7.0.0/build/riscv64-linux-user:$PATH"

#刷新环境变量
source ~/.bashrc
```

### 6. 拉取代码并运行
```bash
# 拉取仓库
cd ~ 
git clone https://github.com/rcore-os/rCore-Tutorial-v3.git

# 运行
cd rCore-Tutorial-v3/
cd os/
make run
```

## 3. 配置Git

### 1. 安装git
```bash
# 更新源，并安装
sudo apt update
sudo apt install git -y
git --version
```

### 2. 全局基础配置（配置身份）
```bash
# 1. 设置用户名（建议与Github用户名一致）
git config --global user.name "username"

# 2. 设置邮箱
git config --global user.email "email@gmail.com"

# 3. 设置默认主分支名为 main (替代旧的 master)
git config --global init.defaultBranch main

# 4. 解决跨平台换行符问题 (Windows与Linux混用时非常重要)
# 如果你在 Windows 上写代码：
git config --global core.autocrlf true
# 如果你在 WSL/Linux 上写代码 (推荐)：
git config --global core.autocrlf input
```

### 3. 配置SSH密钥（免密登录Github）
```bash
# 邮箱换成 GitHub 邮箱，-C 是注释，一直按回车确认即可
ssh-keygen -t ed25519 -C "email@example.com"

# 查看并复制输出的内容
cat ~/.ssh/id_ed25519.pub

# 将复制的内容，粘贴到Github上
Setting -> SSH and GPG keys -> New SSH key

# 验证连接，需要yes确认一下
ssh -T git@github.com
```

### 4. 创建仓库并上传本地代码
```bash
# A：从零开始
# 1. 先在Github上新建仓库
# 2. 在本地创建文件夹，并初始化，然后提交
mkdir my_pro
cd my_pro
git init
git add .
git commit -m "First commit"
git remote add origin git@github.com:用户名/仓库名.git
git push -u origin main

# B：已有本地代码，上传到仓库
# 1. 先创建仓库
# 2. 本地初始化并提交
git init
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:用户名/仓库名.git
git branch -M main
git push -u origin main
```

# 什么是操作系统

图例示意：
红色：代表 硬件/物理层
蓝色：代表 内核态/OS
绿色：代表 用户态/App

## 1. 操作系统

操作系统是软件，帮助用户和应用程序使用和管理计算机资源
![ch0-001.excalidraw](ch0-001.excalidraw.md)

操作系统对用户是不可见的，但是它控制其他各种计算机系统
![ch0-002.excalidraw](ch0-002.excalidraw.md)

## 2. 系统软件

系统软件包括操作系统内核、驱动程序、工具软件、用户界面、软件库等
操作系统内核是核心，负责控制硬件资源并为用户和应用程序提供服务
驱动程序用于控制硬件设备，一般情况下，**驱动程序是操作系统内核的一部分**
工具软件用于维护、调试和优化计算机系统
用户界面用来帮助用户使用操作系统
软件库提供系统调用的接口
![ch0-003.excalidraw](ch0-003.excalidraw.md)

在微内核架构中，驱动独立与内核之外
在宏内核架构中，驱动包含在内核里面，或者作为模块加载进内核

## 3. 执行环境

在应用程序的角度：

执行环境提供运行应用软件所需要的运行时服务
包括：内存管理、文件系统访问、网络连接等，这些服务多数都由操作系统提供

简化了操作系统的概念：**操作系统就是应用程序的软件执行环境。**
软件在执行时，需要什么，操作系统就提供什么，也就是软件的执行环境。

ABI：
为什么是Binary，因为应用程序运行时，已经编译好了，是exe等形式，不是源码
所以和应用程序沟通，不能使用API，也就是函数名
需要使用寄存器、机器码指令等
通过编译器把API变成ABI

狭义的操作系统：操作系统内核
广义的操作系统：内核+库+GUI+工具等

![ch0-004.excalidraw](ch0-004.excalidraw.md)

一般情况下，APP通过API调用库，库通过ABI调用内核
在这里，APP通过ABI直接调用内核，也是rCore第一章要完成的

**执行环境 = System Software + Hardware** 或者 **执行环境 = OS + Hardware**

## 4. 操作系统的定义和组成

![ch0-005.excalidraw](ch0-005.excalidraw.md)
操作系统是向上为应用程序提供服务，向下管理硬件资源

操作系统怎么和硬件打交道，怎么提供服务?

操作系统主要组成：
- 操作系统内核
- 系统工具和软件库
- 用户接口

操作系统内核包括：
- 进程线程管理
- 内存管理
- 文件系统
- 网络通信
- 设备驱动
- 同步互斥
- 系统调用接口

这些也是rCore这本书讲述的重点

# 操作系统的系统调用

## API 和 ABI

