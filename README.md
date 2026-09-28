# sonic-results

在 MuJoCo 里部署 NVIDIA 的 SONIC，让它在 Unitree G1 上跟几段它没见过的动作。这里先放一部分成果，代码还在整理，之后会传上来。

## 做了些什么

SONIC 是 NVIDIA GEAR 实验室放出来的人形动作跟踪模型，用上亿帧动捕数据训出来，一个模型就能跟各种动作。我在另一个仓库 mimic-results 里做的是 BeyondMimic，一段动作训一个模型。两条路子想放在一起比一比，所以这边用的是完全相同的三段动作：同样的 LAFAN1 帧范围，同一份 G1 数据。

没有重新训练，直接用官方的预训练模型。部署用的是官方的 C++ 程序，TensorRT 推理，跑在 Docker 里；MuJoCo 仿真跑在宿主机上，两边通过消息通信。我的显卡是 RTX 5070 Ti，官方容器默认的 CUDA 12.4 和它对不上，改成了 12.8。

BeyondMimic 的动作文件和 SONIC 要的格式差不多，字段名都一样，只是 SONIC 只取 30 个身体里的 14 个。写了个小脚本做转换，转之前先用正运动学把每一帧重算一遍，和原数据对上了才写文件。三段动作都是 0.00 mm。为了确认这个检查本身不是摆设，又故意把关节顺序弄错算了一遍，误差在 540 到 900 毫米之间，说明它确实能发现问题。

做这一步时发现，官方部署用的 MuJoCo 模型和训练用的 URDF，腰部差了大约 1 cm。躯干的位置因此偏了 10 mm。转换时改用 URDF 的尺寸后就对上了。这也是官方 sim2sim 里本来就有的一处小差异。

结果是：走路很稳；搏击那段跑了十来次，一次没摔；冲刺那段有两个圆弧，大概每三次有一次会在第二个圆弧上摔倒。摔之前能看出步子先乱，然后才失去平衡。

## 效果

MuJoCo 里的录屏，每段截 6 秒。

| 走路 | 搏击 |
|---|---|
| ![走路](media/g1_walk1.gif) | ![搏击](media/g1_fight1.gif) |

| 冲刺，跑完第二个圆弧 | 冲刺，在第二个圆弧摔倒 |
|---|---|
| ![冲刺成功](media/g1_sprint1_ok.gif) | ![冲刺摔倒](media/g1_sprint1_fall.gif) |

三段动作在 LAFAN1 里的位置：`walk1_subject1` 122–722、`fightAndSports1_subject1` 1558–2158、`sprint1_subject2` 1727–2327（30 fps 的原始帧号）。每段 20 秒，转成 50 Hz 后是 1000 帧。

## 几个观察

同一段动作，有时摔有时不摔。翻了代码才明白：仿真和控制程序各自按自己的时钟跑，谁也不等谁，所以控制程序每次读到的状态都晚几毫秒，而且每次晚得不一样。这意味着跑一次看结果是不够的，得多跑几次看比例。

按 N 键切换动作，顺序不是固定的。部署程序扫描动作文件夹时，顺序由文件系统决定。我第一次录数据就录错了，以为是走路，其实是搏击。所以后来分析数据时，都是拿录下来的内容反推是哪段动作，不靠记按了几下。

为了把"看着像不像"变成数字，写了个小程序在旁边听部署程序的调试广播，把每一帧的实际关节角和目标关节角都记下来。官方自带的订阅器只保留最新一条，会丢帧，我的版本去掉了这个设置。40 秒收到 2001 条，每秒 50 条，一条没丢。

拿搏击那段先算了一次。20 秒里每个关节平均差 8 度左右，差得最多的是两边肩膀的旋转和手腕。机器人基本是同步跟着参考动作走的，只落后大约一帧，也就是 20 毫秒。这只是一次的结果，用来验证工具能用，不能当结论。

## 还没做完

- 机器人身体在世界里的位置还没接进来，所以"有没有坚持到最后"和"走偏了多少"这两项还算不了。
- BeyondMimic 那边在 G1 上的 MuJoCo 验证还没做，两边的数字暂时没法放在一起比。

## 环境

| | |
|---|---|
| 显卡 | RTX 5070 Ti |
| 部署容器 | 官方 Dockerfile，CUDA 改为 12.8.1 |
| TensorRT | 10.13.0，官方要求必须是这个版本 |
| SONIC 代码 | commit `b042411` |
| 模型 | Hugging Face 上 `nvidia/GEAR-SONIC` 的默认版本 |

## 用到的项目和工具

[SONIC / GR00T-WholeBodyControl](https://github.com/NVlabs/GR00T-WholeBodyControl)、[BeyondMimic](https://github.com/HybridRobotics/whole_body_tracking)、[MuJoCo](https://github.com/google-deepmind/mujoco)、[TensorRT](https://developer.nvidia.com/tensorrt)

## 关于数据

动作数据来自 Ubisoft 的 [LAFAN1](https://github.com/ubisoft/ubisoft-laforge-animation-dataset)，许可证是 CC BY-NC-ND 4.0，不允许分发改编后的版本，所以转换后的动作文件不放在这里。SONIC 的模型权重用的是 NVIDIA Open Model License，请从官方仓库下载。
