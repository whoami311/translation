# ffmpeg

## 1. 概要

```shell
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...
```

## 2. 描述

`ffmpeg` 是一款通用的媒体转换器。 它能读取各种输入（包括实时抓取/录制设备），并将其过滤和转码为大量输出格式。

`ffmpeg` 从任意数量的输入（可以是普通文件、管道、网络流、抓取设备等）中读取（由 `-i` 选项指定），并写入任意数量的输出（由纯输出 url 指定）。 命令行中任何不能被解释为选项的内容都被视为输出网址。

原则上，每个输入或输出可以包含任意数量不同类型的基本流（视频/音频/字幕/attachment/数据），但允许的流数量和/或类型可能会受到容器格式的限制。 自动或使用 `-map` 选项可以选择哪些输入流将进入哪些输出流（请参阅“流选择”章节）。

要在选项中引用输入/输出，必须使用它们的索引（以 0 为基础）。 例如，第一个输入为`0`，第二个输入为`1`，等等。同样，输入/输出中的数据流也用其索引来表示。 例如，2:3 指的是第三个输入或输出中的第四个数据流。 另请参阅“流指定符”章节。

一般来说，选项会应用于下一个指定文件。 因此，顺序很重要，同一选项可以在命令行中出现多次。 每次出现的选项都会应用到下一个输入或输出文件。 这条规则的例外情况是全局选项（如冗长程度），应首先指定全局选项。

不要混用输入和输出文件 - 先指定所有输入文件，然后再指定所有输出文件。 也不要混用属于不同文件的选项。 所有选项只适用于下一个输入或输出文件，并在文件之间重置。

下面是一些简单的例子。

- 通过重新编码媒体流，将输入媒体文件转换为不同格式
    ffmpeg -i input.avi output.mp4
- 将输出文件的视频比特率设置为 64 kbit/s
    ffmpeg -i input.avi -b:v 64k -bufsize 64k output.mp4
- 将输出文件的帧频强制设置为 24 fps：
    ffmpeg -i input.avi -r 24 output.mp4
- 强制将输入文件（仅对原始格式有效）的帧频设为 1 fps，将输出文件的帧频设为 24 fps
    ffmpeg -r 1 -i input.m2v -r 24 output.mp4

原始输入文件可能需要格式选项。

## 3. 详细说明

`ffmpeg` 由以下组件组成转码管道。程序运行时，输入数据块从数据源沿着管道流向数据汇，并被沿途遇到的组件转换。

可使用的组件有以下几种：

- *Demuxers*（"demultiplexers"的简称）读取输入源，以提取
    - 全局属性，如元数据或章节；
    - 输入基本流及其属性列表
    每个 -i 选项都会创建一个解复用器实例，并将编码后的*packet*发送到*decoders*或*muxers*。
    
    在其他文献中，解复用器有时也被称为分割器，因为它们的主要功能是将文件分割成基本流（尽管有些文件只包含一个基本流）。
    
    解复用器的示意图如下：

    ```
    ┌──────────┬───────────────────────┐
    │ demuxer  │                       │ packets for stream 0
    ╞══════════╡ elementary stream 0   ├──────────────────────►
    │          │                       │
    │  global  ├───────────────────────┤
    │properties│                       │ packets for stream 1
    │   and    │ elementary stream 1   ├──────────────────────►
    │ metadata │                       │
    │          ├───────────────────────┤
    │          │                       │
    │          │     ...........       │
    │          │                       │
    │          ├───────────────────────┤
    │          │                       │ packets for stream N
    │          │ elementary stream N   ├──────────────────────►
    │          │                       │
    └──────────┴───────────────────────┘
         ▲
         │
         │ read from file, network stream,
         │     grabbing device, etc.
         │
    ```

- *Decoders*接收音频、视频或字幕基本流的编码（压缩）*packets*，并将其解码为原始*frame*（视频为像素阵列，音频为 PCM）。 解码器通常与*demuxer*中的基本流相关联（并从基本流接收输入），但有时也可能单独存在（见回环解码器）。
    解码器的原理图如下所示：
    ```
              ┌─────────┐
     packets  │         │ raw frames
    ─────────►│ decoder ├────────────►
              │         │
              └─────────┘
    ```

- *Filtergraphs*处理和转换原始音频或视频*frames*。 滤波器图由一个或多个单独的*filters*组成，这些滤波器连接成一个图。 过滤图分为*simple*和*complex*两种，分别使用 -filter 和 -filter_complex 选项进行配置。
    一个简单的过滤图与一个*output elementary stream*相关联；它从*decoder*接收要过滤的输入，并将过滤后的输出发送到该输出流的*encoder*。 一个简单的视频过滤图可以执行去隔行（使用 `yadif` 去隔行器），然后调整大小（使用`scale`过滤器），看起来就像这样：
    ```
                 ┌────────────────────────┐
                 │  simple filtergraph    │
     frames from ╞════════════════════════╡ frames for
     a decoder   │  ┌───────┐  ┌───────┐  │ an encoder
    ────────────►├─►│ yadif ├─►│ scale ├─►│────────────►
                 │  └───────┘  └───────┘  │
                 └────────────────────────┘
    ```
    复合滤波器图是独立的，不与任何特定的数据流相关联。 它可能有多个（或零个）不同类型（音频或视频）的输入，每个输入接收来自解码器或其他复合滤波器输出的数据。 它还有一个或多个输出端，为编码器或另一个复杂滤波图的输入端提供数据。
    下图示例表示一个复杂的滤波器图，有 3 个输入端和 2 个输出端（均为视频）：
    ```
              ┌─────────────────────────────────────────────────┐
              │               complex filtergraph               │
              ╞═════════════════════════════════════════════════╡
     frames   ├───────┐  ┌─────────┐      ┌─────────┐  ┌────────┤ frames
    ─────────►│input 0├─►│ overlay ├─────►│ overlay ├─►│output 0├────────►
              ├───────┘  │         │      │         │  └────────┤
     frames   ├───────┐╭►│         │    ╭►│         │           │
    ─────────►│input 1├╯ └─────────┘    │ └─────────┘           │
              ├───────┘                 │                       │
     frames   ├───────┐ ┌─────┐ ┌─────┬─╯              ┌────────┤ frames
    ─────────►│input 2├►│scale├►│split├───────────────►│output 1├────────►
              ├───────┘ └─────┘ └─────┘                └────────┤
              └─────────────────────────────────────────────────┘
    ```
    第二路输入的帧叠加在第一路输入的帧上。 来自第三个输入端的帧被重新缩放，然后复制成两个相同的流。 其中一个叠加在前两个输入的组合上，结果作为过滤图的第一个输出。 另一个副本最终成为过滤图的第二个输出。
- *Encoders*接收原始音频、视频或字幕*frame*，并将其编码成编码*packet*。 编码（压缩）过程通常是*lossy*--它会降低数据流的质量，使输出变小；有些编码器是*lossless*，但代价是输出变大。 视频或音频编码器从某个过滤图的输出接收输入，字幕编码器从解码器接收输入（因为目前还不支持字幕过滤）。 每个编码器都与某个多路复用器的*output elementary stream*相关联，并将其输出发送到该多路复用器。
    编码器的示意图如下所示：
    ```
                 ┌─────────┐
     raw frames  │         │ packets
    ────────────►│ encoder ├─────────►
                 │         │
                 └─────────┘
    ```

- *Muxers*（"multiplexers"的缩写）从编码器（*transcoding*路径）或直接从解复用器（*streamcopy*路径）接收其基本流的编码*packet*，将其交错（当有多个基本流时），并将生成的字节写入输出文件（或管道、网络流等）。
    多路复用器的原理图如下所示：
    ```
                           ┌──────────────────────┬───────────┐
     packets for stream 0  │                      │   muxer   │
    ──────────────────────►│  elementary stream 0 ╞═══════════╡
                           │                      │           │
                           ├──────────────────────┤  global   │
     packets for stream 1  │                      │properties │
    ──────────────────────►│  elementary stream 1 │   and     │
                           │                      │ metadata  │
                           ├──────────────────────┤           │
                           │                      │           │
                           │     ...........      │           │
                           │                      │           │
                           ├──────────────────────┤           │
     packets for stream N  │                      │           │
    ──────────────────────►│  elementary stream N │           │
                           │                      │           │
                           └──────────────────────┴─────┬─────┘
                                                        │
                         write to file, network stream, │
                             grabbing device, etc.      │
                                                        │
                                                        ▼
    ```

### 3.1 Streamcopy

`ffmpeg`中最简单的流水线是 single-stream *streamcopy*，即复制一个*input elementary stream*的数据包，而不对其进行解码、过滤或编码。 举例来说，一个名为 INPUT.mkv 的输入文件包含 3 个基本流，我们从中提取第二个流并将其写入文件 OUTPUT.mp4。 这种流水线的示意图如下：

```
┌──────────┬─────────────────────┐
│ demuxer  │                     │ unused
╞══════════╡ elementary stream 0 ├────────╳
│          │                     │
│INPUT.mkv ├─────────────────────┤          ┌──────────────────────┬───────────┐
│          │                     │ packets  │                      │   muxer   │
│          │ elementary stream 1 ├─────────►│  elementary stream 0 ╞═══════════╡
│          │                     │          │                      │OUTPUT.mp4 │
│          ├─────────────────────┤          └──────────────────────┴───────────┘
│          │                     │ unused
│          │ elementary stream 2 ├────────╳
│          │                     │
└──────────┴─────────────────────┘
```
上述管道可通过以下命令行构建：
```shell
ffmpeg -i INPUT.mkv -map 0:1 -c copy OUTPUT.mp4
```
在该命令行中
- 只有一个输入 INPUT.mkv；
- 该输入没有输入选项；
- 只有一个输出 OUTPUT.mp4；
- 该输出有两个输出选项：
  - `-map 0:1` 选择要使用的输入流 - 从索引 0 的输入（即第一个）到索引 1 的输入（即第二个）；
  - `-c copy` 选择`copy`编码器，即不进行解码或编码的流复制。

流拷贝可用于更改基本流计数、容器格式或修改容器级元数据。 由于不需要解码或编码，因此速度非常快，也不会造成质量损失。 不过，在某些情况下，由于各种因素（如目标容器所需的某些信息在源文件中不存在），这种方法可能无法奏效。 应用过滤器显然也是不可能的，因为过滤器是在解码帧上工作的。 

还可以构建更复杂的流拷贝方案，例如将两个输入文件的流合并为一个输出文件：
```
┌──────────┬────────────────────┐         ┌────────────────────┬───────────┐
│ demuxer 0│                    │ packets │                    │   muxer   │
╞══════════╡elementary stream 0 ├────────►│elementary stream 0 ╞═══════════╡
│INPUT0.mkv│                    │         │                    │OUTPUT.mp4 │
└──────────┴────────────────────┘         ├────────────────────┤           │
┌──────────┬────────────────────┐         │                    │           │
│ demuxer 1│                    │ packets │elementary stream 1 │           │
╞══════════╡elementary stream 0 ├────────►│                    │           │
│INPUT1.aac│                    │         └────────────────────┴───────────┘
└──────────┴────────────────────┘
```
可以通过命令行
```shell
ffmpeg -i INPUT0.mkv -i INPUT1.aac -map 0:0 -map 1:0 -c copy OUTPUT.mp4
```
这里使用了两次输出 -map 选项，在输出文件中创建了两个数据流--一个由第一个输入数据流提供，另一个由第二个输入数据流提供。 单个 -c 选项实例会为这两个流选择流复制。 你也可以将该选项的多个实例与流指定符一起使用，对每个流应用不同的值，这将在下面的章节中演示。

