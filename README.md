环境配置
----
我们在 ubuntu 下进行开发，本文所有命令都只针对 ubuntu，请安装最新版的 ubuntu。
首先安装开发所需要的包：

    apt-get install build-essential cmake
    apt-get install libavcodec-dev libavformat-dev libavdevice-dev libavfilter-dev libx264-dev

也可以自行下载 libav 的代码进行手动安装。

在编译和链接相关代码的时候，推荐大家使用 cmake 进行编译，因为库比较多，用 cmake 管理会简单得多。不然 Makefile 写起来太麻烦了，libav 的每个库又依赖与其他的库，难以处理。再次声明： *ubuntu 一定要最新版的* ，不然 cmake 可能找不到 libav 的库。

本文给出的所有范例代码，都是一个目录，里面有一个 CMakeList.txt 和一些 c 代码。编译的命令为：

    cmake .
    make

打开一个 mp4 文件
-----

这是一个简单的程序，它打开了一个 mp4 文件并且解析文件中存在哪些流。找到 h264 流对应的解码器。

    #include <stdio.h>
    #include <libavcodec/avcodec.h>
    #include <libavformat/avformat.h>
    #include <libavutil/avutil.h>
    #include <libavutil/dict.h>
    #include <ctype.h>
    #include <stdlib.h>
    #include <unistd.h>
    
    int main() 
    {
      AVFormatContext *ifc;
    	char *input_filename = "1.mp4", *output_filename;
    	AVStream *st_h264 = NULL;
    	int i;
    
      // 在使用 libav 之前，必须使用这两句做初始化
    	av_log_set_level(AV_LOG_DEBUG);
    	av_register_all();
    
      // 使用 avformat_open_input 打开文件
    	printf("opening %s\n", input_filename);
    	i = avformat_open_input(&ifc, input_filename, NULL, NULL);
    	printf("open %d, nb_streams: %d\n", i, ifc->nb_streams);
    
      // 使用 avformat_find_stream_info 解析文件中有哪些 stream
    	avformat_find_stream_info(ifc, NULL);
    	for (i = 0; i < ifc->nb_streams; i++) {
    		AVStream *st = ifc->streams[i];
    		AVCodec *c = avcodec_find_decoder(st->codec->codec_id);
    		printf("%s\n", c->name);
    		if (!strcmp(c->name, "h264")) // h264 流的 codec name 为 "h264"
    			st_h264 = st;
    	}
    
      // 先 find_decoder 然后用 avcodec_open2 来找到解码器
    	AVCodec *c = avcodec_find_decoder(st_h264->codec->codec_id);
    	i = avcodec_open2(st_h264->codec, c, NULL);
    	printf("codec_open %d\n", i);
    }
    
代码编译之后的输出为

    opening 1.mp4
    open 0, nb_streams: 2
    h264
    aac
    codec_open 0

可见 1.mp4 中有两个流，一个是 aac 流，一个是 h264 流。目前流行的封装就是如此。

读取视频帧
----



libx264 的简单编码
----
这是一段简单的编码代码。

    #include <stdio.h>
    #include <stdint.h>
    #include <string.h>
    #include <x264.h>
    
    int main() 
    {
        x264_param_t param;
    
    	x264_param_default_preset(&param, "ultrafast", "");
    	param.i_log_level = X264_LOG_DEBUG;
    	param.i_width = 640;
    	param.i_height = 360;
    	param.i_csp = X264_CSP_I420;
    
    	x264_t *h = x264_encoder_open(&param);
    	printf("264 %p\n", h);
    
    	static char ydata[672];
    	static char udata[336];
    	static char vdata[336];
    
    	int n;
    	for (n = 0; n < 10; n++) {
    		x264_picture_t picin, picout;
    		memset(&picin, 0, sizeof(picin));
    		// TODO: 我还没有搞明白 YUV 的 linesize 换算关系
    		picin.img.i_stride[0] = sizeof(ydata);
    		picin.img.i_stride[1] = sizeof(udata);
    		picin.img.i_stride[2] = sizeof(vdata);
    		picin.img.plane[0] = ydata;
    		picin.img.plane[1] = udata;
    		picin.img.plane[2] = vdata;
    		picin.img.i_csp = X264_CSP_I420;
    		// TODO: 弄明白 pts 的单位
    		picin.i_pts = n*10000;
    
    		x264_nal_t *nal;
    		int i_nal;
    		int r;
    		// TODO: 怎么释放 picout 占用的内存？
    		r = x264_encoder_encode(h, &nal, &i_nal, &picin, &picout);
    		printf("#%d encode=%d\n", n, r);
    		if (r) {
                // 编码后的裸帧在 nal[0].p_payload
                // x264_encoder_encode 的返回值为裸帧的大小
    			printf("#%d payload len=%d data=%p\n", n, r, nal[0].p_payload);
    		}
    	}
    }

输出为

    x264 [debug]: using mv_range_thread = 24
    x264 [info]: using cpu capabilities: MMX2 SSE2Fast SSSE3 FastShuffle SSE4.2
    x264 [info]: profile Constrained Baseline, level 3.0
    264 0x9398020
    #0 encode=0
    #1 encode=0
    #2 encode=0
    #3 encode=0
    #4 encode=0
    #5 encode=0
    #6 encode=0
    x264 [debug]: frame=   0 QP=20.00 NAL=3 Slice:I Poc:0   I:920  P:0    SKIP:0    size=42664 bytes
    #7 encode=42664
    #7 payload len=42664 data=0xb73aa020
    x264 [debug]: frame=   1 QP=7.00 NAL=2 Slice:P Poc:2   I:724  P:196  SKIP:0    size=40974 bytes
    #8 encode=40974
    #8 payload len=40974 data=0xb73aa020
    x264 [debug]: frame=   2 QP=5.00 NAL=2 Slice:P Poc:4   I:76   P:159  SKIP:685  size=14991 bytes
    #9 encode=14991
    #9 payload len=14991 data=0xb73aa020
    
可以观察到，x264 在第 7 帧的时候真正开始编码，因为 x264 编码是需要一开始积攒几帧然后再做一些预测类的处理，才能输出。

