dtc 工具和 fdtdump 工具:

```shell
dtc –I dts –O dtb –o xxx.dtb xxx.dts
dtc –I dtb –O dts –o xxx.dts xxx.dtb

fdtdump -sd xxx.dtb > 1.txt # 有dtb header信息
```