相反的情况是从一个输入分割成多个输出的多个流：

```
┌──────────┬─────────────────────┐          ┌───────────────────┬───────────┐
│ demuxer  │                     │ packets  │                   │ muxer 0   │
╞══════════╡ elementary stream 0 ├─────────►│elementary stream 0╞═══════════╡
│          │                     │          │                   │OUTPUT0.mp4│
│INPUT.mkv ├─────────────────────┤          └───────────────────┴───────────┘
│          │                     │ packets  ┌───────────────────┬───────────┐
│          │ elementary stream 1 ├─────────►│                   │ muxer 1   │
│          │                     │          │elementary stream 0╞═══════════╡
└──────────┴─────────────────────┘          │                   │OUTPUT1.mp4│
                                            └───────────────────┴───────────┘
```
通过以下命令行构建
```shell
ffmpeg -i INPUT.mkv -map 0:0 -c copy OUTPUT0.mp4 -map 0:1 -c copy OUTPUT1.mp4
```
请注意，尽管每个输出文件的值都是一样的，但每个输出文件都需要一个单独的 -c 选项实例。 这是因为非全局选项（即大多数选项）只适用于其所在文件的上下文。 当然，这些例子可以进一步推广到任意数量的输入到任意数量的输出的任意重映射。

### 3.2 Trancoding

*Transcoding*是将数据流解码后再重新编码的过程。由于编码的计算成本往往很高，而且在大多数情况下会降低视频流的质量（即*lossy*），因此只有在需要时才进行转码，否则就进行流复制。 转码的典型原因有：
- 应用滤波器--如调整大小、去隔行或叠加视频；重新采样或混合音频；
- 您想将流传输到无法解码原始编解码器的设备上。

请注意，除非指定 -c 复制，否则 `ffmpeg` 将对所有音频、视频和字幕流进行转码。
请看一个管道示例，它读取包含一个音频流和一个视频流的输入文件，然后将视频和音频复制到一个输出文件中。其示意图如下
```
┌──────────┬─────────────────────┐
│ demuxer  │                     │       audio packets
╞══════════╡ stream 0 (audio)    ├─────────────────────────────────────╮
│          │                     │                                     │
│INPUT.mkv ├─────────────────────┤ video    ┌─────────┐     raw        │
│          │                     │ packets  │  video  │ video frames   │
│          │ stream 1 (video)    ├─────────►│ decoder ├──────────────╮ │
│          │                     │          │         │              │ │
└──────────┴─────────────────────┘          └─────────┘              │ │
                                                                     ▼ ▼
                                                                     │ │
┌──────────┬─────────────────────┐ video    ┌─────────┐              │ │
│ muxer    │                     │ packets  │  video  │              │ │
╞══════════╡ stream 0 (video)    │◄─────────┤ encoder ├──────────────╯ │
│          │                     │          │(libx264)│                │
│OUTPUT.mp4├─────────────────────┤          └─────────┘                │
│          │                     │                                     │
│          │ stream 1 (audio)    │◄────────────────────────────────────╯
│          │                     │
└──────────┴─────────────────────┘
```
通过以下命令行实现：
```shell
ffmpeg -i INPUT.mkv -map 0:v -map 0:a -c:v libx264 -c:a copy OUTPUT.mp4
```
请注意它是如何使用流指定符 `:v` 和 `:a` 来选择输入流，并将 -c 选项的不同值应用于这些输入流的；更多细节，请参阅 "流指定符 "章节。

### 3.3 Filtering

转码时，可在编码前使用 *simple* 或 *complex* 的过滤图过滤音频和视频流。

#### 3.3.1 Simple filtergraphs

简单滤波器图是指输入和输出均为同一类型（音频或视频）的滤波器图。 它们使用 per-stream -filter 选项进行配置（-vf 和 -af 别名分别表示 -filter:v（视频）和 -filter:a（音频））。 请注意，简单过滤图与输出流相关联，因此，如果您有多个音频流，-af 将为每个音频流创建一个单独的过滤图。

以上面的转码为例，加上过滤（为清晰起见，省略音频）后，看起来就像这样：

```
┌──────────┬───────────────┐
│ demuxer  │               │          ┌─────────┐
╞══════════╡ video stream  │ packets  │  video  │ frames
│INPUT.mkv │               ├─────────►│ decoder ├─────►───╮
│          │               │          └─────────┘         │
└──────────┴───────────────┘                              │
                                  ╭───────────◄───────────╯
                                  │   ┌────────────────────────┐
                                  │   │  simple filtergraph    │
                                  │   ╞════════════════════════╡
                                  │   │  ┌───────┐  ┌───────┐  │
                                  ╰──►├─►│ yadif ├─►│ scale ├─►├╮
                                      │  └───────┘  └───────┘  ││
                                      └────────────────────────┘│
                                                                │
                                                                │
┌──────────┬───────────────┐ video    ┌─────────┐               │
│ muxer    │               │ packets  │  video  │               │
╞══════════╡ video stream  │◄─────────┤ encoder ├───────◄───────╯
│OUTPUT.mp4│               │          │         │
│          │               │          └─────────┘
└──────────┴───────────────┘
```

#### 3.3.2 Complex filtergraphs

复杂过滤图是指那些不能简单描述为适用于一个数据流的线性处理链的过滤图。 例如，当过滤图有多个输入和/或输出，或输出流类型与输入不同时，就属于这种情况。复杂过滤图使用 -filter_complex 选项进行配置。 需要注意的是，该选项是全局性的，因为复杂过滤图就其本质而言，无法明确地与单个数据流或文件相关联。 每个 -filter_complex 实例都会创建一个新的复杂过滤图，而且数量不限。

复杂滤波器图的一个简单的例子是 `overlay` 滤波器，它有两个视频输入和一个视频输出，其中一个视频叠加在另一个视频之上。 与之对应的音频滤波器是 `amix` 滤波器。

### 3.4 Loopback decoders

通常解码器与解复用流（demuxer streams）相关联，但也可以创建“回环”（loopback）解码器，这种解码器可以解码某些编码器的输出，并允许将其反馈给复杂的滤波图（filtergraphs）。这是通过使用带有参数的 `-dec` 指令来完成的，该参数是应该被解码的输出流的索引。每个这样的指令都会创建一个新的回环解码器，这些解码器按照连续的整数进行索引，从零开始。然后应该在复杂的滤波图链接标签中使用这些索引来引用回环解码器，如 -filter_complex 的文档中所述。

可以在 `-dec` 之前放置解码 AVOptions 来传递给回环解码器，这类似于输入/输出选项的处理方式。

例如，以下示例：

```shell
ffmpeg -i INPUT                                        \
  -map 0:v:0 -c:v libx264 -crf 45 -f null -            \
  -threads 3 -dec 0:0                                  \
  -filter_complex '[0:v][dec:0]hstack[stack]'          \
  -map '[stack]' -c:v ffv1 OUTPUT
```

读取一个输入视频并

- （第2行）使用 `libx264` 以低质量对其进行编码；
- （第3行）使用3个线程解码这个已编码的流；
- （第4行）将解码后的视频与原始输入视频并排放置；
- （第5行）然后将合并的视频无损编码并写入OUTPUT。

这样的转码管道可以用以下图表表示：

```
┌──────────┬───────────────┐
│ demuxer  │               │   ┌─────────┐            ┌─────────┐    ┌────────────────────┐
╞══════════╡ video stream  │   │  video  │            │  video  │    │ null muxer         │
│   INPUT  │               ├──►│ decoder ├──┬────────►│ encoder ├─┬─►│(discards its input)│
│          │               │   └─────────┘  │         │(libx264)│ │  └────────────────────┘
└──────────┴───────────────┘                │         └─────────┘ │
                                 ╭───────◄──╯   ┌─────────┐       │
                                 │              │loopback │       │
                                 │ ╭─────◄──────┤ decoder ├────◄──╯
                                 │ │            └─────────┘
                                 │ │
                                 │ │
                                 │ │  ┌───────────────────┐
                                 │ │  │complex filtergraph│
                                 │ │  ╞═══════════════════╡
                                 │ │  │  ┌─────────────┐  │
                                 ╰─╫─►├─►│   hstack    ├─►├╮
                                   ╰─►├─►│             │  ││
                                      │  └─────────────┘  ││
                                      └───────────────────┘│
                                                           │
┌──────────┬───────────────┐  ┌─────────┐                  │
│ muxer    │               │  │  video  │                  │
╞══════════╡ video stream  │◄─┤ encoder ├───────◄──────────╯
│  OUTPUT  │               │  │ (ffv1)  │
│          │               │  └─────────┘
└──────────┴───────────────┘
```

## 4 Stream selection

`ffmpeg` 提供了 `-map` 选项，用于手动控制每个输出文件中的流选择。用户可以跳过 `-map`，让 FFmpeg 根据以下描述执行自动流选择。`-vn / -an / -sn / -dn` 选项可以分别用来跳过包含视频、音频、字幕和数据流，无论是手动映射还是自动选择的流，但不包括那些来自复杂滤波图（filtergraphs）的输出流。

### 4.1 Description

接下来的子章节描述了流选择过程中涉及的各种规则。随后的例子展示了这些规则在实际应用中的情况。

尽管我们会尽力准确反映程序的行为，但 FFmpeg 处于持续开发中，自本文撰写以来代码可能已经发生了变化。

#### 4.1.1 Automatic stream selection

在没有为特定输出文件指定任何映射（map）选项的情况下，FFmpeg 会检查输出格式以确定可以包含哪些类型的流，即视频、音频和/或字幕。对于每种可接受的流类型，FFmpeg 将从所有输入中选择一个可用的流。

它将根据以下标准选择流：

- 对于视频，选择分辨率最高的流，
- 对于音频，选择声道数最多的流，
- 对于字幕，选择找到的第一个字幕流，但有一个注意事项：输出格式的默认字幕编码器可以是基于文本或基于图像的，只有相同类型的字幕流会被选择。
- 如果有多个相同类型的流评分相等，则选择索引最低的流。

数据或附件流不会被自动选择，只能通过使用 `-map` 选项来包含。

#### 4.1.2 Manual stream selection

当使用 `-map` 时，只有用户映射的流会被包含在该输出文件中，有一个可能的例外情况是下面描述的滤波图（filtergraph）输出。

#### 4.1.3 Complex filtergraphs

如果存在具有未标记（unlabeled）输出端（pads）的复杂滤波图输出流，它们将被添加到第一个输出文件中。如果流类型不被输出格式支持，这将导致致命错误。在没有 -map 选项的情况下，这些流的存在会导致自动跳过该类型流的自动选择。如果有 -map 选项存在，这些滤波图流将会与用户映射的流一起被包含。

具有已标记（labeled）输出端的复杂滤波图输出流必须且只能被映射一次。

