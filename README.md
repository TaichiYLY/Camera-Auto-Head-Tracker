# 相机自动锁头（Camera Auto Head Tracker）

电脑摄像头实时捕捉画面 → 识别人脸眉心 → 计算头部中心坐标与角度偏差 → 画面绘制
“锁定准星”跟随头部 → 串口驱动二自由度舵机云台跟随转动。

## 功能效果

- 全屏摄像头视频流（也可窗口模式）
- MediaPipe 478 点面部关键点，头部中心取**两眉之间（眉心）**
- 头部中心：**红色实心圆点 + 红色空心圆环 + 四角旋转锁定标**
- 贯穿全屏的黑色十字线（交点跟随头部移动）
- 画面中心固定的微弱十字辅助线，用于判断偏差
- 准星旁蓝紫色坐标文字 `[x, y]`
- 左上角角度 HUD：`Y: 水平偏差 deg`、`P: 垂直偏差 deg`
- 左下角 PiP 云台调试窗：连接状态、Yaw/Pitch 目标与实际角度、仪表盘、帧率
- 检测线程与显示线程分离：720p 下实测检测、显示均约 30 FPS（i5-10200H）
- 指数滑动平均（EMA）跟随，丢帧短时保持，避免抖动/乱跳
- 没接硬件时自动进入**模拟模式**，PiP 里模拟舵机跟随，可先只调视觉

## 目录结构

```
锁头/
├── main.py               # 主程序入口
├── config.py             # 全部可调参数（摄像头、串口、平滑、颜色）
├── detector.py           # 人脸检测（MediaPipe 主，Haar 兜底）
├── tracker.py            # 目标平滑、丢帧保持、锁定状态机
├── geometry.py           # 像素坐标 → 角度偏差（针孔模型）
├── gimbal.py             # 串口后台发送 + 无硬件模拟
├── overlay.py            # 所有 UI 绘制
├── models/
│   └── face_landmarker.task   # MediaPipe 人脸关键点模型（已下载）
├── firmware/
│   └── gimbal_firmware.ino    # Arduino 二自由度舵机固件
├── assets/               # 参考效果图
├── outputs/              # 运行截图（S 键保存）
├── requirements.txt
└── .venv/                # 已配置好的虚拟环境
```

## 一、快速开始（不接硬件，先看效果）

```powershell
cd C:\Users\taichi\Desktop\锁头
.\.venv\Scripts\python.exe main.py --sim
```

没有任何硬件也能跑：PiP 中显示 SIMULATOR，红点模拟云台慢慢跟随头部。

- 用静态图片测试（不打开摄像头）：
  ```powershell
  .\.venv\Scripts\python.exe main.py --source assets\Snipaste_2026-10-16_11-59-05.jpg
  ```
- 窗口模式：加 `--windowed`
- 换第二个摄像头：加 `--camera 1`（本机 1 = OsmoAction4）
- 列出所有可用摄像头（编号 + 后端 + 实际分辨率）：
  ```powershell
  .\.venv\Scripts\python.exe main.py --list-cameras
  ```

> 多摄像头说明：程序自动按 **MSMF → DSHOW** 顺序尝试，且**真正读到画面**才算打开成功；
> 运动相机（如 Osmo Action）启动预热慢，会等待最多约 5 秒。
> 若摄像头不支持 1280x720，会自动使用其原生分辨率（如 1920x1080）并同步修正角度计算。
> 运动相机多为大广角，务必用 `--hfov` 告诉程序真实视场角（如 `--hfov 120`），否则算出的角度偏小；
> 也可直接改 `config.py` 的 `camera_index`、`camera_hfov` 作为默认值。

## 二、硬件接线

| 部件 | 接线 |
|---|---|
| 水平舵机 PAN | 信号线 → Arduino D9；红 → 5V；棕 → GND |
| 俯仰舵机 TILT | 信号线 → Arduino D10；红 → 5V；棕 → GND |
| 供电 | 舵机**强烈建议独立 5V 供电**（如 4×AA 电池盒 / 5V 电源），只把 GND 与 Arduino 共地；USB 口电流通常不够 |

