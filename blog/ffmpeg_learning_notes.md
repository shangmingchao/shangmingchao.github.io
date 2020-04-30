# 音视频技术基础

## 概述

保存视频的每一帧，每一个像素没要必要，而且也是不现实的，因为这个数据量太大了，以至于没办法存储和传输，比如说，一个视频大小是 1280×720 像素，一个像素占 12 个比特位，每秒 30 帧，那么一分钟这样的视频就要占 1280×720×12×30×60/8/1024/1024=2.3GB 的空间，所以视频数据肯定要进行压缩存储和传输的。  
而可以压缩的冗余数据有很多，从空间上来说，一帧图像中的像素之间并不是毫无关系的，相邻像素有很强的相关性，可以利用这些相关性抽象地存储。同样在时间上，相邻的视频帧之间内容相似，也可以压缩。每个像素值出现的概率不同，从编码上也可以压缩。人类视觉系统（HVS）对高频信息不敏感，所以可以丢弃高频信息，只编码低频信息。对高对比度更敏感，可以提高边缘信息的主观质量。对亮度信息比色度信息更敏感，可以降低色度的解析度。对运动的信息更敏感，可以对感兴趣区域（ROI）进行特殊处理。  
视频数据压缩和传输的实现与最终将这些数据还原成视频播放出来的实现是紧密相关的，也就是说视频信息的压缩和解压缩需要一个统一标准，即音视频编码标准。  

## 视频编码

