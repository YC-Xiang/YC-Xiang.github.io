# Chapter9 Low Level Protocol

Features:

- 8-bit word size
- 一条 link 支持最高 4 条交替的 virtual channels
- frame start, frame end, line start, line end 有特殊的 packet
- type, pixel depth and format 有描述符
- 16-bit checksum

// TODO: 插图

## 9.1 Low Level Protocol Packet Format

有short 和 long packet.

都以 SoT(Start of Transmission) 开始和 EoT(End of Transmission) 结束.

### 9.1.1 Low Level Protocol Long Packet Format

// TODO: 长包图

DI: 定义了 virtual channel 和 data type, 长包的 data type 0x10~0x37.

### 9.1.2 Low Level Protocol Short Packet Format

// TODO: 短包图

短包Data type 0x0~0xf.

## 9.2 Data Identifier(DI)

// TODO: 插图

## 9.3 Virtual Channel Identifier

最多支持 4 条 virtual channel.

## 9.4 Data Type(DT)

// TODO: 插图

## 9.5 Packet Header Error Correction Code

skip

## 9.6 Checksum Generation

skip

## 9.7 Packet Spacing

长短包之间必须要有LPS(Low Power State)的进出.

## 9.8 Synchronization Short Packet Data Type Codes

frame start/end, line start/end 短包.

// TODO: 插图

### 9.8.1 Frame Synchronization Packets

### 9.8.2 Line Synchronization Packets

## 9.9 Generic Short Packet Data Type Codes

// TODO: 插图

mipi csi协议层对generic short packet的含义没有定义, 交给receiver来自己处理不同的data types.
