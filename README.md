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

        int n = 0;
    	for (n = 0; n < 10; n++) {
    		AVPacket pkt;
    		int i = av_read_frame(ifc, &pkt);
    		printf("read %d, pkt: size=%d index=%d\n", i, pkt.size, pkt.stream_index);
    		if (pkt.stream_index != st->index)
    			continue;
    		int got_pic; 
    		i = avcodec_decode_video2(st->codec, frm, &got_pic, &pkt);
    		printf("decode %d, w=%d h=%d\n", i, frm->width, frm->height);
    		if (got_pic && frm->key_frame)
    			break;

pkt 中保存的是视频文件中的帧，可能是音频帧，也可能是视频帧。如果是原始的 h264 帧，调用解码函数。

如果解码后有图像生成，则 got_pic 为 1。因为 h264 不一定每帧都是保存图像的，有可能保存别的信息。

frm 保存了解码后图像的信息。如果是关键帧，那么 key_frame 为 1。

获取所有关键帧的位置
----

        AVStream *st = st_h264;
    	int i, r;
    
    	printf("nb_index_entries: %d\n", st->nb_index_entries); 
    	for (i = 0; i < st->nb_index_entries; i++) {
    		AVIndexEntry *ie = &st->index_entries[i];
    		printf("#%d pos=%lld ts=%lld\n", i, ie->pos, ie->timestamp);
    	}

st->index_entries 中保存了视频流所有关键帧的位置。

跳转到指定位置
----

    av_seek_frame(ifc, st_h264->index, 1000*200, 0);
    
跳转到视频流的第 200 秒。

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
    		picin.img.i_stride[0] = sizeof(ydata);
    		picin.img.i_stride[1] = sizeof(udata);
    		picin.img.i_stride[2] = sizeof(vdata);
    		picin.img.plane[0] = ydata;
    		picin.img.plane[1] = udata;
    		picin.img.plane[2] = vdata;
    		picin.img.i_csp = X264_CSP_I420;
    		// TODO: pts 的单位是什么？
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

plane 数组分别指向图像的 Y，U，V 原始数据。在 h264 的编码中默认使用的就是 I420 形式的原始数据。特点如下：

![](yplane.gif?raw=true)
![](u2plane.gif?raw=true)
![](v2plane.gif?raw=true)

http://www.fourcc.org/yuv.php#IYUV 这里有更详细的解释。

i_stride 数组分别保存了 Y，U，V 数据的大小。这个大小应该怎么计算呢？

    i_stride = ( ( h->param.i_width + 15 )&0xfffff0 )+ 64; //宽度352+64=416

http://hi.baidu.com/rhzrfwwrtojsvxr/item/220262e3d1ccc9f5e0a5d4da 这里有更详细的解释。

大概原因是由于 x264 库需要额外保存一些东西，数组要开大一点。

程序输出为

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

解码后再编码
----
libav 与 libx264 的数据结构对应关系为

        	picin.img.i_stride[0] = frm->linesize[0];
    		picin.img.i_stride[1] = frm->linesize[1];
    		picin.img.i_stride[2] = frm->linesize[2];
            
            picin.img.plane[0] = frm->data[0];
    		picin.img.plane[1] = frm->data[1];
    		picin.img.plane[2] = frm->data[2];

frm 为 AVFrame 类型。

rtmp 简介
----
简单的说，rtmp 服务器就是一个电视台，有多个频道。在服务器端，多个视频提供者。macromedia 官方的 Flash Media Server 里面包含了“电视台”和“视频提供者”两个组件。客户端就是 Flash Player，每个浏览器都有，协议也都一致。

在开源项目中，推荐使用 nginx-rtmp-module 和 libav 作为“电视台”和“视频提供者”。这两个开源项目相当稳定和成熟，而且便于部署。我曾经使用过 gstreamer，但发现比较难弄，因为 gstreamr 是属于 freedesktop 项目的东西，严重依赖于 glib 底层结构，导致它的代码虽然是用 C 写的，但是比 C++ 还难懂，用起来相当麻烦。

nginx 是一个 http 服务器，有人专门为它开发了一个模块用于 rtmp 的转发，如果正好要做流媒体的 web 应用，选择 nginx 就是一举两得。

libav 中的 avconv 命令可以方便的将任何视频封转发成 rtmp 流。

安装 nginx-rtmp-module
----
项目地址在 [这儿](https://github.com/arut/nginx-rtmp-module)。推荐先看一下它的 README。

下载 nginx 的源码，并添加 nginx-rtmp-module 模块

    git clone https://github.com/arut/nginx-rtmp-module
    apt-get build-dep nginx
    apt-get source nginx
    cd nginx-1.0.5/
    ./configure --prefix=/usr --add-module=../nginx-rtmp-module
    make
    make install
    
编译安装完之后，将 /etc/nginx.conf 修改如下：

    worker_processes  1;
    
    error_log  logs/error.log debug;
    
    
    events {
        worker_connections  1024;
    }
    
    http {
    
        server {
    
            listen      8080;
    
            location /publish {
                return 201;
            }
    
            location /play {
                return 202;
            }
    
            location /record_done {
                return 203;
            }
    
            location /stat {
                rtmp_stat all;
                rtmp_stat_stylesheet stat.xsl;
            }
    
            location /stat.xsl {
                root /tmp/;
            }
    
            location /rtmp-publisher {
                root /tmp/test;
            }
    
            location / {
                root /tmp/test/www;
            }
    
        }
    }
    
    rtmp {
    
        server {
    
            listen 1935;
    
            chunk_size 128;
    
            publish_time_fix off;
    
            application myapp {
    
                live on;
    
                record keyframes;
                record_path /tmp;
    
                record_max_size 128K;
                record_interval 30s;
    
                record_suffix .this.is.flv;
    
                on_publish http://localhost:8080/publish;
                on_play http://localhost:8080/play;
                on_record_done http://localhost:8080/record_done;
            }
    
            application myapp2 {
                live on;
            }
    
            application mypull {
                live on;
            }
    
            application mypush {
                live on;
            }
        }
    }
    
使用 avconv 命令发送视频给 nginx-rtmp-module
----

    avconv -re -i a.mp4 -c:a copy -c:v copy -f flv rtmp://localhost/myapp/1

其中 `-re` 参数，man avconv 给出的解释是“Read input at native frame rate. Mainly used to simulate a grab device.”

`-f flv` 参数的意思是改变封装格式为 flv，以便输出到 rtmp。

`-c:a copy` 参数的意思是保持音频编码器不变，就是只是改变最外层的格式封装（由 mp4 改为 flv），不重新编码。原来是 aac 格式，现在还是 aac 格式。

`-c:v copy` 参数的指的是视频。保持 h264 格式不变。

`rtmp://localhost/myapp/1` 这个 url 跟上面 nginx.conf 的里面的 application myapp 起头的那行，对应起来了。

此外 avconv 是一个很强大的命令，可以指定从那一秒开始播放文件，也可以改变视频的大小，也可以加一些滤镜效果。

在网页上接收 rtmp 流
----
需要 Flash Player 的支持。但是各大浏览器全部都支持 Flash Player，而且 rtmp 又是比较老的技术，所以不用担心播放不了的情况。
可以使用封装好了的 jwplayer 。jwplayer 是一个网页 flash 视频播放器开源项目，能够通过 javascript 很方便的嵌入播放器，以及调整大小之类的操作。

首先在 [这里](https://github.com/luminosity-group/jwplayer) git clone 一份 jwplayer

    git clone https://github.com/luminosity-group/jwplayer
    
然后从 jwplayer 下面复制 jwplayer.min.js 和 player.swf 出来。

网页代码如下

    <script type="text/javascript" src="jwplayer.min.js"></script>
    <script >
        function setup(file) {
    		jwplayer("container").setup({
    			'width': '720px',
    			'height': '576px',
    			'controlbar': 'none',
    			'logo.hide': true,
    			events: {
    			},
    			modes: [{
    				type: "flash",
    				src: "player.swf",
    				config: {
    					'viral.onpause': false,
    					'autostart': true,
    					'viral.oncomplete': false,
    					'stretching':'none',
    					smoothing:false,
    					file: file,
    					streamer: "rtmp://192.168.1.66/myapp",
    					provider: "rtmp",
    				}
    			}]
    		});
    	}
        setup(1);
    </script>
    <div id="container">
    	Loading the player ...
    </div>

setup 函数的参数中。`logo.hide` 是设置是否隐藏播放器右下角的 logo。controlbar 是设置是否显示进度条。streamer 的地址指向我们的服务器地址。file 是频道名称。

先使用刚刚那段 avconv 的脚本创建频道1。然后再打开网页，就可以看到有视频在直播了。

在直播中循环播放视频
----

值得一提的是，采用 rtmp 做网页直播非常方便。在直播一段视频的时候，如果遇到以下情况：

* 视频文件需要切换
* 视频的大小和帧率都会发生改变

采用 rtmp 做服务器则不会出问题。avconv 进程退出后，频道会被释放，客户端的画面会马上停止，然后变成黑屏。再启动一个 avconv 进程，则直播会继续。
视频的大小帧率发生改变不用另外写代码处理，jwplayer 也不会受到影响，视频大小无论是多少，画面始终居中。

那么我们可以在直播中一直循环播放一些视频文件。只需要在后台不停的使用 avconv 发送视频流给 rtmp 服务器即可。脚本如下：

    while true; do
        avconv -re -i a.mp4 -c:a copy -c:v copy -f flv rtmp://localhost/myapp/1
        avconv -re -i b.mp4 -c:a copy -c:v copy -f flv rtmp://localhost/myapp/1
    done
    
这段代码在 a.mp4 播放完之后接着播放 b.mp4。不停止的一直播放下去。a.mp4 和 b.mp4 的大小和帧率不一样也没有任何问题。

切换频道
----

    avconv -re -i a.mp4 -c:a copy -c:v copy -f flv rtmp://localhost/myapp/1 &
    avconv -re -i b.mp4 -c:a copy -c:v copy -f flv rtmp://localhost/myapp/2 &

这样写的话，则分别在 频道1 和 频道2 播放 a.mp4 和 b.mp4。如果要在网页上添加多个按钮来选择频道，应该怎么做呢？
用 javascript 可以很方便的控制它们。

    <button onclick="setup(1)" >channel 1</button>
    <button onclick="setup(2)" >channel 2</button>

在刚刚的页面上加上这段代码就可以了。简单的来说，切换频道就是重新新建一次 jwplayer 播放器。它应该还更好的方法，有待研究。

查看 rtmp 频道的状态
----
在 http://localhost:8080/stat 页面可以看到当前 rtmp 的状态。包括有几个频道，有几个视频流正在直播，视频流的大小和帧率，等等。

