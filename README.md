# FURP-2026-YijiaChen-RL-AMR

## 基本信息
- 学生姓名：陈易佳
- 项目名称：End-to-End Navigation for an AMR with Reinforcement Learning
- 赛道：研究赛道
- 指导老师：Dr.Tianxiang Cui
- 项目形式：个人项目
- 仓库地址：https://github.com/genius2137416075/FURP-2025-YijiaChen-RL-AMR

## 项目简介
本项目研究基于强化学习的自主移动机器人（AMR）端到端导航方法。
我们在Habitat-Sim仿真平台搭建完整训练与测试流程，训练强化学习策略，让机器人从原始视觉观测直接输出导航动作。
研究重点包括奖励函数设计、环境状态表示、导航成功率评估等内容，属于具身智能、机器人学与机器学习的交叉方向。

第一周核心目标：完成开发环境搭建与Habitat仿真环境验证。

## 开发环境
1. 系统：Ubuntu 22.04 ARM64（Apple M1虚拟机）
2. Conda环境：`pytorch_env`（Python 3.9）
3. 核心依赖：
   - PyTorch 2.12.0（CPU版本）
   - Habitat-Sim v0.2.2（源码编译，无头模式）
   - habitat-lab、habitat-baselines

## 环境安装步骤
1. 通过apt安装系统编译依赖包
2. 安装Anaconda并配置清华镜像源
3. 创建Conda虚拟环境`pytorch_env`
4. 安装CPU版PyTorch（使用清华PyPI源加速）
5. 克隆Habitat-Sim v0.2.2源码，无头模式编译安装
6. 下载Habitat官方测试场景数据集
7. 运行官方烟雾测试验证环境

## 烟雾测试运行方法
# 激活Conda环境
conda activate pytorch_env

# 下载测试数据集
python -m habitat_sim.utils.datasets_download --uids habitat_test_scenes habitat_test_assets --data-path ./data

# 执行烟雾测试
python -m habitat.utils.test_examples

