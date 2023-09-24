---
title: ffmpeg与rtsp源代码分析
date: 2022-03-10 09:57:22
tags: 编程 ffmpeg rtsp
---

## 分析的源码

```c++
#include <fstream>
#include <iostream>
#include <sstream>
#include <stdio.h>
#include <stdlib.h>

extern "C"
{
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavformat/avio.h>
#include <libswscale/swscale.h>
}

int main(int argc, char **argv)
{

    // Open the initial context variables that are needed
    SwsContext *img_convert_ctx;
    AVFormatContext *format_ctx = avformat_alloc_context();
    AVCodecContext *codec_ctx = NULL;
    int video_stream_index;

    // Register everything
    //av_register_all();
    // avformat_network_init();

    // open RTSP
    if (avformat_open_input(&format_ctx, "rtsp://134.169.178.187:8554/h264.3gp",
                            NULL, NULL) != 0)
    {
        return EXIT_FAILURE;
    }

    if (avformat_find_stream_info(format_ctx, NULL) < 0)
    {
        return EXIT_FAILURE;
    }

    // search video stream
    for (int i = 0; i < format_ctx->nb_streams; i++)
    {
        if (format_ctx->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO)
            video_stream_index = i;
    }

    AVPacket packet;
    av_init_packet(&packet);

    // open output file
    AVFormatContext *output_ctx = avformat_alloc_context();

    AVStream *stream = NULL;
    int cnt = 0;

    // start reading packets from stream and write them to file
    av_read_play(format_ctx); // play RTSP

    // Get the codec
    AVCodec *codec = NULL;
    codec = avcodec_find_decoder(AV_CODEC_ID_H264);
    if (!codec)
    {
        exit(1);
    }

    // Add this to allocate the context by codec
    codec_ctx = avcodec_alloc_context3(codec);

    avcodec_get_context_defaults3(codec_ctx, codec);
    avcodec_copy_context(codec_ctx, format_ctx->streams[video_stream_index]->codec);
    std::ofstream output_file;

    if (avcodec_open2(codec_ctx, codec, NULL) < 0)
        exit(1);

    img_convert_ctx = sws_getContext(codec_ctx->width, codec_ctx->height,
                                     codec_ctx->pix_fmt, codec_ctx->width, codec_ctx->height, AV_PIX_FMT_RGB24,
                                     SWS_BICUBIC, NULL, NULL, NULL);

    int size = avpicture_get_size(AV_PIX_FMT_YUV420P, codec_ctx->width,
                                  codec_ctx->height);
    uint8_t *picture_buffer = (uint8_t *)(av_malloc(size));
    AVFrame *picture = av_frame_alloc();
    AVFrame *picture_rgb = av_frame_alloc();
    int size2 = avpicture_get_size(AV_PIX_FMT_RGB24, codec_ctx->width,
                                   codec_ctx->height);
    uint8_t *picture_buffer_2 = (uint8_t *)(av_malloc(size2));
    avpicture_fill((AVPicture *)picture, picture_buffer, AV_PIX_FMT_YUV420P,
                   codec_ctx->width, codec_ctx->height);
    avpicture_fill((AVPicture *)picture_rgb, picture_buffer_2, AV_PIX_FMT_RGB24,
                   codec_ctx->width, codec_ctx->height);

    while (av_read_frame(format_ctx, &packet) >= 0 && cnt < 1000)
    { // read ~ 1000 frames

        std::cout << "1 Frame: " << cnt << std::endl;
        if (packet.stream_index == video_stream_index)
        { // packet is video
            std::cout << "2 Is Video" << std::endl;
            if (stream == NULL)
            { // create stream in file
                std::cout << "3 create stream" << std::endl;
                stream = avformat_new_stream(output_ctx,
                                             format_ctx->streams[video_stream_index]->codec->codec);
                avcodec_copy_context(stream->codec,
                                     format_ctx->streams[video_stream_index]->codec);
                stream->sample_aspect_ratio =
                    format_ctx->streams[video_stream_index]->codec->sample_aspect_ratio;
            }
            int check = 0;
            packet.stream_index = stream->id;
            std::cout << "4 decoding" << std::endl;
            int result = avcodec_decode_video2(codec_ctx, picture, &check, &packet);
            std::cout << "Bytes decoded " << result << " check " << check
                      << std::endl;
            if (cnt > 100) // cnt < 0)
            {
                sws_scale(img_convert_ctx, picture->data, picture->linesize, 0,
                          codec_ctx->height, picture_rgb->data, picture_rgb->linesize);
                std::stringstream file_name;
                file_name << "test" << cnt << ".ppm";
                output_file.open(file_name.str().c_str());
                output_file << "P3 " << codec_ctx->width << " " << codec_ctx->height
                            << " 255\n";
                for (int y = 0; y < codec_ctx->height; y++)
                {
                    for (int x = 0; x < codec_ctx->width * 3; x++)
                        output_file
                            << (int)(picture_rgb->data[0] + y * picture_rgb->linesize[0])[x] << " ";
                }
                output_file.close();
            }
            cnt++;
        }
        av_free_packet(&packet);
        av_init_packet(&packet);
    }
    av_free(picture);
    av_free(picture_rgb);
    av_free(picture_buffer);
    av_free(picture_buffer_2);

    av_read_pause(format_ctx);
    avio_close(output_ctx->pb);
    avformat_free_context(output_ctx);

    return (EXIT_SUCCESS);
}
```

## 初始化封装上下文

```c++
// Open the initial context variables that are needed
    SwsContext *img_convert_ctx;
    AVFormatContext *format_ctx = avformat_alloc_context();
    AVCodecContext *codec_ctx = NULL;
    int video_stream_index;
```

