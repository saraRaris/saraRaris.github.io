---
layout: default
title: Cameras
parent: Hardware
nav_order: 2
---


**Camera specs:**

- Cellular camera -> the data traffic should be small enough (10-24 GB per month)
- At least FULL HD camera or above
- Night Vision
- Motion detection would be super (I think) -> to reduce data traffic
- CSMO and rolling shutter should be enough
- 15-20 FPS should be enough for people detection -> 25 if we are dealing with vehicles
- Which FoV
  - The smaller the focal length the widest will be the FoV
    - 3.6mm will give 72 degrees and can be used for small offices or residential
    - 4mm I think would work better gives you 60 degrees -> sees at 16 m no problem https://www.networkwebcams.co.uk/blog/2020/01/30/cctv-camera-lens-guide/
- What distance do I want it to work at? 10-30m
  -  Sensor resolution? between 2MP and 3MP
- Lightning conditions - autoexposure for exteriors
- For what I have seen 100 dolar cameras are very bad quality -> https://www.amazon.com/Foscam-FI8905W-Outdoor-Wireless-Nightvision/dp/B003YUEF0E?th=1



**More info**

For prototyping: cheap industrial CCTV camera, aroun 5MP, CMOS sensor

For detection 3MP at 60 degrees FoV is enough to detect at 30m http://www.golbong.com/Quality-image-identify-face-with-CCTV-camera.asp 



**Detection calculation:**

object height = [focal length (mm) * real height (m) * image height (pixels)] / [distance m * sensor height (mm)]

A typical sensor height is 1/2.8" --> 4,826 x 3,556 mm

With this calculations: With a 2MP camera, 4mm focal length (60 degrees FoV), at 30 m
                                                a peson of 1.80 m -> 72 pixels
                                                a hat of 0.25 cm -> 10 pixels

Yolov3 can detect objects of 15-30 pixelsYolov3 can detect objects of 15-30 pixels: [source]( https://github.com/pjreddie/darknet/issues/1535)



##### **Data transmission calculations**

How much data will be transmitting? [link1](https://www.cctvcameraworld.com/ip-cameras-frame-rate-bandwidth/), [reolink](https://reolink.com/ip-camera-bandwidth-calculation/)

For an HD camera: 1920x1080 resolution x 24 bits / 8 bytes (8 bit in 1 byte) = 6.220.800
                                        x 20 FPS = 124.416.000 bytes/second
                                        118,65Mbytes/second
                                        417,12Gigabytes/hour

With a H.264 codec: compression of 2000:1 -> 0,0593 Mbytes/second ---> 5 GB/day



**Other data consumtion numbers**

Twitch is able to use 27.61MB/min --> 0.460MB/sec https://www.quora.com/Livestreaming-How-to-calculate-consumed-data-for-streaming-live-video

Data plans: https://support.reolink.com/hc/en-us/articles/360008997773-How-Much-Data-Does-Reolink-Go-Need-in-Normal-Usage ---> esta camera serian unos **21GB per day**

This camera https://www.blick-store.de/Important-questions-about-our-LTE-4G-cctv-cameras-with-SIM-card-and-mobile-transmission consumes **9,6GB per day** trasmitting everyday
Esto se traduce en 6Mbytes/min ---> 0.1137 Mbytes/sec

https://info.verkada.com/surveillance-features/bandwidth/
The average bandwidth consumption of an IP cloud camera is 1-2 Mbps (assuming 1080p using H.264 codec at 6-10fps). A hybrid cloud camera averages a fraction of that, ranging 5-50 Kbps in steady-state.


These are very off ---> 

https://www.ezs3.com/public/What_bitrate_should_I_use_when_encoding_my_video_How_do_I_optimize_my_video_for_the_web.cfm

https://help.encoding.com/knowledge-base/article/understanding-bitrates-in-video-files/ 
Segun esto HD H264 -> 35MB/min



**Secure network**

https://www.oliviawireless.com/networking-for-ip-cameras



**Plan and blockers**

The easiest thing: would be to grab one of the camera that we know work with AWS IoT Core

Otherwise: It's simple to visualize but it's not that simple to access the data -> i.e put it in your own platform

  - I need to know if the camera will allow me to access the RTSP 
  - For accessing from a client in the same network: can use python opencv
      - Use ImageZMQ [link](https://www.pyimagesearch.com/2019/04/15/live-video-streaming-over-network-with-opencv-and-imagezmq/) instead of videocapture [link](https://stackoverflow.com/questions/49978705/access-ip-camera-in-python-opencv)
      - Video streaming in web browsers (might not be relevant) [link](https://towardsdatascience.com/video-streaming-in-web-browsers-with-opencv-flask-93a38846fe00)
  - Ways to streams to aws  - How to stream data from IP Cameras over AWS?
      - Video Kinesis tutorials [link1](https://studymachinelearning.com/stream-cctv-ip-camera-rtsp-feed-into-aws-kinesis-video-streams/) [link2](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/examples-rtsp.html)
      - Port forward --> to access the camera stream
        - [link](https://getlockers1.medium.com/how-to-remotely-view-security-cameras-using-the-internet-264d58d97768), [link2](https://www.dacast.com/blog/how-to-connect-a-network-camera-to-a-rtmp-online-video-platform/)
      - Add camera stream to aws panorama (?) [link](https://github.com/awsdocs/aws-panorama-developer-guide/blob/main/docs-source/gettingstarted-setup.md)
  - To build the whole solution in a secure way (integrate camera api to aws)
      - You need a developer SDK -> Axis has these
      - And you need to build some things around that
      - Tutorial on how to build an app to connect a camera to aws [link](https://spiegelmock.com/2016/03/27/diving-into-iot-development-using-aws/) [github](https://github.com/revmischa/cloudcam)
- **Security**
  - Look into aws iot security
    - AWS IoT
      - In [this](https://github.com/aws-quickstart/quickstart-onica-connected-camera/issues?q=is%3Aissue+is%3Aclosed) github there info on how to connect cameras
      - Intructions and cameras that work with AWS IoT core [here](https://aws-quickstart.s3.amazonaws.com/quickstart-onica-connected-camera/doc/aws-iot-camera-connector-on-the-aws-cloud.pdf)
  - There are other companies suck as [this](https://www.nabto.com/esp32/) trying to make connections to cameras more secure
  - How to hack a camera [here](https://www.contextis.com/en/blog/push-hack-reverse-engineering-ip-camera) 

192.168.188.34
