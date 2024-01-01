# Selecting and Configuring Cameras for Driver Dashboards

## Overview

For many years our team has spent a more-than-reasonable amount of time each year trying to configure our cameras to work well on our dashboard displays. This fall I decided to look into the situation. I found a bunch of useful information that is not easily discovered. 

The difficulty stems largely from the fact that WPILib documentation for cameras is mostly focused on vision tasks and thus doesn't cover cameras for the driver station very thoroughly. 

One obstacle is that the camera specifications and documentation typically available are woefully inadequate to predict whether a particular camera will be a good choice for a driving and/or vision camera. Finding out even the set of supported pixel formats, resolutions, and frame rates is essentially impossible until you hook up the camera. Discussion of inherent camera latency and the way camera settings interact with one another to constrain achievable frame rate is nowhere to be found.

On the WPILib side also there are interactions between settings in different parts of the code and configurations that can have profound impact on achieved performance. 

This article aims to fill some of that gap and tries to give a solid foundation for experimenting with cameras, camera configurations, dashboards and dashboard viewer configurations that will lead to repeatable behavior that meets teams' needs for  viewing cameras on their driving dashboards.

## Considerations for a driver dashboard camera

The uses for a driver dashboard camera vary from game to game but there are a number of common characteristics. From the perspective of the drive team, the most important are likely: camera field of view, image clarity, and image delay -- how long does it take for something that is seen by the camera to show up on the dashboard? 

The field of view is governed by the camera, its lens, and the aspect  ratios that it supports, typically 4:3, 16:9, or 16:10. The latency is affected by the camera itself, the frame rate, and the time to do any processing such as resolution scaling and compression -- typically performed by the roboRIO or a coprocessor as well as the time to transmit the image over the network to the dashboard computer.

Apart from how the display looks, the camera's configuration also affects how much network bandwidth is required between the robot and the driver station computer, and how much cpu load is placed on the roboRIO and/or coprocessor. Both of these can affect the controllability of the robot so care must be taken to make good choices.

Also, sometimes the same camera will be used for vision tasks and for the dashboard camera. This will lead to more tradeoffs as you balance the competing needs of the two uses.

## An important distinction

It may not be immediately obvious but there are at least two sets of configurations that interact to determine the overall system performance: each kind of camera has a set of pixel formats, native resolutions and frame rates that it supports. We will call that the camera-native configuration. The chosen camera-native configuration constrains the maximum resolution and frame rate that users of the camera can see.

In addition to the camera-native configuration there are separate configuration settings for each video stream supplied by the camera. These settings default to the camera-native configuration which often results in too-high a bit rate for practical use in competition (see below about "too high"). Also, if the camera is being used for vision as well as dashboard display, it is necessary to choose a high native resolution and frame rate to support vision and a lower streaming resolution and frame rate for viewing.

The next two sections cover how to set the camera-native and stream configurations. We then cover considerations in selecting cameras and finally offer some opinions, based on our experience, with setting up and configuring the whole system.

## Configuring camera-native parameters -- VideoMode