#### 4.1.4 Stream handling

流处理独立于流选择，但下面描述的字幕情况是个例外。流处理是通过针对特定 *output* 文件内的流设置 `-codec` 选项来完成的。特别是，编解码选项是在流选择过程之后由 FFmpeg 应用的，因此不会影响流选择。如果未为某种流类型指定 `-codec` 选项，ffmpeg 将选择由输出文件复用器注册的默认编码器。

对于字幕存在一个例外。如果为输出文件指定了字幕编码器，那么找到的第一个字幕流（无论是文本还是图像）将被包含。FFmpeg 不会验证所指定的编码器是否能够转换选定的流，或者转换后的流是否在输出格式中是可以接受的。这通常也适用：当用户手动设置编码器时，流选择过程无法检查编码后的流是否可以被复用到输出文件中。如果不能，FFmpeg 将终止，并且所有输出文件都将无法处理。

### 4.2 Examples

以下示例说明了 ffmpeg 的流选择方法的行为、特点和限制。

这些示例假设存在以下三个输入文件。

```
input file 'A.avi'
      stream 0: video 640x360
      stream 1: audio 2 channels

input file 'B.mp4'
      stream 0: video 1920x1080
      stream 1: audio 2 channels
      stream 2: subtitles (text)
      stream 3: audio 5.1 channels
      stream 4: subtitles (text)

input file 'C.mkv'
      stream 0: video 1280x720
      stream 1: audio 2 channels
      stream 2: subtitles (image)
```

示例：automatic stream selection
```shell
ffmpeg -i A.avi -i B.mp4 out1.mkv out2.wav -map 1:a -c:a copy out3.mov
```

指定了三个输出文件，并且对于前两个输出文件，没有设置 `-map` 选项，因此 ffmpeg 将自动为这两个文件选择流。

out1.mkv 是一个 Matroska 容器文件，接受视频、音频和字幕流，所以 ffmpeg 将尝试为每种类型选择一个流。
对于视频，它将选择来自 B.mp4 的 `stream 0`，因为这是所有输入视频流中分辨率最高的。
对于音频，它将选择来自 B.mp4 的 `stream 3`，因为它拥有最多的声道数。
对于字幕，它将选择来自 B.mp4 的 `stream 2`，这是从 A.avi 和 B.mp4 中找到的第一个字幕流。
out2.wav 只接受音频流，因此只会选择来自 B.mp4 的`stream 3`。
对于 out3.mov，由于设置了 `-map` 选项，不会发生自动流选择。`-map 1:a` 选项会选择第二个输入 B.mp4 中的所有音频流。不会有其他流被包含在这个输出文件中。
对于前两个输出文件，所有包含的流都将被转码。选择的编码器将是每个输出格式注册的默认编码器，这可能与选定输入流的编解码器不匹配。

对于第三个输出文件，音频流的编解码选项被设置为 `copy`，因此不会发生解码-过滤-编码操作。选定流的数据包将从输入文件直接传递并复用到输出文件中。

示例：automatic subtitles selection

```shell
ffmpeg -i C.mkv out1.mkv -c:s dvdsub -an out2.mkv
```

尽管 out1.mkv 是一个接受字幕流的 Matroska 容器文件，但只会选择视频和音频流。由于 C.mkv 的字幕流是基于图像的，而 Matroska 复用器的默认字幕编码器是基于文本的，因此可以预期字幕的转码操作会失败，所以该字幕流不会被选择。然而，在 out2.mkv 中，命令中指定了字幕编码器，因此除了视频流之外，字幕流也会被选择。同时，存在 `-an` 选项禁用了 out2.mkv 的音频流选择。

示例：unlabeled filtergraph outputs

```shell
ffmpeg -i A.avi -i C.mkv -i B.mp4 -filter_complex "overlay" out1.mp4 out2.srt
```

这里使用 `-filter_complex` 选项设置了一个滤波图，该滤波图包含一个视频滤波器。`overlay` 滤波器需要正好两个视频输入，但没有指定任何输入，因此会使用前两个可用的视频流，即 A.avi 和 C.mkv 的视频流。滤波器的输出端没有标签，所以它被发送到第一个输出文件 out1.mp4。由于这个原因，自动选择视频流的过程被跳过，原本会从 B.mp4 中选择的视频流不再被选择。音频流则自动选择了声道数最多的流，即 B.mp4 中的`stream 3`。然而，没有任何字幕流被选择，因为 MP4 格式没有注册默认的字幕编码器，而且用户也没有指定字幕编码器。

第二个输出文件 out2.srt 只接受基于文本的字幕流。因此，尽管第一个可用的字幕流属于 C.mkv，但由于它是基于图像的，所以被跳过。选定的流是 B.mp4 中的`stream 2`，这是第一个基于文本的字幕流。

示例：labeled filtergraph outputs

```shell
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0[outv];overlay;aresample" \
       -map '[outv]' -an        out1.mp4 \
                                out2.mkv \
       -map '[outv]' -map 1:a:0 out3.mkv
```

上述命令将会失败，因为输出端标签为 `[outv]` 的 pad 被映射了两次。没有任何输出文件会被处理。

```
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0[outv];overlay;aresample" \
       -an        out1.mp4 \
                  out2.mkv \
       -map 1:a:0 out3.mkv
```

上述命令同样会失败，因为 hue 滤波器的输出有一个标签 `[outv]`，但没有被映射到任何地方。

命令应该如下修改：

```
ffmpeg -i A.avi -i B.mp4 -i C.mkv -filter_complex "[1:v]hue=s=0,split=2[outv1][outv2];overlay;aresample" \
        -map '[outv1]' -an        out1.mp4 \
                                  out2.mkv \
        -map '[outv2]' -map 1:a:0 out3.mkv
```

来自 B.mp4 的视频流被发送到 hue 滤波器，其输出使用 split 滤波器克隆一次，并且两个输出都被标记。然后，每个副本分别映射到第一个和第三个输出文件。

overlay 滤波器需要两个视频输入，它使用前两个未使用的视频流，即来自 A.avi 和 C.mkv 的流。overlay 输出没有被标记，因此它会被发送到第一个输出文件 out1.mp4，无论 `-map` 选项是否存在。

aresample 滤波器接收第一个未使用的音频流，即来自 A.avi 的音频流。由于此滤波器的输出也没有被标记，因此它同样被映射到第一个输出文件。`-an` 选项仅抑制音频流的自动或手动选择，而不影响从滤波图发送的输出。这两个映射的流将在 out1.mp4 中已映射的流之前被处理。

映射到 `out2.mkv` 的视频、音频和字幕流完全由自动流选择决定。

out3.mkv 包含来自 hue 滤波器的克隆视频输出以及来自 B.mp4 的第一个音频流。

## 5 Options

所有数值选项，除非另有说明，接受一个表示数字的字符串作为输入，该字符串后面可以跟一个 SI 单位前缀，例如：’K’、’M’ 或 ’G’。

如果在 SI 单位前缀后附加了 ’i’，则整个前缀将被解释为二进制倍数的单位前缀，这些基于 1024 的幂而不是 1000 的幂。在 SI 单位前缀后附加 ’B’ 会将值乘以 8。这允许使用例如：’KB’、’MiB’、’G’ 和 ’B’ 作为数字后缀。

不带参数的选项是布尔选项，设置相应的值为 true。可以通过在选项名称前加上 "no" 将其设置为 false。例如，使用 -nofoo 将把名为 "foo" 的布尔选项设置为 false。

支持参数的选项还提供了一种特殊语法，其中命令行上给出的参数被视为加载实际参数值的文件路径。要使用此功能，在选项名称前（在引导短横线之后）立即添加正斜杠 ’/’。例如：

```shell
ffmpeg -i INPUT -/filter:v filter.script OUTPUT
```

将从名为 filter.script 的文件中加载滤波图（filtergraph）描述。

### 5.1 Stream specifiers

一些选项是按流应用的，例如比特率或编解码器。流指定符用于精确指定给定选项属于哪个流（或哪些流）。

流指定符通常附加在选项名称之后，并用冒号与其分隔。例如，`-codec:a:1 ac3` 包含流指定符 `a:1`，它匹配第二个音频流。因此，它会选择 AC3 编解码器用于第二个音频流。

流指定符可以匹配多个流，因此选项将应用于所有匹配的流。例如，流指定符 `-b:a 128k` 匹配所有音频流。

空的流指定符匹配所有流。例如，`-codec copy` 或 `-codec: copy` 将复制所有流而不重新编码。

流指定符的可能形式包括：

*stream_index*

    匹配具有此索引的流。例如，-threads:1 4 将第二个流的线程数设置为 4。如果 stream_index 用作附加的流指定符（见下文），那么它会选择匹配流中的第 stream_index 个流。流编号基于 libavformat 检测到的流顺序，除非还指定了流组指定符或节目 ID。在这种情况下，它基于组或节目中的流顺序。

*stream_type[:additional_stream_specifier]*
    stream_type 是以下之一：’v’ 或 ’V’ 表示视频，’a’ 表示音频，’s’ 表示字幕，’d’ 表示数据，以及 ’t’ 表示附件。’v’ 匹配所有视频流，而 ’V’ 仅匹配不是附加图片、视频缩略图或封面艺术的视频流。如果使用了 additional_stream_specifier，则匹配既具有该类型又符合 additional_stream_specifier 的流。否则，它匹配所有指定类型的流。

*g:group_specifier[:additional_stream_specifier]*

    匹配在 group_specifier 指定的组中的流。如果使用了 additional_stream_specifier，则匹配既属于该组又符合 additional_stream_specifier 的流。group_specifier 可以是以下之一：

    *group_index*

        匹配具有此组索引的流。

    *#group_id 或 i:group_id*

        匹配具有此组 ID 的流。

*p:program_id[:additional_stream_specifier]*

    匹配在 program_id 指定的节目中的流。如果使用了 additional_stream_specifier，则匹配既属于该节目又符合 additional_stream_specifier 的流。

*#stream_id 或 i:stream_id*

    通过流 ID（例如 MPEG-TS 容器中的 PID）匹配流。

*m:key[:value]*

    匹配包含元数据标签 key 并具有指定值的流。如果没有给出 value，则匹配包含给定标签且具有任何值的流。key 或 value 中的冒号字符 ’:’ 需要用反斜杠转义。

*disp:dispositions[:additional_stream_specifier]*

    匹配具有给定属性（disposition）的流。dispositions 是一个由一个或多个属性（如 -dispositions 选项所打印）用 ’+’ 连接的列表。

*u*

    匹配配置可用的流，编解码器必须已定义，并且必须存在诸如视频尺寸或音频采样率等必要信息。
    注意，在 `ffmpeg` 中，基于元数据的匹配仅对输入文件有效。

### 5.2 Generic options

这些选项在 ff* 工具之间共享。

-L

    显示许可证。

