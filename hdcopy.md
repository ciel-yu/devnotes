- [HD-copy Image(.IMG) Format](#hd-copy-imageimg-format)
	- [Format variants](#format-variants)
	- [Header](#header)
		- [18](#18)
		- [17](#17)
		- [1](#1)
		- [Tracks map](#tracks-map)
	- [Tracks data](#tracks-data)
	- [raw](#raw)

# HD-copy Image(.IMG) Format

HD-Copy 的镜像文件有四种变体（18 / 17 / 1 / raw）。写入（Put to file）时由 `/I` 开关决定生成压缩格式（默认 `/I+`）还是无头 raw 格式（`/I-`）；读取（Get from file）时按文件首字节自动识别格式，`/I` 不影响读取。

18 / 17 / 1 三种压缩格式的区别仅在于文件头的控制信息，磁道数据的编码方式是一样的。

## Format variants

| # | 格式 | 首字节识别 | 头部 | 产生者 |
|---|------|-----------|------|--------|
| 1 | 18 | `0xFF`，次字节 `0x18` | 184 字节 | HD-Copy 2.x，`/I+`（默认） |
| 2 | 17 | `0xFF`，次字节 `0x17` | 180 字节 | HD-Copy 1.7x |
| 3 | 1 | `0x4F..0x53`（`'O'..'S'`） | 166 字节 | 无魔数/卷标的同类镜像 |
| 4 | raw | 其它任意值 | 无 | HD-Copy `/I-` 及其他工具的无头镜像 |

识别流程：读入首字节；若为 `0xFF`，次字节必须为 `0x17` 或 `0x18`（压缩格式，接着读入 12 字节卷标）；若落在 `'O'..'S'` 范围内，则该字节本身就是轨数字节，按 1 格式（几何头）处理；其余情况按 raw 处理（校验引导扇区 BPB 与 FAT，失败报 "No disk image file"）。

18 整体布局示例：
```
FF 18                          ; 格式：HD-Copy 2.x 压缩
08 "HD-COPY  "                 ; 卷标（Pascal 串：len=8）
50 12                          ; T=0x50(80 轨) S=0x12(18 扇区)
[168 字节 tracks_map]           ; 非 0 槽后跟数据记录
-- 槽 0（track 0, head 0, boot）--
[size_lo size_hi]              ; token 流长度
[esc]                          ; 本轨最多字节（如 0xF6）
[token 流 ...]                 ; 字面量 / [esc][val][n] 包
-- 其后有数据的槽 -- ……
```

## Header

### 18
Length: 184 bytes
```
struct {
	byte    magic[2]; // 0xFF 0x18
	char    label[12]; // 卷标，Pascal 串：byte[0]=长度(<=11)，后跟字符，余下补 0x20(空格)
	byte    tracks; // total tracks - 1，合法范围 0x4F..0x53（79..83）
	byte    sectors; // sectors per track，合法范围 0x08..0x2B（8..43）
	byte	tracks_map[168]; // 槽状态表
}
```

### 17
Length: 180 bytes

结构同 18，区别仅在：

- 次字节为 `0x17`；
- `tracks_map` 只有 164 字节（读取后槽 164..167 一律按空处理，以兼容 ≤82 轨的镜像）。

### 1
Length: 166 bytes
```
struct {
	byte    tracks; // total tracks - 1，合法范围 0x4F..0x53（79..83）
	byte    sectors; // sectors per track，合法范围 0x08..0x2B（8..43）
	byte	tracks_map[164]; // 槽状态表
}
```
无魔数与卷标，首字节即轨数字节。

### Tracks map

- 头部 `tracks` 字节为 total tracks - 1，实际轨数 `T = tracks + 1`。
- 槽（slot）总数 = `T * 2`（每条磁道两个面各占一个槽）。
- 槽号编码：`slot = track * 2 + head`（每轨起始槽号 = `track * 2`；`slot & 1` = 磁头号，`slot >> 1` = 磁道号，偶数槽为 0 面）。
- `tracks_map[k]` 为槽 k 的状态：`0x00` = 空，镜像中不存在该槽的数据记录（读取时直接跳过）；非 0 = 有数据。HD-Copy 读完一轨后会把这个状态写回 3。

例如 tracks_map 开头为：
```
01 00 00 00 01 01 00 .. .. ..
```
表示槽 0、4、5 有数据，即 0 道 0 面、2 道两个面……其余槽没有记录。

18 的 168 字节 map 可容纳 84 轨 × 2 槽；17 / 1 的 164 字节对应 82 轨 × 2 槽。

## Tracks data

压缩格式（18 / 17 / 1）按 tracks_map 里非 0 的槽来保存数据。镜像数据区以每个磁道作为一个数据块，按槽号顺序存放。数据块的结构如下：
```
struct {
	short	size; // little endian，token 流的大小（整个记录 = size + 2 字节）
	byte	data[size]; // RLE 压缩的磁道内容
}
```
`data` 字段的第一个字节（`data[0]`）为指示符（escape），取该磁道内容中出现次数最多的字节；`data[1..]` 为 token 流：

- 字节 ≠ escape：字面量，直接输出；
- 字节 = escape：包 `[escape][value][count]`，3 字节一并消耗，输出 value 重复 count 次（count 为 8 位，上限 255，为 0 时无输出）。

需要单独输出 escape 字节本身时，用包 `[escape][escape][count]` 表达。

例子（escape = FF）：
```
FF 01 02 03 FF 04 03 05 06 FF 07 00 FF FF 01
```
其解码结果为：
```
01 02 03 04 04 04 05 06 FF
```
`04`重复了`03`次，`07`重复了`00`次，`FF`重复了`01`次。

解码结束条件：token 流耗尽，或输出长度达到 `0x5400`（21504 = 42×512，缓冲上限）。最终输出长度必须恰好为 `sectors * 512` 字节，否则报 "Error unpacking file"。

编码（Put）时的规则：escape 取整条磁道中出现次数最多的字节；长度 ≥3 的重复行程、以及 escape 字节的行程（任意长度）写成包，其余按字面量输出；行程长度上限 0xFF。
