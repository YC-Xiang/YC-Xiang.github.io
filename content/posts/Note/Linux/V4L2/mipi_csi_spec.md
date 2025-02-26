# Chapter 9 Low Level Protocol

Features:

- 8-bit word size
- 一条 link 支持最高 4 条交替的 virtual channels
- frame start, frame end, line start, line end 有特殊的 packet
- type, pixel depth and format 有描述符
- 16-bit checksum

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101506.png)

## 9.1 Low Level Protocol Packet Format

有short 和 long packet.

都以 SoT(Start of Transmission) 开始和 EoT(End of Transmission) 结束.

### 9.1.1 Low Level Protocol Long Packet Format

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101604.png)

DI: 定义了 virtual channel 和 data type, 长包的 data type 0x10~0x37.

### 9.1.2 Low Level Protocol Short Packet Format

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101641.png)

短包Data type 0x0~0xf.

## 9.2 Data Identifier(DI)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101709.png)

高 2bit 表示 virtual channel 号, 低 6bit 表示 data type.

## 9.3 Virtual Channel Identifier

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101835.png)

最多支持 4 条 virtual channel.

## 9.4 Data Type(DT)

所有的data type类型:

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224101913.png)

## 9.5 Packet Header Error Correction Code

skip

## 9.6 Checksum Generation

skip

## 9.7 Packet Spacing

长短包之间必须要有LPS(Low Power State)的进出.

## 9.8 Synchronization Short Packet Data Type Codes

短包的data type有: frame start/end, line start/end 短包.

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20250224102142.png)

### 9.8.1 Frame Synchronization Packets

### 9.8.2 Line Synchronization Packets

## 9.9 Generic Short Packet Data Type Codes

短包 data type 0x8~0xf 是 generic short packet.

mipi csi 协议层对 generic short packet 的含义没有定义, 交给 receiver 来自己处理不同的 data types.

## 9.10 Packet Spacing Examples

展示了 line blanking 和 frame blanking 的情况.

## 9.11 Packet Data Payload Size Rules

## 9.12 Frame Format Examples

# Chapter 10 Color Spaces

# Chapter 11 Data Formats