-h, -?, -help, --help [arg]

    显示帮助。可以指定一个可选参数来打印关于特定项目的帮助信息。如果没有指定参数，则只显示基本（非高级）工具选项。

    可能的 arg 值包括：

    long

        除了基本工具选项外，还打印高级工具选项。

    full

        打印完整的选项列表，包括编解码器、解复用器、复用器、过滤器等的共享和私有选项。

    decoder=*decoder_name*

        打印名为 *decoder_name* 的解码器的详细信息。使用 -decoders 选项获取所有解码器的列表。

    encoder=*encoder_name*

        打印名为 *encoder_name* 的编码器的详细信息。使用 -encoders 选项获取所有编码器的列表。

    demuxer=*demuxer_name*

        打印名为 *demuxer_name* 的解复用器的详细信息。使用 -formats 选项获取所有解复用器和复用器的列表。

    muxer=*muxer_name*

        打印名为 *muxer_name* 的复用器的详细信息。使用 -formats 选项获取所有复用器和解复用器的列表。

    filter=*filter_name*

        打印名为 *filter_name* 的过滤器的详细信息。使用 -filters 选项获取所有过滤器的列表。

    bsf=*bitstream_filter_name*

        打印名为 *bitstream_filter_name* 的比特流过滤器的详细信息。使用 -bsfs 选项获取所有比特流过滤器的列表。

    protocol=*protocol_name*

        打印名为 *protocol_name* 的协议的详细信息。使用 -protocols 选项获取所有协议的列表。

-version

    显示版本。

-buildconf

    逐行显示构建配置。

-formats

    显示可用格式（包括设备）。

-demuxers

    显示可用解复用器。

-muxers

    显示可用复用器。

-devices

    显示可用设备。

-codecs

    显示 libavcodec 知道的所有编解码器。

    注意，文档中使用的术语“编解码器”是媒体比特流格式的缩写。

-decoders

    显示可用解码器。

-encoders

    显示所有可用编码器。

-bsfs

    显示可用比特流过滤器。

-protocols

    显示可用协议。

-filters

    显示可用的 libavfilter 过滤器。

-pix_fmts

    显示可用像素格式。

-sample_fmts

    显示可用采样格式。

-layouts

    显示通道名称和标准通道布局。

-dispositions

    显示流属性。

-colors

    显示识别的颜色名称。

-sources *device[,opt1=val1[,opt2=val2]...]*

    显示输入设备的自动检测源。某些设备可能提供系统依赖的源名称，这些名称无法自动检测。返回的列表不能总是被认为是完整的。

```shell
    ffmpeg -sources pulse,server=192.168.0.4
```

-sinks *device[,opt1=val1[,opt2=val2]...]*

    显示输出设备的自动检测接收端。某些设备可能提供系统依赖的接收端名称，这些名称无法自动检测。返回的列表不能总是被认为是完整的。

```shell
ffmpeg -sinks pulse,server=192.168.0.4
```

-loglevel *[flags+]loglevel | -v [flags+]loglevel*

    设置库使用的日志记录级别和标志。

    可选的 flags 前缀可以包含以下值：

    ‘repeat’

        表示应重复的日志输出不应被压缩到第一行，并且将省略“上次消息重复 n 次”的行。

    ‘level’

        表示日志输出应在每条消息行添加 `[level]` 前缀。这可以用作日志着色的替代方案，例如在将日志转储到文件时。

    也可以通过添加 ’+’/’-’ 前缀来单独设置或重置单个标志而不影响其他标志或更改日志级别。当同时设置标志和日志级别时，期望在最后一个标志值之后和 *loglevel* 之前使用 ’+’ 分隔符。

    *loglevel* 是一个字符串或包含以下值之一的数字：

    ‘quiet, -8’

        完全不显示任何内容；保持静默。

    ‘panic, 0’

        仅显示可能导致进程崩溃的致命错误，例如断言失败。这目前没有用于任何东西。

    ‘fatal, 8’

        仅显示致命错误。这些是在发生后进程绝对无法继续的错误。

    ‘error, 16’

        显示所有错误，包括可以恢复的错误。

    ‘warning, 24’

        显示所有警告和错误。任何与可能不正确或意外事件相关的消息都会显示。

    ‘info, 32’

        在处理期间显示信息性消息。这是除警告和错误之外的信息。这是默认值。

    ‘verbose, 40’

        与 `info` 相同，只是更加详细。

    ‘debug, 48’

        显示所有内容，包括调试信息。

    ‘trace, 56’

    例如，要启用重复的日志输出，添加`level`前缀，并将 *loglevel* 设置为 `verbose`：

```shell
ffmpeg -loglevel repeat+level+verbose -i input output
```

    另一个例子是仅启用重复的日志输出而不影响当前级别的前缀标志或日志级别：

```shell
ffmpeg [...] -loglevel +repeat
```

    默认情况下，程序会记录到 stderr。如果终端支持颜色，则颜色会被用来标记错误和警告。可以通过设置环境变量 `AV_LOG_FORCE_NOCOLOR` 来禁用日志颜色，或者通过设置环境变量 `AV_LOG_FORCE_COLOR` 来强制使用日志颜色。

-report

    将完整的命令行和日志输出转储到当前目录中的名为 `*program-YYYYMMDD-HHMMSS*.log` 的文件中。此文件可用于错误报告。它也意味着 `-loglevel debug`。

    将环境变量 `FFREPORT` 设置为任何值都有相同的效果。如果值是一个以 ’:’ 分隔的 key=value 序列，这些选项将影响报告；选项值必须在包含特殊字符或选项分隔符 ’:’ 时进行转义（参见 ffmpeg-utils 手册中的“引用和转义”部分）。

    识别以下选项：

    file

        设置报告使用的文件名；`%p` 展开为程序名，`%t` 展开为时间戳，`%%` 展开为普通 `%`。

    level

        使用数值设置日志详细程度级别（参见 `-loglevel`）。

    例如，要将报告输出到名为 ffreport.log 的文件并使用 `32` 日志级别（即 `info` 级别）：

```shell
FFREPORT=file=ffreport.log:level=32 ffmpeg -i input output
```

    解析环境变量中的错误不会出现在报告中。

-hide_banner

    抑制打印横幅。

    所有 FFmpeg 工具通常会显示版权声明、构建选项和库版本。可以使用此选项抑制打印此信息。

-cpuflags flags (*global*)

    允许设置和清除 CPU 标志。此选项用于测试目的。除非你明白你在做什么，否则不要使用它。

```shell
ffmpeg -cpuflags -sse+mmx ...
ffmpeg -cpuflags mmx ...
ffmpeg -cpuflags 0 ...
```

    可能的标志包括：

    ‘x86’

        ‘mmx’
        ‘mmxext’
        ‘sse’
        ‘sse2’
        ‘sse2slow’
        ‘sse3’
        ‘sse3slow’
        ‘ssse3’
        ‘atom’
        ‘sse4.1’
        ‘sse4.2’
        ‘avx’
        ‘avx2’
        ‘xop’
        ‘fma3’
        ‘fma4’
        ‘3dnow’
        ‘3dnowext’
        ‘bmi1’
        ‘bmi2’
        ‘cmov’

    ‘ARM’

        ‘armv5te’
        ‘armv6’
        ‘armv6t2’
        ‘vfp’
        ‘vfpv3’
        ‘neon’
        ‘setend’

    ‘AArch64’

        ‘armv8’
        ‘vfp’
        ‘neon’

    ‘PowerPC’

        ‘altivec’

    ‘Specific Processors’
        ‘pentium2’
        ‘pentium3’
        ‘pentium4’
        ‘k6’
        ‘k62’
        ‘athlon’
        ‘athlonxp’
        ‘k8’

-cpucount *count* (*global*)

    覆盖 CPU 数量的检测。此选项用于测试目的。除非你明白你在做什么，否则不要使用它。
```shell
ffmpeg -cpucount 2
```

-max_alloc bytes

    设置 ffmpeg 的 malloc 函数族在堆上分配块的最大大小限制。使用此选项时需极度小心。如果你不了解这样做带来的全部后果，请勿使用。默认值为 INT_MAX。

### 5.3 AVOptions

这些选项是由 libavformat、libavdevice 和 libavcodec 库直接提供的。要查看可用的 AVOptions 列表，可以使用 -help 选项。它们被分为两类：

generic

    这些选项可以为任何容器、编解码器或设备设置。通用选项在容器/设备下列为 AVFormatContext 选项，在编解码器下列为 AVCodecContext 选项。

private

    这些选项特定于给定的容器、设备或编解码器。私有选项在其对应的容器/设备/编解码器下列出。

例如，要写入 ID3v2.3 标头而不是默认的 ID3v2.4 到 MP3 文件中，可以使用 MP3 复用器的 id3v2_version 私有选项：

```shell
ffmpeg -i input.flac -id3v2_version 3 out.mp3
```

所有编解码器的 AVOptions 都是按流设置的，因此应该附带一个流指定符。

```shell
ffmpeg -i multichannel.mxf -map 0:v:0 -map 0:a:0 -map 0:a:0 -c:a:0 ac3 -b:a:0 640k -ac:a:1 2 -c:a:1 aac -b:2 128k out.mp4
```

在上述例子中，一个多声道音频流被映射了两次用于输出。第一次实例使用 ac3 编解码器和 640k 的比特率进行编码。第二次实例被下混（downmixed）到 2 个声道，并使用 aac 编解码器进行编码。通过指定输出流的绝对索引来为其设置 128k 的比特率。

注意：对于布尔类型的 AVOptions，不能使用 -nooption 语法；应使用 -option 0 或 -option 1 来设置布尔值。

注意：旧的、未文档化的方法，即通过在选项名称前添加 v/a/s 来指定每个流的 AVOptions 现已过时，不久将被移除。

### 5.4 Main options

-f *fmt* (*input/output*)

    强制输入或输出文件格式。通常情况下，输入文件的格式会自动检测，而输出文件的格式则根据文件扩展名猜测，所以在大多数情况下不需要使用此选项。

-i *url*(*input*)

    指定输入文件的 URL。

-y (*global*)

    不询问直接覆盖输出文件。

-n (*global*)

    如果指定的输出文件已经存在，则不覆盖并立即退出。

-stream_loop *number* (*input*)

    设置输入流循环播放的次数。0 表示不循环，-1 表示无限循环。

-recast_media (*global*)

    允许强制使用与解复用器检测或指定的不同媒体类型的解码器。对于作为数据流复用的媒体数据解码很有用。

-c[:stream_specifier] codec (*input/output,per-stream*)
-codec[:stream_specifier] codec (*input/output,per-stream*)

    为一个或多个流选择解码器（在输入文件前使用）或编码器（在输出文件前使用）。*codec* 是解码器/编码器的名字，或者是一个特殊的值 `copy`（仅限输出），表示该流不会被重新编码。

    例如
```shell
ffmpeg -i INPUT -map 0 -c:v libx264 -c:a copy OUTPUT
```

    将所有视频流使用 libx264 编码，并复制所有音频流。

    对于每个流，最后匹配的 `c` 选项会被应用，所以

```shell
ffmpeg -i INPUT -map 0 -c copy -c:v:1 libx264 -c:a:137 libvorbis OUTPUT
```

    除了第二个视频流将使用 libx264 编码和第 138 个音频流将使用 libvorbis 编码外，其他所有流都将被复制。

-t *duration* (*input/output*)

    当作为输入选项使用时（在 `-i` 之前），限制从输入文件读取的数据持续时间。

    当作为输出选项使用时（在输出 URL 之前），在持续时间达到 duration 后停止写入输出。

    *duration* 必须是时间持续时间规格，参见 ffmpeg-utils(1) 手册中的“时间持续时间”部分。

    -to 和 -t 互斥，且 -t 优先。

