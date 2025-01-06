# LIVE555 Streaming Media

这组 C++ 库用于多媒体流式传输，使用开放标准协议（RTP/RTCP、RTSP、SIP）。这些库可以编译用于 Unix（包括 Linux 和 Mac OS X）、QNX（及其他 POSIX 兼容系统），可用于构建流媒体应用程序。这些库已经被用来实现诸如 "LIVE555 Media Server"、"LIVE555 Proxy Server"、"LIVE555 HLS Proxy" 和 "vobStreamer"（用于通过 RTP/RTCP/RTSP 流式传输 DVD 内容）等应用程序。此外，这些库还可以用于流式传输、接收和处理 MPEG、H.265、H.264、H.263+、DV 或 JPEG 视频以及多种音频编码格式。它们可以轻松扩展以支持额外的（音频或视频）编码格式，并且可以用来构建基本的 RTSP 或 SIP 客户端和服务器。这些库还被用来为现有的媒体播放器应用程序（如 "VLC" 和 "MPlayer"）添加流式传输支持。（有关这些库如何使用的具体示例，请参阅下面的测试程序。）

## Description

该代码包含以下各个库，每个库都有自己的子目录：

### UsageEnvironment
"UsageEnvironment" 和 "TaskScheduler" 类用于调度延迟事件、分配异步读取事件的处理程序以及输出错误/警告消息。此外，"HashTable" 类定义了一个通用哈希表的接口，供其余代码使用。

这些都是抽象基类，必须在实现中进行子类化才能使用。这些子类可以利用程序运行环境（例如其图形用户界面和/或脚本环境）的特定属性。

### groupsock
此库中的类封装了网络接口和套接字。特别是 "Groupsock" 类封装了一个用于发送（和/或接收）多播数据报的套接字。

### liveMedia
此库定义了一个以 "Medium" 类为根的类层次结构，用于各种流媒体类型和编解码器。

### BasicUsageEnvironment
此库定义了 "UsageEnvironment" 类的一个具体实现（即子类），用于简单的控制台应用程序。读取事件和延迟操作通过 `select()` 循环来处理。

### testProgs
此目录实现了一些简单程序，使用 "BasicUsageEnvironment" 来演示如何使用这些库开发应用程序。

### RTSP client

- **testRTSPClient** 是一个命令行程序，展示了如何打开并接收由 RTSP URL（即以 rtsp:// 开头的 URL）指定的媒体流。在这个演示应用程序中，接收到的音频/视频数据没有被处理或使用。但是，您可以在自己的应用程序中使用和调整这段代码，例如解码并播放接收到的数据。

