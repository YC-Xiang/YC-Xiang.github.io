IPCam 3917 isp 模块图:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250304105727.png)

**信号处理路径模块：**

Crop：裁剪模块，用于裁剪图像的特定区域

BLC：黑电平补偿(Black level Correction)

DPC: 坏点校正(Defect Pixel Correction)

2DNR: 2D降噪(2D Noise Reduction)

GE: 边缘增强(Gain Enhancement)

3DNR & GE：3D降噪和增益增强

AE Short/Long Gain：自动曝光短/长时间增益控制

HDR Fusion：高动态范围图像融合，合并不同曝光的图像

**镜头校正模块：**

Circle Lens Shading：圆形镜头阴影校正

Grid Lens Shading：网格镜头阴影校正

**色彩处理模块：**

YUV to RGB：YUV色彩空间转换为RGB

INTP：demosaic颜色插值, 从raw格式转换到rgb格式

CCM：色彩校正矩阵(Color Correction Matrix)

RGB Gamma：RGB伽马校正

WDR：宽动态范围(Wide Dynamic Range)处理

**YUV处理模块：**

YUV420 To YUV422：YUV色彩格式转换

YUV422 To YUV444：YUV色彩格式转换

RGB2YUV444：RGB转YUV444格式

Global Curve：全局曲线调整

UV Tune & UVS：UV通道调整和UV抑制

**统计和分析模块：**

AF Statistics：自动对焦统计数据

AE Short/Long Statistics：自动曝光短/长时间统计数据

Raw Statistics：原始图像统计数据

AWB Statistics：自动白平衡统计数据

AE Dyn. Statistics：自动曝光动态统计数据

Y Statistics：亮度统计数据

**后处理模块：**

Adaptive Tone Mapping：自适应色调映射

Flicker Detection：闪烁检测

Mask：遮罩处理

Y Gamma：亮度伽马校正

HUE：色调调整

SPE：空间增强(Spatial Enhancement)

FEH：特征增强(Feature Enhancement)

H-LDC：水平畸变校正(Horizontal Lens Distortion Correction)

**压缩和输出模块：**

JPEG：JPEG压缩

DDR：动态随机存取存储器，用于图像缓存

OSD：屏幕显示(On-Screen Display)

Zoom LPF：变焦低通滤波器

MD：运动检测(Motion Detection)
