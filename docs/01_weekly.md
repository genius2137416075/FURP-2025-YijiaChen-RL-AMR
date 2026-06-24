# 因为Mac跑不了，所以这周等硬盘到了重新配置了一遍环境
## 项目概况
Ubuntu22.04，ROS2 Humble + RL视觉导航项目，安装ROS2、Miniconda、独立Python3.10虚拟环境、Gazebo仿真，Habitat视觉仿真。

---
## 一、ROS2 Humble 安装相关问题
### 问题1：执行安装命令提示「无法定位软件包 ros-humble-dwa-local-planner」
原因：ROS2不再使用ROS1的DWA包，包名已重构
解决：
1. 开启系统universe仓库
```bash
sudo add-apt-repository universe
sudo apt update
```
2. 修正安装包名，替换为`ros-humble-dwb-plugins`
分段安装避免依赖报错：
```bash
# 桌面基础+导航SLAM
sudo apt install -y ros-humble-desktop ros-humble-turtlebot3 ros-humble-navigation2 ros-humble-slam-toolbox ros-humble-cv-bridge ros-humble-rviz2 ros-humble-dwb-plugins
# Gazebo仿真套件
sudo apt install -y ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control
```

### 问题2：找不到curl命令，无法导入ROS密钥
解决：先安装curl工具
```bash
sudo apt install curl -y
# 再导入密钥
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```
---
## 二、Miniconda 安装 & 虚拟环境全套403报错问题
### 问题1：wget下载Miniconda脚本报403 Forbidden
原因：Anaconda官方源国内访问拦截
解决：换清华镜像下载脚本
```bash
wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh
bash miniconda.sh -b -p $HOME/miniconda3
rm miniconda.sh
# 配置conda环境
echo "source $HOME/miniconda3/etc/profile.d/conda.sh" >> ~/.bashrc
source ~/.bashrc
conda init bash
```

### 问题2：conda create创建环境持续报 CondaHTTPError 403 Forbidden
根源：conda通道里残留`defaults`（国外官方源），检索包时会自动访问被墙地址
#### 踩坑点1：`conda config --remove defaults` 语法报错
错误：--remove 需要两个参数，不能直接删通道名
#### 踩坑点2：单纯添加清华源无法删除defaults，仍会触发403
#### 解决方案
1.临时强制绕过所有默认通道创建环境
```bash
conda create -n vln_rl_nav python=3.10 -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r --override-channels -y
```

### 问题3：不创建独立conda虚拟环境会有什么影响
1. 全局Python版本、库版本冲突，项目依赖不兼容；
2. 污染系统base环境，大量依赖难以清理；
3. 项目无法复现，换设备后代码大概率运行失败；
4. RL项目需要Python3.10、特定版本PyTorch，全局环境极易出现版本bug。

### 虚拟环境基础操作命令
```bash
# 创建环境
conda create -n vln_rl_nav python=3.10 -y
# 激活环境
conda activate vln_rl_nav
# 退出环境
conda deactivate
# 查看所有环境
conda env list
```

---
## 三、硬件/系统相关问题
### 问题1：内置nvme硬盘在项目里的作用
1. 存储Ubuntu系统、ROS、Conda、全部代码、训练数据集；
2. Gazebo/Habitat仿真加载地图、纹理、RGB-D场景依赖硬盘读写速度；
3. RL训练产生模型权重、日志文件频繁读写固态；
4. 内存不足时swap分区依托硬盘防止程序崩溃。

---
## 四、仿真器选择：Gazebo vs Habitat（RL视觉导航项目）
### Gazebo（已随ROS2安装）
适用：激光雷达导航、底盘运动控制、ROS真机迁移、2D路径规划；

### Habitat-Sim
适用：室内RGB-D视觉、自然语言指令导航、端到端视觉强化学习；
安装命令：
```bash
sudo apt-get install -y cmake build-essential libgl1-mesa-dev libglm-dev libpng-dev libjpeg-dev libtiff-dev
conda install -c aihabitat habitat-sim headless=0.2.4 cudatoolkit=11.8
pip install habitat-lab habitat-baselines
```

### 区分
仅做激光RL：只用Gazebo；
做视觉语言VLN RL：Gazebo+Habitat都需要。

---
## 五、环境验证&烟雾测试命令汇总
1. ROS2验证
```bash
printenv ROS_DISTRO
```
2. Conda版本验证
```bash
conda -V
```
3. Conda通道校验
```bash
conda config --show channels
```
4. Habitat安装校验（安装后）
```python
import habitat_sim
print(habitat_sim.__version__)
```
# 后续完整任务清单
## 1. 激活conda环境，安装全部依赖
1. 激活环境：`conda activate vln_rl_nav`
2. 安装GPU PyTorch、强化学习、图像处理、ROS相关pip库
3. 导出两份环境依赖文档，方便复现项目

## 2. 搭建ROS2强化学习工作空间
1. 创建`rl_ws`工作目录与rl导航功能包
2. 编写核心Python节点：订阅传感器、执行PPO推理、下发小车速度、计算奖励、环境重置
3. colcon编译，刷新环境变量

## 3. Gazebo仿真完整训练闭环调试
1. 启动Turtlebot3仿真世界
2. 运行RL训练节点，完成整套训练流程，保存模型权重
3. 测试训练完成的模型，统计导航成功率，保证算法稳定收敛

## 4. 特色拓展
1. 安装Habitat视觉仿真环境
2. 实现图像+激光多模态输入PPO网络
3. 添加动态障碍物、域随机化优化仿真转真机效果

## 5. 真机部署前置准备
1. 电脑与小车配置同一局域网、ROS多机通信
2. 代码适配真实硬件传感器话题
3. 将仿真训练权重迁移到真机运行

## 6. 收尾整理
1. 整理启动脚本、README说明
2. 汇总实验数据、对比实验结果，完善项目报告