-to *position* (*input/output*)

    在位置处停止写入输出或读取输入。position 必须是时间持续时间规格，参见 ffmpeg-utils(1) 手册中的“时间持续时间”部分。

    -to 和 -t 互斥，且 -t 优先。

-fs *limit_size* (*output*)

    设置文件大小限制，以字节表示。超过限制后不再写入任何字节块。输出文件的实际大小可能略大于请求的文件大小。

-ss *position* (*input/output*)

    当作为输入选项使用时（在 `-i` 之前），在此输入文件中跳转到 position。请注意，在大多数格式中无法精确跳转，因此 `ffmpeg` 将跳转到最接近 position 的跳转点之前。当进行转码且启用了 -accurate_seek（默认启用）时，这个额外段落将在 position 和跳转点之间的部分被解码并丢弃。当执行流复制或使用了 -noaccurate_seek 时，它将被保留。

    当作为输出选项使用时（在输出 URL 之前），解码但丢弃直到时间戳到达 *position* 的输入。

    *position* 必须是时间持续时间规格，参见 ffmpeg-utils(1) 手册中的“时间持续时间”部分。

-sseof *position* (*input*)

    类似于 `-ss` 选项，但相对于文件的“末尾”。即负值表示文件更早的位置，0 表示文件末尾（EOF）。

-isync *input_index* (*input*)
    
    将一个输入指定为同步源。

    这会取目标和参考输入开始时间之间的差值，并根据该差值调整目标文件的时间戳。为了获得预期的结果，两个输入的时间戳应源自相同的时钟源。如果设置了 `copyts`，则必须同时设置 `start_at_zero`。如果任一输入没有起始时间戳，则不会进行同步调整。

    可接受的值是指向有效 ffmpeg 输入索引的值。如果同步参考是目标索引本身或 -1，则不对目标时间戳进行调整。同步参考不能被同步到任何其他输入。

    默认值为 -1。

-itsoffset *offset* (*input*)

    设置输入时间偏移。

    *offset* 必须是一个时间持续时间规格，参见 ffmpeg-utils(1) 手册中的“时间持续时间”部分。

    此偏移量会被添加到输入文件的时间戳上。指定正偏移量意味着相应的流将被延迟指定在 *offset* 中的时间长度。

-itsscale *scale* (*input,per-stream*)

    重新缩放输入时间戳。*scale* 应该是一个浮点数。

-timestamp *date* (输出)

    在容器中设置录制时间戳。

    *date* 必须是一个日期规格，参见 ffmpeg-utils(1) 手册中的“日期”部分。

-metadata[:metadata_specifier] *key=value* (output,per-metadata)

    设置一个元数据键/值对。

    可以提供一个可选的 *metadata_specifier* 来在流、章节或节目上设置元数据。详情请参阅 `-map_metadata` 文档。

    此选项会覆盖使用 `-map_metadata` 设置的元数据。也可以通过使用空值来删除元数据。

    例如，对于设置输出文件的标题：

```shell
ffmpeg -i in.avi -metadata title="my title" out.flv
```

要设置第一个音频流的语言：

```shell
ffmpeg -i INPUT -metadata:s:a:0 language=eng OUTPUT
```

-disposition[:stream_specifier] *value* (*output,per-stream*)

    为一个流设置属性标志。

    默认值：默认情况下，所有属性标志都会从输入流复制过来，除非该选项适用的输出流是由复杂滤波图供给的——在这种情况下，默认不会设置任何属性标志。

    *value* 是由 ‘+’ 或 ‘-’ 分隔的属性标志序列。‘+’ 前缀表示添加给定的属性，‘-’ 表示移除它。如果第一个标志也带有 ‘+’ 或 ‘-’ 前缀，则最终的属性是默认值更新为 value 的结果。如果第一个标志没有前缀，则最终的属性就是 value。也可以通过将其设置为 0 来清除属性。

    如果没有为输出文件指定 `-disposition` 选项，当输出文件中有多个此类类型的流且没有标记为默认的流时，ffmpeg 将自动为每种类型的第一个流设置 ‘default’ 属性标志。

    `-dispositions` 选项列出了已知的属性标志。

    例如，要使第二个音频流成为默认流：

```shell
ffmpeg -i in.mkv -c copy -disposition:a:1 default out.mkv
```

    要使第二个字幕流成为默认流并移除第一个字幕流的默认属性：

```shell
ffmpeg -i in.mkv -c copy -disposition:s:0 0 -disposition:s:1 default out.mkv
```

    要添加嵌入封面/缩略图：

```shell
ffmpeg -i in.mp4 -i IMAGE -map 0 -map 1 -c copy -c:v:1 png -disposition:v:1 attached_pic out.mp4
```

    要为第一个音频流添加 ‘original’ 属性并移除 ‘comment’ 属性而不移除其其他属性：

```shell
ffmpeg -i in.mkv -c copy -disposition:a:0 +original-comment out.mkv
```

    要从第一个音频流移除 ‘original’ 属性并添加 ‘comment’ 属性而不移除其其他属性：

```shell
ffmpeg -i in.mkv -c copy -disposition:a:0 -original+comment out.mkv
```

    只为第一个音频流设置 ‘original’ 和 ‘comment’ 属性（并移除其他属性）：

```shell
ffmpeg -i in.mkv -c copy -disposition:a:0 original+comment out.mkv
```

    移除第一个音频流的所有属性：

```shell
ffmpeg -i in.mkv -c copy -disposition:a:0 0 out.mkv
```

    并非所有的复用器都支持嵌入式缩略图，并且那些支持的复用器只支持几种格式，比如 JPEG 或 PNG。

-program [title=*title*:][program_num=*program_num*:]st=*stream*[:st=*stream*...] (*output*)

    创建一个具有指定标题（*title*）、程序编号（*program_num*）的节目，并将指定的*stream(s)*添加到其中。

-stream_group [map=*input_file_id*=*stream_group*][type=*type*:]st=*stream*[:st=*stream*][:stg=*stream_group*][:id=*stream_group_id*...] (*output*)

    创建一个指定类型和流组ID（*stream_group_id*）的流组，或通过映射输入组创建，将指定的*stream(s)*和/或先前定义的*stream_group(s)*添加到其中。

    *type* 可以是以下之一：

    iamf_audio_element

        将属于同一 IAMF 音频元素的流分组

        对于这种类型的组，有以下可用选项：

        audio_element_type

            音频元素类型。支持以下值：

            channel

                可扩展声道音频表示

            scene

                Ambisonics 表示

        demixing

            用于重建可扩展声道音频表示的解混信息。此选项必须用 ‘,’ 与其他部分分开，并采用 key=value 的格式

            parameter_id

                帧中的参数块可能引用的标识符

            dmixp_mode

                预定义的解混参数组合

        recon_gain

            用于重建可扩展声道音频表示的重构造增益信息。此选项必须用 ‘,’ 与其他部分分开，并采用 key=value 的格式

            parameter_id

                帧中的参数块可能引用的标识符

        layer

            定义音频元素中通道布局的层。此选项必须用 ‘,’ 与其他部分分开。可以定义多个 ‘,’ 分隔的条目，且至少需要设置一个。

            它采用以下 “:” 分隔的 key=value 格式

            ch_layout

                层的通道布局

            flags

                可用的标志如下：

                recon_gain

                    是否在帧内的参数块中标记 recon_gain 是否存在作为元数据

            output_gain
            output_gain_flags

                output_gain 应用于哪些通道。可用的标志如下：

                FL
                FR
                BL
                BR
                TFL
                TFR

            ambisonics_mode

                如果设置了 audio_element_type 为 channel，则该选项无效。

                支持以下值：

                mono

                    每个 Ambisonics 通道被编码为组中的独立单声道流

        default_w

            默认权重值

    iamf_mix_presentation

        将属于同一 IAMF 混合演示的所有 IAMF 音频元素的流分组

        对于这种类型的组，有以下可用选项：

        submix

            混合演示中的子混音。此选项必须用 ‘,’ 与其他部分分开。可以定义多个 ‘,’ 分隔的条目，且至少需要设置一个。

            它采用以下 “:” 分隔的 key=value 格式

            parameter_id

                帧中的参数块可能引用的标识符，用于后处理混合后的音频信号以生成播放用的音频信号

            parameter_rate

                帧中引用此 parameter_id 的参数块的采样率持续时间字段的表达方式

            default_mix_gain

                当给定帧没有共享相同 parameter_id 的参数块时应用的默认混音增益值

            element

                引用在此混合演示中使用的音频元素以生成最终输出音频信号用于播放。此选项必须用 ‘|’ 与其他部分分开。可以定义多个 ‘|’ 分隔的条目，且至少需要设置一个。

                它采用以下 “:” 分隔的 key=value 格式：

                stg

                    此子混音引用的音频元素的 *stream_group_id*

                parameter_id

                    帧中的参数块可能引用的标识符，用于对引用和渲染的音频元素进行任何处理，然后再与其他处理过的音频元素相加

                parameter_rate

                    帧中引用此 *parameter_id* 的参数块的采样率持续时间字段的表达方式

                default_mix_gain

                    当给定帧没有共享相同 *parameter_id* 的参数块时应用的默认混音增益值

                annotations

                    描述子混音元素的关键字 = 值字符串，其中 "key" 是符合 BCP-47 规范的字符串，用于指定 "value" 字符串的语言。"key" 必须与混音的*annotations*中的键相同

                headphones_rendering_mode

                    指示当在耳机上播放时，基于输入通道的音频元素是渲染为立体声扬声器还是使用双耳渲染器进行空间化。如果引用的音频元素的 *audio_element_type* 设置为 channel，则该选项无效。

                    支持以下值：

                    stereo
                    binaural

            layout

                指定测量响度信息的此子混音的布局。此选项必须用 ‘|’ 与其他部分分开。可以定义多个 ‘|’ 分隔的条目，且至少需要设置一个。

                它采用以下 “:” 分隔的 key=value 格式：

                layout_type

                    loudspeakers

                        布局遵循 ITU-2051-3 的扬声器音响系统规范。

                    binaural

                        布局是双耳的。

                sound_system

                    匹配 ITU-2051-3 中的 A 到 J 音响系统之一，加上 7.1.2 和 3.1.2。如果 layout_type 设置为 binaural，则此选项无效。

                integrated_loudness

                    程序集成响度信息，如 ITU-1770-4 所定义。

                digital_peak

                    数字（采样）峰值，如 ITU-1770-4 所定义。

                true_peak

                    真实峰值，如 ITU-1770-4 所定义。

                dialog_anchored_loudness

                    对话响度信息，如 ITU-1770-4 所定义。

                album_anchored_loudness

                    专辑响度信息，如 ITU-1770-4 所定义。

        annotations

            描述混音的关键字 = 值字符串，其中 "key" 是符合 BCP-47 规范的字符串，用于指定 "value" 字符串的语言。"key" 必须与所有子混音元素注释中的键相同

    例如，要从几个 WAV 输入文件创建一个可扩展的 5.1 IAMF 文件：

