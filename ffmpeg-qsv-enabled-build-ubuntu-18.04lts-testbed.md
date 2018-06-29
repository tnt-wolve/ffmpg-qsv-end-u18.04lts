**Build FFmpeg with Intel's QSV enablement on an Intel-based validation test-bed:**

Build platform: Ubuntu

**Install baseline dependencies first**

`sudo apt-get -y install autoconf automake build-essential libass-dev libtool pkg-config texinfo zlib1g-dev libva-dev cmake mercurial libdrm-dev libvorbis-dev libogg-dev git libx11-dev libperl-dev libpciaccess-dev libpciaccess0 xorg-dev intel-gpu-tools`

Then add the Oibaf PPA, needed to install the latest development headers for libva:

```
sudo add-apt-repository ppa:oibaf/graphics-drivers
sudo apt-get update && sudo apt-get -y upgrade && sudo apt-get -y dist-upgrade

```

**Build the latest libva and all drivers from source:**


**Setup build environment:**

Work space init:

```
mkdir -p ~/vaapi
mkdir -p ~/ffmpeg_build
mkdir -p ~/ffmpeg_sources
mkdir -p ~/bin
```

Build the dependency chain as shown:

**1. [Libva :](https://github.com/intel/libva)**

Libva is an implementation for VA-API (Video Acceleration API)

VA-API is an open-source library and API specification, which provides access to graphics hardware acceleration capabilities for video processing. It consists of a main library and driver-specific acceleration backends for each supported hardware vendor. It is a prerequisite for building the VAAPI driver components below.

```
cd ~/vaapi
git clone https://github.com/01org/libva
cd libva
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
time make -j$(nproc) VERBOSE=1
sudo make -j$(nproc) install
sudo ldconfig -vvvv


```

**2. [Gmmlib:](https://github.com/intel/gmmlib)**

The Intel(R) Graphics Memory Management Library provides device specific and buffer management for the Intel(R) Graphics Compute Runtime for OpenCL(TM) and the Intel(R) Media Driver for VAAPI.

The component is a prerequisite to the Intel Media driver build step below.

To build this, create a workspace directory within the vaapi sub directory and run the build:

```
mkdir -p ~/vaapi/workspace
cd ~/vaapi/workspace
git clone https://github.com/intel/gmmlib
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE= Release -DARCH= 64 ../gmmlib
make -j$(nproc)


```

**3. [Intel Media driver:](https://github.com/intel/media-driver)**

The Intel(R) Media Driver for VAAPI is a new VA-API (Video Acceleration API) user mode driver supporting hardware accelerated decoding, encoding, and video post processing for GEN based graphics hardware, released under the MIT license.

```
cd ~/vaapi/workspace
git clone https://github.com/intel/media-driver
cd media-driver
git submodule init
git pull
mkdir -p ~/vaapi/workspace/build_media
cd ~/vaapi/workspace/build_media


```

Configure the project with cmake:

```
cmake ../media-driver \
-DMEDIA_VERSION="2.0.0" \
-DBUILD_ALONG_WITH_CMRTLIB=1 \
-DBS_DIR_GMMLIB=$PWD/../gmmlib/Source/GmmLib/ \
-DBS_DIR_COMMON=$PWD/../gmmlib/Source/Common/ \
-DBS_DIR_INC=$PWD/../gmmlib/Source/inc/ \
-DBS_DIR_MEDIA=$PWD/../media-driver \
-DCMAKE_INSTALL_PREFIX=/usr \
-DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu \
-DINSTALL_DRIVER_SYSCONF=OFF \
-DLIBVA_DRIVERS_PATH=/usr/lib/x86_64-linux-gnu/dri


```

Then build the media driver:

```
time make -j$(nproc) VERBOSE=1


```

Then install the project:

```
sudo make -j$(nproc) install VERBOSE=1


```

Add yourself to the video group:

```
sudo usermod -a -G video $USER


```

Now, export environment variables as shown below:

```
LIBVA_DRIVERS_PATH=/usr/lib/x86_64-linux-gnu/dri
LIBVA_DRIVER_NAME=iHD

```

Put that in `/etc/environment`.

**4. [libva-utils:](https://github.com/intel/libva-utils)**

This package provides a collection of tests for VA-API, such as `vainfo`, needed to validate a platform's supported features (encode, decode & postproc attributes on a per-codec basis by VAAPI entry points information).

```
cd ~/vaapi
git clone https://github.com/intel/libva-utils
cd libva-utils
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
time make -j$(nproc) VERBOSE=1
sudo make -j$(nproc) install
```

At this point, issue a reboot:

```
sudo systemctl reboot
```

Then on resume, proceed with the steps below.

**4. Build [Intel's MSDK](https://github.com/Intel-Media-SDK/MediaSDK):**

This package provides an API to access hardware-accelerated video decode, encode and filtering on Intel® platforms with integrated graphics. See the supported platform details below:

**Supported platforms:**

Compared to the open source VAAPI driver, this project supports the following SKU generations:

```
BDW (Broadwell)

SKL (Skylake)

BXT (Broxton) / APL (Apollolake)

CNL (Cannonlake)
```

Unless explicitly listed, any platform not in that list should be considered as unsupported. For these platforms, stick to upstream VAAPI.

**Build steps:**

(a). Fetch the sources into the working directory `~/vaapi`:

```
cd ~/vaapi
git clone https://github.com/Intel-Media-SDK/MediaSDK msdk
cd msdk
git submodule init
git pull
```

(b). Configure the build with GCC:

(i). For Apollo Lake:

```
perl tools/builder/build_mfx.pl --cmake=intel64.make.release --target=BXT
```

(ii). For SKL:

```
perl tools/builder/build_mfx.pl --cmake=intel64.make.release 
```

This will build MSDK binaries and MSDK samples.

(c ). Run the build:

```
make -j$(nproc) -C __cmake/intel64.make.release
```

(d). And install:

`cd __cmake/intel64.make.release`

`sudo make -j$(nproc) install`

Apply this workaround:

```
sudo mkdir -p /opt/intel/mediasdk/include/mfx
sudo cp -vr /opt/intel/mediasdk/include/*.h  /opt/intel/mediasdk/include/mfx
```

Create a library config file for the iMSDK:

```
sudo nano /etc/ld.so.conf.d/imsdk.conf
```

Content:

```
/opt/intel/mediasdk/lib
/opt/intel/mediasdk/plugins
```

Then run:

```
sudo ldconfig -vvvv
```

To proceed.

Confirm that `/opt/intel/mediasdk/lib/pkgconfig/libmfx.pc` and `/opt/mediasdk/lib/pkgconfig/mfx.pc` are populated correctly:

```
Name: mfx
Description: Intel(R) Media SDK Dispatcher
Version: 1.26  ((MFX_VERSION) % 1000)

prefix=/opt/intel/mediasdk
libdir=/opt/intel/mediasdk/lib
includedir=/opt/intel/mediasdk/include
Libs: -L${libdir} -lmfx -lstdc++ -ldl -lva -lstdc++ -ldl -lva-drm -ldrm
Cflags: -I${includedir} -I/usr/include/libdrm
```

When done, issue a reboot:

```
 sudo systemctl reboot
```

**Known Issues and Limitations:**

For validation, consider the following known issues:

1.  SKL: Green or other incorrect color will be observed in output frames when using YV12/I420 as input format for csc/scaling/blending/rotation, etc. on Ubuntu 16.04 stock (with kernel 4.10). The issue can be addressed with the kernel patch: [WaEnableYV12BugFixInHalfSliceChicken7 commit](https://cgit.freedesktop.org/drm-tip/commit/?id=0b71cea29fc29bbd8e9dd9c641fee6bd75f68274).
    
2.  CNL: Functionalities requiring HuC including AVC BRC for low power encoding, HEVC low power encoding, and VP9 low power encoding are pending on the kernel patch for GuC support which is expected in Q1’2018.
    
3.  BXT/APL: Limited validation was performed; product quality expected in Q1’2018.
    

**Build a usable FFmpeg binary with the iMSDK:**

Include extra components as needed:

**(a). Build and deploy nasm:** [Nasm](https://www.nasm.us/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow.

Note that we've now switched away from Yasm to nasm, as this is the current assembler that x265,x264, among others, are adopting.

```
cd ~/ffmpeg_sources
wget wget http://www.nasm.us/pub/nasm/releasebuilds/2.14rc0/nasm-2.14rc0.tar.gz
tar xzvf nasm-2.14rc0.tar.gz
cd nasm-2.14rc0
./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" 
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
```

**(b). Build and deploy libx264 statically:** This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples. This requires ffmpeg to be configured with `--enable-gpl --enable-libx264`.

```
cd ~/ffmpeg_sources
git clone http://git.videolan.org/git/x264.git -b stable
cd x264/
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --enable-static --disable-opencl
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install VERBOSE=1
make -j$(nproc) distclean
```

**(c ). Build and configure libx265:** This library provides a H.265/HEVC video encoder. See the [H.265 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.265) for more information and usage examples.

```
cd ~/ffmpeg_sources
hg clone https://bitbucket.org/multicoreware/x265
cd ~/ffmpeg_sources/x265/build/linux
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
make -j$(nproc) VERBOSE=1
make -j$(nproc) install VERBOSE=1
make -j$(nproc) clean VERBOSE=1
```

**(d). Build and deploy the libfdk-aac library:** This provides an AAC audio encoder. See the [AAC Audio Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC) for more information and usage examples. This requires ffmpeg to be configured with `--enable-libfdk-aac` (and `--enable-nonfree` if you also included `--enable-gpl`).

```
cd ~/ffmpeg_sources
wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
tar xzvf fdk-aac.tar.gz
cd mstorsjo-fdk-aac*
autoreconf -fiv
./configure --prefix="$HOME/ffmpeg_build" --disable-shared
make -j$(nproc)
make -j$(nproc) install
make -j$(nproc) distclean
```

**(e). Build and configure libvpx:**

```
   cd ~/ffmpeg_sources
   git clone https://chromium.googlesource.com/webm/libvpx
   cd libvpx
   ./configure --prefix="$HOME/ffmpeg_build" --enable-runtime-cpu-detect --enable-vp9 --enable-vp8 \
   --enable-postproc --enable-vp9-postproc --enable-multi-res-encoding --enable-webm-io --enable-better-hw-compatibility --enable-vp9-highbitdepth --enable-onthefly-bitpacking --enable-realtime-only --cpu=native --as=nasm 
   time make -j$(nproc)
   time make -j$(nproc) install
   time make clean -j$(nproc)
   time make distclean
```

**(f). Build LibVorbis:**

```
   cd ~/ffmpeg_sources
   wget -c -v http://downloads.xiph.org/releases/vorbis/libvorbis-1.3.6.tar.xz
   tar -xvf libvorbis-1.3.6.tar.xz
   cd libvorbis-1.3.6
   ./configure --enable-static --prefix="$HOME/ffmpeg_build"
   time make -j$(nproc)
   time make -j$(nproc) install
   time make clean -j$(nproc)
   time make distclean
```

**(g). Build FFmpeg:**

Notes on API support:

The hardware can be accessed through a number of different APIs:

libmfx on Linux

> This is a library from Intel which can be installed as part of the Intel Media SDK, and supports a subset of encode and decode cases.

In our use case, that is the backend that we will be using throughout this validation with this FFmpeg build:

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/opt/intel/mediasdk/lib/pkgconfig" ./configure \
  --pkg-config-flags="--static" \
  --prefix="$HOME/bin" \
  --bindir="$HOME/bin" \
  --extra-cflags="-I$HOME/bin/include" \
  --extra-ldflags="-L$HOME/bin/lib" \
  --extra-cflags="-I/opt/intel/mediasdk/include" \
  --extra-ldflags="-L/opt/intel/mediasdk/lib" \
  --extra-ldflags="-L/opt/intel/mediasdk/plugins" \
  --enable-libmfx \
  --enable-vaapi \
  --disable-debug \
  --enable-libvorbis \
  --enable-libvpx \
  --enable-libdrm \
  --enable-gpl \
  --cpu=native \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --extra-libs=-lpthread \
  --enable-nonfree 
PATH="$HOME/bin:$PATH" make -j$(nproc) 
make -j$(nproc) install 
make -j$(nproc) distclean 
hash -r


```

Note: To get debug builds, add the `--enable-debug=3` configuration flag and omit the `distclean` step and you'll find the `ffmpeg_g` binary under the sources subdirectory.

We only want the debug builds when an issue crops up and a gdb trace may be required for debugging purposes. Otherwise, leave this omitted for production environments.

**Sample snippets to test the new encoders:**

Confirm that the VAAPI & QSV-based encoders have been built successfully:

```
ffmpeg  -hide_banner -encoders | grep vaapi 

 V..... h264_vaapi           H.264/AVC (VAAPI) (codec h264)
 V..... hevc_vaapi           H.265/HEVC (VAAPI) (codec hevc)
 V..... mjpeg_vaapi          MJPEG (VAAPI) (codec mjpeg)
 V..... mpeg2_vaapi          MPEG-2 (VAAPI) (codec mpeg2video)
 V..... vp8_vaapi            VP8 (VAAPI) (codec vp8)

```

See the help documentation for each encoder in question:

```
ffmpeg -hide_banner -h encoder='encoder name'

```

**Test the encoders;**

Using GNU parallel, we will encode some mp4 files (4k H.264 test samples, 40 minutes each, AAC 6-channel audio) on the ~/src path on the system to VP8 and HEVC respectively using the examples below. Note that I've tuned the encoders to suit my use-cases, and re-scaling to 1080p is enabled. Adjust as necessary (and compensate as needed if `~/bin` is not on the system path).

**To VP8, launching 10 encode jobs simultaneously:**

```
parallel -j 10 --verbose 'ffmpeg -loglevel debug -threads 4 -hwaccel vaapi -i "{}"  -vaapi_device /dev/dri/renderD129 -c:v vp8_vaapi -loop_filter_level:v 63 -loop_filter_sharpness:v 15 -b:v 4500k -maxrate:v 7500k -vf 'format=nv12,hwupload,scale_vaapi=w=1920:h=1080' -c:a libvorbis -b:a 384k -ac 6 -f webm "{.}.webm"' ::: $(find . -type f -name '*.mp4')

```

**To HEVC with GNU Parallel:**

To HEVC Main Profile, launching 10 encode jobs simultaneously:

```
parallel -j 4 --verbose 'ffmpeg -loglevel debug -threads 4 -hwaccel vaapi -i "{}"  -vaapi_device /dev/dri/renderD129 -c:v hevc_vaapi -qp:v 19 -b:v 2100k -maxrate:v 3500k -vf 'format=nv12,hwupload,scale_vaapi=w=1920:h=1080' -c:a libvorbis -b:a 384k -ac 6 -f matroska "{.}.mkv"' ::: $(find . -type f -name '*.mp4')

```

**FFmpeg QSV encoder usage notes:**

**Complex Filter chain usage for variant stream encoding with Intel's QSV encoders:**

Take the example snippet below:

```
    ffmpeg -re -stream_loop -1 -threads n -loglevel debug -filter_complex_threads n \
    -init_hw_device qsv=qsv:MFX_IMPL_hw_any -hwaccel qsv -filter_hw_device qsv \
    -i 'udp://$stream_url:$port?fifo_size=9000000' \
    -filter_complex "[0:v]split=6[s0][s1][s2][s3][s4][s5]; \
    [s0]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=1920:1080:format=nv12[v0]; \
    [s1]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=1280:720:format=nv12[v1];
    [s2]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=960:540:format=nv12[v2];
    [s3]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=842:480:format=nv12[v3];
    [s4]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=480:360:format=nv12[v4];
    [s5]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=426:240:format=nv12[v5]" \
    -b:v:0 2250k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -b:v:1 1750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -b:v:2 1000k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -b:v:3 875k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -b:v:4 750k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -b:v:5 640k -c:v h264_qsv -a53cc 1 -rdo 1 -pic_timing_sei 1 -recovery_point_sei 1 -profile high -aud 1 \
    -c:a aac -b:a 128k -ar 48000 -ac 2 \
    -flags -global_header -f tee -use_fifo 1 \
    -map "[v0]" -map "[v1]" -map "[v2]" -map "[v3]" -map "[v4]" -map "[v5]" -map 0:a:0 -map 0:a:1 \
    "[select=\'v:0,a\':f=mpegts]udp:$stream_url_out:$port_out| \
    [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:0,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:1,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:2,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:3,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:4,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out| \
    [select=\'v:5,a\':f=mpegts]udp://$stream_url_out:$port_out"

```

**Template breakdown:**

The ffmpeg snippet assumes the following:

1.  The UDP ingest stream specifier has one video stream and two separate audio streams, as shown in the explicit mapping above: `-map "[v0]" -map "[v1]" -map "[v2]" -map "[v3]" -map "[v4]" -map "[v5]" -map 0:a:0 -map 0:a:1`

The mapping is needed because the [tee muxer](https://www.ffmpeg.org/ffmpeg-formats.html#tee-1) (`-f tee`) makes no assumptions about the capabilities of the underlying muxer launched beneath the fifo process.

2.  We encode audio only once. Consider audio as a [blocking encoder and minimize unnecessary encoder duplication](https://blog.twitch.tv/live-video-transmuxing-transcoding-ffmpeg-vs-twitchtranscoder-part-ii-4973f475f8a3) to save on CPU cycles.
    
3.  We split the incoming streams into six, and in so doing:
    

(a). Allocate a single thread to each filter complex chain. Thus the value `n` set for `-filter_complex_threads` should match the number of `split=n` value.

(b). Allocate the total thread count for FFmpeg to the value specified above. This ensures that each encoder is fed only through a single thread, the optimal value for hardware-accelerated encoding.

The following notes about thread allocation in FFmpeg applies doubly so:

More encoder threads beyond a certain threshold increases latency and will have a higher encoding memory footprint. Quality degradation is more prominent with higher thread counts in constant bitrate modes and near-constant bitrate mode called VBV ([video buffer verifier](https://en.wikipedia.org/wiki/Video_buffering_verifier)), due to increased encode delay. Keyframes need more data then other frame types to avoid pulsing poor quality keyframes.

Zero-delay or sliced thread mode (on supported encoders) has no delay, but this option farther worsens multi-threads quality in supported encoders.

It's therefore wise to limit thread counts on encodes where latency matters, as the perceived encoder throughput increase offsets any advantages it may bring in the long term.

4.  Through the tee muxer, we then allocate each encoded stream variant to an output, through the select statement in the bracketed clauses above. This allows us to generate exponentially more elementary streams than there are encoders.
    
5.  On the tuning options passed to the `h264_qsv` encoder:
    

(a). The `hwupload` filter must be appended with the `extra_hw_frames=10` option, because the QSV encoder expects a fixed initial pool size.  
Conceptually, this should happen automatically, but the problem is that the mfx plugins lack sufficient negotiation with the encoder to know how big the pool should be - if it's feeding into a scaler (such as the complex scale filter chain above), then probably 2 or 3 frames are sufficient, but if it's feeding into an encoder with look-ahead enabled then you might need over 100. As such, it's currently under user control and must be applied to ensure proper encoder initialization.

Without this option, the encoder will fail as shown below:

```
[AVHWFramesContext @ 0x3e26ec0] QSV requires a fixed frame pool size
[AVHWFramesContext @ 0x3e26ec0] Error creating an internal frame pool
[Parsed_hwupload_1 @ 0x3e26880] Failed to configure output pad on Parsed_hwupload_1
Error reinitializing filters!
Failed to inject frame into filter network: Invalid argument
Error while processing the decoded data for stream #0:0
[AVIOContext @ 0x3a68f80] Statistics: 0 seeks, 0 writeouts
[AVIOContext @ 0x3a6cd40] Statistics: 1022463 bytes read, 0 seeks
Conversion failed!

```

(b). When initializing the encoder, the hardware device node for [libmfx](https://trac.ffmpeg.org/wiki/Hardware/QuickSync) _must_ be initialized as shown:

```
-init_hw_device qsv=qsv:MFX_IMPL_hw_any -hwaccel qsv -filter_hw_device qsv

```

That ensures that the proper hardware accelerator node (`qsv`) is initialized with the proper device context (`-init_hw_device qsv=qsv:MFX_IMPL_hw_any`) with device nodes for a hardware accelerator implementation being inherited `(-hwaccel qsv`) with an appropriate filter device (`-filter_hw_device qsv`) are initialized for resource allocation by the `hwupload` filter, the `vpp_qsv` post-processor (needed for advanced deinterlacing) and the `scale_vpp` filter (needed for texture format conversion to nv12, otherwise the encoder will fail).

If hardware scaling is undesired, the filter chain can be modified from:

```
[sn]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=W:H:format=nv12[vn]

```

To:

```
[sn]hwupload=extra_hw_frames=10,vpp_qsv=deinterlace=2,scale_qsv=format=nv12[vn]

```

Where `n` is the stream specifier id inherited from the complex filter chain. Note that the texture conversion is mandatory, and cannot be skipped.

The other arguments passed to the encoder are optimal for smooth streaming, enabling automatic detection and use of closed captions, an advanced [rate distortion algorithm](https://en.wikipedia.org/wiki/Rate%E2%80%93distortion_optimization) and sensible bitrates and profile limits per encoder variant.

4. The open source iMSDK has a frame encoder limit of 1000 for the HEVC-based encoder, and as such, the HEVC encoder components should only be used for evaluation purposes. These that require these functions should consult the [proprietary licensed SDK](https://software.intel.com/en-us/media-sdk).

5. The `iHD` libva driver also provides similar VAAPI functionality as the opensource `i965` driver. 