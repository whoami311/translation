# Live555 Media Server

## About

"LIVE555 Media Server" 是一个完整的 RTSP 服务器应用程序。它可以流式传输多种媒体文件（这些文件必须存储在当前工作目录中——即启动应用程序的目录——或其子目录中）：

- MPEG 传输流文件（文件名后缀为 ".ts"）
- Matroska 或 WebM 文件（文件名后缀为 ".mkv" 或 ".webm"）
- Ogg 文件（文件名后缀为 ".ogg", ".ogv" 或 ".opus"）
- MPEG-1 或 2 程序流文件（文件名后缀为 ".mpg"）
- MPEG-4 视频基本流文件（文件名后缀为 ".m4e"）
- H.264 视频基本流文件（文件名后缀为 ".264"）
- H.265 视频基本流文件（文件名后缀为 ".265"）
- VOB 视频+音频文件（文件名后缀为 ".vob"）
- DV 视频文件（文件名后缀为 ".dv"）
- MPEG-1 或 2（包括第 III 层——即 'MP3'）音频文件（文件名后缀为 ".mp3"）
- WAV（PCM）音频文件（文件名后缀为 ".wav"）
- AMR 音频文件（文件名后缀为 ".amr"）
- AC-3 音频文件（文件名后缀为 ".ac3"）
- AAC（ADTS 格式）音频文件（文件名后缀为 ".aac"）

这些流可以被标准兼容的 RTSP/RTP 媒体客户端接收/播放，包括：

- VLC 媒体播放器
- Amino 机顶盒（仅用于播放 MPEG 传输流）
- openRTSP 命令行 RTSP 客户端（它接收/存储流数据，但不播放）

注意事项：
- 服务器可以同时传输多个流（来自相同或不同的文件）
- 默认情况下，服务器以 RTP/UDP 数据包的形式传输其流。如果 RTSP 客户端请求，服务器将通过 TCP 流式传输其 RTP（和 RTCP）数据包。（这对位于防火墙后的客户端很有用。）
- 某些非标准 RTSP 客户端——如 Amino 机顶盒——请求原始 UDP 流，而不是标准 RTP 流。尽管如此，该服务器将适应此类请求，并根据需要通过原始 UDP 流式传输 MPEG 传输流。

## Running

服务器是一个控制台应用程序（目前没有图形用户界面版本）。要运行服务器，只需在命令行中输入 "live555MediaServer"。

## Trick Play functionality

服务器支持 RTSP 'trick play' 操作，但并非所有媒体类型都受支持：

- 暂停：所有媒体类型均支持
- 跳转：MPEG 传输流、DV 视频、WAV（PCM）音频、MPEG-1 或 2 音频、MPEG-1 或 2 程序流（部分工作）、Matroska 或 WebM 文件
- 快进：MPEG 传输流、WAV（PCM）音频、MPEG-1 或 2 音频
- 倒放：MPEG 传输流、WAV（PCM）音频

请注意，为了对流式传输的 MPEG 传输流文件提供 'trick play' 操作，必须为每个这样的文件创建一个特殊的 '索引文件'，可以使用我们的 "MPEG2TransportStreamIndexer" 工具来创建。

未来将添加对更多媒体类型的 trick play 支持。