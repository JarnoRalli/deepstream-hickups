# Deepstrem RTSP

Depending on the capabilities of the media server serving RTSP streams, sometimes Deepstream pipelines don't start properly and there are no clear
error messages indicating what the error might be caused by.

Tested using
* Setup 1
  * Desktop x64 with AMD CPU
  * NVIDIA-SMI 565.57.01 Driver Version: 565.57.01 CUDA Version: 12.7
  * NVIDIA Corporation GP104 [GeForce GTX 1070] (rev a1)
  * Deepstream 7.0
* Setup 2
  * Jetson Xavier NX
  * NVIDIA Jetson Xavier NX Developer Kit - Jetpack 5.1 [L4T 35.2.1]
  * Deepstream 6.2

## 1. Contents

* [sample_1080p_h264.mp4](./sample_1080p_h264.mp4)
  * Sample video from Deepstream
* [docker-compose.yml](./docker-compose.yml)
  * A Docker compose file used for starting streaming
* [rtsp-simple-server.yml](./rtsp-simple-server.yml)
  * Configuration file used by docker-compose.yml
* [gst-rtsp-server.py](./gst-rtsp-server.py)

## 2. MediaMTX Media Server Test

This example uses the [MediaMTX](https://github.com/bluenviron/mediamtx) media server for streaming the same video over two streams. The Docker Compose
file is as follows:

```bash
services:
  rtsp-server:
    image: bluenviron/mediamtx #aler9/rtsp-simple-server
    container_name: rtsp-server
    ports:
      - "8554:8554"
      - "1935:1935"
      - "8888:8888"
    volumes:
      - ./rtsp-simple-server.yml:/rtsp-simple-server.yml

  camera1:
    image: jrottenberg/ffmpeg
    command: >
      -re -stream_loop -1
      -i /data/sample_1080p_h264.mp4
      -map 0:v:0
      -c:v libx264
      -f rtsp
      rtsp://rtsp-server:8554/camera1
    depends_on:
      - rtsp-server
    volumes:
      - ./:/data

  camera2:
    image: jrottenberg/ffmpeg
    command: >
      -re -stream_loop -1
      -i /data/sample_1080p_h264.mp4
      -map 0:v:0
      -c:v libx264
      -f rtsp
      rtsp://rtsp-server:8554/camera2
    depends_on:
      - rtsp-server
    volumes:
      - ./:/data
```

Once streaming has been started using:

```bash
docker compose up
```

, the streams can be played back with ffmpeg:

```bash
ffplay -rtsp_transport tcp rtsp://localhost:8554/camera1
```

, or with GStreamer:

```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost:8554/camera1 protocols=tcp latency=200 \
! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

### 2.1 x64 Setup

However, when trying to playback either of the streams using a pipeline that uses Deepstream's `nvv4l2decoder` as follows:

```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost:8554/camera1 protocols=tcp latency=500 \
! rtph264depay ! h264parse ! nvv4l2decoder ! queue ! nvvideoconvert ! queue \
! mux.sink_1 nvstreammux name=mux width=1920 height=1080 batch-size=1 live-source=1 \
! queue ! nvvideoconvert ! queue ! nvdsosd ! queue ! nveglglessink
```

, the pipeline does not show anything. Following is the output to the terminal:

```bash
Setting pipeline to PAUSED ...
Pipeline is live and does not need PREROLL ...
Got context from element 'eglglessink0': gst.egl.EGLDisplay=context, display=(GstEGLDisplay)NULL;
Progress: (open) Opening Stream
Pipeline is PREROLLED ...
Prerolled, waiting for progress to finish...
Progress: (connect) Connecting to rtsp://localhost:8554/camera1
Progress: (open) Retrieving server options
Progress: (open) Retrieving media info
Progress: (request) SETUP stream 0
Progress: (open) Opened Stream
Setting pipeline to PLAYING ...
New clock: GstSystemClock
Progress: (request) Sending PLAY request
Progress: (request) Sending PLAY request
Progress: (request) Sent PLAY request
Redistribute latency...
```

The only indication that something might be wrong is:

```bash
Got context from element 'eglglessink0': gst.egl.EGLDisplay=context, display=(GstEGLDisplay)NULL;
```

Starting the pipeline with GST debug level 3 doesn't reveal anything obvious either. If a proper stream format cannot be negoatiated, 
typically GStreamer components print an error message out.

### 2.2 Jetson Setup

Running the following in the Jetson Xavier NX:


```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost.213:8554/camera1 protocols=tcp latency=500 \
! rtph264depay ! h264parse ! nvv4l2decoder ! queue ! nvvideoconvert ! queue \
! mux.sink_1 nvstreammux name=mux width=1920 height=1080 batch-size=1 live-source=1 \
! queue ! nvvideoconvert ! queue ! nvdsosd ! queue ! nvegltransform ! nveglglessink
```

Produces the following error message.

```bash
Stream format not found, dropping the frame
```

, so at least the developer has something to start debugging from. 

## 3. GStreamer RTSP Server

GStreamer's RTSP server appears to work fine, though. [gst-rtsp-server.py](gst-rtsp-server.py), found in this same directory, is a simple Python application
that plays back a file. The server can be started directly in the host:

```bash
python gst-rtsp-server.py --file=./sample_1080p_h264.mp4
```

Or inside a [Docker container](Dockerfile-rtsp-server). You can build the container as follows:

```bash
docker build -t gst-rtsp-server -f ../docker/Dockerfile-rtsp-server .
```

, and run it:

```bash
docker run -p 8554:8554 -v $(pwd):/home -it gst-rtsp-server bash
cd /home
python3 gst-rtsp-server.py --file=./sample_1080p_h264.mp4
```

Once the GStreamer RTSP server is running, you should be able to playback the streams with ffmpeg, GStreamer and GStreamer with Deepstream's `nvv4l2decoder`.

```bash
ffplay -rtsp_transport tcp rtsp://localhost:8554/camera1
```

, or with GStreamer:

```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost:8554/camera1 protocols=tcp latency=200 \
! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
```

### 3.1 x64 Setup

Following works in the x64 setup:

```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost:8554/camera1 protocols=tcp latency=500 \
! rtph264depay ! h264parse ! nvv4l2decoder ! queue ! nvvideoconvert ! queue \
! mux.sink_1 nvstreammux name=mux width=1920 height=1080 batch-size=1 live-source=1 \
! queue ! nvvideoconvert ! queue ! nvdsosd ! queue ! nveglglessink
```

### 3.2 Jetson

Following works in the Jetson setup:

```bash
gst-launch-1.0 rtspsrc location=rtsp://localhost.213:8554/camera1 protocols=tcp latency=500 \
! rtph264depay ! h264parse ! nvv4l2decoder ! queue ! nvvideoconvert ! queue \
! mux.sink_1 nvstreammux name=mux width=1920 height=1080 batch-size=1 live-source=1 \
! queue ! nvvideoconvert ! queue ! nvdsosd ! queue ! nvegltransform ! nveglglessink
```

## 4 Conclusion

I often see differences in the level 
of error messages being printed out depending on the HW that Deepstream components are run on. Sometimes these differences can be found from Nvidia's forums, but quite often it happens
that there are no proper error messages and the developer is left at guessing what might be the causes for the problem. From time-to-market perspective, the longer it takes to iron out the kinks,
the less sales we can generate.
