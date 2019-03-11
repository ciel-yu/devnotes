- [HD-copy Image(.IMG) Format](#hd-copy-imageimg-format)
	- [Header](#header)
		- [v1](#v1)
		- [v2](#v2)
	- [Tracks data](#tracks-data)

# HD-copy Image(.IMG) Format

HD-Copy的镜像至少有两个版本，一个版本支持最多82磁道，另一个版本支持最多84磁道。
两个版本的区别仅在于文件头的控制信息，磁盘内容的编码方式是一样的。

## Header

### v1
Length: 166 bytes
```
struct {
	byte    tracks; // total tracks - 1
	byte    sectors; // sectors per track
	byte	tracks_map[164]; 
}
```

### v2
Length: 184 bytes
```
struct {
	char    magic[3]; // 0xFF 0x18 0x??
	char    label[11]; // padded with 0x20(space)
	byte    tracks; // total tracks - 1
	byte    sectors; // sectors per track
	byte	tracks_map[168]; 
}
```

## Tracks data
HD-Copy按照tracks_map里对应值为1的磁道来保存数据。比如下列结构：
```
01 00 00 00 01 01 00 .. .. ..
```
镜像里就保存了第0，4，5...磁道的数据。
所以，磁盘镜像数据区是以每个磁道作为一个数据块，按照磁道顺序来存放数据。数据块的结构如下：
```
struct {
	short	size; // little endian, 数据块的大小
	byte	data[size]; // RLE压缩的磁道内容
}
```
`data`字段的的第一个字节(`data[0]`)为指示符，意味着当后续的内容出现指示符时，指示符后面的第一个字节为需要重复的内容，第二个字节为重复的次数。
例子：
```
FF 01 02 03 FF 04 03 05 06 FF 07 00 FF FF 01
```
其解码结果为：
```
01 02 03 04 04 04 05 06 FF
```
`04`重复了`03`次，`07`重复了`00`次，`FF`重复了`01`次。

最终，每个数据块解码后的长度应该与`512 * sectors`一致。