avformat_alloc_context 初始化封装上下文, 其实这一步不太需要, 因为在后面的 avformat_open_input 方法中也会检查 format_ctx 参数, 如果没有进行初始化的话, 也会将其初始化

```c++
AVFormatContext *avformat_alloc_context(void)
{
    FFFormatContext *const si = av_mallocz(sizeof(*si));
    AVFormatContext *s;

    if (!si)
        return NULL;

  	//初始化AVStream的操作方法
    s = &si->pub;
    s->av_class = &av_format_context_class;
    s->io_open  = io_open_default;
    s->io_close = ff_format_io_close_default;
    s->io_close2= io_close2_default;

    av_opt_set_defaults(s);

	  //包含音视频数据的包
    si->pkt = av_packet_alloc(); 
		//临时数据包,只会被解析或编码程序使用, 不会被av_read_frame与ff_read_packet覆盖
  	si->parse_pkt = av_packet_alloc(); 
    if (!si->pkt || !si->parse_pkt) {
        avformat_free_context(s);
        return NULL;
    }

    si->shortest_end = AV_NOPTS_VALUE;

    return s;
}
```

## 打开 rtsp 流

```c++
if (avformat_open_input(&format_ctx, "rtsp://134.169.178.187:8554/h264.3gp",
                            NULL, NULL) != 0)
    {
        return EXIT_FAILURE;
    }
```

### avformat_open_input

avformat_open_input 在不指定 AVInputFormat 的情况下, 会根据 filename 自动查找合适的流处理器与解封器

```c++
int avformat_open_input(AVFormatContext **ps, const char *filename,
                        const AVInputFormat *fmt, AVDictionary **options)
{
    AVFormatContext *s = *ps;
    FFFormatContext *si;
    AVDictionary *tmp = NULL;
    ID3v2ExtraMeta *id3v2_extra_meta = NULL;
    int ret = 0;

  	// 如果事先没有初始化过上下文, 这里会初始化
    if (!s && !(s = avformat_alloc_context()))
        return AVERROR(ENOMEM);
	  //统一转换为FFFormatContext指针
    si = ffformatcontext(s); 
    if (!s->av_class) {
        av_log(NULL, AV_LOG_ERROR, "Input context has not been properly allocated by avformat_alloc_context() and is not NULL either\n");
        return AVERROR(EINVAL);
    }
    if (fmt)
        s->iformat = fmt;

    if (options)
        av_dict_copy(&tmp, *options, 0);

    if (s->pb) // must be before any goto fail
        s->flags |= AVFMT_FLAG_CUSTOM_IO;

    if ((ret = av_opt_set_dict(s, &tmp)) < 0)
        goto fail;

    if (!(s->url = av_strdup(filename ? filename : ""))) {
        ret = AVERROR(ENOMEM);
        goto fail;
    }

  	// init_input 会根据 filename 自动查找合适的解析器
    if ((ret = init_input(s, filename, &tmp)) < 0)
        goto fail;
    s->probe_score = ret;

    if (!s->protocol_whitelist && s->pb && s->pb->protocol_whitelist) {
        s->protocol_whitelist = av_strdup(s->pb->protocol_whitelist);
        if (!s->protocol_whitelist) {
            ret = AVERROR(ENOMEM);
            goto fail;
        }
    }

    if (!s->protocol_blacklist && s->pb && s->pb->protocol_blacklist) {
        s->protocol_blacklist = av_strdup(s->pb->protocol_blacklist);
        if (!s->protocol_blacklist) {
            ret = AVERROR(ENOMEM);
            goto fail;
        }
    }

    if (s->format_whitelist && av_match_list(s->iformat->name, s->format_whitelist, ',') <= 0) {
        av_log(s, AV_LOG_ERROR, "Format not on whitelist \'%s\'\n", s->format_whitelist);
        ret = AVERROR(EINVAL);
        goto fail;
    }

    avio_skip(s->pb, s->skip_initial_bytes);

    /* Check filename in case an image number is expected. */
    if (s->iformat->flags & AVFMT_NEEDNUMBER) {
        if (!av_filename_number_test(filename)) {
            ret = AVERROR(EINVAL);
            goto fail;
        }
    }

    s->duration = s->start_time = AV_NOPTS_VALUE;

    /* Allocate private data. */
    if (s->iformat->priv_data_size > 0) {
        if (!(s->priv_data = av_mallocz(s->iformat->priv_data_size))) {
            ret = AVERROR(ENOMEM);
            goto fail;
        }
        if (s->iformat->priv_class) {
            *(const AVClass **) s->priv_data = s->iformat->priv_class;
            av_opt_set_defaults(s->priv_data);
            if ((ret = av_opt_set_dict(s->priv_data, &tmp)) < 0)
                goto fail;
        }
    }

    /* e.g. AVFMT_NOFILE formats will not have an AVIOContext */
    if (s->pb)
        ff_id3v2_read_dict(s->pb, &si->id3v2_meta, ID3v2_DEFAULT_MAGIC, &id3v2_extra_meta);

    if (s->iformat->read_header)
        if ((ret = s->iformat->read_header(s)) < 0) {
            if (s->iformat->flags_internal & FF_FMT_INIT_CLEANUP)
                goto close;
            goto fail;
        }

    if (!s->metadata) {
        s->metadata    = si->id3v2_meta;
        si->id3v2_meta = NULL;
    } else if (si->id3v2_meta) {
        av_log(s, AV_LOG_WARNING, "Discarding ID3 tags because more suitable tags were found.\n");
        av_dict_free(&si->id3v2_meta);
    }

    if (id3v2_extra_meta) {
        if (!strcmp(s->iformat->name, "mp3") || !strcmp(s->iformat->name, "aac") ||
            !strcmp(s->iformat->name, "tta") || !strcmp(s->iformat->name, "wav")) {
            if ((ret = ff_id3v2_parse_apic(s, id3v2_extra_meta)) < 0)
                goto close;
            if ((ret = ff_id3v2_parse_chapters(s, id3v2_extra_meta)) < 0)
                goto close;
            if ((ret = ff_id3v2_parse_priv(s, id3v2_extra_meta)) < 0)
                goto close;
        } else
            av_log(s, AV_LOG_DEBUG, "demuxer does not support additional id3 data, skipping\n");
        ff_id3v2_free_extra_meta(&id3v2_extra_meta);
    }

    if ((ret = avformat_queue_attached_pictures(s)) < 0)
        goto close;

    if (s->pb && !si->data_offset)
        si->data_offset = avio_tell(s->pb);

    si->raw_packet_buffer_size = 0;
		
  	// 设置编解码器
    update_stream_avctx(s);

    if (options) {
        av_dict_free(options);
        *options = tmp;
    }
    *ps = s;
    return 0;

close:
    if (s->iformat->read_close)
        s->iformat->read_close(s);
fail:
    ff_id3v2_free_extra_meta(&id3v2_extra_meta);
    av_dict_free(&tmp);
    if (s->pb && !(s->flags & AVFMT_FLAG_CUSTOM_IO))
        avio_closep(&s->pb);
    avformat_free_context(s);
    *ps = NULL;
    return ret;
}

```