```shell
ffmpeg -i front.wav -i back.wav -i center.wav -i lfe.wav
-map 0:0 -map 1:0 -map 2:0 -map 3:0 -c:a opus
-stream_group type=iamf_audio_element:id=1:st=0:st=1:st=2:st=3,
demixing=parameter_id=998,
recon_gain=parameter_id=101,
layer=ch_layout=stereo,
layer=ch_layout=5.1,
-stream_group type=iamf_mix_presentation:id=2:stg=0:annotations=en-us=Mix_Presentation,
submix=parameter_id=100:parameter_rate=48000|element=stg=0:parameter_id=100:annotations=en-us=Scalable_Submix|layout=sound_system=stereo|layout=sound_system=5.1
-streamid 0:0 -streamid 1:1 -streamid 2:2 -streamid 3:3 output.iamf
```

    要将两个流组（音频元素和混合演示）从具有四个流的输入 IAMF 文件复制到 mp4 输出：

```shell
ffmpeg -i input.iamf -c:a copy -stream_group map=0=0:st=0:st=1:st=2:st=3 -stream_group map=0=1:stg=0
-streamid 0:0 -streamid 1:1 -streamid 2:2 -streamid 3:3 output.mp4
```

-target *type* (*output*)

    指定目标文件类型（`vcd`, `svcd`, `dvd`, `dv`, `dv50`）。可以在类型前加上 `pal-`、`ntsc-` 或 `film-` 前缀以使用相应的标准。所有格式选项（比特率、编解码器、缓冲区大小）将自动设置。您可以直接输入以下命令：

```shell
ffmpeg -i myfile.avi -target vcd /tmp/vcd.mpg
```

    不过，只要您确保这些选项不会与标准冲突，也可以指定额外的选项，例如：

```shell
ffmpeg -i myfile.avi -target vcd -bf 2 /tmp/vcd.mpg
```

    为每个目标设置的参数如下。

    VCD

```
pal:
-f vcd -muxrate 1411200 -muxpreload 0.44 -packetsize 2324
-s 352x288 -r 25
-codec:v mpeg1video -g 15 -b:v 1150k -maxrate:v 1150k -minrate:v 1150k -bufsize:v 327680
-ar 44100 -ac 2
-codec:a mp2 -b:a 224k

ntsc:
-f vcd -muxrate 1411200 -muxpreload 0.44 -packetsize 2324
-s 352x240 -r 30000/1001
-codec:v mpeg1video -g 18 -b:v 1150k -maxrate:v 1150k -minrate:v 1150k -bufsize:v 327680
-ar 44100 -ac 2
-codec:a mp2 -b:a 224k

film:
-f vcd -muxrate 1411200 -muxpreload 0.44 -packetsize 2324
-s 352x240 -r 24000/1001
-codec:v mpeg1video -g 18 -b:v 1150k -maxrate:v 1150k -minrate:v 1150k -bufsize:v 327680
-ar 44100 -ac 2
-codec:a mp2 -b:a 224k
```

    SVCD

```
pal:
-f svcd -packetsize 2324
-s 480x576 -pix_fmt yuv420p -r 25
-codec:v mpeg2video -g 15 -b:v 2040k -maxrate:v 2516k -minrate:v 0 -bufsize:v 1835008 -scan_offset 1
-ar 44100
-codec:a mp2 -b:a 224k

ntsc:
-f svcd -packetsize 2324
-s 480x480 -pix_fmt yuv420p -r 30000/1001
-codec:v mpeg2video -g 18 -b:v 2040k -maxrate:v 2516k -minrate:v 0 -bufsize:v 1835008 -scan_offset 1
-ar 44100
-codec:a mp2 -b:a 224k

film:
-f svcd -packetsize 2324
-s 480x480 -pix_fmt yuv420p -r 24000/1001
-codec:v mpeg2video -g 18 -b:v 2040k -maxrate:v 2516k -minrate:v 0 -bufsize:v 1835008 -scan_offset 1
-ar 44100
-codec:a mp2 -b:a 224k
```

    DVD

```
pal:
-f dvd -muxrate 10080k -packetsize 2048
-s 720x576 -pix_fmt yuv420p -r 25
-codec:v mpeg2video -g 15 -b:v 6000k -maxrate:v 9000k -minrate:v 0 -bufsize:v 1835008
-ar 48000
-codec:a ac3 -b:a 448k

ntsc:
-f dvd -muxrate 10080k -packetsize 2048
-s 720x480 -pix_fmt yuv420p -r 30000/1001
-codec:v mpeg2video -g 18 -b:v 6000k -maxrate:v 9000k -minrate:v 0 -bufsize:v 1835008
-ar 48000
-codec:a ac3 -b:a 448k

film:
-f dvd -muxrate 10080k -packetsize 2048
-s 720x480 -pix_fmt yuv420p -r 24000/1001
-codec:v mpeg2video -g 18 -b:v 6000k -maxrate:v 9000k -minrate:v 0 -bufsize:v 1835008
-ar 48000
-codec:a ac3 -b:a 448k
```

    DV

```
pal:
-f dv
-s 720x576 -pix_fmt yuv420p -r 25
-ar 48000 -ac 2

ntsc:
-f dv
-s 720x480 -pix_fmt yuv411p -r 30000/1001
-ar 48000 -ac 2

film:
-f dv
-s 720x480 -pix_fmt yuv411p -r 24000/1001
-ar 48000 -ac 2
```

    `dv50` 目标与 `dv` 目标相同，唯一的区别是为所有三种标准（PAL、NTSC、FILM）设置的像素格式为 `yuv422p`。

    用户为上述任何参数设置的值都将覆盖目标预设值。在这种情况下，输出可能不符合目标标准。

-dn (*input/output*)

    作为输入选项时，阻止文件中的所有数据流被过滤或自动选择或映射到任何输出。参见 `-discard` 选项以单独禁用流。

    作为输出选项时，禁用数据记录，即自动选择或映射任何数据流。对于完全的手动控制，请参阅 `-map` 选项。

-dframes *number* (*output*)

    设置要输出的数据帧数量。这是 `-frames:d` 的过时别名，建议使用后者代替。

-frames[:*stream_specifier*] *framecount* (*output,per-stream*)

    在写入流后停止，当达到 framecount 帧数时。

-q[:*stream_specifier*] *q* (*output,per-stream*)
-qscale[:*stream_specifier*] *q* (*output,per-stream*)

    使用固定的质量比例（VBR）。*q/qscale* 的含义取决于编解码器。如果在没有 *stream_specifier* 的情况下使用 *qscale*，则它仅适用于视频流，这是为了保持与以前行为的兼容性，并且通常不是在没有 stream_specifier 的情况下使用时所期望的结果，因为不打算对音频和视频两个不同编解码器指定相同的编解码器特定值。

-filter[:*stream_specifier*] *filtergraph* (*output,per-stream*)

    创建由 *filtergraph* 描述的滤镜图并用它来过滤流。

    *filtergraph* 是应用于流的滤镜图的描述，必须有一个与流类型相同的单一输入和单一输出。在滤镜图中，输入关联到标签 `in`，输出关联到标签 `out`。有关滤镜图语法的更多信息，请参阅 ffmpeg-filters 手册。

    如果您想创建具有多个输入和/或输出的滤镜图，请参阅 -filter_complex 选项。

-reinit_filter[:*stream_specifier*] *integer* (*input,per-stream*)

    这个布尔选项决定当输入帧参数中途改变时，是否重新初始化与此流关联的滤镜图。此选项默认启用，因为大多数视频和所有音频滤镜无法处理输入帧属性的变化。重新初始化时，现有的滤镜状态将丢失，例如一些滤镜中可用的帧计数 `n` 引用。重新初始化时缓冲的任何帧也将丢失。对于视频，触发重新初始化的属性包括帧分辨率或像素格式；对于音频，包括样本格式、采样率、声道数量或声道布局。

-filter_threads *nb_threads* (*global*)

    定义用于处理滤镜管道的线程数。每个管道将生成一个线程池，其中包含此数量的线程用于并行处理。默认值是可用的 CPU 数量。

-pre[:*stream_specifier*] *preset_name* (*output,per-stream*)

    为匹配的流指定预设名称。

-stats (*global*)

    以 "info" 级别的日志记录编码进度/统计信息（参见 `-loglevel`）。默认是开启的，要显式禁用它需要指定 `-nostats`。

-stats_period time (*global*)

    设置更新编码进度/统计信息的时间间隔。默认是 0.5 秒。

-progress *url* (*global*)

    将程序友好的进度信息发送到指定的 URL。

    进度信息会在编码过程中定期以及结束时写入。它由 "*key=value*" 格式的行组成，其中 *key* 仅包含字母数字字符。每组进度信息的最后一项 key 总是 "progress"，其值为 "continue" 或 "end"。

    更新周期可以通过 `-stats_period` 设置。

    例如，将进度信息记录到标准输出：

```shell
ffmpeg -progress pipe:1 -i in.mkv out.mkv
```

-stdin

    启用标准输入上的交互。默认情况下启用，除非标准输入用作输入。要显式禁用交互，需要指定 `-nostdin`。

    在 ffmpeg 处于后台进程组时，禁用标准输入上的交互是有用的。大约相同的效果可以通过 `ffmpeg ... < /dev/null` 实现，但这需要一个 shell。

-debug_ts (*global*)

    打印时间戳/延迟信息。默认情况下关闭。此选项主要用于测试和调试目的，且输出格式可能会从一个版本到另一个版本发生变化，因此不应被便携脚本使用。

    另见选项 `-fdebug ts`。

-attach *filename* (*output*)

    向输出文件添加附件。这支持一些格式，如 Matroska，用于例如字幕渲染中使用的字体。附件实现为特定类型的流，因此这个选项会向文件添加一个新的流。然后可以像平常一样在这个流上使用每个流的选项。通过此选项创建的附件流将在所有其他流（即通过 `-map` 或自动映射创建的流）之后创建。

    注意：对于 Matroska，您还需要设置 mimetype 元数据标签：

```shell
ffmpeg -i INPUT -attach DejaVuSans.ttf -metadata:s:2 mimetype=application/x-truetype-font out.mkv
```

（假设附件流将是输出文件中的第三个流）。

-dump_attachment[:*stream_specifier*] *filename* (*input,per-stream*)

    将匹配的附件流提取到名为 *filename* 的文件中。如果 *filename* 为空，则使用 `filename` 元数据标签的值。

    例如，将第一个附件提取到名为 'out.ttf' 的文件中：

```shell
ffmpeg -dump_attachment:t:0 out.ttf -i INPUT
```

    将所有附件提取到由 `filename` 标签确定的文件中：

```shell
ffmpeg -dump_attachment:t "" -i INPUT
```

技术说明 - 附件是作为编解码器额外数据实现的，因此实际上此选项可用于从任何流（而不仅仅是附件）中提取额外数据。

### 5.5 Video Options

-vframes *number* (*output*)

&nbsp;&nbsp;&nbsp;&nbsp;设置要输出的视频帧数量。这是 `-frames:v` 的过时别名，建议使用后者代替。

-r[:*stream_specifier*] *fps* (input/output,per-strea)

&nbsp;&nbsp;&nbsp;&nbsp;设置帧率（Hz 值、分数或缩写）。