> 注意：舵机为模拟/数字标准舵机（SG90、MG90S 等），水平舵机装在云台下层负责左右转，俯仰舵机装在上层负责上下。

## 三、烧录 Arduino 固件

1. Arduino IDE 打开 `firmware/gimbal_firmware.ino`；
2. 开发板选 Arduino Uno / Nano，上传；
3. 保持波特率 **115200**（与 `config.py` 一致）；
4. 上电后舵机会自动回中（90°，90°）。

## 四、连接硬件运行

```powershell
# 自动扫描串口（识别 Arduino / CH340 / CP210x）
.\.venv\Scripts\python.exe main.py

# 或手动指定串口号
.\.venv\Scripts\python.exe main.py --port COM5
```

PiP 中状态点变绿、MODE 显示 HARDWARE 即连上。

### 串口协议（115200，纯文本，`\n` 结尾）

| 上位机 → 单片机 | 含义 |
|---|---|
| `Y+12.30 P-4.50\n` | Yaw/Pitch 角度偏差（度），云台据此消偏 |
| `C\n` | 回中 |
| `D\n` / `E\n` | 舵机卸载 / 使能 |

单片机回传：`PAN 102.3 TILT 87.6\n`（两个舵机当前绝对角度，0~180，90 为中位）。

## 五、按键

| 按键 | 功能 |
|---|---|
| Q / ESC | 退出 |
| F | 全屏 / 窗口切换 |
| S | 截图，保存到 `outputs/` |

## 六、调校指南（都在 `config.py`）

1. **角度不准（最常见）**
   - 调整 `camera_hfov`：摄像头水平视场角。普通 webcam 约 60°，越广角度越大。
   - 粗测方法：让头移到画面最左边缘，看 `Y` 是否接近 HFOV/2（如 60° → 30°），不符就微调 `camera_hfov`。
2. **准星抖** → 调小 `ema_alpha`（如 0.3）；**跟得慢** → 调大（如 0.7）。
3. **锁点位置**：默认眉心 = 关键点 105、334 的中点；
   - 想锁双眼之间鼻梁：`glabella_indices = (168,)`
   - 想锁额头中部：`glabella_indices = (9,)`
4. **云台方向反了**：
   - 水平反：固件里 `panPos += step;` 改为 `panPos -= step;`
   - 俯仰反：固件里 `tiltPos -= step;` 改为 `tiltPos += step;`
5. **云台反应太猛/太肉**：`gain_yaw`、`gain_pitch`；`max_send_angle` 是限幅保护。
6. **丢目标后**：`center_on_lost=False` 保持原位；`True` 自动回中。

## 七、常见问题

| 问题 | 解决 |
|---|---|
| 打不开 / 只有一个摄像头能用 | 先 `--list-cameras` 看编号；再用 `--camera 0` 或 `--camera 1`；关闭占用摄像头的其他软件；运动相机先确认已进入 USB 摄像头模式 |
| 启动后退回 Haar 或报模型错误 | 确认 `models/face_landmarker.task` 存在（约 3.7MB）；程序已用字节方式加载，中文路径不影响 |
| 连不上串口 | 设备管理器查 COM 号，用 `--port COMx`；检查驱动（CH340/CP210x）；没硬件时自动模拟 |
| 画面只有 10 FPS | 程序默认 MSMF 后端；个别摄像头 720p 物理上只支持 10fps，可把 `config.py` 里 `width/height` 改为 640/480（通常 30fps） |
| 中文乱码 | 程序已统一 UTF-8；如仍乱码，用 Windows Terminal 运行 |

## 八、角度计算原理

针孔相机模型，焦距由水平视场角估算：

```
fx = (画面宽 / 2) / tan(HFOV / 2)
Yaw   = atan2(头x - 中心x, fx)
Pitch = atan2(中心y - 头y, fy)   # y 轴向下，取反
```

## 九、依赖

见 `requirements.txt`，已安装在 `.venv` 中：
`opencv-contrib-python 5.0.0`、`mediapipe 1.1.1`、`numpy 2.0.2`、`pyserial 3.5`。