####  init_input

负责查找解封器

```c++
static int init_input(AVFormatContext *s, const char *filename,
                      AVDictionary **options)
{
    int ret;
    AVProbeData pd = { filename, NULL, 0 };
    int score = AVPROBE_SCORE_RETRY;

    if (s->pb) {
        s->flags |= AVFMT_FLAG_CUSTOM_IO;
        if (!s->iformat)
          	//如果没有制定 iformat, 但是有拿到数据的话, 则 av_probe_input_buffer2 会调用 av_probe_input_format2(pd, 1, &score) 来查找解封器.
            return av_probe_input_buffer2(s->pb, &s->iformat, filename,
                                          s, 0, s->format_probesize);
        else if (s->iformat->flags & AVFMT_NOFILE)
            av_log(s, AV_LOG_WARNING, "Custom AVIOContext makes no sense and "
                                      "will be ignored with AVFMT_NOFILE format.\n");
        return 0;
    }

  	//如果没有制定 iformat, 也没有制定的 probe 块, 则直接调用 av_probe_input_format2(pd, 0, &score) 来查找解封器.
    if ((s->iformat && s->iformat->flags & AVFMT_NOFILE) ||
        (!s->iformat && (s->iformat = av_probe_input_format2(&pd, 0, &score))))
        return score;

  	// 在这里 io_open 调用的是 avformat_alloc_context 中初始化的缺省 io_open_default, 
  	// 这个方法在 options.c 文件中
    if ((ret = s->io_open(s, &s->pb, filename, AVIO_FLAG_READ | s->avio_flags, options)) < 0)
        return ret;

    if (s->iformat)
        return 0;
    return av_probe_input_buffer2(s->pb, &s->iformat, filename,
                                  s, 0, s->format_probesize);
}	
```

##### av_probe_input_format3

av_probe_input_format2 调用 av_probe_input_format3 进行实际操作

```c++
const AVInputFormat *av_probe_input_format3(const AVProbeData *pd,
                                            int is_opened, int *score_ret)
{
    AVProbeData lpd = *pd;
    const AVInputFormat *fmt1 = NULL;
    const AVInputFormat *fmt = NULL;
    int score, score_max = 0;
    void *i = 0;
    const static uint8_t zerobuffer[AVPROBE_PADDING_SIZE];
    enum nodat {
        NO_ID3,
        ID3_ALMOST_GREATER_PROBE,
        ID3_GREATER_PROBE,
        ID3_GREATER_MAX_PROBE,
    } nodat = NO_ID3;

    if (!lpd.buf)
        lpd.buf = (unsigned char *) zerobuffer;

  	//如果有数据则特别处理 ID3V2 的歌词格式
    if (lpd.buf_size > 10 && ff_id3v2_match(lpd.buf, ID3v2_DEFAULT_MAGIC)) {
        int id3len = ff_id3v2_tag_len(lpd.buf);
        if (lpd.buf_size > id3len + 16) {
            if (lpd.buf_size < 2LL*id3len + 16)
                nodat = ID3_ALMOST_GREATER_PROBE;
            lpd.buf      += id3len;
            lpd.buf_size -= id3len;
        } else if (id3len >= PROBE_BUF_MAX) {
            nodat = ID3_GREATER_MAX_PROBE;
        } else
            nodat = ID3_GREATER_PROBE;
    }

   	// ffmpeg 在 demuxer_list.c 文件中定义的 demuxer_list[] 常量, 
  	// 这个数组常量包含了本版本源代码中所有的 demuxer. 
  	// 通过 av_demuxer_iterate 方法可以对这些 demuxer 进行查找.
    while ((fmt1 = av_demuxer_iterate(&i))) {
        if (fmt1->flags & AVFMT_EXPERIMENTAL)
            continue;
        if (!is_opened == !(fmt1->flags & AVFMT_NOFILE) && strcmp(fmt1->name, "image2"))
            continue;
        score = 0;
       	// 以 rtsp 的 demuxer 为例, 它提供了 read_probe 方法, 
      	// 并在方法中对 rtsp:// 开头的链接进行了确认
        if (fmt1->read_probe) {
            score = fmt1->read_probe(&lpd);
            if (score)
                av_log(NULL, AV_LOG_TRACE, "Probing %s score:%d size:%d\n", fmt1->name, score, lpd.buf_size);
            if (fmt1->extensions && av_match_ext(lpd.filename, fmt1->extensions)) {
                switch (nodat) {
                case NO_ID3:
                    score = FFMAX(score, 1);
                    break;
                case ID3_GREATER_PROBE:
                case ID3_ALMOST_GREATER_PROBE:
                    score = FFMAX(score, AVPROBE_SCORE_EXTENSION / 2 - 1);
                    break;
                case ID3_GREATER_MAX_PROBE:
                    score = FFMAX(score, AVPROBE_SCORE_EXTENSION);
                    break;
                }
            }
          // rtsp 的 demuxer 没有提供 extensions
        } else if (fmt1->extensions) {
            if (av_match_ext(lpd.filename, fmt1->extensions))
                score = AVPROBE_SCORE_EXTENSION;
        }
      	// rtsp 的 demuxer 没有提供 mime_type
        if (av_match_name(lpd.mime_type, fmt1->mime_type)) {
            if (AVPROBE_SCORE_MIME > score) {
                av_log(NULL, AV_LOG_DEBUG, "Probing %s score:%d increased to %d due to MIME type\n", fmt1->name, score, AVPROBE_SCORE_MIME);
                score = AVPROBE_SCORE_MIME;
            }
        }
      	// 只有分数最高的才返回
        if (score > score_max) {
            score_max = score;
            fmt       = fmt1;
        } else if (score == score_max)
            fmt = NULL;
    }
    if (nodat == ID3_GREATER_PROBE)
        score_max = FFMIN(AVPROBE_SCORE_EXTENSION / 2 - 1, score_max);
    *score_ret = score_max;

    return fmt;
}
```