**Note about host addresses in URLS**
In the URLs below I write `<host>` to identify the network host where the camera server is running. `<host>` has to be replaced by the network address or name of the computer, typically an on-robot roboRIO or a Raspberry Pi (or similar) coprocessor. An example network address is `10.TE.AM.2` for the robot roboRIO (Replace `TE.AM` with your team's team number spread over 4 digits -- e.g. `40.61`. The roboRIO will have the name `roboRIO-TEAM-frc.local.` A Raspberry Pi running WPILibPi will have the name `wpilibpi.local`. How to find the network address of particular computer is beyond the scope of this article, but the `nmap` tool is often helpful.

From the above, we know that each camera supports a specific set of possible `VideoMode`s (pixel format, resolution and frame rate). If a USB camera is plugged into the roboRIO or Raspberry Pi running the WPILib camera server, you can use a web browser to visit `http://<host>:1181/` and see a list of supported `VideoMode`s as well as view the camera stream at the current settings. (Additional cameras on the same host can be seen by visiting `http://<host>:1182`, `http:<host>:1183`, etc.) You _cannot_ change the `VideoMode` from this page.

On the RPi you change the `VideoMode` using the `Vision Settings` tab on the WPILibPi web interface.  ![RPi-mainpage](https://github.com/chauser/CameraConfiguration/assets/834131/38d0e491-2905-4c5d-8465-56ae5edb9afc) ![RPi-camerasettings-select-camera](https://github.com/chauser/CameraConfiguration/assets/834131/c5f17893-fd32-4424-a7e8-67ee5c1e7d26)
![RPi-camerasettings-videomode](https://github.com/chauser/CameraConfiguration/assets/834131/eb79f73b-4d50-4381-91c6-5342cfeee5e5)



On the roboRIO you control the camera-native settings in your robot code.  Basic code for this in Java is:
```
UsbCamera cam = CameraServer.startAutomaticCapture();
cam.setVideoMode(VideoMode.PixelFormat.MJPEG, 360, 240, 15); // configure camera for MJPEG at 360x240 resolution and 15fps.
```
 C++ code is similar (see the WPILib documentation Vision documentation [here](wpilibdoc).
  
### Other camera settings
Cameras typically have additional settings beyond the VideoMode including things like exposure, brightness, hue, backlight compensation powerline frequency, etc. The ones that are applicable for an attached camera can be seen _and controlled_ on the `http://<host>:118x/` page. They can also be controlled through robot code, by loading camera configuration `.json` files on the Raspberry Pi camera configuration page, by changing values in the NetworkTables (look under the key `/CameraServer/cameraname/`) or by sending `http` requests -- se the next section.

**Important:** Changes to these values, regardless of how made, affect _all_ users of the camera -- i.e. they are specific to the camera and not to the stream. 

### Configuring other camera settings on the roboRIO with code 

### Configuring other camera settings on WPILibPi

On wpilibpi these settings can be controlled by `.json` files that can be generated, edited, and loaded through the web interface. Use the `Copy Source Config From Camera` button 
to populate the `Custom Properties JSON` box with the current configuration. ![copy-source-config-highlighted](https://github.com/chauser/CameraConfiguration/assets/834131/c0e37dba-931e-4815-b830-ad232c0192b4)


Copy that text into a `.json` file and save it on the Pi. Then use the Browse button to find that file to 
populate the `Load Source Config From JSON` box with the file name. Finally, click the `Save` button at the bottom the of page. The camera settings, apart from the VideoMode will now be loaded from the file each time the system starts up.

## Stream requests, configuring per-stream parameters, and stream defaults

The wpilib camera server has the ability to deliver streams at resolutions and frame rates different from the native camera settings. This approach, however, requires the roboRIO or Raspberry Pi to convert the resolution, which is a potentially expensive operation in terms of CPU percentage continuously required for the conversion. The camera server can also compress the images making up the stream which is controlled by the `compression` parameter in a stream request or the camera server's default compression setting (More about camera server defaults in a moment.). The value of the parameter ranges from -1 to 100 with -1 meaning "use the camera server's default compression', 0 meaning, somewhat counterintuitively, "maximum compression" which yields unusable image quality, and 100 meaning "full quality without compression at all".

### Requesting a stream and controlling stream configuration parameters in http requests
The basic way to start streaming is with the request
```
http://host:1181/?action=stream
```
You can use this in a browser. It will give you the camera stream with the camera server's default resolution, frame rate, and compression settings.
You can control other characteristics of the camera stream with the `resolution`, `fps`, and `compression` parameters in the URL. Any that are not supplied take the default value.  An example that sets all three is 
```
http://host:1181/?action=stream&resolution=320x240&fps=15&compression=15
```
Other camera parameters such as exposure, brightness, hue, can also be set as part of the request URL, for example
```
http://host:1181/?action=stream&brightness=50
```
but remember that changing these other parameters affect _all_ users of the camera.

### Setting stream defaults
Remember that the stream default settings are the frame rate and mode from the camera's VideoMode and no compression. The defaults for a particular stream can be set either in robot code for the roboRIO and on the Video Settings web page of the Raspberry Pi.
Here's how to do it in code for the first camera. (There's no explicit WPILib documentation for this technique which takes advantage of starting the server based on an already-created Camera object. This is the only way I could figure out that gave access to both the Camera object and MjpegServer object -- and you need to have both in order to fully configure the system.) 
```
UsbCamera cam = new UsbCamera("First Camera", 0); // the string is an arbitary name, 0 is the device number
cam.setVideoMode(VideoMode.PixelFormat.MJPEG, 640, 480, 30); // camera is 640x480 at 30fps
MjpegServer server = CameraServer.startAutomaticCapture(cam);
// now set the defaults for streams to 320x240 at 15fps and level 20 compression
server.setCompression(20);
server.setFPS(15);
server.setResolution(320, 240);
```

Here's what it looks like on the Raspberry Pi

![RPi-camerasettings-streamdefaults](https://github.com/chauser/CameraConfiguration/assets/834131/be193a1e-8812-4ea8-80a1-73df14e7fc69)

What's the difference between `Compression` and `Default Compression`? `Compression` is for sources using MJPEG format. `Default Compression` is for sources using other formats such as YUYV. Normally you should try to use an MJPEG source
  

## Cameras

Different cameras have different lists of supported video modes. As mentioned above it is almost impossible to learn the list of supported modes until you hook up the camera to a computer. A camera that has a mode that matches your needs will ease your experimentation and implementation tasks, but the wpilib camera server, at the cost of cpu load, is able to adapt most USB camera's capabilities to what you'll need.

In addition to having different mode capabilities, different cameras have different performance. Part of choosing a camera involves experimentation to determine which camera or cameras will perform well enough to meet your needs. It is also important to realize that the frame rates in camera modes are estimates: some cameras may deliver lower frame rates than the mode would suggest depending on many different things: for example, some cameras deliver lower frame rates if the scene being viewed is complex; others' frame rate may depend on settings of other camera parameters such as exposure type and level, blink compensation, etc.

Also relevant is the cord length of the camera -- for USB cameras probably not a problem but the Raspberry Pi camera, which has a lot of positive attributes, is location limited by the short length of the cable with which it is attached to the Pi. Furthermore, the Pi only supports a single Raspberry Pi camera, although it can also simultaneously support multiple USB cameras.

Latency: the best latency I've been able to achieve with USB cameras on a Robo Rios and Raspberry Pis 130-200ms. I suspect this is about the lowest can be achieved--the latency of the built-in camera on the dashboard laptop is about the same. Of course, lower is better, but 200ms is likely adequate, depending on the driving task involved.

Pixel format: the available dashboard viewers support only the MJPEG pixel format. Some USB cameras do not support MJPEG format. For example, I have one that only supports the YUYV format. The good news is that the wpilib camera server can convert other formats to MJPEG. The bad news is that this is a cpu-intensive operation that grows as the product of the stream resolution's width and height.

What about IP cameras? IP cameras plug directly into a network switch on the robot and operate independently of the robot code. For several years our favorite camera was the Axis M-1065L, a sophisticated security system camera with many possible configurations. These cameras can be completely independent of the robot code, having no effect on roboRIO or Pi performance since those cpus never see the data from the camera. Bandwidth, resolution, frame rate and compression for an IP camera are controlled using on-camera mechanisms that depend on the specific camera. Consult the camera documentation.

Some IP cameras these days do not support mjpeg streaming format and instead use rtsp (Real-time Streaming Protocol). None of the FRC standard dashboards support this protocol. To use these you'll need to use a different program on the driverstation computer -- VLC works (but imposes very high latency). Other people talk about using ffMJPEG, and various security camera systems and tools to view video from rtsp-only cameras.

## Dashboard-specific information 

### SmartDashboard

SmartDashboard offers two camera widgets: one establishes a connection to a camera managed by the wpilib camera server (and advertised via NetworkTables). The other allows connection to an arbitrary mjpeg ip camera. Unfortunately, SmartDashboard offers no way to configure stream parameters when using the camera server widget so you just get the defaults which are likely not suitable -- at the very least you're going to want to control the compression and fps settings.

The good news is that the general mjpeg camera widget can be used to connect to a wpilib camera server stream -- but it is not easy to find documentation about how to do it. Camera URLs described above work just fine in this widget : `http://<host>:1181/?action=stream&compression=50&fps=15&resolution=320x240`

Or you can configure the default stream parameters for the camera using the techniques described in the **Setting the defaults** section above.

### Shuffleboard

Shuffleboard has only one camera widget whose stream source must be a camera served by the wpilib camera server via NetworkTables. However, this widget allows you to configure the stream parameters. The settings for compression, fps, and resolution default to -1 meaning: accept the server's defaults which will be native camera resolution, and frame rate with no compression.

To use an mjpeg IP camera (one not connected to the wpilib cameraserver) you can tell the camera server about the IP camera by, in your robot code, including a call to CameraServer.addIPCamera(name, ipaddress). You'll need to configure the camera according to the manufacturer's instructions. And be aware that stream configuration parameters that the widget controls adjust will likely not be understood by the camera so the camera will need to be configured appropriately.

### Default Dashboard (provided by the NI Game Tools package)

Like Shuffleboard, this dashboard has only one camera widget that can only connected to camera streams listed by the wpilib camera server in NetworkTables. The widget offers a limited set of resolutions (all 4:3) and fps choices, although compression can be set to any value from 0 to 100. I do not know if there are any settings that you can make on the dashboard to use the stream defaults from the server.

## Switched cameras
In some situations you will find a need for two cameras and the ability to select which one is displayed on the dashboard. For example, if the robot has to be driven with precision both forward and back. Displaying both cameras at once may require more bandwidth than available. The WpiLib camera server supports attaching two (or more) cameras to the server and creating a third, virtual, switched camera whose display of one or the other is controlled by a NetworkTable entry. 

I'm going to leave out discussion of switched cameras -- it's an advanced topic and fairly easily figured out from the WpiLib docs.

## Recommendations and observations on specific cameras

The opinions here are based on one team's experience which covers only limited use cases and solutions. Our frustrations over the years stemmed in large part from experimentation that gave varying results when performed at different times and in different settings. It's now pretty clear that those variations came from not understanding the different settings and their interactions as discussed above. We encourage you to do experiments relevant for your own needs and hope that the information provided here will make that experimental journey more pleasant than ours has been.

Our experience is that limiting the camera bandwidth to the dashboard to 1-1.5 Mbps is desirable/necessary to avoid interfering with the driverstation's control role. Be aware that on-field bandwidth is limited to a maximum of 4Mbps. In some venues, wifi interference has caused lower limits to be imposed.

Getting under 1Mbps typically requires a resolution sent to the dashboard of about 320x240 (4:3) or 384x240 (~16:10), a frame rate of 15fps, and a compression setting of 15-50. Recommendation: for a camera used only as a driving camera, set the video mode to whatever the camera has that is close to 320x240 or 384x240, fps to 15. Set the camera server stream default for compression to something in the range 15 to 50 but do not set the resolution or frame rate -- thereby accepting what the camera is doing natively and thereby minimizing the cpu load.

If you want to use a camera both as a driving camera and a vision camera, you'll probably need the native resolution to be higher -- at least 640x480, perhaps much higher. It will require significant cpu resources to sufficiently compress and rescale the video to achieve the desired bandwidth target. This is probably a really bad idea on a roboRIO 1, not a great idea on a roboRIO 2, and would probably be fine with a raspberry pi (or the like) as your vision processor.

Avoid RTSP-only IP cameras. Although you can probably make them work it seems like a waste of time given that adequate USB cameras are available and relatively easy to use.

### Camera idiosyncracies:

 -   choose a camera that supports MJPEG pixel modes; others require work from the cpu that is avoided by MJPEG modes. All the cameras discussed below support MJPEG modes.
    
 -   the **Microsoft Lifecam HD-3000** (currently $25 at Amazon) has a few video modes close to what is recommended above. It also has a fairly wide field of view. For configured video modes above 15fps, regardless of resolution, it may only achieve 15fps (or even 10 fps) depending on the scene complexity and exposure settings
    
 -   [this ELP camera](https://www.amazon.com/gp/product/B071D3S43S/ref=ppx_yo_dt_b_asin_title_o04_s00?ie=UTF8&th=1)  has an ultra-wide-angle lens and a few video modes near the sweet spot. Video modes with frame rates above 15 may be curtailed by scene complexity and exposure settings. Also, weirdly, this camera seems to only do higher frame rates if the 60Hz power_line_frequency is selected -- though this setting seems to have little or no effect on image quality.
    
 -   the **Logitech C260** camera (obsolete; the C270 model is available at about the same price as the HD-3000; of course, differences in capabilities and performance are unknowable without having one in hand) has very limited choices for VideoMode near the sweet spot. It also has a somewhat narrower field of view than the HD-3000 and exhibits color/exposure anomalies sometimes, especially when using wpilibpi.
    
 - **The Raspberry Pi Camera Module** (experience with V2): this has a large sensor, good field of view and pretty much arbitrary choice of resolution and frame rate. The main disadvantages are: it doesn't work on the roboRIO, only on a Pi or Pi clone; each Raspberry Pi can only support one camera module; and the camera must be located within a few inches of the Raspberry Pi.

### Latency and latency experiments

Our experience is that latency (time from when the camera sees a change to when the change is displayed on the dashboard) in the neighborhood of 150ms is good enough, 100ms is outstanding but hard to achieve, and anything over 200ms is approaching annoying and results in reduction of usefulness of the driver camera.

I suggest experimenting with camera and stream settings until you achieve a 15fps frame rate with acceptable bandwidth and image quality. Just by moving in front of the camera while watching the video you'll get a sense of how much the latency affects your perception of what's happening. A better sense of the latency can be achieved by pointing the camera at the same screen where the video is displayed and doing something in another window. 

The _point-the-camera at the screen_ technique can be further refined to get actual measurements of latency. I strongly suggest doing these experiments initially with a wired connection (either ethernet or USB) to the robot or coprocessor. WiFi variability just adds confusion. Of course, after figuring out satisfactory settings, test it on WiFi, preferably with the robot radio.

-   Create a program that causes an area at least 1cm x 1cm on the driverstation screen to blink at 1 to 2 Hz rate -- it can be: a boolean widget on a dashboard driven by a robot program running on a real robot or in simulation; a separate program running on the computer; a video of something blinking that you play on the computer, etc. There's nothing critical about this -- it doesn't have to be strictly periodic, the rate isn't critical, etc. It's just important that it be easy to visually distinguish the two states.
    
-   Install a program that let's you record the screen of your computer. I used Open Broadcast Studio (OBS). 
-       
-   Run OBS and configure a video source that captures the screen; a 60Hz frame capture rate is good.
    
-   Hook up and configure the camera and stream viewer you want to test. Arrange the windows so the camera stream display is close to the blinking area of the screen. This is really easy if both are displayed on the same dashboard but even if using a separate blinking program it's not hard.
    
-   In OBS start recording. Point the camera at the screen so it picks up both the blinking and the camera viewer's display. Record for 5-10 blinks. Stop the recording and save the recorded video file (OBS does this automatically when you stop the recording).
    
-   Install a viewing program that let's you look at video frame by frame. I used `kdenlive`. This doesn't have to be on the same computer.
-   Load the video into the viewing program and step through video looking for places where the blinker changes state. The number of frames between that point and when the camera image shows the changed state, multiplied by 1/recordingframerate, is the latency. Average over a few state changes and you'll have measured the average latency for that configuration. It takes much longer to describe than it does to do once the programs are set up. 
    
-   For 30fps recording, 4-6 keypresses cover the range of 130ms-200ms. For 60fps, 8-12 keypresses cover that same range.

## Sources
Information for this article was gleaned from the [WPILib vision documentation](https://docs.wpilib.org/en/stable/docs/software/vision-processing/index.html), the `cscore` source code within WPILib, and quite a few different threads on [ChiefDelphi](https://www.chiefdelphi.com/). A good search term there is "camera". Also quite a lot of experimentation was done to figure out the way various settings interact. I would happy to receive suggestions for improvement, especially corrections for any mistakes but more generally improvements to make the document more useful. Please submit as pull requests.

