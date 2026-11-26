# FR3 / π0.5 部署复现教程

每节课从一个可观察的问题开始：找到入口，追踪一个数据变化，先写预测，再离线验证。源码阅读、手算和程序运行分别记录，便于回查结论是怎么来的。

| 顺序 | 练习 | 状态 |
| --- | --- | --- |
| 1 | Quest 手柄移动后，怎样算出 FR3 的末端目标？ | 本节可开始 |
| 2 | 图像和机器人状态怎样进入一条示教轨迹？ | 后续补充 |
| 3 | π0.5 的观测和动作格式如何衔接？ | 后续补充 |
| 4 | 动作怎样按时发送，人怎样接管？ | 后续补充 |

源码命令在原项目的 FR3 扩展工作区执行。展示仓库提供教程和复现记录，所需源码路径见[架构说明](architecture.md)。

## 第一节 · 从手柄平移算出机器人末端目标

**目标**：能从代码指出两个起点在哪里保存，并用一个数字例子解释目标位置如何得出。预计 15 分钟。所有命令只读取文件或做 Python 数值计算。

### 1. 用终端进入源码根目录

```bash
# 改成你电脑上原项目的源码目录
cd /path/to/reactive_diffusion_policy
pwd
test -f teleop.py && test -f tools/start_teleop_collection.sh && echo "源码目录正确"
```

当前工作区路径为 `/run/media/hty/NEW/reactive_diffusion_policy`。检查通过后可运行 `code .` 在 VS Code 打开整个目录。也可以继续用 shell 完成本节。

### 2. 找到数据从哪里来、最后交给谁

```bash
src=reactive_diffusion_policy/real_world/teleoperation/teleop_server.py
test -f "$src" && echo "遥操作源码已找到"
rg -n "post\('/unity'\)|unity2robot_frame|set_start_tcp\(left_tcp|calc_relative_target\(l_pos_from_unity|send_left_arm_waypoint\(left_target" "$src"
```

按下面顺序看匹配的几行。可以先忽略行号之间的其他逻辑。

| 搜索结果 | 这一步的数据 |
| --- | --- |
| `post('/unity')` | 接收 Quest/Unity 位姿消息，把消息放进缓冲区 |
| `unity2robot_frame` | 把手柄位姿换到机器人基座坐标系 |
| `set_start_tcp(left_tcp, ...)` | 开始跟踪时，记住机器人末端的实测位姿和已转换的手柄位姿 |
| `calc_relative_target(l_pos_from_unity, ...)` | 用当前手柄位姿和两个参考起点计算末端目标 |
| `send_left_arm_waypoint(left_target)` | 把算好的目标交给发送函数，后者向机器人服务提交请求 |

### 3. 只读位置计算

```bash
rg -n -A 31 'def calc_relative_target' "$src"
```

VS Code 中也能按 `Ctrl+Shift+F` 搜索 `def calc_relative_target`。先找 `relative_translation` 和 `target[:3]`，其中 `[:3]` 是位置的 x、y、z。TCP 可以先理解为机器人用于描述夹爪末端的参考点。

在允许平移且未触发限幅时，先做减法，再乘缩放系数，最后做加法：

```python
hand_delta = hand_now - hand_start
target_position = robot_start + scale * hand_delta
```

想一想，如果刚开始跟踪时手柄还停在自己的起点，`hand_delta` 应该是多少？此时算出来的机器人目标应落在哪里？实际代码还处理仅旋转模式、平移限幅和姿态，本节先看上述条件中的平移。

### 4. 先预测，再用终端核对

假设机器人起点 x 是 0.30 m，手柄起点 x 是 0.50 m，当前手柄 x 是 0.54 m，缩放系数是 0.5，忽略限幅。这里的手柄 x 已经在机器人基座坐标系中。

先写三项预测：手柄位移、机器人目标位移、机器人目标 x。写完后可以将下面的 `TODO` 替换为你的表达式，再将整个代码块粘贴到终端。

```bash
python3 - <<'PY'
robot_start_x = 0.30
hand_start_x = 0.50
hand_now_x = 0.54
scale = 0.5

hand_dx = TODO
robot_dx = TODO
target_x = TODO

print(f"手柄位移：{hand_dx:.3f} m")
print(f"机器人目标位移：{robot_dx:.3f} m")
print(f"机器人目标 x：{target_x:.3f} m")
PY
```

这是简化公式的离线算例。完整遥操作链路还涉及消息接收、跟踪状态、坐标转换和机器人服务。**课程记录：源码已核读，学习者的预测与输出待补。**

### 5. 卡住时从最近的一步查起

| 看到什么 | 先检查什么 |
| --- | --- |
| `cd` 报目录不存在 | 检查路径是否已换为本机目录，路径含空格时用双引号 |
| `test -f teleop.py` 没有输出 | 单独运行 `pwd` 和 `ls`，确认打开的是源码根目录。单独的 `test` 成功时也没有输出，组合命令成功才会打印提示 |
| `code: command not found` | 先用 shell 阅读，或从 VS Code 的“打开文件夹”打开同一目录 |
| `rg: command not found` | 可用 `grep -n -A 31 'def calc_relative_target' "$src"` 读同一个函数 |
| 报文件不存在或搜不到函数 | 先在当前终端重新设置 `src`，再检查 `test -f "$src"`。文件存在但函数缺失时，用 `git status --short -- "$src"` 和 `git log -1 --oneline` 记录当前工作区版本 |
| `NameError: name 'TODO' is not defined` | 将三个 `TODO` 全部改成自己的计算表达式 |
| 数值与预测不一致 | 检查减法顺序、缩放乘在哪一项、最后加的是哪个起点 |

做完后，用一句话回答面试追问：**为什么进入跟踪时要分别记录手柄起点和机器人起点？** 把预测、实际输出和疑问放进[实验记录](experiments/README.md)的模板。下一节再看机器人状态和相机画面怎样进入录制器。