#### s->iformat->read_header

读取 rtsp 头信息

```c++
static int rtsp_read_header(AVFormatContext *s)
{
    RTSPState *rt = s->priv_data;
    int ret;

    if (rt->initial_timeout > 0)
        rt->rtsp_flags |= RTSP_FLAG_LISTEN;

  	//如果作为服务器的话, 开始监听端口
    if (rt->rtsp_flags & RTSP_FLAG_LISTEN) {
        ret = rtsp_listen(s);
        if (ret)
            return ret;
    } else {
      // 否则连接 rtsp 服务器
        ret = ff_rtsp_connect(s);
        if (ret)
            return ret;

        rt->real_setup_cache = !s->nb_streams ? NULL :
            av_calloc(s->nb_streams, 2 * sizeof(*rt->real_setup_cache));
        if (!rt->real_setup_cache && s->nb_streams) {
            ret = AVERROR(ENOMEM);
            goto fail;
        }
        rt->real_setup = rt->real_setup_cache + s->nb_streams;

      // 如果设置的 initial_pause 则连接上后不立刻开始播放 
        if (rt->initial_pause) {
            /* do not start immediately */
        } else {
          // 开始播放流
            ret = rtsp_read_play(s);
            if (ret < 0)
                goto fail;
        }
    }

    return 0;

fail:
    rtsp_read_close(s);
    return ret;
}
```

##### ff_rtsp_connect

