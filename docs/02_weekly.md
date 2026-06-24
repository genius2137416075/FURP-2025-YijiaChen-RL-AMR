### 1. 找不到功能包 `rl_amr_nav`
- 问题：运行`ros2 run rl_amr_nav rl_train_node`时提示`Package 'rl_amr_nav' not found`
- 原因：工作空间的`install`环境没有被加载，ROS 2无法识别自建功能包
- 解决：
  1. 进入工作空间：`cd ~/rl_ws`
  2. 编译包：`colcon build`
  3. 加载环境：`source install/setup.bash`
  4. 验证包：`ros2 pkg list | grep rl_amr_nav`

---

### 2. 找不到Python模块 `stable_baselines3`
- 问题：运行节点时提示`ModuleNotFoundError: No module named 'stable_baselines3'`
- 原因：
  - 终端没有激活`vln_rl_nav` conda虚拟环境，使用了系统Python
  - 或ROS 2节点没有使用conda环境的Python解释器
- 解决：
  1. 激活虚拟环境：`conda activate vln_rl_nav`
  2. 安装依赖：`pip install stable-baselines3`
  3. 临时解决ROS路径问题：`export PYTHONPATH=/home/debbie/miniconda3/envs/vln_rl_nav/lib/python3.10/site-packages:$PYTHONPATH`
  4. 永久解决：在节点入口文件顶部添加conda环境的shebang：`#!/home/debbie/miniconda3/envs/vln_rl_nav/bin/python`

---

### 3. 关键背景
- 项目

基于ROS 2的强化学习导航项目，依赖`stable_baselines3`等第三方库
- 这些库安装在`vln_rl_nav` conda虚拟环境中，而ROS 2默认使用系统Python，两者环境不互通是核心矛盾

---

**总结**：后续问题都围绕**ROS 2与Conda环境的兼容性**展开，核心解决思路是让ROS 2节点正确调用`vln_rl_nav`环境的Python解释器，并确保依赖库已安装✅
