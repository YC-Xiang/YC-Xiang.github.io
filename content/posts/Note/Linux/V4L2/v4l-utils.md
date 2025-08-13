# v4l-ctl

# media-ctl

`media-ctl.c`

`-p`: 打印 device topology

```shell
$ media-ctl -p
```

</br>

`--print-dot`: 输出 graphviz DOT 格式图形描述。

```shell
# 将输出保存到文件并转换为图像
$ media-ctl --print-dot > topology.dot
$ dot -Tpng topology.dot -o topology.png
```

</br>

`-e`: 指定某个 entity name, 打印属于哪个 device

```shell
$ media-ctl -e rtsisp_video0
/dev/video0
```

</br>

`--get-v4l2` or `--get-format`: 查看某个 pad 的格式

```shell
# 使用entity ID
$ media-ctl --get-v4l2 0:1         # entity 0的pad 1
$ media-ctl --get-v4l2 5:0       # entity 5的pad 0

# 使用entity 名称
$ media-ctl --get-v4l2 '"sensor":1'   # 名为"sensor"的entity的pad 1
$ media-ctl --get-v4l2 '"isp":0'    # 名为"isp"的entity的pad 0
```

</br>

`-r`: reset all links to inactive

```shell
media-ctl -r
```

`-l`: 设置 links

```shell
media-ctl -l "0:1 -> 5:0 [1]" # 基本link设置 - entity 0的pad1连接到entity 5的pad0，设置为active
media-ctl -l '"sensor":0 -> "isp":0 [1]' # 使用entity名称
media-ctl -l "0:1 -> 5:0 [1], 5:1 -> 6:0 [1]" # 多个link设置，用逗号隔开
media-ctl -l "0:1 -> 5:0 [0]" # 禁用链接
```

</br>

`-V`：设置 pad 格式

```shell
media-ctl -V '"sensor":0 [fmt:YUYV8_2X8/1920x1080]' # 设置基本格式
media-ctl -V '"sensor":0 [fmt:YUYV8_2X8/1920x1080], "isp":0 [fmt:YUYV/1920x1080]' # 设置多个 pad 的格式
media-ctl -V '"sensor":0 [fmt:YUYV8_2X8/1920x1080, colorspace:srgb]' # 设置格式和颜色空间
media-ctl -V '"sensor":0 [fmt:YUYV8_2X8/1920x1080, field:none, @30/1]' # 设置格式、field 和帧间隔
media-ctl -V '"isp":1 [fmt:YUYV8_2X8/1920x1080, crop:(100,100)/1720x880, compose:(0,0)/1920x1080]' # 设置格式、裁剪和合成区域
```

`--known-mbus-fmts`: 枚举出所有的 mbus formats

# v4l2-ctl

```shell
$ v4l2-ctl --help

$ v4l2-ctl --all # 列出很多信息
$ v4l2-ctl --device /dev/video0 # 指定 device
$ v4l2-ctl --out-device
$ v4l2-ctl --export-device
$ v4l2-ctl --media-bus-info
$ v4l2-ctl --get-fmt-video # VIDIOC_G_FMT
$ v4l2-ctl --set-fmt-video width=2560,height=1440,pixelformat=YUYV # VIDIOC_S_FMT 可以指定width,height,pixelformat,field,colorspace,ycbcr等等
$ v4l2-ctl --try-fmt-video width=2560,height=1440,pixelformat=YUYV # VIDIOC_TRY_FMT

$ v4l2-ctl --list-formats # 列出支持的格式 VIDIOC_ENUM_FMT
$ v4l2-ctl --list-formats-ext #  除了格式还有frame size&interval信息, 但interval这里只会打印discrete的 VIDIOC_ENUM_FMT+VIDIOC_ENUM_FRAMESIZES+VIDIOC_ENUM_FRAMEINTERVALS
$ v4l2-ctl --list-fields # 支持的fields
$ v4l2-ctl --list-framesizes YUYV # 列出指定格式的frame size 信息 VIDIOC_ENUM_FRAMESIZES
$ v4l2-ctl --list-frameintervals width=1920,height=1080,pixelformat=YUYV # 列出指定格式的frame size 信息 VIDIOC_ENUM_FRAMEINTERVALS
$ v4l2-ctl --list-formats-meta # meta device format VIDIOC_ENUM_FMT
$ v4l2-ctl -d /dev/v4l-subdev0 --list-subdev-mbus-code # VIDIOC_SUBDEV_ENUM_MBUS_CODE
$ v4l2-ctl -d /dev/v4l-subdev0 --list-subdev-framesizes pad=0,code=0x2008 # code为上面打印出来的format code. VIDIOC_SUBDEV_ENUM_FRAME_SIZE
$ v4l2-ctl -d /dev/v4l-subdev6 --list-subdev-frameintervals pad=0,width=2560,height=1440,code=0x3008 # width和height为上面返回的framesizes VIDIOC_SUBDEV_ENUM_FRAME_INTERVAL
$ v4l2-ctl --subset
$ v4l2-ctl --get-parm # get fps
$ v4l2-ctl --set-parm 60 # set fps
$ v4l2-ctl --info # 获取 device 信息

$ v4l2-ctl --list-ctrls
$ v4l2-ctl --list-ctrls-menus
$ v4l2-ctl --set-ctrl
$ v4l2-ctl --get-ctrl


$ v4l2-ctl --list-devices



$ v4l2-ctl --stream-mmap/user/dambuf

```