```c++
int ff_rtsp_connect(AVFormatContext *s)
{
    RTSPState *rt = s->priv_data;
    char proto[128], host[1024], path[1024];
    char tcpname[1024], cmd[MAX_URL_SIZE], auth[128];
    const char *lower_rtsp_proto = "tcp";
    int port, err, tcp_fd;
    RTSPMessageHeader reply1, *reply = &reply1;
    int lower_transport_mask = 0;
    int default_port = RTSP_DEFAULT_PORT;
    int https_tunnel = 0;
    char real_challenge[64] = "";
    struct sockaddr_storage peer;
    socklen_t peer_len = sizeof(peer);

    if (rt->rtp_port_max < rt->rtp_port_min)
    {
        av_log(s, AV_LOG_ERROR, "Invalid UDP port range, max port %d less "
                                "than min port %d\n",
               rt->rtp_port_max,
               rt->rtp_port_min);
        return AVERROR(EINVAL);
    }

    if (!ff_network_init())
        return AVERROR(EIO);

    if (s->max_delay < 0) /* Not set by the caller */
        s->max_delay = s->iformat ? DEFAULT_REORDERING_DELAY : 0;

    //调用 ff_network_init 初始化网络
    // 设置缺省使用rtsp传输协议, 
    rt->control_transport = RTSP_MODE_PLAIN;
  	// 如果设置了使用 http 或者 https 则设置使用 http tunnel 协议
    if (rt->lower_transport_mask & ((1 << RTSP_LOWER_TRANSPORT_HTTP) |
                                    (1 << RTSP_LOWER_TRANSPORT_HTTPS)))
    {
        https_tunnel = !!(rt->lower_transport_mask & (1 << RTSP_LOWER_TRANSPORT_HTTPS));
        rt->lower_transport_mask = 1 << RTSP_LOWER_TRANSPORT_TCP;
        rt->control_transport = RTSP_MODE_TUNNEL;
    }
    /* Only pass through valid flags from here */
    rt->lower_transport_mask &= (1 << RTSP_LOWER_TRANSPORT_NB) - 1;

redirect:
    memset(&reply1, 0, sizeof(reply1));
    /* extract hostname and port */
    av_url_split(proto, sizeof(proto), auth, sizeof(auth),
                 host, sizeof(host), &port, path, sizeof(path), s->url);
		// 在 rtsps 协议情况下会使用322端口, 否则使用缺省的554端口
    if (!strcmp(proto, "rtsps"))
    {
        lower_rtsp_proto = "tls";
        default_port = RTSPS_DEFAULT_PORT;
        rt->lower_transport_mask = 1 << RTSP_LOWER_TRANSPORT_TCP;
    }
    else if (!strcmp(proto, "satip"))
    {
        av_strlcpy(proto, "rtsp", sizeof(proto));
        rt->server_type = RTSP_SERVER_SATIP;
    }

    if (*auth)
    {
        av_strlcpy(rt->auth, auth, sizeof(rt->auth));
    }
    if (port < 0)
        port = default_port;

    lower_transport_mask = rt->lower_transport_mask;

    if (!lower_transport_mask)
        lower_transport_mask = (1 << RTSP_LOWER_TRANSPORT_NB) - 1;
		// 在 RTSP_MODE_TUNNEL 情况下是不支持上传流数据的
    if (s->oformat)
    {
        /* Only UDP or TCP - UDP multicast isn't supported. */
        lower_transport_mask &= (1 << RTSP_LOWER_TRANSPORT_UDP) |
                                (1 << RTSP_LOWER_TRANSPORT_TCP);
        if (!lower_transport_mask || rt->control_transport == RTSP_MODE_TUNNEL)
        {
            av_log(s, AV_LOG_ERROR, "Unsupported lower transport method, "
                                    "only UDP and TCP are supported for output.\n");
            err = AVERROR(EINVAL);
            goto fail;
        }
    }

    /* Construct the URI used in request; this is similar to s->url,
     * but with authentication credentials removed and RTSP specific options
     * stripped out. */
  	// 生成 rtsp 控制地址
    ff_url_join(rt->control_uri, sizeof(rt->control_uri), proto, NULL,
                host, port, "%s", path);

    if (rt->control_transport == RTSP_MODE_TUNNEL)
    {
        /* set up initial handshake for tunneling */
        char httpname[1024];
        char sessioncookie[17];
        char headers[1024];
        AVDictionary *options = NULL;

        av_dict_set_int(&options, "timeout", rt->stimeout, 0);

        ff_url_join(httpname, sizeof(httpname), https_tunnel ? "https" : "http", auth, host, port, "%s", path);
        snprintf(sessioncookie, sizeof(sessioncookie), "%08x%08x",
                 av_get_random_seed(), av_get_random_seed());

        /* GET requests */
        if (ffurl_alloc(&rt->rtsp_hd, httpname, AVIO_FLAG_READ,
                        &s->interrupt_callback) < 0)
        {
            err = AVERROR(EIO);
            goto fail;
        }

        /* generate GET headers */
        snprintf(headers, sizeof(headers),
                 "x-sessioncookie: %s\r\n"
                 "Accept: application/x-rtsp-tunnelled\r\n"
                 "Pragma: no-cache\r\n"
                 "Cache-Control: no-cache\r\n",
                 sessioncookie);
        av_opt_set(rt->rtsp_hd->priv_data, "headers", headers, 0);

        if (!rt->rtsp_hd->protocol_whitelist && s->protocol_whitelist)
        {
            rt->rtsp_hd->protocol_whitelist = av_strdup(s->protocol_whitelist);
            if (!rt->rtsp_hd->protocol_whitelist)
            {
                err = AVERROR(ENOMEM);
                goto fail;
            }
        }

        /* complete the connection */
        if (ffurl_connect(rt->rtsp_hd, &options))
        {
            av_dict_free(&options);
            err = AVERROR(EIO);
            goto fail;
        }

        /* POST requests */
        if (ffurl_alloc(&rt->rtsp_hd_out, httpname, AVIO_FLAG_WRITE,
                        &s->interrupt_callback) < 0)
        {
            err = AVERROR(EIO);
            goto fail;
        }

        /* generate POST headers */
        snprintf(headers, sizeof(headers),
                 "x-sessioncookie: %s\r\n"
                 "Content-Type: application/x-rtsp-tunnelled\r\n"
                 "Pragma: no-cache\r\n"
                 "Cache-Control: no-cache\r\n"
                 "Content-Length: 32767\r\n"
                 "Expires: Sun, 9 Jan 1972 00:00:00 GMT\r\n",
                 sessioncookie);
        av_opt_set(rt->rtsp_hd_out->priv_data, "headers", headers, 0);
        av_opt_set(rt->rtsp_hd_out->priv_data, "chunked_post", "0", 0);
        av_opt_set(rt->rtsp_hd_out->priv_data, "send_expect_100", "0", 0);

        /* Initialize the authentication state for the POST session. The HTTP
         * protocol implementation doesn't properly handle multi-pass
         * authentication for POST requests, since it would require one of
         * the following:
         * - implementing Expect: 100-continue, which many HTTP servers
         *   don't support anyway, even less the RTSP servers that do HTTP
         *   tunneling
         * - sending the whole POST data until getting a 401 reply specifying
         *   what authentication method to use, then resending all that data
         * - waiting for potential 401 replies directly after sending the
         *   POST header (waiting for some unspecified time)
         * Therefore, we copy the full auth state, which works for both basic
         * and digest. (For digest, we would have to synchronize the nonce
         * count variable between the two sessions, if we'd do more requests
         * with the original session, though.)
         */
        ff_http_init_auth_state(rt->rtsp_hd_out, rt->rtsp_hd);

        /* complete the connection */
        if (ffurl_connect(rt->rtsp_hd_out, &options))
        {
            av_dict_free(&options);
            err = AVERROR(EIO);
            goto fail;
        }
        av_dict_free(&options);
    }
    else //使用缺省的 rtsp 协议
    {
        int ret;
        /* open the tcp connection */
        ff_url_join(tcpname, sizeof(tcpname), lower_rtsp_proto, NULL,
                    host, port,
                    "?timeout=%" PRId64, rt->stimeout);
      	// 打开 rtsp 流地址, 在 ffurl_open_whitelist 中会
      	// 通过 ffurl_alloc 函数生成 rtsp_hd 这个 URLContext 对象
        if ((ret = ffurl_open_whitelist(&rt->rtsp_hd, tcpname, AVIO_FLAG_READ_WRITE,
                                        &s->interrupt_callback, NULL, s->protocol_whitelist, s->protocol_blacklist, NULL)) < 0)
        {
            err = ret;
            goto fail;
        }
        rt->rtsp_hd_out = rt->rtsp_hd;
    }
    rt->seq = 0;

    tcp_fd = ffurl_get_file_handle(rt->rtsp_hd);
    if (tcp_fd < 0)
    {
        err = tcp_fd;
        goto fail;
    }
    if (!getpeername(tcp_fd, (struct sockaddr *)&peer, &peer_len))
    {
        getnameinfo((struct sockaddr *)&peer, peer_len, host, sizeof(host),
                    NULL, 0, NI_NUMERICHOST);
    }

    /* request options supported by the server; this also detects server
     * type */
    if (rt->server_type != RTSP_SERVER_SATIP)
        rt->server_type = RTSP_SERVER_RTP;
  
  	// 向服务器发出 OPTIONS 命令, 直到返回错误或者正确的信息, 
  	// OPTIONS 命令会告诉客户端, 服务端的能力, 并且也表明服务端正确响应了客户端,
  	// 可以开始进一步的通讯了
    for (;;)
    {
        cmd[0] = 0;
        if (rt->server_type == RTSP_SERVER_REAL)
            av_strlcat(cmd,
                       /*
                        * The following entries are required for proper
                        * streaming from a Realmedia server. They are
                        * interdependent in some way although we currently
                        * don't quite understand how. Values were copied
                        * from mplayer SVN r23589.
                        *   ClientChallenge is a 16-byte ID in hex
                        *   CompanyID is a 16-byte ID in base64
                        */
                       "ClientChallenge: 9e26d33f2984236010ef6253fb1887f7\r\n"
                       "PlayerStarttime: [28/03/2003:22:50:23 00:00]\r\n"
                       "CompanyID: KnKV4M4I/B2FjJ1TToLycw==\r\n"
                       "GUID: 00000000-0000-0000-0000-000000000000\r\n",
                       sizeof(cmd));
        ff_rtsp_send_cmd(s, "OPTIONS", rt->control_uri, cmd, reply, NULL);
        if (reply->status_code != RTSP_STATUS_OK)
        {
            err = ff_rtsp_averror(reply->status_code, AVERROR_INVALIDDATA);
            goto fail;
        }

        /* detect server type if not standard-compliant RTP */
        if (rt->server_type != RTSP_SERVER_REAL && reply->real_challenge[0])
        {
            rt->server_type = RTSP_SERVER_REAL;
            continue;
        }
        else if (!av_strncasecmp(reply->server, "WMServer/", 9))
        {
            rt->server_type = RTSP_SERVER_WMS;
        }
        else if (rt->server_type == RTSP_SERVER_REAL)
            strcpy(real_challenge, reply->real_challenge);
        break;
    }

#if CONFIG_RTSP_DEMUXER
    if (s->iformat)
    {
        if (rt->server_type == RTSP_SERVER_SATIP)
            err = init_satip_stream(s);
        else
          	// 设置解析流
            err = ff_rtsp_setup_input_streams(s, reply);
    }
    else
#endif
        if (CONFIG_RTSP_MUXER)
        err = ff_rtsp_setup_output_streams(s, host);
    else
        av_assert0(0);
    if (err)
        goto fail;

    do
    {
        int lower_transport = ff_log2_tab[lower_transport_mask &
                                          ~(lower_transport_mask - 1)];

        if ((lower_transport_mask & (1 << RTSP_LOWER_TRANSPORT_TCP)) && (rt->rtsp_flags & RTSP_FLAG_PREFER_TCP))
            lower_transport = RTSP_LOWER_TRANSPORT_TCP;

        err = ff_rtsp_make_setup_request(s, host, port, lower_transport,
                                         rt->server_type == RTSP_SERVER_REAL ? real_challenge : NULL);
        if (err < 0)
            goto fail;
        lower_transport_mask &= ~(1 << lower_transport);
        if (lower_transport_mask == 0 && err == 1)
        {
            err = AVERROR(EPROTONOSUPPORT);
            goto fail;
        }
    } while (err);

    rt->lower_transport_mask = lower_transport_mask;
    av_strlcpy(rt->real_challenge, real_challenge, sizeof(rt->real_challenge));
    rt->state = RTSP_STATE_IDLE;
    rt->seek_timestamp = 0; /* default is to start stream at position zero */
    return 0;
fail:
    ff_rtsp_close_streams(s);
    ff_rtsp_close_connections(s);
    if (reply->status_code >= 300 && reply->status_code < 400 && s->iformat)
    {
        char *new_url = av_strdup(reply->location);
        if (!new_url)
        {
            err = AVERROR(ENOMEM);
            goto fail2;
        }
        ff_format_set_url(s, new_url);
        rt->session_id[0] = '\0';
        av_log(s, AV_LOG_INFO, "Status %d: Redirecting to %s\n",
               reply->status_code,
               s->url);
        goto redirect;
    }
fail2:
    ff_network_close();
    return err;
}

```

