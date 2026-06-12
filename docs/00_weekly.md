### 2. `docs/00_weekly.md`（第一周日志）
# 每周工作日志 - 强化学习导航

## 第一周
### 已完成任务
1. 按照官方规范创建了个人FURP仓库，设置为公开可见
2. 完成系统环境准备：
   - 安装了Habitat所需的所有Ubuntu编译依赖包
   - 安装Miniconda并配置清华Conda镜像源，关闭默认自动激活base环境
3. 搭建独立Python虚拟环境`pytorch_env`：
   - 安装CPU版PyTorch，通过清华PyPI源解决网络中断问题
   - 修正了检查PyTorch版本时的语法错误
4. 从源码编译安装Habitat-Sim v0.2.2（ARM架构无预编译包）：
   - 克隆指定版本源码并拉取所有子模块
   - 安装Python依赖，在无头模式下编译（适配虚拟机）
5. 准备烟雾测试材料：
   - 记录了完整的安装日志与终端截图
   - 下载了Habitat官方测试场景数据集

### 前期遇到的问题与解决方法
1. 直接用pip安装habitat-sim 0.3.1失败：无ARM64架构的预编译包
   解决：改用v0.2.2版本并从源码编译
2. PyTorch下载因网络问题中断
   解决：使用清华PyPI镜像源，清除pip缓存后重新安装
3. Conda包下载速度慢
   解决：配置清华Conda镜像源
   # 我在 ARM Ubuntu 虚拟机编译运行 Habitat‑Sim 项目全流程问题&解决方案总结

> 环境背景：Apple Silicon Mac + ARM aarch64 架构 Ubuntu 22.04 虚拟机，目标编译、运行 `habitat‑sim` + `habitat‑lab` 仿真项目，使用 `pytorch_env` Conda 环境

## 一、硬件 / 虚拟机资源配置相关问题
### 问题1：编译过程中虚拟机自动关机/重启
- 现象：编译中途系统直接进入关机日志、进程被系统终止
- 原因：
  1. `cc1plus`（C++编译进程）CPU满载，虚拟机双核心打满
  2. 物理内存耗尽，触发 Linux OOM 杀手，强制杀死高占用进程/直接关机
- 解决方案：
  1. 内存配置选择：
     - 8GB(8192MB)：可使用但紧张，易大量调用Swap
     - 12GB(12288MB)：折中最优，兼顾Mac主机压力与编译稳定性
     - 16GB(16384MB)：性能充足，但占用Mac大量内存，本机易卡顿
  2. 创建 Swapfile 虚拟内存兜底：
     ```bash
     sudo fallocate -l 4G /swapfile
     sudo chmod 600 /swapfile
     sudo mkswap /swapfile
     sudo swapon /swapfile
     echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
     sudo sysctl vm.swappiness=60
     ```
  3. 编译强制限制线程数，降低峰值负载

### 问题2：如何监控编译时硬件占用
- 工具选择：`btop` / `htop`，均为终端资源监控工具
  - `btop`：界面更炫酷，可视化更强
  - `htop`：轻量简洁，启动更快
- 安装示例(htop)：
  ```bash
  sudo apt update && sudo apt install htop -y
  ```
- 监控重点：内存占用、Swap占用、`cc1plus`/`python`编译进程资源占用

## 二、源码编译 Habitat‑Sim 核心问题
### 问题1：编译100%后，依然报 `ModuleNotFoundError: No module named 'habitat_sim'`
- 现象：编译进度走完，但Python无法找到该模块
- 原因：
  1. Conda 环境丢失，编译安装到了系统默认Python，而非`pytorch_env`
  2. 链接/安装阶段进程被OOM杀手静默终止，未完成部署
  3. 缓存残留导致编译产物异常
- 解决方案：
  1. 编译前确认环境激活：`conda activate pytorch_env`，用`which python`核对路径为conda环境内Python
  2. 彻底清理旧缓存：
     ```bash
     cd ~/桌面/habitat-sim
     rm -rf build build_release dist *.egg-info _skbuild
     ```
  3. 带日志编译，排查隐藏报错：
     ```bash
     CMAKE_BUILD_PARALLEL_LEVEL=2 python setup.py install --headless 2>&1 | tee build_log.txt
     ```
  4. 备选开发模式安装：`python setup.py develop --headless`

## 三、编译后导入&运行报错问题
### 问题1：`ImportError: cannot import name 'ReplayRenderer' from 'habitat_sim'`
- 现象：`habitat‑lab` 上层代码尝试从`habitat_sim`导入`ReplayRenderer`类失败
- 核心原因：
  1. 我编译的是`--headless`无头模式，该模式裁剪了图形/回放渲染相关模块，原生不包含`ReplayRenderer`
  2. `habitat‑lab` 与编译的`habitat‑sim`版本/编译模式不匹配
- 解决方案（3种优先级方案）：
  1. 临时绕过（快速验证）：注释报错文件中对应导入行
  2. 规范匹配：卸载现有包，编译适配的非无头版本（ARM虚拟机无显卡，成功率极低）或匹配对应版本的`habitat‑lab`
  3. 架构兜底：ARM aarch64 对该项目兼容性极差，切换 x86‑64 架构设备/服务器，使用官方预编译包

### 问题2：Gym 版本弃用警告、NumPy 版本兼容警告
- 警告原文：`Gym has been unmaintained since 2022... upgrade to Gymnasium`
- 性质：**非致命警告，不直接导致程序崩溃**
- 解决方案：
  1. 临时规避：降级NumPy到1.x版本：`pip install "numpy<2.0"`
  2. 长期方案：按照官方迁移指南，将项目中`gym`替换为维护版`gymnasium`

### 问题3：Magnum 插件路径警告
- 警告原文：`none of the plugin search paths ... exists, skipping plugin discovery`
- 性质：**无头模式下正常现象，可忽略**
- 原因：无头编译禁用了图形导入功能，图形库无需加载对应插件

### 下周计划
1. 完成Habitat烟雾测试并保存完整运行日志作为提交材料
2. 学习habitat-baselines中的PointNav强化学习基线算法
3. 阅读关于强化学习导航奖励设计的相关论文