制定音视频编码标准的有两个组织机构，一个是国际电联下属的机构 ITU-T（ITU Telecommunication Standardization Sector），一个是国际标准化组织 ISO 和国际电工委员会 IEC 下属的 MPEG（Moving Picture Experts Group） 专家组。  
1988 年，ITU-T 制定了第一个实用的视频编码标准 [H.261](https://en.wikipedia.org/wiki/H.261)，这也是第一个 H.26x 家族的视频编码标准，之后的一些视频编码标准大多都是以此为基础的。它的的基本处理单元称为宏块，H.261 是宏块概念出现的第一个标准。每个宏块由 16×16 阵列的亮度样本和两个对应的 8×8 色度样本阵列组成，使用 4:2:0 采样和 YCbCr 色彩空间。编码算法使用运动补偿的图片间预测和空间变换编码的混合，涉及标量量化，Z 字形扫描和熵编码。  
1993 年，ISO/IEC 制定了有损压缩标准 [MPEG-1](https://en.wikipedia.org/wiki/MPEG-1)，其中最著名的部分是它引入的 MP3 音频格式。  
2003 年，ITU-T 和 MPEG 共同组成的 JVT（Joint Video Team）联合视频小组开发了优秀且广为流行的 [H.264](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC) 标准，该标准既是 ITU-T 的 H.264 标准，也是 MPEG-4 的第十部分（第十部分也叫 AVC（Advanced Video Coding）），所以 H.264/AVC, AVC/H.264, H.264/MPEG-4 AVC, MPEG-4/H.264 AVC 都是指 H.264。而之后的 [HEVC](https://en.wikipedia.org/wiki/High_Efficiency_Video_Coding)（High Efficiency Video Coding）视频压缩标准既是指 H.265 也是指 MPEG-H 第二部分。  
2003 年，微软基于 WMV9（Windows Media Video 9）格式开发了视频编码标准 [VC-1](https://en.wikipedia.org/wiki/VC-1)。  
2008 年，Google 基于 VP7 开源了 [VP8](https://en.wikipedia.org/wiki/VP8) 视频压缩格式。 VP8 可以与 Vorbis 和 Opus 音频一起多路复用到基于 Matroska 的容器格式 WebM 中。图像格式 WebP 基于 VP8 的帧内编码。之后的 VP9 和 AOMedia（Alliance for Open Media）开发的 AV1（AOMedia Video 1）都是基于 VP8 的。这个系列编码标准的最大优势是它是开放的，免版权税的。  

## 术语

### 多媒体容器格式（封装格式）

一个多媒体文件或者多媒体流可能包含多个视频、音频、字幕、同步信息，章节信息以及元数据等数据。也就是说我们通常看到的 .mp4 、.avi、.rmvb 等文件中的 MP4、AVI 其实是一种容器格式（container formats），用来封装这些数据，而不是视频的编码格式。  

### muxer 和 demuxer

muxer 就是用来封装多媒体容器格式的封装器，比如把一个 rmvb 视频文件，mp3 音频文件以及 srt 字幕文件，封装成为一个新的 mp4 文件。而 demuxer 就是解封装器，可以将容器格式分解成视频流、音频流、附加数据等信息。  

### Codec

编解码器，是编码器（Encoder）和 解码器（Decoder）的统称。  

### I 帧

Intra-frame，也被称为 I-pictures 或 keyframes，也就是说俗称的关键帧，是指不依赖于其他任何帧进行渲染的视频帧，简单呈现一个固定图像。两个关键帧之间的视频帧是可以预测计算出来的，但两个 I 帧之间的帧数不可能特别大，因为解码的复杂度，解码器缓冲区大小，数据错误后的恢复时间，搜索能力以及在硬件解码器中最常见的低精度实现中 IDCT 错误的累积，限制了 I 帧之间的最大帧数。  

### P 帧

Predicted-frame，也被称为向前预测帧或帧间帧，仅存储与紧邻它的前一个帧（I 帧或 P 帧，这个参考帧也称为锚帧）的图像差异。使用帧的每个宏块上的运动矢量计算 P 帧与其锚帧之间的差异，这种运动矢量数据将嵌入 P 帧中以供解码器使用。除了任何前向预测的块之外，P 帧还可以包含任意数量的帧内编码块。如果视频从一帧到下一帧（例如剪辑）急剧变化，则将其编码为 I 帧会更有效。如果 P 帧丢失，视频画面可能会出现花屏或者马赛克的现象。  

### B 帧

Bidirectional-frame，代表双向帧，也被称为向后预测帧或 B-pictures。 B 帧与 P 帧非常相似，B 帧可以使用前一帧和后一帧（即两个锚帧）进行预测。因此，在可以解码和显示 B 帧之前，播放器必须首先在 B 帧之后顺序解码下一个 I 或 P 锚帧。这意味着解码 B 帧需要更大的数据缓冲器，并导致解码和编码期间的延迟增加。这还需要容器/系统流中的解码时间戳（DTS）特征。因此，B 帧长期以来一直备受争议，它们通常在视频中被避免，有时硬件解码器不能完全支持它们。不存在从 B 帧 预测的帧的，因此，可以在需要时插入非常低比特率的 B 帧，以帮助控制比特率。如果这是用 P 帧完成的，则可以从中预测未来的 P 帧，并且会降低整个序列的质量。除了向后预测或双向预测的块之外，B帧还可以包含任意数量的帧内编码块和前向预测块。  

### NAL 和 VCL

网络抽象层 NAL（Network Abstraction Layer）和 视频编码层 VCL（Video Coding Layer）是 H.264/AVC 和 HEVC 标准的一部分，NAL 的主要目的是对访问“会话”（视频通话）和“非会话”（存储、传播、转成媒体流）应用的网络友好的视频表示一个规定。NAL 用来格式化 VCL 的视频表示，并以适当的方式为通过各种传输层和存储介质进行的传输提供头信息。也就是说 NAL 有助于将 VCL 数据映射到传输层。  
NALU（NAL units）是已编码的视频数据用来存储和传输的基本单元，NAL 单元的前一个（H.264/AVC）或两个（HEVC）字节是 Header 字节，用来标明该 NAL 单元中数据的类型。其它字节是有效载荷。  
NAL 单元分为 VCL 和非 VCL 的 NAL 单元。VCL NAL 单元包含表示视频图像中样本值的数据，非 VCL NAL 单元包含任何相关的附加信息，例如参数集 parameter sets（可应用于大量 VCL NAL 单元的重要 header 数据）和补充增强信息 SEI（Supplemental enhancement information）（定时信息和其他可以增强解码视频信号可用性的补充数据，但对于解码视频图像中的样本的值不是必需的）。  
参数集分为两种类型:  SPS（sequence parameter sets）和 PPS（picture parameter sets）。SPS 应用于一系列连续的已编码的视频图像（即已编码视频序列），PPS 应用于已编码视频序列中一个或多个单独图像的解码。也就是说 SPS 和 PPS 将不频繁改变信息的传输和视频图像中样本值编码表示的传输分离开来。每个 VCL NAL 单元包含一个指向相关 PPS 内容的标识符，而每个 PPS 都包含一个指向相关 SPS 内容的标识符。因此仅仅通过少量数据（标识符）就可以引用大量的信息（参数集）而无需在每个 VCL NAL 单元中重复该信息了。SPS 和 PPS 可以在它们要应用的 VCL NAL 单元之前发送，并且可以重复发送以提升针对数据丢失的顽健性。  
NAL Header 字节中的 nal\_ref\_idc 用于表示当前 NALU 的重要性，值越大，越重要，解码器在解码处理不过来的时候，可以丢掉重要性为 0 的 NALU。SPS/PPS 时，nal\_ref\_idc 不可为 0。当某个图像的 slice 的 nal\_ref\_id 等于 0 时，该图像的所有片均应等 0。nal\_unit\_type 表示 NALU 的类型，7 表示这个 NALU 是 SPS，8 表示这个 NALU 是 PPS。5 表示这个 NALU 是 IDR（instantaneous decoding refresh，即 I 帧） 的 slice，1 表示这个 NALU 所在的帧是 P 帧。

### DTS 和 PTS

PS（Program Streams）指将多个打包的基本码流 PES （通常是一个音频 PES 和一个视频 PES）组合成的单个流，以确保同时传送并保持同步，PS 也被称为多路传输（multiplex）或容器格式（container format）。  
PTS（Presentation time stamps）: PS 中的 PTS 用来校正音频和视频 SCR（system clock reference）值之间的不可避免的差异（时基校正），如 PS 头中的 90 kHz PTS 值告诉解码器哪些视频 SCR 值与哪些音频 SCR 值匹配。PTS 决定了何时显示 MPEG program 的一部分，并且解码器还使用它来确定何时可以从缓冲器中丢弃数据。解码器将延迟视频或音频中的一个，直到另一个的相应片段到达并且可以被解码。  
DTS（Decoding Time Stamps）: 对于视频流中的 B 帧，必须对相邻帧进行无序编码和解码（重新排序的帧）。DTS 与 PTS 非常相似，但它不仅仅处理顺序帧，而是包含适当的时间戳，在它的锚帧（P 帧 或 I 帧）之前，告诉解码器何时解码并显示下一个 B 帧。如果视频中没有B帧，那么 PTS 和 DTS 值是相同的。  

## FFMPEG

### FFMPEG 概述

[FFMPEG](http://ffmpeg.org/) 项目是在 2000 年由法国著名程序员 Fabrice Bellard 发起的，名字是受到 MPEG 专家组的启发，前面的 “FF” 是 “fast forward” 快进的意思。FFMPEG 是一个可以录制音视频，转码音视频的格式，将音视频转成媒体流的完整的、跨平台的 **解决方案**。它是一个自由的软件项目，任何人都可以免费使用和修改，只要遵循 GPL 或者 LGPL 协议引用或公开源码就行。它中的编解码库也是 VLC 播放器所使用的核心编解码库，B 站（Bilibili）开源的 [ijkplayer](https://github.com/Bilibili/ijkplayer) 、著名的 [MPlayer](http://www.mplayerhq.hu/design7/news.html) 等基本所有主流播放器也都是基于 FFMPEG 开发的。  

### FFMPEG 使用

#### 注册编解码器

`libavcodec/allcodecs.c` 文件中的 `avcodec_register_all()` 函数用来注册所有的编解码器（包括硬件加速、视频、音频、PCM、DPCM、ADPCM、字幕、文本、外部库、解析器）。  
`libavformat/allformats.c` 文件中的 `av_register_all()` 函数中调用了 `avcodec_register_all()` 注册所有的编解码器并注册了所有 muxer 和 demuxer。  
因此使用 FFMPEG 一般都要先调用 `av_register_all()`。  

#### 打开输入流

要读取一个媒体文件，可以使用 `libavformat/utils.c` 文件中的 `avformat_open_input()` 函数:  

```c
int avformat_open_input(AVFormatContext **ps, const char *filename,
                        AVInputFormat *fmt, AVDictionary **options)
```

`ps` 包含了媒体相关的基本所有数据，随后函数中调用的 `libavformat/options.c` 文件中的 `avformat_alloc_context()` 函数会为它分配空间，而 `avformat_alloc_context()` 中会调用 `avformat_get_context_defaults()` 给 `s->io_open` 设置默认值 `io_open_default()` 函数。  
`filename` 是想要读取的媒体文件的路径表示，可以是本地或者网络的。  
`fmt` 是自定义的读取格式，可以为 `NULL` 也可以提前通过 `av_find_input_format()` 函数获取。  
`options` 是特殊操作参数，如设置 `timeout` 参数的值。  
`avformat_open_input()` 中会调用 `init_input()` 函数打开输入文件并尽可能地解析出文件格式:  

```c
static int init_input(AVFormatContext *s, const char *filename,
                      AVDictionary **options)
```

`init_input()` 中的关键代码是:  

```c
if ((ret = s->io_open(s, &s->pb, filename, AVIO_FLAG_READ | s->avio_flags, options)) < 0)
        return ret;
```

而前面说的 `s->io_open` 默认指向的 `libavformat/option.c` 文件中的 `io_open_default()` 函数会调用 `libavformat/aviobuf.c` 文件中的 `ffio_open_whitelist()` 函数。  
`ffio_open_whitelist()` 函数会先调用 `libavformat/avio.c` 文件中的 `ffurl_open_whitelist()` 函数初始化 `URLContext`，再调用 `libavformat/aviobuf.c` 文件中的 `ffio_fdopen()` 函数根据 `URLContext` 的真正类型（如 `HTTPContext`）初始化 `AVIOContext`，这个 `AVIOContext` 就是常见的 `s->pb`，也就是说从这时开始 `pb` 已经被初始化了。  
`ffurl_open_whitelist()` 函数中会先调用 `ffurl_alloc()` 函数找到协议真正类型并根据类型为 `URLContext` 分配空间，再调用 `ffurl_connect()` 函数打开媒体文件。  
`ffurl_connect()` 函数中的主要调用是这样的:  

```c
err =
        uc->prot->url_open2 ? uc->prot->url_open2(uc,
                                                  uc->filename,
                                                  uc->flags,
                                                  options) :
        uc->prot->url_open(uc, uc->filename, uc->flags);
```

而位于 `libavformat/http.c` 文件中的 HTTP 协议 `ff_http_protocol` 的 `url_open2` 指向了 `http_open()` 函数，`http_open()` 中通过 `HTTPContext` 中的 `AVApplicationContext` 可以跟上层进行通讯，比如告诉上层正在进行 HTTP 请求，但主要调用的 `http_open_cnx()` 函数调用了 `http_open_cnx_internal()`。  
`http_open_cnx_internal()` 中先是对视频 URL 进行分析，比如如果使用了代理那么还要重新组装 URL 以避免将一些信息暴露给代理服务器，如果是 HTTPS 那么底层协议就是 TLS 否则底层协议就是 TCP，然后调用 `ffurl_open_whitelist()` 进行底层协议的处理（如 DNS 解析，TCP 握手建立 Socket 连接）。然后调用 `http_connect()` 函数进行 HTTP 请求，当然请求前要给 Header 设置默认值并且添加用户自定义的 Header，然后调用 `libavformat/avio.c` 文件中的 `ffurl_write()` 函数发送请求数据，它调用底层协议的 `url_write`，而位于 `libavformat/tcp.c` 文件中的 TCP 协议 `ff_tcp_protocol` 的 `url_write` 指向了 `tcp_write()` 函数，`tcp_write()` 主要是调用系统函数 `send()` 发送数据（`tcp_read` 调用系统函数 `recv()`）。最后，在发送完数据后会调用 `http_read_header()` 函数读取响应报文的 Header，而 `http_read_header()` 中有个死循环，就是不停地 `http_get_line()` 和 `process_line()` 直到所有 Header 数据处理完毕，`http_get_line()` 内部其实也是调用了 `ffurl_read()`（跟 `ffurl_write()` 逻辑类似）。  
至此，如果 `avformat_open_input()` 返回了大于等于零的数，就算是第一次拿到了媒体文件的数据，播放器就可以向上层发一个 `FFP_MSG_OPEN_INPUT` 的消息表示成功打开了输入流。  

#### 分析输入流

打开输入流并一定能精确地知道媒体流实际的时长、帧率等信息，一般情况下还需要调用 `libavformat/utils.c` 文件中的 `avformat_find_stream_info()` 函数对输入流进行探测分析:  

```c
int avformat_find_stream_info(AVFormatContext *ic, AVDictionary **options)
```

由于读取一部分媒体数据进行分析的过程还是非常耗时的，所以需要一个时间限制，这个时间限制不能太短以避免成功率太低。`max_analyze_duration` 如果不指定那么默认是 `5 * AV_TIME_BASE`（时间都是基于时基的，而时基 `AV_TIME_BASE` 是 `1000000`），对于 `mpeg` 或 `mpegts` 格式的视频流 `max_stream_analyze_duration = 90 * AV_TIME_BASE`。  
对于媒体中的所有流（包括视频流、音频流、字幕流），先根据之前的 `codec_id` 调用 `find_probe_decoder()` 函数寻找合适的解码器，再调用 `libavcodec/utils.c` 文件中的 `avcodec_open2()` 函数打开解码器，再调用 `read_frame_internal()` 函数读取一个完整的 `AVPacket`，再调用 `try_decode_frame()` 函数尝试解码 packet。  

#### 获取各个媒体类型的流的索引

一般媒体流中都会包括 `AVMEDIA_TYPE_VIDEO`、`AVMEDIA_TYPE_AUDIO` 和 `AVMEDIA_TYPE_SUBTITLE` 等媒体类型的流，可以通过 `libavformat/utils.c` 文件中的 `av_find_best_stream()` 函数获取他们的索引。  

#### 打开各个媒体流

根据各个媒体流的索引就可以打开各个媒体流了，首先调用 `libavcodec/utils.c` 文件中的 `avcodec_find_decoder()` 函数找到该媒体流的解码器，然后调用 `libavcodec/options.c` 文件中的 `avcodec_alloc_context3()` 为解码器分配空间，然后调用 `libavcodec/utils.c` 文件中的 `avcodec_parameters_to_context()` 为解码器复制上下文参数，然后调用 `libavcodec/utils.c` 文件中的 `avcodec_open2()` 打开解码器，然后调用 `libavutil/frame.c` 文件中的 `av_frame_alloc()` 为 `AVFrame` 分配空间，然后调用 `libavutil/imgutils.c` 文件中的 `av_image_get_buffer_size()` 获取需要的缓冲区大小并为其分配空间，然后调用 `libavcodec/avpacket.c` 文件中的 `av_init_packet()` 对 `AVPacket` 进行初始化。  

#### 循环读取每一帧

通过 `libavformat/utils.c` 文件中的 `av_read_frame()` 函数就可以读取完整的一帧数据了:  

```c
    do {
        if (!end_of_stream)
            if (av_read_frame(fmt_ctx, &pkt) < 0)
                end_of_stream = 1;
        if (end_of_stream) {
            pkt.data = NULL;
            pkt.size = 0;
        }
        if (pkt.stream_index == video_stream || end_of_stream) {
            got_frame = 0;
            if (pkt.pts == AV_NOPTS_VALUE)
                pkt.pts = pkt.dts = i;
            result = avcodec_decode_video2(ctx, fr, &got_frame, &pkt);
            if (result < 0) {
                av_log(NULL, AV_LOG_ERROR, "Error decoding frame\n");
                return result;
            }
            if (got_frame) {
                number_of_written_bytes = av_image_copy_to_buffer(byte_buffer, byte_buffer_size,
                                        (const uint8_t* const *)fr->data, (const int*) fr->linesize,
                                        ctx->pix_fmt, ctx->width, ctx->height, 1);
                if (number_of_written_bytes < 0) {
                    av_log(NULL, AV_LOG_ERROR, "Can't copy image to buffer\n");
                    return number_of_written_bytes;
                }
                printf("%d, %10"PRId64", %10"PRId64", %8"PRId64", %8d, 0x%08lx\n", video_stream,
                        fr->pts, fr->pkt_dts, av_frame_get_pkt_duration(fr),
                        number_of_written_bytes, av_adler32_update(0, (const uint8_t*)byte_buffer, number_of_written_bytes));
            }
            av_packet_unref(&pkt);
            av_init_packet(&pkt);
        }
        i++;
    } while (!end_of_stream || got_frame);
```

### 编译 ijkplayer

如果编译过程中出现 `linux-perf` 相关文件未找到的错误可以在编译脚本文件中添加下面这一行以禁用相关调试功能:  

```text
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-linux-perf"
```

如果想支持 webm 格式视频的播放需要修改编译脚本，添加 decoder，demuxer，parser 对相关格式的支持:  

```text
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=opus"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vp6"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vp6a"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vp8_cuvid"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vp8_mediacodec"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vp8_qsv"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=flac"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=theora"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-decoder=zlib"

export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=matroska"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=ogg"

export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=vp8"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=vp9"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=vorbis"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-parser=opus"
```

如果想支持分段视频（`ffconcat` 协议），首先需要修改编译脚本以支持拼接协议:  

```text
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-protocol=concat"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-demuxer=concat"
```

然后在 Java 层将 `ffconcat` 协议加入白名单并允许访问不安全的路径:  

```java
ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "safe", 0);
ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "protocol_whitelist", "ffconcat,file,http,https");
ijkMediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_FORMAT, "protocol_whitelist", "concat,http,tcp,https,tls,file");
```

ijkplayer k0.8.8 版本， 支持常见格式的 lite 版本，支持 HTTPS 协议的 .so 文件的编译命令如下:  

```text
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
cd ijkplayer-android
git checkout -B latest k0.8.8
cd config
rm module.sh
ln -s module-lite.sh module.sh
cd ..
./init-android.sh
./init-android-openssl.sh
cd android/contrib
./compile-openssl.sh clean
./compile-openssl.sh all
./compile-ffmpeg.sh clean
./compile-ffmpeg.sh all
cd ..
./compile-ijk.sh clean
./compile-ijk.sh all
```

也可以简化成一个命令:  

```text
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android && cd ijkplayer-android && git checkout -B latest k0.8.8 && cd config && rm module.sh && ln -s module-lite.sh module.sh && cd .. && ./init-android.sh && ./init-android-openssl.sh && cd android/contrib && ./compile-openssl.sh clean && ./compile-openssl.sh all && ./compile-ffmpeg.sh clean && ./compile-ffmpeg.sh all && cd .. && ./compile-ijk.sh clean && ./compile-ijk.sh all
```

生成的 `libijkffmpeg.so`，`libijkplayer.so`，`libijksdl.so` 文件目录位于如下目录:  

```text
ijkplayer-android/android/ijkplayer/ijkplayer-armv7a/src/main/libs/armeabi-v7a/libijkffmpeg.so
```

## 参考

- [MPEG-1 Wiki](https://en.wikipedia.org/wiki/MPEG-1)
- [H.261 Wiki](https://en.wikipedia.org/wiki/H.261)
- [FFmpeg Wiki](https://en.wikipedia.org/wiki/FFmpeg)
- [FFmpeg](http://ffmpeg.org/)
- [ijkplayer](https://github.com/Bilibili/ijkplayer)