##### ffurl_open_whitelist

```c++
int ffurl_open_whitelist(URLContext **puc, const char *filename, int flags,
                         const AVIOInterruptCB *int_cb, AVDictionary **options,
                         const char *whitelist, const char* blacklist,
                         URLContext *parent)
{
    AVDictionary *tmp_opts = NULL;
    AVDictionaryEntry *e;
    // 通过 avio.c 文件中的 url_find_protocol 函数, 使用 		
    // ffurl_get_protocols 在 protocol_list.c 文件定义的常量 url_protocols 中找到 
  	// ff_rtp_protocol 这个 rtp 传输协议解析器
  	// 然后把传输协议放入 puc->prot
    int ret = ffurl_alloc(puc, filename, flags, int_cb);
    if (ret < 0)
        return ret;
    if (parent) {
        ret = av_opt_copy(*puc, parent);
        if (ret < 0)
            goto fail;
    }
    if (options &&
        (ret = av_opt_set_dict(*puc, options)) < 0)
        goto fail;
    if (options && (*puc)->prot->priv_data_class &&
        (ret = av_opt_set_dict((*puc)->priv_data, options)) < 0)
        goto fail;

    if (!options)
        options = &tmp_opts;

    av_assert0(!whitelist ||
               !(e=av_dict_get(*options, "protocol_whitelist", NULL, 0)) ||
               !strcmp(whitelist, e->value));
    av_assert0(!blacklist ||
               !(e=av_dict_get(*options, "protocol_blacklist", NULL, 0)) ||
               !strcmp(blacklist, e->value));

    if ((ret = av_dict_set(options, "protocol_whitelist", whitelist, 0)) < 0)
        goto fail;

    if ((ret = av_dict_set(options, "protocol_blacklist", blacklist, 0)) < 0)
        goto fail;

    if ((ret = av_opt_set_dict(*puc, options)) < 0)
        goto fail;

  	// 使用得到的 rtp 传输协议连接 rtsp 服务
    ret = ffurl_connect(*puc, options);

    if (!ret)
        return 0;
fail:
    ffurl_closep(puc);
    return ret;
}

```

