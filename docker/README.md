# Docker Images

This directory contains docker files used for generating docker images where the examples can be run.

* [Dockerfile-rtsp-server](Dockerfile-rtsp-server)
  * Docker container with RTSP GStreamer components
  * Based on ubuntu:20.04
  * gstreamer1.0-plugins-base
  * gstreamer1.0-plugins-good
  * gstreamer1.0-plugins-bad
  * gstreamer1.0-plugins-ugly
  * gstreamer1.0-rtsp

# 1 Creating Docker Images

You can create Docker images as follows:

```bash
docker build -t <NAME-OF-THE-IMAGE> -f ./<NAME-OF-THE-DOCKERFILE> .
```

, replace `<NAME-OF-THE-IMAGE>` with the name you want the built Docker image to have and `<NAME-OF-THE-DOCKERFILE>` with
the Dockerfile you want to use for building an image.

