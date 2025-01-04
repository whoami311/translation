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

-aframes number (output)
&nbsp;&nbsp;&nbsp;&nbsp;设置要输出的音频帧数。这是 `-frames:a` 的过时别名，建议使用 `-frames:a` 代替。

-ar[:stream_specifier] freq (input/output,per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频采样频率。对于输出流，默认设置为相应输入流的频率。对于输入流，此选项仅对音频捕获设备和原始解复用器有意义，并映射到相应的解复用器选项。

-aq q (output)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频质量（编解码器特定，VBR）。这是 -q:a 的别名。

-ac[:stream_specifier] channels (input/output,per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频声道数。对于输出流，默认设置为输入音频声道的数量。对于输入流，此选项仅对音频捕获设备和原始解复用器有意义，并映射到相应的解复用器选项。

-an (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;作为输入选项，阻止文件中的所有音频流被过滤或自动选择或映射到任何输出。参见 `-discard` 选项以单独禁用流。

&nbsp;&nbsp;&nbsp;&nbsp;作为输出选项，禁用音频录制，即自动选择或映射任何音频流。对于完全的手动控制，请参阅 `-map` 选项。

-acodec codec (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频编解码器。这是 `-codec:a` 的别名。

-sample_fmt[:stream_specifier] sample_fmt (output,per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频样本格式。使用 `-sample_fmts` 获取支持的样本格式列表。

-af filtergraph (output)
&nbsp;&nbsp;&nbsp;&nbsp;创建由 filtergraph 描述的滤镜图并用它来过滤流。

&nbsp;&nbsp;&nbsp;&nbsp;这是 `-filter:a` 的别名，详见 `-filter` 选项。

### 5.8 Advanced Audio options

-atag fourcc/tag (output)
&nbsp;&nbsp;&nbsp;&nbsp;强制音频标签/fourcc。这是 `-tag:a` 的别名。

-ch_layout[:stream_specifier] layout (input/output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;`-channel_layout` 的别名。

-channel_layout[:stream_specifier] layout (input/output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置音频声道布局。对于 output streams，默认设置为输入声道布局。对于 input streams，它会覆盖输入的声道布局。并非所有解码器都会尊重被覆盖的声道布局。此选项还设置了音频捕获设备和原始解复用器的声道布局，并映射到相应的解复用器选项。

-guess_layout_max channels (input, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;如果某些输入声道布局未知，则仅在声道数不超过指定数量时尝试猜测。例如，2 告诉 `ffmpeg` 将 1 个声道识别为单声道，将 2 个声道识别为立体声，但不会将 6 个声道识别为 5.1 声道。默认情况下是总是尝试猜测。使用 0 禁用所有猜测。使用 `-channel_layout` 选项显式指定输入布局也会禁用猜测。

### 5.9 Subtitle options

-scodec codec (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;设置字幕编解码器。这是 `-codec:s` 的别名。

-sn (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;作为 input 选项，阻止文件中的所有字幕流被过滤或自动选择或映射到任何输出。参见 `-discard` 选项以单独禁用流。

&nbsp;&nbsp;&nbsp;&nbsp;作为 output 选项，禁用字幕录制，即自动选择或映射任何字幕流。对于完全的手动控制，请参阅 `-map` 选项。

### 5.10 Advanced Subtitle options

-fix_sub_duration
&nbsp;&nbsp;&nbsp;&nbsp;修复字幕持续时间。对于每个字幕，等待同一流中的下一个数据包，并调整第一个字幕的持续时间以避免重叠。这对于某些字幕编解码器（尤其是 DVB 字幕）是必要的，因为在原始数据包中的持续时间只是一个粗略的估计，实际结束是由一个空的字幕帧标记的。在必要时未能使用此选项可能导致持续时间过长或由于非单调的时间戳而导致复用失败。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，这个选项会延迟所有数据的输出，直到解码了下一个字幕数据包：它可能会显著增加内存消耗和延迟。

-canvas_size size
&nbsp;&nbsp;&nbsp;&nbsp;设置用于渲染字幕的画布大小。

### 5.11 Advanced options

-map [-]input_file_id[:stream_specifier][:view_specifier][:?] | [linklabel] (output)
&nbsp;&nbsp;&nbsp;&nbsp;创建一个或多个输出文件中的流。此选项有两种形式来指定数据源：第一种是从某些输入文件（使用 `-i` 指定）中选择一个或多个流；第二种是采用复杂滤镜图（使用 `-filter_complex` 指定）的一个输出。

&nbsp;&nbsp;&nbsp;&nbsp;在第一种形式中，为具有索引 input_file_id 的输入文件中的每个流创建一个输出流。如果给出了 stream_specifier，则仅使用匹配该说明符的流（有关 stream_specifier 语法，请参阅流说明符部分）。

&nbsp;&nbsp;&nbsp;&nbsp;stream 标识符前的 `-` 字符创建“负”映射。它会禁用已经创建的映射中匹配的流。

&nbsp;&nbsp;&nbsp;&nbsp;可选的 view_specifier 可能在 stream_specifier 之后给出，对于多视图视频指定要使用的视图。view specifier 可能有以下格式之一：

&nbsp;&nbsp;&nbsp;&nbsp;view:view_id
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过其 ID 选择一个视图；view_id 可以设置为 'all' 以交错使用所有视图到一个流中；

&nbsp;&nbsp;&nbsp;&nbsp;vidx:view_idx
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过其索引选择一个视图；即 0 是基础视图，1 是第一个非基础视图等。

&nbsp;&nbsp;&nbsp;&nbsp;vpos:position
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过其显示位置选择一个视图；position 可以是 `left` 或 `right`。

&nbsp;&nbsp;&nbsp;&nbsp;转码的默认行为是仅使用基础视图，即相当于 `vidx:0`。对于流复制，不支持视图说明符，并且总是复制所有视图。

&nbsp;&nbsp;&nbsp;&nbsp;stream 索引后的尾随 `?` 使得映射可选：如果映射不匹配任何流，映射将被忽略而不是失败。请注意，如果使用了无效的输入文件索引，映射仍然会失败；例如，如果映射引用了一个不存在的输入。

&nbsp;&nbsp;&nbsp;&nbsp;另一种 [linklabel] 形式将复杂滤镜图的输出（参见 -filter_complex 选项）映射到输出文件。linklabel 必须对应于图中定义的输出链接标签。

&nbsp;&nbsp;&nbsp;&nbsp;此选项可以多次指定，每次都会向输出文件中添加更多流。任何给定的输入流也可以作为不同输出流的来源被映射任意次数，例如为了使用不同的编码选项和/或滤镜。这些流按照命令行上给出 `-map` 选项的顺序在输出中创建。

&nbsp;&nbsp;&nbsp;&nbsp;使用此选项会禁用此输出文件的默认映射。

&nbsp;&nbsp;&nbsp;&nbsp;示例：

&nbsp;&nbsp;&nbsp;&nbsp;map everything
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将第一个输入文件的所有流映射到输出：
```shell
ffmpeg -i INPUT -map 0 output
```
&nbsp;&nbsp;&nbsp;&nbsp;select specific stream
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果有两个音频流在第一个输入文件中，这些流由 0:0 和 0:1 标识。可以使用 -map 来选择哪些流放入输出文件中。例如：
```shell
ffmpeg -i INPUT -map 0:1 out.wav
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将 INPUT 中第二个输入流映射到 out.wav 中的（单个）输出流。

&nbsp;&nbsp;&nbsp;&nbsp;create multiple streams
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从输入文件 a.mov（由标识符 0:2 指定）中选择索引为 2 的流，以及从 b.mov（由标识符 1:6 指定）中选择索引为 6 的流，并复制到输出文件 out.mov：
```shell
ffmpeg -i a.mov -i b.mov -c copy -map 0:2 -map 1:6 out.mov
```
&nbsp;&nbsp;&nbsp;&nbsp;create multiple streams 2
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择所有视频和第三个音频流：
```shell
ffmpeg -i INPUT -map 0:v -map 0:a:2 OUTPUT
```
&nbsp;&nbsp;&nbsp;&nbsp;negative map
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;映射所有流，除了第二个音频流，使用负映射：
```shell
ffmpeg -i INPUT -map 0 -map -0:a:1 OUTPUT
```
&nbsp;&nbsp;&nbsp;&nbsp;optional map
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;映射第一个输入的视频和音频流，并使用尾随的 ?，如果第一个输入中没有音频流则忽略音频映射：
```shell
ffmpeg -i INPUT -map 0:v -map 0:a? OUTPUT
```
&nbsp;&nbsp;&nbsp;&nbsp;map by language
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;选择英语音频流：
```shell
ffmpeg -i INPUT -map 0:m:language:eng OUTPUT
```
-ignore_unknown
&nbsp;&nbsp;&nbsp;&nbsp;尝试复制未知类型的流时，忽略输入流而不是失败。

-copy_unknown
&nbsp;&nbsp;&nbsp;&nbsp;允许复制未知类型的输入流而不是在尝试复制此类流时失败。

-map_metadata[:metadata_spec_out] infile[:metadata_spec_in] (output, per-metadata)
&nbsp;&nbsp;&nbsp;&nbsp;从 infile 设置下一个输出文件的元数据信息。请注意这些是文件索引（从零开始），而不是文件名。可选的 metadata_spec_in/out 参数指定要复制哪些元数据。元数据说明符可以有以下形式：

&nbsp;&nbsp;&nbsp;&nbsp;g
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;全局元数据，即适用于整个文件的元数据

&nbsp;&nbsp;&nbsp;&nbsp;s[:stream_spec]
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每流元数据。stream_spec 是如流说明符章节中所述的流说明符。在输入元数据说明符中，会从第一个匹配的流复制。在输出元数据说明符中，所有匹配的流都会被复制。

&nbsp;&nbsp;&nbsp;&nbsp;c:chapter_index
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每章节元数据。chapter_index 是基于零的章节索引。

&nbsp;&nbsp;&nbsp;&nbsp;p:program_index
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每节目元数据。program_index 是基于零的节目索引。

&nbsp;&nbsp;&nbsp;&nbsp;如果省略了元数据说明符，默认为全局。

&nbsp;&nbsp;&nbsp;&nbsp;默认情况下，全局元数据是从第一个输入文件复制的，每流和每章节元数据是随流/章节一起复制的。通过创建任何相关类型的映射来禁用这些默认映射。可以使用负数文件索引来创建一个虚拟映射，以仅禁用自动复制。

&nbsp;&nbsp;&nbsp;&nbsp;例如，将输入文件的第一个流的元数据复制到输出文件的全局元数据：
```shell
ffmpeg -i in.ogg -map_metadata 0:s:0 out.mp3
```
&nbsp;&nbsp;&nbsp;&nbsp;反向操作，即将全局元数据复制到所有音频流：
```shell
ffmpeg -i in.mkv -map_metadata:s:a 0:g out.mkv
```
&nbsp;&nbsp;&nbsp;&nbsp;在这个例子中，简单的 `0` 也可以工作，因为默认假设为全局元数据。

-map_chapters input_file_index (output)
&nbsp;&nbsp;&nbsp;&nbsp;从具有索引 input_file_index 的输入文件复制章节到下一个输出文件。如果没有指定章节映射，则从至少有一个章节的第一个输入文件复制章节。使用负数文件索引以禁用任何章节复制。

-benchmark (global)
&nbsp;&nbsp;&nbsp;&nbsp;在编码结束时显示基准测试信息。显示实际、系统和用户时间以及最大内存消耗。并非所有系统都支持最大内存消耗，通常不支持时会显示为 0。

-benchmark_all (global)
&nbsp;&nbsp;&nbsp;&nbsp;在编码期间显示基准测试信息。显示各种步骤（音频/视频编码/解码）所使用的实际、系统和用户时间。

-timelimit duration (global)
&nbsp;&nbsp;&nbsp;&nbsp;在 ffmpeg 运行 CPU 用户时间 duration 秒后退出。

-dump (global)
&nbsp;&nbsp;&nbsp;&nbsp;将每个输入包转储到标准错误输出。

-hex (global)
&nbsp;&nbsp;&nbsp;&nbsp;在转储包时，也转储有效负载。

-readrate speed (input)
&nbsp;&nbsp;&nbsp;&nbsp;限制输入读取速度。

&nbsp;&nbsp;&nbsp;&nbsp;其值是一个浮点正数，表示应在一秒实际时间内摄取的媒体的最大持续时间（以秒为单位）。默认值为零，表示不对摄取速度施加任何限制。值 `1` 表示实时速度，并等同于 `-re`。

&nbsp;&nbsp;&nbsp;&nbsp;主要用于模拟捕获设备或直播输入流（例如从文件读取时）。当输入是实际的捕获设备或直播流时，不应使用低值，因为它可能导致丢包。

&nbsp;&nbsp;&nbsp;&nbsp;当输出包的流速重要时（如直播流）非常有用。

-re (input)
&nbsp;&nbsp;&nbsp;&nbsp;以原生帧速率读取输入。这相当于设置 `-readrate 1`。

-readrate_initial_burst seconds
&nbsp;&nbsp;&nbsp;&nbsp;设置初始读取爆发时间，以秒为单位，在此之后将强制执行 -re/-readrate。

-vsync parameter (global)
-fps_mode[:stream_specifier] parameter (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置视频同步方法/帧率模式。vsync 应用于所有输出视频流，但可以通过为特定流设置 fps_mode 来覆盖它。vsync 已被弃用，并将在未来移除。

&nbsp;&nbsp;&nbsp;&nbsp;出于兼容性原因，vsync 的某些值可以以数字形式指定（在下表中以括号显示）。

&nbsp;&nbsp;&nbsp;&nbsp;passthrough (0)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每个帧都从解复用器传递到复用器，保持其时间戳不变。

&nbsp;&nbsp;&nbsp;&nbsp;cfr (1)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过复制和丢弃帧来实现请求的确切恒定帧率。

&nbsp;&nbsp;&nbsp;&nbsp;vfr (2)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧会带着它们的时间戳传递或被丢弃，以防止两个帧具有相同的时间戳。

&nbsp;&nbsp;&nbsp;&nbsp;auto (-1)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据复用器的能力选择 cfr 和 vfr 之一。这是默认方法。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，时间戳可能会进一步被复用器修改。例如，在启用格式选项 avoid_negative_ts 的情况下。

&nbsp;&nbsp;&nbsp;&nbsp;使用 -map 可以选择从哪个流获取时间戳。您可以保留视频或音频中的一个不变，并将其他流同步到未更改的那个。

-frame_drop_threshold parameter
&nbsp;&nbsp;&nbsp;&nbsp;帧丢弃阈值，指定了视频帧可以在多大程度上滞后才会被丢弃。以帧率为单位，所以 1.0 表示一帧。默认值是 -1.1。一个可能的应用场景是在时间戳有噪声的情况下避免帧丢弃，或者在时间戳精确的情况下提高帧丢弃的精度。

-apad parameters (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;填充输出音频流。这相当于应用 `-af apad`。参数是一个由与 `apad` 滤镜相同的参数组成的字符串。要使此选项生效，必须为此输出设置 `-shortest`。

-copyts
&nbsp;&nbsp;&nbsp;&nbsp;不处理输入时间戳，而是保留它们的值而不尝试进行清理。特别是，不会移除初始开始时间偏移值。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，即使选择了此选项，由于 vsync 选项或特定复用器处理（例如，在启用格式选项 avoid_negative_ts 的情况下），输出时间戳也可能与输入时间戳不匹配。

-start_at_zero
&nbsp;&nbsp;&nbsp;&nbsp;当与 copyts 一起使用时，将输入时间戳调整为从零开始。

&nbsp;&nbsp;&nbsp;&nbsp;这意味着，例如使用 `-ss 50` 将使输出时间戳从 50 秒开始，无论输入文件从什么时间戳开始。

-copytb mode
&nbsp;&nbsp;&nbsp;&nbsp;指定在流复制时如何设置编码器的时间基。mode 是一个整数值，可以取以下值之一：

&nbsp;&nbsp;&nbsp;&nbsp;1
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用解复用器的时间基。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;时间基从相应的输入解复用器复制到输出编码器。有时需要这样做以避免在复制具有可变帧率的视频流时产生非单调递增的时间戳。

&nbsp;&nbsp;&nbsp;&nbsp;0
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用解码器的时间基。

&nbsp;&nbsp;&nbsp;&nbsp;时间基从相应的输入解码器复制到输出编码器。

&nbsp;&nbsp;&nbsp;&nbsp;-1
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;尝试自动选择，以便生成合理的输出。

&nbsp;&nbsp;&nbsp;&nbsp;默认值是 -1。

-enc_time_base[:stream_specifier] timebase (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;设置编码器时间基。timebase 可以取以下值之一：

&nbsp;&nbsp;&nbsp;&nbsp;0
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据媒体类型分配默认值。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于视频 - 使用 1/帧率，对于音频 - 使用 1/采样率。

&nbsp;&nbsp;&nbsp;&nbsp;demux
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用解复用器的时间基。

&nbsp;&nbsp;&nbsp;&nbsp;filter
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用滤镜图的时间基。

&nbsp;&nbsp;&nbsp;&nbsp;a positive number
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用提供的数字作为时间基。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;此字段可以表示为两个整数的比例（例如 1:24, 1:48000）或小数（例如 0.04166, 2.0833e-5）。

&nbsp;&nbsp;&nbsp;&nbsp;默认值是 0。

-bitexact (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;启用比特精确模式用于（解）复用器和（解/编）码器。

-shortest (output)
&nbsp;&nbsp;&nbsp;&nbsp;当最短的输出流结束时完成编码。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，此选项可能需要缓冲帧，这会引入额外的延迟。可以通过 -shortest_buf_duration 选项控制此延迟的最大量。

-shortest_buf_duration duration (output)
&nbsp;&nbsp;&nbsp;&nbsp;当至少有一个流是“稀疏”的（即帧之间有较大间隔——这通常是字幕的情况）时，`-shortest` 选项可能需要缓冲潜在的大量数据。

&nbsp;&nbsp;&nbsp;&nbsp;此选项控制缓冲帧的最大持续时间（以秒为单位）。较大的值可能允许 `-shortest` 选项产生更准确的结果，但会增加内存使用和延迟。

&nbsp;&nbsp;&nbsp;&nbsp;默认值是 10 秒。

-dts_delta_threshold threshold
&nbsp;&nbsp;&nbsp;&nbsp;时间戳不连续性差值阈值，表示为秒的小数值。

&nbsp;&nbsp;&nbsp;&nbsp;由这个选项启用的时间戳不连续性校正仅应用于接受时间戳不连续性的输入格式（对于这些格式启用了 `AVFMT_TS_DISCONT` 标志），例如 MPEG-TS 和 HLS，并且在使用 `-copyts` 选项时自动禁用（除非检测到环绕）。

&nbsp;&nbsp;&nbsp;&nbsp;如果检测到绝对值大于阈值的时间戳不连续性，ffmpeg 将通过减去/加上相应的 delta 值来移除不连续性。

&nbsp;&nbsp;&nbsp;&nbsp;默认值是 10。

-dts_error_threshold threshold
&nbsp;&nbsp;&nbsp;&nbsp;时间戳错误差值阈值，表示为秒的小数值。

&nbsp;&nbsp;&nbsp;&nbsp;由这个选项启用的时间戳校正仅应用于不接受时间戳不连续性的输入格式（对于这些格式未启用 `AVFMT_TS_DISCONT` 标志）。

&nbsp;&nbsp;&nbsp;&nbsp;如果检测到绝对值大于阈值的时间戳不连续性，ffmpeg 将丢弃 PTS/DTS 时间戳值。

&nbsp;&nbsp;&nbsp;&nbsp;默认值是 `3600*30`（30 小时），这是任意选择且相当保守的。

-muxdelay seconds (output)
&nbsp;&nbsp;&nbsp;&nbsp;设置最大的解复用-解码延迟。

-muxpreload seconds (output)
&nbsp;&nbsp;&nbsp;&nbsp;设置初始的解复用-解码延迟。

-streamid output-stream-index:new-value (output)
&nbsp;&nbsp;&nbsp;&nbsp;为输出流分配一个新的流ID值。此选项应在适用的输出文件名之前指定。在存在多个输出文件的情况下，可以将流ID重新分配给不同的值。

&nbsp;&nbsp;&nbsp;&nbsp;例如，要将输出 mpegts 文件中的流 0 PID 设置为 33 和流 1 PID 设置为 36：
```shell
ffmpeg -i inurl -streamid 0:33 -streamid 1:36 out.ts
```

-bsf[:stream_specifier] bitstream_filters (input/output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;对匹配的流应用比特流滤镜。这些滤镜在从解复用器接收每个数据包时（作为输入选项使用）或在发送到复用器之前（作为输出选项使用）应用于每个数据包。

&nbsp;&nbsp;&nbsp;&nbsp;bitstream_filters 是以逗号分隔的比特流滤镜规范列表，每个规范的形式如下：
```shell
filter[=optname0=optval0:optname1=optval1:...]
```
&nbsp;&nbsp;&nbsp;&nbsp;需要成为选项值一部分的任何 ‘,=:’ 字符需要用反斜杠转义。

&nbsp;&nbsp;&nbsp;&nbsp;使用 `-bsfs` 选项获取比特流滤镜列表。

&nbsp;&nbsp;&nbsp;&nbsp;例如：
```shell
ffmpeg -bsf:v h264_mp4toannexb -i h264.mp4 -c:v copy -an out.h264
```
&nbsp;&nbsp;&nbsp;&nbsp;将 `h264_mp4toannexb` 比特流滤镜（它将 MP4 封装的 H.264 流转换为附录 B 格式）应用于输入视频流。

&nbsp;&nbsp;&nbsp;&nbsp;另一方面，
```shell
ffmpeg -i file.mov -an -vn -bsf:s mov2textsub -c:s copy -f rawvideo sub.txt
```
&nbsp;&nbsp;&nbsp;&nbsp;将 `mov2textsub` 比特流滤镜（它从 MOV 字幕中提取文本）应用于输出字幕流。但是请注意，由于两个示例都使用了 `-c copy`，因此滤镜是在输入还是输出应用影响不大——如果发生了转码则会有所不同。

-tag[:stream_specifier] codec_tag (input/output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;为匹配的流强制指定一个标签/fourcc。

-timecode hh:mm:ssSEPff
&nbsp;&nbsp;&nbsp;&nbsp;指定用于写入的时间码。SEP 在非丢帧时间码时是 ‘:’，而在丢帧时是 ‘;’（或 ‘.’）。
```shell
ffmpeg -i input.mpg -timecode 01:02:03.04 -r 30000/1001 -s ntsc output.mpg
```

-filter_complex filtergraph (global)
&nbsp;&nbsp;&nbsp;&nbsp;定义一个复杂滤镜图，即具有任意数量输入和/或输出的滤镜图。对于简单滤镜图（即那些具有一个输入和一个输出且类型相同的情况），请参阅 -filter 选项。filtergraph 是对滤镜图的描述，如 ffmpeg-filters 手册中“滤镜图语法”部分所述。此选项可以多次指定——每次使用都会创建一个新的复杂滤镜图。

&nbsp;&nbsp;&nbsp;&nbsp;复杂滤镜图的输入可能来自不同的源类型，通过相应链接标签的格式加以区分：

- 要连接输入流，请使用 [file_index:stream_specifier]（即与 -map 相同的语法）。如果 stream_specifier 匹配多个流，则使用第一个匹配的流。对于多视图视频，流说明符后面可以跟随视图说明符，详见 -map 选项文档中的语法。
- 要连接回环解码器，请使用 [dec:dec_idx]，其中 dec_idx 是要连接给定输入的回环解码器索引。对于多视图视频，解码器索引后面可以跟随视图说明符，详见 -map 选项文档中的语法。
- 要连接另一个复杂滤镜图的输出，请使用其链接标签。例如，以下示例：
    ```shell
    ffmpeg -i input.mkv \
    -filter_complex '[0:v]scale=size=hd1080,split=outputs=2[for_enc][orig_scaled]' \
    -c:v libx264 -map '[for_enc]' output.mkv \
    -dec 0:0 \
    -filter_complex '[dec:0][orig_scaled]hstack[stacked]' \
    -map '[stacked]' -c:v ffv1 comparison.mkv
    ```
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;读取输入视频并

    - （第2行）使用一个具有一个输入和两个输出的复杂滤镜图将视频缩放到1920x1080，并将结果复制到两个输出；
    - （第3行）使用 `libx264` 编码一个已缩放的输出，并将结果写入 output.mkv；
    - （第4行）使用回环解码器解码该编码的流；
    - （第5行）将回环解码器的输出（即 `libx264` 编码的视频）与原始缩放后的输入并排放置；
    - （第6行）然后将组合的视频无损编码并写入 comparison.mkv。
    请注意，由于转码管道中存在循环（滤镜图输出去编码，从那里解码，再回到同一个图），这两个滤镜图不能合并成一个，这是不允许的。

&nbsp;&nbsp;&nbsp;&nbsp;未标记的输入将被连接到第一个未使用的匹配类型的输入流。

&nbsp;&nbsp;&nbsp;&nbsp;输出链接标签用 -map 引用。未标记的输出将被添加到第一个输出文件。

&nbsp;&nbsp;&nbsp;&nbsp;请注意，使用此选项可以在没有正常输入文件的情况下仅使用 lavfi 源。

&nbsp;&nbsp;&nbsp;&nbsp;例如，要将图像叠加在视频上：
```shell
ffmpeg -i video.mkv -i image.png -filter_complex '[0:v][1:v]overlay[out]' -map
'[out]' out.mkv
```
&nbsp;&nbsp;&nbsp;&nbsp;这里 `[0:v]` 指向第一个输入文件中的第一个视频流，它连接到 overlay 滤镜的第一个（主）输入。同样地，第二个输入中的第一个视频流连接到 overlay 的第二个（叠加）输入。

&nbsp;&nbsp;&nbsp;&nbsp;假设每个输入文件中只有一个视频流，我们可以省略输入标签，因此上述命令等价于：
```shell
ffmpeg -i video.mkv -i image.png -filter_complex 'overlay[out]' -map
'[out]' out.mkv
```
&nbsp;&nbsp;&nbsp;&nbsp;此外，我们可以省略输出标签，滤镜图的单个输出将自动添加到输出文件，所以我们只需写：
```shell
ffmpeg -i video.mkv -i image.png -filter_complex 'overlay' out.mkv
```
&nbsp;&nbsp;&nbsp;&nbsp;作为特殊情况，您可以使用位图字幕流作为输入：它将被转换为与文件中最大的视频相同大小的视频，如果没有视频则为 720x576。请注意，这是一个实验性和临时性的解决方案。一旦 libavfilter 对字幕有适当支持，这个特性将被移除。

&nbsp;&nbsp;&nbsp;&nbsp;例如，为了将字幕硬编码到顶部，延迟1秒的 DVB-T 录像（存储在 MPEG-TS 格式中）：
```shell
ffmpeg -i input.ts -filter_complex \
  '[#0x2ef] setpts=PTS+1/TB [sub] ; [#0x2d0] [sub] overlay' \
  -sn -map '#0x2dc' output.mkv
```
&nbsp;&nbsp;&nbsp;&nbsp;(0x2d0、0x2dc 和 0x2ef 分别是视频、音频和字幕流的 MPEG-TS PID；0:0、0:3 和 0:7 也可以工作)

&nbsp;&nbsp;&nbsp;&nbsp;为了使用 lavfi `color` 源生成5秒纯红色视频：
```shell
ffmpeg -filter_complex 'color=c=red' -t 5 out.mkv
```

-filter_complex_threads nb_threads (global)
&nbsp;&nbsp;&nbsp;&nbsp;定义用于处理 -filter_complex 图的线程数。类似于 filter_threads 但仅用于 `-filter_complex` 图。默认值是可用的CPU数量。

-lavfi filtergraph (global)
&nbsp;&nbsp;&nbsp;&nbsp;定义一个复杂滤镜图，即具有任意数量输入和/或输出的滤镜图。等价于 -filter_complex。

-accurate_seek (input)
&nbsp;&nbsp;&nbsp;&nbsp;此选项启用或禁用使用 -ss 选项时在输入文件中的精确查找功能。默认是启用的，所以在转码时查找是精确的。使用 -noaccurate_seek 禁用它，这可能在复制一些流并转码其他流时有用。

-seek_timestamp (input)
&nbsp;&nbsp;&nbsp;&nbsp;此选项启用或禁用使用 -ss 选项时在输入文件中按时间戳查找的功能。默认是禁用的。如果启用，-ss 选项的参数将被视为实际的时间戳，并不会被文件的开始时间所偏移。这对于不从时间戳0开始的文件（如传输流）很重要。

-thread_queue_size size (input/output)
&nbsp;&nbsp;&nbsp;&nbsp;对于输入，此选项设置从文件或设备读取时的最大排队数据包数量。对于低延迟/高频率的直播流，如果数据包未能及时读取可能会被丢弃；设置这个值可以强制 ffmpeg 使用单独的输入线程并在数据包到达时立即读取。默认情况下，ffmpeg 仅在指定了多个输入时这样做。

&nbsp;&nbsp;&nbsp;&nbsp;对于输出，此选项指定可以排队到每个复用线程的最大数据包数量。

-sdp_file file (global)
&nbsp;&nbsp;&nbsp;&nbsp;将 SDP 信息打印到文件，针对输出流。这允许在至少一个输出不是 RTP 流的情况下转储 SDP 信息。（需要至少一个输出格式为 RTP）。

-discard (input)
&nbsp;&nbsp;&nbsp;&nbsp;允许丢弃特定的流或流中的帧。任何输入流都可以完全丢弃，使用值 `all`；而选择性地丢弃流中的帧发生在解复用器阶段，并不是所有解复用器都支持。

&nbsp;&nbsp;&nbsp;&nbsp;none
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;不丢弃任何帧。

&nbsp;&nbsp;&nbsp;&nbsp;default
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;默认，不丢弃任何帧。

&nbsp;&nbsp;&nbsp;&nbsp;noref
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;丢弃所有非参考帧。

&nbsp;&nbsp;&nbsp;&nbsp;bidir
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;丢弃所有双向帧。

&nbsp;&nbsp;&nbsp;&nbsp;nokey
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;除关键帧外丢弃所有帧。

&nbsp;&nbsp;&nbsp;&nbsp;all
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;丢弃所有帧。

-abort_on flags (global)
&nbsp;&nbsp;&nbsp;&nbsp;在各种条件下停止并中止。可用的标志有：

&nbsp;&nbsp;&nbsp;&nbsp;empty_output
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;没有数据包传递给复用器，输出为空。

&nbsp;&nbsp;&nbsp;&nbsp;empty_output_stream
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在某些输出流中没有数据包传递给复用器。

-max_error_rate (global)
&nbsp;&nbsp;&nbsp;&nbsp;设置跨所有输入的解码帧失败比例，当超过此比例时 ffmpeg 将返回退出代码69。超过这个阈值并不会终止处理。范围是一个介于0到1之间的浮点数，默认值是2/3。

-xerror (global)
&nbsp;&nbsp;&nbsp;&nbsp;遇到错误时停止并退出。

-max_muxing_queue_size packets (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;在转码音频和/或视频流时，ffmpeg 不会在每个这样的流都有一个数据包之前开始写入输出。等待期间，其他流的数据包会被缓冲。此选项设置匹配输出流的该缓冲区大小，以数据包为单位。

&nbsp;&nbsp;&nbsp;&nbsp;此选项的默认值对于大多数用途来说已经足够大，因此只有在确定需要时才应更改此选项。

-muxing_queue_data_threshold bytes (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;这是直到哪个最小阈值为止复用队列大小不被考虑。默认每流50兆字节，并基于传递给复用器的数据包的整体大小。

-auto_conversion_filters (global)
&nbsp;&nbsp;&nbsp;&nbsp;在所有滤镜图中自动插入格式转换滤镜，包括由 -vf、-af、-filter_complex 和 -lavfi 定义的那些。如果滤镜格式协商需要转换，则滤镜初始化将会失败。仍然可以通过在图中插入相应的转换滤镜（scale、aresample）来进行转换。默认是开启的，要明确禁用它，你需要指定 -noauto_conversion_filters。

-bits_per_raw_sample[:stream_specifier] value (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;声明给定输出流的原始样本位数为 value。请注意，此选项设置的是提供给编码器/复用器的信息，它不会改变流以符合这个值。设置与流属性不匹配的值可能导致编码失败或无效的输出文件。

-stats_enc_pre[:stream_specifier] path (output, per-stream)
-stats_enc_post[:stream_specifier] path (output, per-stream)
-stats_mux_pre[:stream_specifier] path (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;将匹配流的每帧编码信息写入由 path 指定的文件。

&nbsp;&nbsp;&nbsp;&nbsp;-stats_enc_pre 写入关于原始视频或音频帧的信息，这些帧即将被发送进行编码；而 -stats_enc_post 写入关于从编码器接收到的已编码数据包的信息。-stats_mux_pre 写入关于即将发送到复用器的数据包的信息。每个帧或数据包在指定文件中产生一行。此行的格式由 -stats_enc_pre_fmt / -stats_enc_post_fmt / -stats_mux_pre_fmt 控制。

&nbsp;&nbsp;&nbsp;&nbsp;当多个流的统计信息写入同一个文件时，不同流对应的行将会交错。这种交错的确切顺序未作规定，并且不能保证在程序的不同调用之间保持稳定，即使使用相同的选项也是如此。

-stats_enc_pre_fmt[:stream_specifier] format_spec (output, per-stream)
-stats_enc_post_fmt[:stream_specifier] format_spec (output, per-stream)
-stats_mux_pre_fmt[:stream_specifier] format_spec (output, per-stream)
&nbsp;&nbsp;&nbsp;&nbsp;指定与 -stats_enc_pre / -stats_enc_post / -stats_mux_pre 一起写入的行的格式。

&nbsp;&nbsp;&nbsp;&nbsp;format_spec 是一个可能包含形式为 {fmt} 的指令的字符串。format_spec 使用反斜杠转义——分别使用 \{, \}, 和 \\ 在输出中写入实际的 {, }, 或 \。

&nbsp;&nbsp;&nbsp;&nbsp;给出的 fmt 指令可以是以下之一：

&nbsp;&nbsp;&nbsp;&nbsp;fidx
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;输出文件的索引。

&nbsp;&nbsp;&nbsp;&nbsp;sidx
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;文件中输出流的索引。

&nbsp;&nbsp;&nbsp;&nbsp;n
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧号。编码前：到目前为止发送给编码器的帧数。编码后：到目前为止从编码器接收的数据包数。复用：到目前为止为此流提交给复用器的数据包数。

&nbsp;&nbsp;&nbsp;&nbsp;ni
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;输入帧号。对应于此输出帧或数据包的输入帧（即解码器输出）的索引。如果不可用则为 -1。

&nbsp;&nbsp;&nbsp;&nbsp;tb
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表示该帧/数据包的时间戳的时间基，作为有理数 num/den。请注意，编码器和复用器可能会使用不同的时间基。

&nbsp;&nbsp;&nbsp;&nbsp;tbi
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ptsi 的时间基，作为有理数 num/den。当 ptsi 可用时可用，否则为 0/1。

&nbsp;&nbsp;&nbsp;&nbsp;pts
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧或数据包的显示时间戳，以整数表示。应乘以时间基以计算显示时间。

&nbsp;&nbsp;&nbsp;&nbsp;ptsi
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;输入帧（见 ni）的显示时间戳，以整数表示。应乘以 tbi 以计算显示时间。不可用时打印为 (2^63 - 1 = 9223372036854775807)。

&nbsp;&nbsp;&nbsp;&nbsp;t
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧或数据包的显示时间，以十进制数表示。等于 pts 乘以 tb。

&nbsp;&nbsp;&nbsp;&nbsp;ti
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;输入帧（见 ni）的显示时间，以十进制数表示。等于 ptsi 乘以 tbi。不可用时打印为 inf。

&nbsp;&nbsp;&nbsp;&nbsp;dts (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据包的解码时间戳，以整数表示。应乘以时间基以计算显示时间。

&nbsp;&nbsp;&nbsp;&nbsp;dt (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧或数据包的解码时间，以十进制数表示。等于 dts 乘以 tb。

&nbsp;&nbsp;&nbsp;&nbsp;sn (frame, audio)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到目前为止发送给编码器的音频样本数。

&nbsp;&nbsp;&nbsp;&nbsp;samp (frame, audio)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;帧中的音频样本数。

&nbsp;&nbsp;&nbsp;&nbsp;size (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;编码后的数据包大小，以字节为单位。

&nbsp;&nbsp;&nbsp;&nbsp;br (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前比特率，以每秒比特数表示。

&nbsp;&nbsp;&nbsp;&nbsp;abr (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到目前为止整个流的平均比特率，以每秒比特数表示，如果此时无法确定则为 -1。

&nbsp;&nbsp;&nbsp;&nbsp;key (packet)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果数据包包含关键帧，则字符为 ‘K’，否则为 ‘N’。

&nbsp;&nbsp;&nbsp;&nbsp;标记为 packet 的指令只能与 -stats_enc_post_fmt 和 -stats_mux_pre_fmt 一起使用。

&nbsp;&nbsp;&nbsp;&nbsp;标记为 frame 的指令只能与 -stats_enc_pre_fmt 一起使用。

&nbsp;&nbsp;&nbsp;&nbsp;标记为 audio 的指令只能用于音频流。

&nbsp;&nbsp;&nbsp;&nbsp;默认的格式字符串是：

&nbsp;&nbsp;&nbsp;&nbsp;pre-encoding
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{fidx} {sidx} {n} {t}

&nbsp;&nbsp;&nbsp;&nbsp;post-encoding
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{fidx} {sidx} {n} {t}

&nbsp;&nbsp;&nbsp;&nbsp;将来，新项可能会添加到默认格式化字符串的末尾。依赖于格式完全不变的用户应该手动指定格式。

&nbsp;&nbsp;&nbsp;&nbsp;注意，写入同一文件的不同流的统计信息可能有不同的格式。

### 5.12 Preset files

预设文件包含一系列 option=value 对，每个选项占一行，指定了在命令行上指定可能较为复杂的选项序列。以井号（'#'）字符开头的行被忽略，用于提供注释。可以在 FFmpeg 源树中的预设目录检查示例。

预设文件有两种类型：ffpreset 和 avpreset 文件。

#### 5.12.1 ffpreset files

ffpreset 文件可以通过 `vpre`（视频预设）、`apre`（音频预设）、`spre`（字幕预设）和 `fpre`（通用预设）选项来指定。`fpre` 选项接受预设文件名作为输入，而不是预设名称，并且可以用于任何类型的编解码器。对于 `vpre`、`apre` 和 `spre` 选项，预设文件中指定的选项将应用于当前选定的与预设选项类型相同的编解码器。

传递给 `vpre`、`apre` 和 `spre` 预设选项的参数根据以下规则确定要使用的预设文件：

首先，FFmpeg 会按照顺序在 $FFMPEG_DATADIR（如果已设置）、$HOME/.ffmpeg 和配置时定义的数据目录（通常是 PREFIX/share/ffmpeg）或在 win32 上与可执行文件一起的 ffpresets 文件夹中搜索名为 arg.ffpreset 的文件。例如，如果参数是 `libvpx-1080p`，则它将搜索名为 libvpx-1080p.ffpreset 的文件。

如果没有找到这样的文件，那么 FFmpeg 将在上述目录中搜索名为 codec_name-arg.ffpreset 的文件，其中 codec_name 是将应用预设文件选项的编解码器名称。例如，如果您使用 `-vcodec libvpx` 选择视频编解码器并使用 `-vpre 1080p`，则它将搜索名为 libvpx-1080p.ffpreset 的文件。

#### 5.12.2 avpreset files

avpreset 文件通过 `pre` 选项来指定。它们的工作方式类似于 ffpreset 文件，但仅允许特定于编码器的选项。因此，指定编码器的 option=value 对是不允许的。

当指定了 `pre` 选项时，FFmpeg 将按照顺序在 $AVCONV_DATADIR（如果已设置）、$HOME/.avconv 和配置时定义的数据目录（通常是 PREFIX/share/ffmpeg）中查找带有 .avpreset 后缀的文件。

首先，FFmpeg 会在上述目录中搜索名为 codec_name-arg.avpreset 的文件，其中 codec_name 是将应用预设文件选项的编解码器名称。例如，如果您使用 `-vcodec libvpx` 选择视频编解码器并使用 `-pre 1080p`，则它将搜索名为 libvpx-1080p.avpreset 的文件。

如果没有找到这样的文件，那么 FFmpeg 将在同一目录中搜索名为 arg.avpreset 的文件。

### 5.13 vstats file format

`-vstats` 和 `-vstats_file` 选项启用生成包含关于生成的视频输出统计信息的文件。

`-vstats_version` 选项控制生成文件的格式版本。

对于版本`1`，格式如下：
```shell
frame= FRAME q= FRAME_QUALITY PSNR= PSNR f_size= FRAME_SIZE s_size= STREAM_SIZEkB time= TIMESTAMP br= BITRATEkbits/s avg_br= AVERAGE_BITRATEkbits/s
```
对于版本`2`，格式如下：
```shell
out= OUT_FILE_INDEX st= OUT_FILE_STREAM_INDEX frame= FRAME_NUMBER q= FRAME_QUALITYf PSNR= PSNR f_size= FRAME_SIZE s_size= STREAM_SIZEkB time= TIMESTAMP br= BITRATEkbits/s avg_br= AVERAGE_BITRATEkbits/s
```
每个键对应的值描述如下：

avg_br
&nbsp;&nbsp;&nbsp;&nbsp;平均比特率，以 Kbits/s 表示

br
&nbsp;&nbsp;&nbsp;&nbsp;比特率，以 Kbits/s 表示

frame
&nbsp;&nbsp;&nbsp;&nbsp;已编码帧的数量（版本1）或帧编号（版本2）

out
&nbsp;&nbsp;&nbsp;&nbsp;输出文件索引（仅版本2）

PSNR
&nbsp;&nbsp;&nbsp;&nbsp;峰值信噪比

q
&nbsp;&nbsp;&nbsp;&nbsp;帧的质量

f_size
&nbsp;&nbsp;&nbsp;&nbsp;已编码数据包大小，以字节数表示

s_size
&nbsp;&nbsp;&nbsp;&nbsp;流大小，以 KiB 表示

st
&nbsp;&nbsp;&nbsp;&nbsp;输出流索引（仅版本2）

time
&nbsp;&nbsp;&nbsp;&nbsp;数据包的时间

type
&nbsp;&nbsp;&nbsp;&nbsp;图像类型（注：在提供的格式中未提及，但通常也是 vstats 的一部分）

另请参见 -stats_enc 选项，以获取显示编码统计信息的另一种方式。

## 6 Examples

### 6.1 Video and Audio grabbing

如果您指定了输入格式和设备，则 ffmpeg 可以直接抓取视频和音频。
```shell
ffmpeg -f oss -i /dev/dsp -f video4linux2 -i /dev/video0 /tmp/out.mpg
```
或者使用 ALSA（Advanced Linux Sound Architecture）音频源（单声道输入，声卡 ID 为 1）代替 OSS：
```shell
ffmpeg -f alsa -ac 1 -i hw:1 -f video4linux2 -i /dev/video0 /tmp/out.mpg
```
请注意，在用任何电视查看器（如 Gerd Knorr 开发的 xawtv）启动 ffmpeg 之前，您必须激活正确的视频源和频道。您还需要使用标准混音器正确设置音频录制级别。

### 6.2 X11 grabbing

通过 FFmpeg 使用 X11grab 捕获 X11 显示：
```shell
ffmpeg -f x11grab -video_size cif -framerate 25 -i :0.0 /tmp/out.mpg
```
这里的 `:0.0` 是您的 X11 服务器的显示.屏幕编号，与 DISPLAY 环境变量相同。

```shell
ffmpeg -f x11grab -video_size cif -framerate 25 -i :0.0+10,20 /tmp/out.mpg
```
这里的 `:0.0` 同样是您的 X11 服务器的显示.屏幕编号。`10` 是 x 偏移量，而 `20` 是 y 偏移量，用于确定捕获区域的起始点。

### 6.3 Video and Audio file format conversion

任何支持的文件格式和协议都可以作为 ffmpeg 的输入：

示例：

- 您可以使用 YUV 文件作为输入：
    ```shell
    ffmpeg -i /tmp/test%d.Y /tmp/out.mpg
    ```
    这将使用以下文件：
    ```shell
    /tmp/test0.Y, /tmp/test0.U, /tmp/test0.V,
    /tmp/test1.Y, /tmp/test1.U, /tmp/test1.V 等...
    ```
    Y 文件的分辨率是 U 和 V 文件的两倍。它们是原始文件，没有头部信息。这些文件可以由所有优秀的视频解码器生成。如果 FFmpeg 无法猜测图像大小，您必须使用 `-s` 选项指定图像大小。

- 您可以从原始 YUV420P 文件中输入：
    ```shell
    ffmpeg -i /tmp/test.yuv /tmp/out.avi
    ```
    test.yuv 是一个包含原始 YUV 平面数据的文件。每个帧由 Y 平面组成，后面跟着水平和垂直分辨率减半的 U 和 V 平面。

- 您可以输出到原始 YUV420P 文件：
    ```shell
    ffmpeg -i mydivx.avi hugefile.yuv
    ```

- 您可以设置多个输入文件和输出文件：
    ```shell
    ffmpeg -i /tmp/a.wav -s 640x480 -i /tmp/a.yuv /tmp/a.mpg
    ```
    将音频文件 a.wav 和原始 YUV 视频文件 a.yuv 转换为 MPEG 文件 a.mpg。

- 您也可以同时进行音频和视频转换：
    ```shell
    ffmpeg -i /tmp/a.wav -ar 22050 /tmp/a.mp2
    ```
    将 a.wav 转换为采样率为 22050 Hz 的 MPEG 音频。

- 您可以同时编码为多种格式，并定义输入流到输出流的映射：
    ```shell
    ffmpeg -i /tmp/a.wav -map 0:a -b:a 64k /tmp/a.mp2 -map 0:a -b:a 128k /tmp/b.mp2
    ```
    将 a.wav 转换为 a.mp2（64 kbit/s）和 b.mp2（128 kbit/s）。`-map file:index` 指定哪个输入流用于每个输出流，按输出流定义的顺序。

- 您可以转码解密的 VOB 文件：
    ```shell
    ffmpeg -i snatch_1.vob -f avi -c:v mpeg4 -b:v 800k -g 300 -bf 2 -c:a libmp3lame -b:a 128k snatch.avi
    ```
    这是一个典型的 DVD 撕取示例；输入是一个 VOB 文件，输出是一个具有 MPEG-4 视频和 MP3 音频的 AVI 文件。注意，在这个命令中我们使用 B 帧使 MPEG-4 流兼容 DivX5，并且 GOP 大小为 300，这意味着对于 29.97 fps 的输入视频每 10 秒有一个 I 帧。此外，音频流被 MP3 编码，因此需要通过传递 `--enable-libmp3lame` 给 configure 来启用 LAME 支持。映射对于从 DVD 转码以获取所需的音频语言特别有用。

注意：要查看支持的输入格式，请使用 `ffmpeg -demuxers`。

- 您可以从视频中提取图像，或从许多图像创建视频：

    对于从视频中提取图像：
    ```shell
    ffmpeg -i foo.avi -r 1 -s WxH -f image2 foo-%03d.jpeg
    ```
    这将从视频中每秒提取一帧，并将它们输出到名为 foo-001.jpeg, foo-002.jpeg 等的文件中。图像将被重新缩放以适应新的 WxH 尺寸。

    如果您只想提取有限数量的帧，可以结合使用上述命令与 `-frames:v` 或 `-t` 选项，或者与 -ss 结合以从某个时间点开始提取。

    对于从许多图像创建视频：
    ```shell
    ffmpeg -f image2 -framerate 12 -i foo-%03d.jpeg -s WxH foo.avi
    ```
    语法 `foo-%03d.jpeg` 表示使用由三个数字组成的十进制数（用零填充），以表示序列号。它使用的是 C printf 函数支持的相同语法，但只适合接受普通整数的格式。

    当导入图像序列时，-i 还支持内部扩展类似 shell 的通配符模式（globbing），通过选择特定于 image2 的 `-pattern_type glob` 选项。

    例如，从匹配通配符模式 `foo-*.jpeg` 的文件名创建视频：
    ```shell
    ffmpeg -f image2 -pattern_type glob -framerate 12 -i 'foo-*.jpeg' -s WxH foo.avi
    ```
- 您可以将多个相同类型的流放入输出中：
    ```shell
    ffmpeg -i test1.avi -i test2.avi -map 1:1 -map 1:0 -map 0:1 -map 0:0 -c copy -y test12.nut
    ```
    结果输出文件 test12.nut 将包含来自输入文件的前四个流，按相反顺序排列。

- 为了强制恒定比特率 (CBR) 视频输出：
    ```shell
    ffmpeg -i myfile.avi -b 4000k -minrate 4000k -maxrate 4000k -bufsize 1835k out.m2v
    ```
- 四个选项 lmin, lmax, mblmin 和 mblmax 使用 ‘lambda’ 单位，但您可以使用 QP2LAMBDA 常量轻松地从 ‘q’ 单位转换：
    ```shell
    ffmpeg -i src.ext -lmax 21*QP2LAMBDA dst.ext
    ```

## 7 See Also

ffmpeg-all, ffplay, ffprobe, ffmpeg-utils, ffmpeg-scaler, ffmpeg-resampler, ffmpeg-codecs, ffmpeg-bitstream-filters, ffmpeg-formats, ffmpeg-devices, ffmpeg-protocols, ffmpeg-filters

## Authors

FFmpeg 的开发者们。

有关作者详情，请参阅项目的 Git 历史记录 (https://git.ffmpeg.org/ffmpeg)，例如，您可以通过在 FFmpeg 源代码目录中输入命令 `git log` 来查看，或者在线浏览仓库 https://git.ffmpeg.org/ffmpeg。

特定组件的维护者名单列在源代码树中的 MAINTAINERS 文件里。

本文档于 2025 年 1 月 3 日使用 makeinfo 生成。