#####  url_protocols

protocol_list.c 文件定义的常量 url_protocols 中找到 ff_rtp_protocol 这个 rtp 传输协议解析器.

```c++
const URLProtocol ff_rtp_protocol = {
    .name                      = "rtp",
    .url_open                  = rtp_open,
    .url_read                  = rtp_read,
    .url_write                 = rtp_write,
    .url_close                 = rtp_close,
    .url_get_file_handle       = rtp_get_file_handle,
    .url_get_multi_file_handle = rtp_get_multi_file_handle,
    .priv_data_size            = sizeof(RTPContext),
    .flags                     = URL_PROTOCOL_FLAG_NETWORK,
    .priv_data_class           = &rtp_class,
};
```

##### ffurl_connect

```c++
int ffurl_connect(URLContext *uc, AVDictionary **options)
{
    int err;
    AVDictionary *tmp_opts = NULL;
    AVDictionaryEntry *e;

    if (!options)
        options = &tmp_opts;

    // Check that URLContext was initialized correctly and lists are matching if set
    av_assert0(!(e=av_dict_get(*options, "protocol_whitelist", NULL, 0)) ||
               (uc->protocol_whitelist && !strcmp(uc->protocol_whitelist, e->value)));
    av_assert0(!(e=av_dict_get(*options, "protocol_blacklist", NULL, 0)) ||
               (uc->protocol_blacklist && !strcmp(uc->protocol_blacklist, e->value)));

    if (uc->protocol_whitelist && av_match_list(uc->prot->name, uc->protocol_whitelist, ',') <= 0) {
        av_log(uc, AV_LOG_ERROR, "Protocol '%s' not on whitelist '%s'!\n", uc->prot->name, uc->protocol_whitelist);
        return AVERROR(EINVAL);
    }

    if (uc->protocol_blacklist && av_match_list(uc->prot->name, uc->protocol_blacklist, ',') > 0) {
        av_log(uc, AV_LOG_ERROR, "Protocol '%s' on blacklist '%s'!\n", uc->prot->name, uc->protocol_blacklist);
        return AVERROR(EINVAL);
    }

    if (!uc->protocol_whitelist && uc->prot->default_whitelist) {
        av_log(uc, AV_LOG_DEBUG, "Setting default whitelist '%s'\n", uc->prot->default_whitelist);
        uc->protocol_whitelist = av_strdup(uc->prot->default_whitelist);
        if (!uc->protocol_whitelist) {
            return AVERROR(ENOMEM);
        }
    } else if (!uc->protocol_whitelist)
        av_log(uc, AV_LOG_DEBUG, "No default whitelist set\n"); // This should be an error once all declare a default whitelist

    if ((err = av_dict_set(options, "protocol_whitelist", uc->protocol_whitelist, 0)) < 0)
        return err;
    if ((err = av_dict_set(options, "protocol_blacklist", uc->protocol_blacklist, 0)) < 0)
        return err;

  	// 使用 rtp 协议中的 rtp_open 方法连接s
    err =
        uc->prot->url_open2 ? uc->prot->url_open2(uc,
                                                  uc->filename,
                                                  uc->flags,
                                                  options) :
        uc->prot->url_open(uc, uc->filename, uc->flags);

    av_dict_set(options, "protocol_whitelist", NULL, 0);
    av_dict_set(options, "protocol_blacklist", NULL, 0);

    if (err)
        return err;
    uc->is_connected = 1;
    /* We must be careful here as ffurl_seek() could be slow,
     * for example for http */
    if ((uc->flags & AVIO_FLAG_WRITE) || !strcmp(uc->prot->name, "file"))
        if (!uc->is_streamed && ffurl_seek(uc, 0, SEEK_SET) < 0)
            uc->is_streamed = 1;
    return 0;
}

```

##### rtp_open

```c++
static int rtp_open(URLContext *h, const char *uri, int flags)
{
   // rtp的实际连接是个udp连接, 在这个函数中根据rtp连接字符串重新构建udp连接字符串
   // 再通过 ffurl_open_whitelist 函数去打开 udp 连接
   // 这里打开的 upd 连接有两个, 一个是 rtp 的数据传输连接, 一个是 rtcp 的协议控制连接
   // fec 前向错误矫正, 如果有这的话, 还要多一个连接
}
```

