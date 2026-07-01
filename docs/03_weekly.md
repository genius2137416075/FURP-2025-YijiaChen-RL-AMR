# 今日完整工作复盘（Isaac Gym / Legged Gym 环境搭建全流程）
## 一、今日完成操作梳理
### 1. Conda 虚拟环境搭建
1. 下载 Miniconda3 安装脚本，完成 Linux 版 Miniconda 部署
2. 创建 `legged_gym` 专用虚拟环境并激活
3. 安装 CUDA11.3 匹配版本 PyTorch：`torch==1.10.0+cu113 torchvision==0.11.1+cu113`
4. 验证环境：执行校验脚本，确认 `torch.cuda.is_available() = True`，PyTorch CUDA 绑定11.3正常
5. 查找环境依赖库：通过 `find $CONDA_PREFIX -name "libpython3.8.so*"` 确认 Python3.8 动态库存在

### 2. 系统环境变量配置（`.bashrc`）
依次写入4类库路径，永久配置动态链接库搜索：
1. CUDA11.3 工具链库：`/usr/local/cuda-11.3/lib64`
2. Conda 虚拟环境Python库：`$CONDA_PREFIX/lib`
3. NVIDIA系统驱动库：`/usr/lib/x86_64-linux-gnu`
4. Isaac Gym 底层绑定库：`isaacgym/_bindings/linux-x86_64`
同时配置 `GYM_USD_PLUG_INFO_PATH` USD渲染插件路径。

### 3. Isaac Gym 示例调试（1080_balls_of_solitude.py）
1. 切换到 `isaacgym/python/examples` 目录运行示例脚本
2. 尝试修改源码开启GPU渲染管线 `sim_params.use_gpu_pipeline = True`
3. 测试无头运行模式 `--headless` 规避窗口卡死
4. 排查 GPU 渲染、库缺失、指针崩溃等报错

### 4. NVIDIA 驱动兼容性修复方案调研
确定CUDA11.3匹配稳定驱动版本为510，整理完整重装驱动、Vulkan图形依赖安装流程。

### 5. 底层依赖文件排查
1. 解决 `libpython3.8.so.1.0: No such file or directory` 动态库找不到报错
2. 定位 Isaac Gym 加载 `gym_38.so` 底层绑定文件正常，物理引擎PhysX可识别 `cuda:0` GPU

---
## 二、今日遇到的全部问题 + 对应解决方案
### 问题1：运行脚本提示 `ls: 无法访问 /usr/local/cuda-11.3/lib64/libcuda.so* 没有那个文件或目录`
- 根源：`libcuda.so` 不属于CUDA Toolkit，由NVIDIA显卡驱动提供，系统未识别驱动库路径
- 解决方案：在 `LD_LIBRARY_PATH` 追加系统驱动路径 `/usr/lib/x86_64-linux-gnu`

### 问题2：ImportError: libpython3.8.so.1.0: cannot open shared object file
- 根源：终端单次`export`环境变量临时生效，新开/重新运行脚本时路径丢失
- 解决方案：
  1. `.bashrc` 永久添加 `$CONDA_PREFIX/lib` 库路径
  2. 运行脚本前一次性完整加载全部LD库路径

### 问题3：运行示例弹出「Isaac Gym无响应」窗口，GUI卡死
- 日志警告：`WARNING: Forcing CPU pipeline. GPU Pipeline: disabled`
- 根源：GPU图形渲染管线启用存在兼容性冲突，图形绘制交给CPU造成窗口卡顿；但**物理计算全程GPU cuda:0不受影响**
- 临时解决方案：使用 `python xxx.py --headless` 无头模式，不启动可视化窗口，纯GPU后台仿真，无卡顿
- 根治方案：重装510版本NVIDIA驱动 + 完整Vulkan图形组件，修复GPU渲染管线兼容问题

### 问题4：开启 `use_gpu_pipeline = True` 直接程序崩溃
报错：`Gym cuda error: invalid resource handle / PxScene::copyBodyData, data has to be valid pointer`
- 根源：当前NVIDIA驱动版本过高/缺失Vulkan ICD，Vulkan渲染、CUDA、PhysX显存指针内存冲突
- 解决方案：
  1. 折中方案：保持 `use_gpu_pipeline = False`，图形走CPU、物理GPU，不会崩溃，仅窗口轻微卡顿
  2. 根治方案：完整卸载现有驱动，重装稳定版510驱动，配套安装`nvidia-vulkan-icd` Vulkan图形依赖，加载Vulkan ICD环境变量

### 问题5：修改代码添加 `sim_params.graphics_device_id = -1` 报属性不存在
- 根源：当前Isaac Gym版本老旧，`SimParams` 无该API
- 解决方案：删除该行代码，改用官方 `--headless` 启动参数实现无窗口运行

### 问题6：无法切换GPU图形渲染，持续强制CPU pipeline
- 根源：缺少NVIDIA Vulkan驱动组件、驱动版本与CUDA11.3不匹配、多显卡/核显干扰
- 配套修复手段：
  1. 锁定单GPU：`export NVIDIA_VISIBLE_DEVICES=0`
  2. 指定Vulkan ICD路径：`export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json`
  3. 调低PhysX迭代参数，降低显存冲突概率

---
## 三、后续长期方案（Legged Gym + URDF 机器人仿真全流程规划）
### 1. 底层环境根治（今日待执行收尾）
1. 完整重装NVIDIA 510驱动：清理旧驱动→黑名单nouveau→安装510驱动+Vulkan全套依赖→重启验证`nvidia-smi`
2. 永久写入Vulkan ICD、显卡锁定环境变量到 `.bashrc`，彻底解决GPU pipeline强制CPU警告
3. 验证1080 balls示例可正常开启GPU图形渲染，窗口不崩溃、不卡顿

### 2. Legged Gym 环境部署
1. 拉取 legged_gym 源码仓库，安装项目全部Python依赖包
2. 匹配现有 `cuda11.3 + torch1.10` 环境，修复项目内部CUDA、Torch版本适配代码
3. 调试训练启动脚本，支持无头后台训练，大幅提升训练效率，规避GUI卡顿问题

### 3. URDF机器人模型仿真调试（核心业务目标）
1. URDF模型导入流程：
   - 规范机器人URDF文件、STL网格材质文件目录结构
   - 编写Isaac Gym加载代码，设置碰撞形状、物理质量、关节限位、驱动参数
2. 仿真参数调优：
   - 匹配PhysX GPU物理参数，调整迭代次数避免显存指针崩溃
   - 配置相机可视化、轨迹记录、仿真数据导出功能
3. 强化学习训练对接：
   - 基于legged_gym封装四足/人形机器人环境
   - 无头模式批量并行仿真训练，GPU全算力分配给动力学计算，不占用图形资源

### 4. 长期优化方案
1. 环境脚本封装：编写一键启动脚本，自动加载全部LD库、Vulkan、显卡锁定变量，无需重复输入命令
2. 多版本隔离：备份当前conda环境配置，避免后续更新驱动、CUDA破坏现有可用仿真环境
3. 问题标准化排查清单：整理库缺失、GPU渲染崩溃、URDF加载失败的快速排查步骤，后续调试提速
