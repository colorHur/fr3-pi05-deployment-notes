# 先用一个场景理解项目

## 场景：让夹爪向前移动一点，再合拢

这是帮助理解流程的例子，不是已完成的任务实验。

第一次由人示范：你操作 Quest，遥操作程序把人的操作转换成机器人目标，FR3 带动夹爪运动。相机拍下过程，机器人报告自己的位置、姿态和夹爪宽度，录制器将这些消息采样、保存为轨迹。

以后由模型尝试：系统取最新的相机画面和机器人状态，加上任务文字，发给 π0.5 策略服务。模型返回未来一小段动作，部署程序将这些动作逐步转换、提交给机器人服务。机器人运动后，系统继续获取新观测。

这两个阶段分别回答“示范数据怎么来”和“模型输出怎么变成机器人运动”。

## 各部分负责什么

| 部分 | 具体职责 | 可以类比为 |
| --- | --- | --- |
| FR3 与夹爪 | 执行运动，并提供当前机器人状态 | 真正干活的手臂 |
| Quest 遥操作 | 把人的操作传入系统，支持人工示教和接管 | 人的操作入口 |
| 外部和腕部相机 | 提供场景与夹爪附近的画面 | 两个观察视角 |
| ROS2 | 在节点间传递图像、位姿和夹爪状态等消息 | 组件间的消息通道 |
| 录制器 | 按采样节奏读取消息并落盘 | 过程记录员 |
| 数据转换工具 | 把录制轨迹整理为模型训练需要的格式 | 数据整理员 |
| π0.5 策略服务 | 根据图片、状态和文字预测未来动作 | 动作预测器 |
| 部署程序 | 整理观测、请求推理、安排动作执行、处理异常和日志 | 现场调度员 |
| 控制权仲裁 | 处理模型与遥操作之间的控制归属 | 判断当前谁能发指令的门卫 |

类比只用于建立直觉，具体语义以实现与实验为准。

## 先区分两条技术路线

- **原版 RDP**：仓库原有研究路线，涉及视觉、触觉、Asymmetric Tokenizer 和潜在扩散策略训练。
- **本次学习的 FR3 / π0.5 链路**：示教采集与数据转换、OpenPI 服务调用、动作调度与机器人执行。当前在线观测代码整理双 RGB、10 维机器人状态与任务文字。

两者共享部分采集和机器人基础设施，但训练方法与模型输入不能混为一谈。可选 GelSight 录制不等于当前 π0.5 在线推理已经使用触觉。

## 有 Python 基础，可以这样理解一次自主执行

下面是帮助阅读的伪代码，省略了并发、通信、心跳与异常处理；不能用于驱动真机：

```python
observation = read_latest_images_and_robot_state()
action_chunk = policy_service.predict(observation, task_text)
for action in action_chunk:
    target = convert_action_to_robot_target(action)
    submit_if_control_is_authorized(target)
```

真正需要学习的系统问题包括：观测是不是旧的，动作格式是否一致，推理返回前动作能否继续，人工接管时模型请求如何处理。本课先记住它们位于什么环节，暂不展开细节。

## 源码地图（本地工作区路径）

以下路径相对于原项目根目录。部分扩展尚未发布到原项目远端，暂不提供未经核实的在线源码行链接。

| 想了解的问题 | 入口 |
| --- | --- |
| 怎样启动示教采集 | `tools/start_teleop_collection.sh` |
| 怎样记录数据 | `reactive_diffusion_policy/real_world/teleoperation/data_recorder.py` |
| 怎样转换为 LeRobot 数据 | `tools/convert_chunked_pkl_to_lerobot_v3.py` |
| 怎样启动模型部署 | `tools/run_pi05_fr3.py` |
| 模型观测如何整理 | `reactive_diffusion_policy/real_world/model_deployment/observation_node.py`、`observation.py` |
| 怎样调用模型服务 | `reactive_diffusion_policy/real_world/model_deployment/policy_client.py` |
| 动作如何安排和转换 | `reactive_diffusion_policy/real_world/model_deployment/rollout.py`、`action_adapter.py` |
| 如何与机器人服务通信 | `reactive_diffusion_policy/real_world/model_deployment/robot_client.py` |
| 模型与人如何交接控制权 | `reactive_diffusion_policy/real_world/robot/control_lease.py` |

验证状态：对照学习路线、采集/部署入口及观测、动作适配、客户端源码整理；未完整追踪所有服务调用链，未运行硬件或模型服务。

## 下一小节

只看示教阶段：从“你动一下 Quest”开始，画出人的操作如何到达 FR3，以及相机和机器人状态如何被记录。随后再读对应的少量源码。
