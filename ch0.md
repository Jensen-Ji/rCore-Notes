# 实验环境

## 1. 安装Ubuntu

#### 在本地环境（win10）上使用WSL2来安装Ubuntu

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

#### 注：导出并在其他盘恢复
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
```

如此，安装的Ubuntu就从C盘迁移到了D:\WSL\Ubuntu。
然后启动，设置用户名和密码即可。

#### 附：注销和彻底删除Ubuntu后再重新安装
```powershell
wsl --list --verbose
wsl --terminate Ubuntu-24.04
wsl --unregister Ubuntu-24.04
wsl --install -d Ubuntu-24.04
```

## 2. 在Ubuntu中安装并配置实验环境

#### 0. 换源
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

#### 1. 基础环境准备（Linux）
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

#### 2. 安装Rust工具链
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

# 附加：更新和卸载Rust命令
rustup update
rustup self uninstall
```

#### 3. 安装QEMU
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

#### 4. 拉取代码并运行
```bash
# 拉取仓库
cd ~ 
git clone https://github.com/rcore-os/rCore-Tutorial-v3.git

# 运行
cd rCore-Tutorial-v3/
cd os/
make run
```