- **openRTSP** 类似于 testRTSPClient，但功能更多。它是一个命令行程序，与 testRTSPClient 不同的是，它旨在作为一个完整且功能齐全的应用程序使用（而不是将代码嵌入其他应用程序）。有关 openRTSP 的更多信息，包括其众多命令行选项，请参阅[在线文档](http://www.live555.com/openRTSP/)。

### RTSP server

- **testOnDemandRTSPServer** 创建了一个 RTSP 服务器，可以通过 RTP 单播按需从各种类型的媒体文件进行流式传输。（支持的媒体类型包括：MPEG-1 或 2 音频或视频（基本流），包括 MP3 音频；MPEG-4 视频（基本流）；H.264 视频（基本流）；H.265 视频（基本流）；MPEG 程序或传输流，包括 VOB 文件；DV 视频；AMR 音频；WAV（PCM）音频。）服务器还可以通过解复用并流式传输文件中的轨道来从 Matroska 或 WebM 文件进行流式传输。如果请求，MPEG 传输流也可以通过原始 UDP 流式传输——例如，由机顶盒请求。
    - 此外，该服务器应用程序还展示了如何通过 RTSP 交付到达服务器的 MPEG 传输流（作为 UDP（原始 UDP 或 RTP/UDP）多播或单播流）。特别是，默认情况下，它设置为接受来自 "testMPEG2TransportStreamer" 演示应用程序的输入。

### SIP client

- **playSIP** 是一个命令行程序（类似于 openRTSP），它向 SIP 会话发起呼叫（使用 sip: URL），然后（可选地）将传入的媒体流记录到文件中。

### MP3 audio test programs

- **testMP3Streamer** 反复从名为 "test.mp3" 的 MP3 音频文件读取，并使用 RTP 将其流式传输到多播组 239.255.42.42，端口 6666（RTCP 使用端口 6667）。该程序也有一个（可选的）内置 RTSP 服务器。

- **testMP3Receiver** 执行相反的操作：它从相同的多播组/端口读取 MP3/RTP 流，并将重构的 MP3 流输出到标准输出（stdout）。它还发送 RTCP 接收报告。
    - 或者，可以使用这些工具之一播放 MP3/RTP 流。

### MPEG video test programs

- **testMPEG1or2VideoStreamer** 反复从名为 "test.mpg" 的 MPEG-1 或 2 视频文件读取，并使用 RTP 协议将视频流式传输到多播组 239.255.42.42，端口 8888（RTCP 使用端口 8889）。该程序还包含一个（可选的）内置 RTSP 服务器。

    - 默认情况下，输入文件被假定为 MPEG 视频基本流（Elementary Stream）。然而，如果输入文件是 MPEG 程序流（Program Stream），则可以插入一个解复用过滤器以提取视频基本流。（详情见 testMPEG1or2VideoStreamer.cpp）
    - Apple 的 "QuickTime Player" 可用于接收和查看此流式传输的视频（前提是视频为 MPEG-1，而不是 MPEG-2）。要使用此功能，请让 QuickTime Player 打开文件 "testMPEG1or2Video.sdp"。（如果启用了 testMPEG1or2VideoStreamer 的 RTSP 服务器，则 QuickTime Player 还可以通过 "rtsp://" URL 播放流。）
    - 开源的 "VLC" 和 "MPlayer" 媒体播放器也可以用于播放此流。
    - RealNetworks 的 "RealPlayer" 也可以用于播放此流。（建议使用最新版本。）

- **testMPEG1or2VideoReceiver** 执行相反的操作：它从相同的多播组/端口读取 MPEG 视频/RTP 流，并将重构的 MPEG 视频（基本流）输出到标准输出（stdout）。它还发送 RTCP 接收报告。

- **testMPEG4VideoStreamer** 反复从名为 "test.m4e" 的 MPEG-4 基本流（Elementary Stream）视频文件读取，并使用 RTP 多播进行流式传输。该程序还包含一个内置的 RTSP 服务器。

    - Apple 的 "QuickTime Player" 可用于接收和播放此视频流。要使用此功能，请让播放器打开会话的 "rtsp://" URL（该 URL 在程序开始流式传输时打印出来）。
    - 开源的 "VLC" 和 "MPlayer" 媒体播放器也可以用于播放此流。

- **testH264VideoStreamer** 反复从名为 "test.264" 的 H.264 基本流（Elementary Stream）视频文件读取，并使用 RTP 多播进行流式传输。该程序还包含一个内置的 RTSP 服务器。

    - Apple 的 "QuickTime Player" 可用于接收和播放此视频流。要使用此功能，请让播放器打开会话的 "rtsp://" URL（该 URL 在程序开始流式传输时打印出来）。
    - 开源的 "VLC" 和 "MPlayer" 媒体播放器也可以用于播放此流。

- **testH265VideoStreamer** 反复从名为 "test.265" 的 H.265 基本流视频文件读取，并使用 RTP 多播进行流式传输。该程序也具有内置的 RTSP 服务器。

### MPEG audio+video (Program Stream) test programs

- **testMPEG1or2AudioVideoStreamer** 读取名为 "test.mpg" 的 MPEG-1 或 2 程序流（Program Stream）文件，从中提取音频和视频基本流（Elementary Stream），并使用 RTP 协议将这些流分别传输到多播组 239.255.42.42，端口 6666/6667（用于音频流）和端口 8888/8889（用于视频流）。该程序还包含一个（可选的）内置 RTSP 服务器。

    - Apple 的 "QuickTime Player" 可用于接收和查看此流式传输的视频（前提是视频为 MPEG-1，而不是 MPEG-2）。要使用此功能，请让 QuickTime Player 打开文件 "testMPEG1or2AudioVideo.sdp"。（如果启用了 testMPEG1or2VideoStreamer 的 RTSP 服务器，则 QuickTime Player 还可以通过 "rtsp://" URL 播放流。）
    - 开源的 "VLC" 和 "MPlayer" 媒体播放器也可以用于播放此流。

- **testMPEG1or2Splitter** 读取名为 "in.mpg" 的 MPEG-1 或 2 程序流文件，从中提取音频和视频基本流。这两个基本流分别写入名为 "out_audio.mpg" 和 "out_video.mpg" 的文件。

### MPEG audio+video (Transport Stream) test programs

- **testMPEG2TransportStreamer** 读取名为 "test.ts" 的 MPEG 传输流文件，并使用 RTP 将其流式传输到多播组 239.255.42.42，端口 1234（RTCP 使用端口 1235）。该程序也有一个（可选的）内置 RTSP 服务器。
    - 开源的 "VLC" 媒体播放器可以用于播放此流。

- **testMPEG2TransportReceiver** 执行相反的操作：它从相同的多播组/端口读取 MPEG 传输/RTP 流，并将重构的 MPEG 传输流输出到标准输出（stdout）。它还发送 RTCP 接收报告。

- **testMPEG1or2ProgramToTransportStream** 读取名为 "in.mpg" 的 MPEG-1 或 2 程序流文件，并将其转换为等效的 MPEG 传输流文件，命名为 "out.ts"。

- **testH264VideoToTransportStream** 读取名为 "in.264" 的 H.264 视频基本流文件，并将其转换为等效的 MPEG 传输流文件，命名为 "out.ts"。

- **testH265VideoToTransportStream** 读取名为 "in.265" 的 H.265 视频基本流文件，并将其转换为等效的 MPEG 传输流文件，命名为 "out.ts"。

- **testMPEG2TransportStreamSplitter** 对传输流文件进行解复用，生成每个组件轨道的输出文件。（注意：此应用程序从 stdin 读取。）

### PCM audio test program

- **testWAVAudioStreamer** 从名为 "test.wav" 的 WAV 格式音频文件读取，并通过 IP 多播使用内置 RTSP 服务器流式传输其中包含的 PCM 音频流。

    - 该程序支持 8 或 16 位 PCM 流，单声道或立体声，任意采样频率。
    - Apple 的 "QuickTime Player" 可用于接收和播放此音频流。要使用此功能，请让播放器打开会话的 "rtsp://" URL（该 URL 在程序开始流式传输时打印出来）。
    - 可选地，在流式传输之前可以将 16 位 PCM 流转换为 8 位 u-law 格式。（请参阅 testWAVAudioStreamer.cpp 获取如何执行此操作的说明）
    - 开源的 "VLC" 和 "MPlayer" 媒体播放器也可以用于播放此流。

### AMR audio test program

- **testAMRAudioStreamer** 从名为 "test.amr" 的 AMR 格式音频文件读取（如 RFC 3267 第 5 节所定义），并通过 IP 多播使用内置 RTSP 服务器流式传输其中包含的音频流。

    - Apple 的 "QuickTime Player" 可用于接收和播放此音频流。要使用此功能，请让播放器打开会话的 "rtsp://" URL（该 URL 在程序开始流式传输时打印出来）。

### DV video test program
- **testDVVideoStreamer** 从名为 "test.dv" 的 DV 视频文件读取，并通过 IP 多播使用内置 RTSP 服务器进行流式传输。

    - 目前，我们尚未发现任何广泛可用的媒体播放器客户端可以播放此流。

### Matroska (or 'Webm') test programs

- **testMKVStreamer** 从名为 "test.mkv" 的 Matroska（或 WebM）文件读取，并通过 IP 多播使用内置 RTSP 服务器进行流式传输。
- **testMKVSplitter** 从 Matroska（或 WebM）文件读取，并将其解复用为单独的输出文件——每个轨道一个文件。

### VOB (DVD) streaming test program

- **vobStreamer**：读取一个或多个 ".vob" 文件（例如来自 DVD），提取音频和视频流，并使用 RTP 多播进行传输。

### Support for server 'trick play' operations on MPEG Transport Stream files

- 应用程序 MPEG2TransportStreamIndexer 和 testMPEG2TransportStreamTrickPlay。

### HLS ("HTTP Live Streaming") test programs

- **testH264VideoToHLSSegments**将名为 "in.264" 的 H.264 基本流视频文件转换为一系列 HLS（“HTTP Live Streaming”）片段，以及一个可以通过网页浏览器访问的 ".m3u8" 文件。（请注意，LIVE555 HLS Proxy 可以实时将 RTSP 流转换为一组 HLS 片段。）

### Miscellaneous test programs

- **testRelay** 反复从 UDP 多播套接字读取，并将每个数据包的有效载荷重新传输（“中继”）到新的（多播或单播）地址和端口。
- **testReplicator** 类似于 testRelay，但它使用 "FrameReplicator" 类复制输入流，并将一个副本流重新传输到另一个（多播或单播）地址和端口，同时将另一个副本流写入文件。
- **sapWatch** 读取并打印在默认 SDP/SAP 目录（224.2.127.254/9875）上发布的 SDP/SAP 公告。
- **registerRTSPStream** 向给定的 RTSP 客户端（或代理服务器）发送自定义的 RTSP "REGISTER" 命令，要求其从给定的 "rtsp://" URL 进行流式传输。
- **mikeyParse** 解析 Base64 字符串（该字符串编码了二进制 MIKEY（多媒体密钥管理）消息），并输出 MIKEY 消息的人类可读描述。（Base64 字符串可以例如用于 SDP "a=key-mgmt:" 属性，如 RFC 4567 所定义。）

### WindowsAudioInputDevice

这是 "liveMedia" 库中 "AudioInputDevice" 抽象类的一个实现。它可以被 Windows 应用程序用于从输入设备读取 PCM 音频采样。

（此项目构建两个库："libWindowsAudioInputDevice_mixer.lib" 使用 Windows 内置的混音器，而 "libWindowsAudioInputDevice_noMixer.lib" 则不使用混音器。）

## How to configure and build the code on Unix (including Linux, Mac OS X, QNX, and other Posix-compliant systems)

源代码包可以在[这里](http://www.live555.com/liveMedia/public/)找到（作为一个 ".tar.gz" 文件）。使用 "tar -x" 和 "gunzip"（或如果可用的话，使用 "tar -xz"）来解压该包；然后进入 "live" 目录。接着运行以下命令：

```shell
./genMakefiles <os-platform>
```
其中 <os-platform> 是您的目标平台——例如 "linux" 或 "solaris"——由一个 "config.<os-platform>" 文件定义。这将在 "live" 目录及其每个子目录中生成一个 Makefile。然后运行 make。

- 如果 make 失败，您可能需要对相应的 "config.<os-platform>" 文件进行小的修改，然后重新运行 genMakefiles <os-platform>。（例如，您可能需要向 COMPILE_OPTS 定义中添加另一个 "-I<dir>" 标志。）
- 一些用户（特别是 FreeBSD 用户）报告说，GNU 版本的 make——通常称为 gmake——比他们默认预安装的 make 版本工作得更好。（特别是，如果您遇到与 ar 命令链接问题时，应该尝试使用 gmake。）
- 如果您使用的是 "gcc" 3.0 或更高版本：您可能还希望在 CPLUSPLUS_FLAGS 中添加 -Wno-deprecated 标志。
- 如果您的目标平台没有对应的 "config.<os-platform>" 文件，则可以尝试使用现有文件中的一个作为模板。

如果您愿意，您还可以通过运行 make install 来“安装”头文件、库和应用程序。