##### ff_rtsp_setup_input_stream

发送 DESCRIBE 命令, 得到媒体流的格式数据, 通过 ff_sdp_parse 解析返回的结果后对解封装器进行设置, 其中

1.  如果不是MP2T的封装格式, 通过 ff_rtp_handler_find_by_id 从 rtpdesc.c 文件中定义的 rtp_dynamic_protocol_handler_list 中找到解码器
2. 如果是私有协议类型, ff_rtp_get_codec_info 函数得到缺省支持的媒体类型,  然后再从 ff_rtp_handler_find_by_id 中得到对应的解码器

```c++
int ff_rtsp_setup_input_streams(AVFormatContext *s, RTSPMessageHeader *reply)
{
    RTSPState *rt = s->priv_data;
    char cmd[MAX_URL_SIZE];
    unsigned char *content = NULL;
    int ret;

    /* describe the stream */
    snprintf(cmd, sizeof(cmd),
             "Accept: application/sdp\r\n");
    if (rt->server_type == RTSP_SERVER_REAL) {
        /**
         * The Require: attribute is needed for proper streaming from
         * Realmedia servers.
         */
        av_strlcat(cmd,
                   "Require: com.real.retain-entity-for-setup\r\n",
                   sizeof(cmd));
    }
    ff_rtsp_send_cmd(s, "DESCRIBE", rt->control_uri, cmd, reply, &content);
    if (reply->status_code != RTSP_STATUS_OK) {
        av_freep(&content);
        return ff_rtsp_averror(reply->status_code, AVERROR_INVALIDDATA);
    }
    if (!content)
        return AVERROR_INVALIDDATA;

    av_log(s, AV_LOG_VERBOSE, "SDP:\n%s\n", content);
    /* now we got the SDP description, we parse it */
    ret = ff_sdp_parse(s, (const char *)content);
    av_freep(&content);
    if (ret < 0)
        return ret;

    return 0;
}
```

##### ff_rtsp_make_setup_request

发送 SETUP 命令给 rtsp 服务器

##### rtsp_read_play

```c++
static int rtsp_read_play(AVFormatContext *s)
{
    RTSPState *rt = s->priv_data;
    RTSPMessageHeader reply1, *reply = &reply1;
    int i;
    char cmd[MAX_URL_SIZE];

    av_log(s, AV_LOG_DEBUG, "hello state=%d\n", rt->state);
    rt->nb_byes = 0;

    if (rt->lower_transport == RTSP_LOWER_TRANSPORT_UDP) {
        for (i = 0; i < rt->nb_rtsp_streams; i++) {
            RTSPStream *rtsp_st = rt->rtsp_streams[i];
            /* Try to initialize the connection state in a
             * potential NAT router by sending dummy packets.
             * RTP/RTCP dummy packets are used for RDT, too.
             */
            if (rtsp_st->rtp_handle &&
                !(rt->server_type == RTSP_SERVER_WMS && i > 1))
                ff_rtp_send_punch_packets(rtsp_st->rtp_handle);
        }
    }
    if (!(rt->server_type == RTSP_SERVER_REAL && rt->need_subscription)) {
        if (rt->transport == RTSP_TRANSPORT_RTP) {
            for (i = 0; i < rt->nb_rtsp_streams; i++) {
                RTSPStream *rtsp_st = rt->rtsp_streams[i];
                RTPDemuxContext *rtpctx = rtsp_st->transport_priv;
                if (!rtpctx)
                    continue;
                ff_rtp_reset_packet_queue(rtpctx);
                rtpctx->last_rtcp_ntp_time  = AV_NOPTS_VALUE;
                rtpctx->first_rtcp_ntp_time = AV_NOPTS_VALUE;
                rtpctx->base_timestamp      = 0;
                rtpctx->timestamp           = 0;
                rtpctx->unwrapped_timestamp = 0;
                rtpctx->rtcp_ts_offset      = 0;
            }
        }
        if (rt->state == RTSP_STATE_PAUSED) {
            cmd[0] = 0;
        } else {
            snprintf(cmd, sizeof(cmd),
                     "Range: npt=%"PRId64".%03"PRId64"-\r\n",
                     rt->seek_timestamp / AV_TIME_BASE,
                     rt->seek_timestamp / (AV_TIME_BASE / 1000) % 1000);
        }
        ff_rtsp_send_cmd(s, "PLAY", rt->control_uri, cmd, reply, NULL);
        if (reply->status_code != RTSP_STATUS_OK) {
            return ff_rtsp_averror(reply->status_code, -1);
        }
        if (rt->transport == RTSP_TRANSPORT_RTP &&
            reply->range_start != AV_NOPTS_VALUE) {
            for (i = 0; i < rt->nb_rtsp_streams; i++) {
                RTSPStream *rtsp_st = rt->rtsp_streams[i];
                RTPDemuxContext *rtpctx = rtsp_st->transport_priv;
                AVStream *st = NULL;
                if (!rtpctx || rtsp_st->stream_index < 0)
                    continue;

                st = s->streams[rtsp_st->stream_index];
                rtpctx->range_start_offset =
                    av_rescale_q(reply->range_start, AV_TIME_BASE_Q,
                                 st->time_base);
            }
        }
    }
    rt->state = RTSP_STATE_STREAMING;
    return 0;
}
```



## 查询流信息

```c++
if (avformat_find_stream_info(format_ctx, NULL) < 0)
    {
        return EXIT_FAILURE;
    }
```

### avformat_find_stream_info



​	