&nbsp;&nbsp;&nbsp;&nbsp;作为输入选项时，忽略文件中存储的任何时间戳，并假设恒定帧率为 *fps* 生成时间戳。这与用于像 image2 或 v4l2 这类输入格式的 -framerate 选项不同（在 FFmpeg 的旧版本中是相同的）。如果不确定，请使用 -framerate 而不是输入选项 -r。

作为输出选项时：

- video encoding
  在编码之前复制或丢弃帧以实现恒定的输出帧率 fps。
- video streamcopy
  指示复用器 *fps* 是流的帧率。在这种情况下不会丢弃或复制数据。如果 *fps* 不匹配由包时间戳确定的实际流帧率，这可能会产生无效文件。另请参阅 `setts` 比特流滤镜。

-fpsmax[:*stream_specifier*] *fps* (*output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置最大帧率（Hz 值、分数或缩写）。

&nbsp;&nbsp;&nbsp;&nbsp;当输出帧率自动设置且高于此值时，限制输出帧率。在批处理或输入帧率被错误检测为非常高时很有用。它不能与 `-r` 同时设置。在流复制期间会被忽略。

-s[:*stream_specifier*] *size* (*input/output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置帧大小。

&nbsp;&nbsp;&nbsp;&nbsp;作为输入选项时，这是某些解复用器识别的 video_size 私有选项的快捷方式，这些解复用器的帧大小要么未存储在文件中，要么是可配置的——例如原始视频或视频采集设备。

&nbsp;&nbsp;&nbsp;&nbsp;作为输出选项时，这会在相应的滤镜图末尾插入 `scale` 视频滤镜。请直接使用 `scale` 滤镜以将其插入到开头或其他位置。

&nbsp;&nbsp;&nbsp;&nbsp;格式为 'wxh'（默认值——与源相同）。

-aspect[:*stream_specifier*] *aspect* (*output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置指定的视频显示宽高比。

&nbsp;&nbsp;&nbsp;&nbsp;aspect 可以是一个浮点数字符串，或者是一个形式为 num:den 的字符串，其中 num 和 den 分别是宽高比的分子和分母。例如 "4:3"、"16:9"、"1.3333" 和 "1.7777" 都是有效的参数值。

&nbsp;&nbsp;&nbsp;&nbsp;如果与 -vcodec copy 一起使用，它将影响容器级别存储的宽高比，但不会影响编码帧中存储的宽高比（如果存在）。

-display_rotation[:*stream_specifier*] *rotation* (*input,per-stream*)
设置视频旋转元数据。

&nbsp;&nbsp;&nbsp;&nbsp;*rotation* 是一个十进制数，指定了视频在显示前应逆时针旋转的度数。

&nbsp;&nbsp;&nbsp;&nbsp;这个选项会覆盖文件中存储的任何旋转/显示变换元数据。当视频被转码（而不是复制）且启用了 `-autorotate` 时，视频将在滤镜阶段进行旋转。否则，如果复用器支持，元数据将被写入输出文件。

&nbsp;&nbsp;&nbsp;&nbsp;如果给出了 `-display_hflip` 和/或 `-display_vflip` 选项，则它们将在本选项指定的旋转之后应用。

-display_hflip[:*stream_specifier*] (*input,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置在显示时图像是否应该水平翻转。

&nbsp;&nbsp;&nbsp;&nbsp;更多详情请参见 `-display_rotation` 选项。

-display_vflip[:*stream_specifier*] (*input,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置在显示时图像是否应该垂直翻转。

&nbsp;&nbsp;&nbsp;&nbsp;更多详情请参见 `-display_rotation` 选项。

-vn (*input/output*)

&nbsp;&nbsp;&nbsp;&nbsp;作为输入选项，阻止文件中的所有视频流被过滤或自动选择或映射到任何输出。参见 `-discard` 选项以单独禁用流。

&nbsp;&nbsp;&nbsp;&nbsp;作为输出选项，禁用视频记录，即自动选择或映射任何视频流。对于完全的手动控制，请参阅 `-map` 选项。

-vcodec *codec* (*output)

&nbsp;&nbsp;&nbsp;&nbsp;设置视频编解码器。这是 `-codec:v` 的别名。

-pass[:*stream_specifier*] n (*output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;选择通过次数（1 或 2）。用于两遍视频编码。第一遍会将视频的统计数据记录到一个日志文件中（另见 -passlogfile 选项），第二遍则使用该日志文件生成符合请求比特率的视频。在第一遍时，您可以仅停用音频并将输出设为 null。以下是在 Windows 和 Unix 上的例子：

```shell
ffmpeg -i foo.mov -c:v libxvid -pass 1 -an -f rawvideo -y NUL
ffmpeg -i foo.mov -c:v libxvid -pass 1 -an -f rawvideo -y /dev/null
```

-passlogfile[:*stream_specifier*] *prefix* (output,per-stream)

&nbsp;&nbsp;&nbsp;&nbsp;将两遍日志文件名前缀设置为 prefix，默认文件名前缀是 "ffmpeg2pass"。完整的文件名将是 PREFIX-N.log，其中 N 是特定于输出流的数字。

-vf *filtergraph* (*output*)

&nbsp;&nbsp;&nbsp;&nbsp;创建由 filtergraph 描述的滤镜图并用它来过滤流。

&nbsp;&nbsp;&nbsp;&nbsp;这是 `-filter:v` 的别名，详见 -filter 选项。

-autorotate

&nbsp;&nbsp;&nbsp;&nbsp;根据文件元数据自动旋转视频。默认启用，使用 -noautorotate 禁用。

-autoscale

&nbsp;&nbsp;&nbsp;&nbsp;根据第一帧的分辨率自动缩放视频。默认启用，使用 -noautoscale 禁用。当禁用 autoscale 时，滤镜图的所有输出帧可能不在同一分辨率，并且可能不适合某些编码器/复用器。因此，除非您确实知道自己在做什么，否则不建议禁用它。自行承担风险禁用 autoscale。

### 5.6 Advanced Video options

-pix_fmt[:stream_specifier] format (*input/output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;设置像素格式。使用 `-pix_fmts` 查看所有支持的像素格式。如果选定的像素格式无法选择，FFmpeg 将打印警告并选择编码器支持的最佳像素格式。如果 pix_fmt 前缀为 `+`，则 FFmpeg 会在请求的像素格式无法选择时退出，并禁用滤镜图内的自动转换。如果 pix_fmt 是一个单独的 `+`，FFmpeg 会选择与输入（或图输出）相同的像素格式，并禁用自动转换。

-sws_flags flags (*input/output*)

&nbsp;&nbsp;&nbsp;&nbsp;设置 libswscale 库的默认标志。这些标志用于自动插入的 `scale` 滤镜以及简单滤镜图中的滤镜，除非在滤镜图定义中被覆盖。

&nbsp;&nbsp;&nbsp;&nbsp;有关缩放器选项列表，请参阅 (ffmpeg-scaler)ffmpeg-scaler 手册。

-rc_override[:stream_specifier] override (*output,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;特定区间的码率控制覆盖，格式化为 "int,int,int" 列表并以斜杠分隔。前两个值是开始和结束帧号，最后一个值是正数时使用的量化器，或负数时的质量因子。

-vstats

&nbsp;&nbsp;&nbsp;&nbsp;将视频编码统计信息转储到 vstats_HHMMSS.log 文件中。有关格式描述，请参阅 vstats 文件格式部分。

-vstats_file file

&nbsp;&nbsp;&nbsp;&nbsp;将视频编码统计信息转储到指定文件中。有关格式描述，请参阅 vstats 文件格式部分。

-vstats_version file

&nbsp;&nbsp;&nbsp;&nbsp;指定要使用的 vstats 格式版本，默认为 `2`。有关格式描述，请参阅 vstats 文件格式部分。

-vtag fourcc/tag (*output*)

&nbsp;&nbsp;&nbsp;&nbsp;强制视频标签/fourcc。这是 `-tag:v` 的别名。

-force_key_frames[:stream_specifier] time[,time...] (*output,per-stream*)
-force_key_frames[:stream_specifier] expr:expr (*output,per-stream*)
-force_key_frames[:stream_specifier] source (*output,per-stream*)

force_key_frames 可以接受以下形式的参数：

&nbsp;&nbsp;&nbsp;&nbsp;time[,time...]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果参数由时间戳组成，FFmpeg 将指定的时间四舍五入到最接近的输出时间戳（根据编码器时间基准），并在第一个时间戳等于或大于计算后时间戳的帧上强制关键帧。请注意，如果编码器时间基准太粗糙，则关键帧可能会被强制在时间戳低于指定时间的帧上。默认编码器时间基准是输出帧率的倒数，但可以通过 `-enc_time_base` 设置为其他值。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果其中一个时间是 "`chapters`[delta]"，它会展开为文件中所有章节开始的时间，并根据 delta 进行偏移，delta 以秒为单位表示的时间。此选项可用于确保在章节标记或输出文件中的任何其他指定位置存在一个寻址点（seek point）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如，在5分钟处插入一个关键帧，并在每个章节开始前0.1秒插入关键帧：

```shell
-force_key_frames 0:05:00,chapters-0.1
```

&nbsp;&nbsp;&nbsp;&nbsp;expr:expr

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果参数以前缀 `expr:` 开头，则字符串 expr 被解释为表达式，并针对每一帧进行求值。如果求值结果非零，则会强制一个关键帧。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表达式 expr 中可以包含以下常量：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;n

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从 0 开始的当前处理帧的编号

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;n_forced

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;已强制帧的数量

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;prev_forced_n

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上一个强制帧的编号，如果没有强制过关键帧则为 `NAN`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;prev_forced_t

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上一个强制帧的时间，如果没有强制过关键帧则为 `NAN`

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;t

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前处理帧的时间

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如，每 5 秒强制一个关键帧，您可以指定：

```shell
-force_key_frames expr:gte(t,n_forced*5)
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从第 13 秒起，每隔 5 秒在上一个强制帧之后强制一个关键帧：

```shell
-force_key_frames expr:if(isnan(prev_forced_t),gte(t,13),gte(t,prev_forced_t+5))
```

&nbsp;&nbsp;&nbsp;&nbsp;source

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果参数为 `source`，FFmpeg 将在当前正在编码的帧在其源中标记为关键帧时强制一个关键帧。如果此特定源帧必须被丢弃，则强制下一个可用帧成为关键帧。

&nbsp;&nbsp;&nbsp;&nbsp;注意：强迫过多的关键帧对某些编码器的 lookahead 算法非常有害：使用固定 GOP 选项或类似选项会更有效。

-apply_cropping[:stream_specifier] source (*input,per-stream*)

&nbsp;&nbsp;&nbsp;&nbsp;根据文件元数据自动裁剪解码后的视频。默认为 all。

&nbsp;&nbsp;&nbsp;&nbsp;none (0)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不应用任何裁剪元数据。

&nbsp;&nbsp;&nbsp;&nbsp;all (1)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;应用编解码器级别和容器级别的裁剪。这是默认模式。

&nbsp;&nbsp;&nbsp;&nbsp;codec (2)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;应用编解码器级别的裁剪。

&nbsp;&nbsp;&nbsp;&nbsp;container (3)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;应用容器级别的裁剪。

-copyinkf[:stream_specifier] (*output,per-stream*)
&nbsp;&nbsp;&nbsp;&nbsp;在进行流复制时，也复制位于开头的非关键帧。

-init_hw_device type[=name][:device[,key=value...]]
&nbsp;&nbsp;&nbsp;&nbsp;初始化一个名为 name 的新硬件设备，类型为 type，并使用给定的设备参数。如果没有指定名称，则会获得 "type%d" 格式的默认名称。

&nbsp;&nbsp;&nbsp;&nbsp;设备和以下参数的意义取决于设备类型：

&nbsp;&nbsp;&nbsp;&nbsp;cuda
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 是 CUDA 设备的编号。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可识别的选项包括：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;primary_ctx
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果设置为 1，则使用主设备上下文而不是创建新的上下文。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device cuda:1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择系统上的第二个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device cuda:0,primary_ctx=1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择第一个设备并使用主设备上下文。

&nbsp;&nbsp;&nbsp;&nbsp;dxva2
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 是 Direct3D 9 显示适配器的编号。

&nbsp;&nbsp;&nbsp;&nbsp;d3d11va
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 是 Direct3D 11 显示适配器的编号。如果没有指定，则尝试使用默认的 Direct3D 11 显示适配器或硬件 VendorId 由 ‘vendor_id’ 指定的第一个 Direct3D 11 显示适配器。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device d3d11va***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在默认 Direct3D 11 显示适配器上创建 d3d11va 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device d3d11va:1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在由索引 1 指定的 Direct3D 11 显示适配器上创建 d3d11va 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device d3d11va:,vendor_id=0x8086***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在硬件 VendorId 为 0x8086 的第一个 Direct3D 11 显示适配器上创建 d3d11va 设备。

&nbsp;&nbsp;&nbsp;&nbsp;vaapi
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 可以是 X11 显示名称、DRM 渲染节点或 DirectX 适配器索引。如果没有指定，则尝试打开默认 X11 显示 ($DISPLAY) 和第一个 DRM 渲染节点 (/dev/dri/renderD128)，或者 Windows 上的默认 DirectX 适配器。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可识别的选项包括：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;kernel_driver
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当未指定 device 时，使用此选项指定与所需设备关联的内核驱动程序名称。仅当启用了硬件加速方法 drm 和 vaapi 时可用。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;vendor_id
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当未指定 device 和 kernel_driver 时，使用此选项指定与所需设备关联的供应商 ID。仅当启用了硬件加速方法 drm 和 vaapi 且未指定 kernel_driver 时可用。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在默认设备上创建 vaapi 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi:/dev/dri/renderD129***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 DRM 渲染节点 /dev/dri/renderD129 上创建 vaapi 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi:1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 DirectX 适配器 1 上创建 vaapi 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi:,kernel_driver=i915***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在与内核驱动程序 ‘i915’ 关联的设备上创建 vaapi 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi:,vendor_id=0x8086***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在与供应商 ID ‘0x8086’ 关联的设备上创建 vaapi 设备。

&nbsp;&nbsp;&nbsp;&nbsp;vdpau
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 是 X11 显示名称。如果没有指定，则尝试打开默认 X11 显示 ($DISPLAY)。

&nbsp;&nbsp;&nbsp;&nbsp;qsv
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 选择 ‘MFX_IMPL_*’ 中的一个值。允许的值有：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;auto
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sw
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hw
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;auto_any
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hw_any
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hw2
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hw3
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hw4

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果没有指定，默认使用 ‘auto_any’。（请注意，通过创建适当的平台子设备（‘dxva2’ 或 ‘d3d11va’ 或 ‘vaapi’）然后从中派生 QSV 设备，可能会更容易实现所需的 QSV 结果。）

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可识别的选项包括：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;child_device
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;指定 Linux 上的 DRM 渲染节点或 Windows 上的 DirectX 适配器。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;child_device_type
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择平台适用的子设备类型。在 Windows 上，如果配置时指定了 `--enable-libvpl`，则默认使用 ‘d3d11va’ 子设备类型；如果指定了 `--enable-libmfx`，则默认使用 ‘dxva2’ 子设备类型。在 Linux 上，用户只能使用 ‘vaapi’ 作为子设备类型。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device qsv:hw,child_device=/dev/dri/renderD129***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 DRM 渲染节点 /dev/dri/renderD129 上创建具有 ‘MFX_IMPL_HARDWARE’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device qsv:hw,child_device=1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 DirectX 适配器 1 上创建具有 ‘MFX_IMPL_HARDWARE’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device qsv:hw,child_device_type=d3d11va***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择类型为 ‘d3d11va’ 的 GPU 子设备并创建具有 ‘MFX_IMPL_HARDWARE’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device qsv:hw,child_device_type=dxva2***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择类型为 ‘dxva2’ 的 GPU 子设备并创建具有 ‘MFX_IMPL_HARDWARE’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device qsv:hw,child_device=1,child_device_type=d3d11va***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在类型为 ‘d3d11va’ 的 DirectX 适配器 1 上创建具有 ‘MFX_IMPL_HARDWARE’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vaapi=va:/dev/dri/renderD129 -init_hw_device qsv=hw1@va***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 /dev/dri/renderD129 上创建名为 ‘va’ 的 VAAPI 设备，然后从设备 ‘va’ 派生名为 ‘hw1’ 的 QSV 设备。

&nbsp;&nbsp;&nbsp;&nbsp;opencl
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device 选择 platform_index.device_index 形式的平台和设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还可以使用键值对过滤器来查找符合特定平台或设备字符串的设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可用作过滤器的字符串有：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;platform_profile
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;platform_version
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;platform_name
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;platform_vendor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;platform_extensions
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_name
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_vendor
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;driver_version
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_version
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_profile
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_extensions
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_type

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;索引和过滤器必须共同唯一地选择一个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device opencl:0.1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择第一个平台上的第二个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device opencl:,device_name=Foo9000***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择名称包含字符串 Foo9000 的设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device opencl:1,device_type=gpu,device_extensions=cl_khr_fp16***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择支持 cl_khr_fp16 扩展的第二个平台上的 GPU 设备。

&nbsp;&nbsp;&nbsp;&nbsp;vulkan
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果 device 是整数，则按系统依赖的设备列表中的索引选择设备。如果 device 是其他字符串，则选择名称包含该字符串作为子串的第一个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可识别的选项包括：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;debug
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果设置为 1，则启用验证层（如果已安装）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;linear_images
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果设置为 1，则 hwcontext 分配的图像将是线性的并且可以局部映射。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;instance_extensions
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;附加实例扩展的加号分隔列表。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;device_extensions
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;附加设备扩展的加号分隔列表。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vulkan:1***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择系统上的第二个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vulkan:RADV***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择名称包含字符串 RADV 的第一个设备。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;***-init_hw_device vulkan:0,instance_extensions=VK_KHR_wayland_surface+VK_KHR_xcb_surface***
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择第一个设备并启用 Wayland 和 XCB 实例扩展。

-init_hw_device type[=name]@source
&nbsp;&nbsp;&nbsp;&nbsp;初始化一个名为 name 的新硬件设备，类型为 type，并从已有的名为 source 的设备派生。

-init_hw_device list
&nbsp;&nbsp;&nbsp;&nbsp;列出此版本 ffmpeg 支持的所有硬件设备类型。

-filter_hw_device name
&nbsp;&nbsp;&nbsp;&nbsp;将名为 name 的硬件设备传递给任何滤镜图中的所有滤镜。这可以用于设置 `hwupload` 滤镜要上传到的设备，或 `hwmap` 滤镜要映射到的设备。其他滤镜在需要硬件设备时也可能使用这个参数。请注意，通常只有当输入不是已经处于硬件帧时才需要这样做——当它是的时候，滤镜会从它们接收到的帧上下文中推导出所需的设备。

&nbsp;&nbsp;&nbsp;&nbsp;这是一个全局设置，因此所有滤镜都将接收相同的设备。

-hwaccel[:stream_specifier] hwaccel (*input,per-stream*)
&nbsp;&nbsp;&nbsp;&nbsp;使用硬件加速解码匹配的流。hwaccel 允许的值有：

&nbsp;&nbsp;&nbsp;&nbsp;none
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不使用任何硬件加速（默认）。

&nbsp;&nbsp;&nbsp;&nbsp;auto
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;自动选择硬件加速方法。

&nbsp;&nbsp;&nbsp;&nbsp;vdpau
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 VDPAU（Unix 视频解码和演示 API）硬件加速。

&nbsp;&nbsp;&nbsp;&nbsp;dxva2
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 DXVA2（DirectX 视频加速）硬件加速。

&nbsp;&nbsp;&nbsp;&nbsp;d3d11va
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 D3D11VA（DirectX 视频加速）硬件加速。

&nbsp;&nbsp;&nbsp;&nbsp;vaapi
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 VAAPI（视频加速 API）硬件加速。

&nbsp;&nbsp;&nbsp;&nbsp;qsv
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用 Intel QuickSync Video 加速进行视频转码。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;与大多数其他值不同，这个选项不会启用加速解码（当选择了 QSV 解码器时会自动使用），而是启用了无需将帧复制到系统内存的加速转码。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了使它工作，解码器和编码器都必须支持 QSV 加速，并且不能使用滤镜。

&nbsp;&nbsp;&nbsp;&nbsp;如果选定的 hwaccel 不可用或所选解码器不支持，则此选项无效。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，大多数加速方法都是为了播放而设计的，在现代 CPU 上可能不会比软件解码更快。此外，`ffmpeg` 通常需要将解码后的帧从 GPU 内存复制到系统内存，导致进一步的性能损失。因此，这个选项主要用于测试。

-hwaccel_device[:stream_specifier] hwaccel_device (输入, 按流)
&nbsp;&nbsp;&nbsp;&nbsp;选择一个用于硬件加速的设备。

&nbsp;&nbsp;&nbsp;&nbsp;只有当 -hwaccel 选项也被指定时才有意义。它可以引用通过 -init_hw_device 创建的现有设备的名称，或者它可以在调用前立即创建一个新设备，就像调用了 ‘-init_hw_device’ 类型:hwaccel_device 一样。

-hwaccels
&nbsp;&nbsp;&nbsp;&nbsp;列出此版本 FFmpeg 启用的所有硬件加速组件。实际运行时的可用性取决于硬件及其适当的驱动程序是否已安装。

-fix_sub_duration_heartbeat[:stream_specifier]
&nbsp;&nbsp;&nbsp;&nbsp;根据随机访问包接收时当前正在进行的字幕，按照特定输出视频流作为心跳流来分割并推送。

&nbsp;&nbsp;&nbsp;&nbsp;这降低了那些结束包或下一个字幕尚未收到的字幕的延迟。缺点是这可能会导致字幕事件的重复，以覆盖整个持续时间，因此在字幕事件传递给输出的时间延迟无关紧要的情况下不应使用此选项。

&nbsp;&nbsp;&nbsp;&nbsp;这需要为相关的输入字幕流设置了 -fix_sub_duration 才能生效，而且输入字幕流必须直接映射到包含心跳流的同一输出。

### 5.7 Audio Options
