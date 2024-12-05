人眼对亮度的感知是非线性的，对暗处的变化比亮处的变化更敏感。如果图像没有进行 gamma correction，那么会分配了太多的比特来突出人类无法区分的亮点，并且分配了太少的比特显示阴影。因此需要 gamma correction 来校准，尽量做到线性化。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241104154318.png)

可以想象 video out 和 video in 之间的公式：

$V_{out} = V_{in}^\gamma$

if gamma < 1, encoding gamma，称作 gamma compression。
if gamma > 1, decoding gamma，称作 gamma expansion。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241104154625.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20241104160157.png)

蓝色为输入图像亮度变化曲线，红色为 gamma 值的曲线，紫色为最终显示的亮度变化曲线。

display gamma 即 lcd 运用的 gamma 变换，可以看出 gamma 的值在比如 0~255 色深中不是一个定值，通常需要 LUT 查找表来对某一范围内的亮度进行 gamma 变换。在 0~255 低位区域使用更小的 gamma，而高位区域使用更大的 gamma 值，gamma > 1。
