Leopard USB3.0 Camera Linux Camera Tool
=======
This is the sample code for Leopard USB3.0 camera linux camera tool. It is a simple interface for capturing, viewing, controlling video stream from Leopard's uvc compatible devices, with a special emphasis for the linux v4l2 driver. 

For Leopard USB3.0 Firmware Update Tool, please refer to [FW_UPDATE_README](/li_firmware_updates/README.md)

Table of Contents
-----------------  

- [Leopard USB3.0 Camera Linux Camera Tool](#leopard-usb30-camera-linux-camera-tool)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
    - [Hardware Prerequisites](#hardware-prerequisites)
    - [Dependencies](#dependencies)
      - [OpenCV Prerequisites](#opencv-prerequisites)
      - [OpenCV CUDA support](#opencv-cuda-support)
    - [Build Camera Tool](#build-camera-tool)
    - [Install leopard_cam](#install-leopard_cam)
    - [Uninstall leopard_cam](#uninstall-leopard_cam)
  - [Code Structure](#code-structure)
  - [Declarations](#declarations)
    - [Image Processing Functionalities](#image-processing-functionalities)
      - [Benchmark Results](#benchmark-results)
    - [Datatype Definition for Data Stream](#datatype-definition-for-data-stream)
    - [Auto Exposure](#auto-exposure)
    - [Exposure Time Calculation](#exposure-time-calculation)
  - [Run Camera Tool](#run-camera-tool)
    - [Test OS Platforms](#test-os-platforms)
    - [Test OpenCV Revisions](#test-opencv-revisions)
  - [Known Bugs](#known-bugs)
    - [Exposure & Gain Control Momentarily Split Screen](#exposure--gain-control-momentarily-split-screen)
    - [SerDes Camera Experiences First Frame Bad When Uses Trigger](#serdes-camera-experiences-first-frame-bad-when-uses-trigger)
  - [Reporting Bugs](#reporting-bugs)
  - [Contributing](#contributing)

---
## Installation
### Hardware Prerequisites
- __Camera__: any Leopard USB3.0/2.0 cameras
- __Host PC__: with USB3.0 port connectivity 
  Some boards like OV580 has more strict voltage requirements toward USB3.0 port. It is suggested to use a USB3.0 hub to regulate the voltage out from PC's USB3.0 port. 
  Do a `lsusb` and found your camera device after connection: VID of the device is usually `2a0b`.
- __USB3.0, USB2.0 connectivity__: 
  If you plug in your USB camera device slow enough to a USB3.0 port, your host PC might still recognize the USB3.0 device as USB2.0. 
  To be able to tell if your host PC has been recognized this USB3.0 camera truly as a USB3.0 device, do a `v4l2-ctl --device=/dev/videoX --all`, where `X` is the device number that being recognized by your PC. And check if its resolution, frame rate listed is the right one, (USB2.0 camera will have much lower bandwidth, so usually small resolution and frame rate).
  
### Dependencies
Make sure the libraries have installed. Run [configure.sh](configure.sh) in current directory for completing installing all the required dependencies.
```sh
$ chmod +X configure.sh
$ ./configure.sh
```

#### OpenCV Prerequisites
Make sure you have GTK 3 and OpenCV (3 at least) installed. The way you do to install this package varies by operational system.

Gtk3 and Gtk2 don't live together peaceful. If you try to run camera tool and got this error message:

    Gtk-ERROR **: GTK+ 2.x symbols detected. Using GTK+ 2.x and GTK+ 3 in the same proc

It is mostly you have OpenCV build with GTk2 support. The only way to fix it is rebuild OpenCV without GTk2:

    opencv_dir/release$cmake [other options] -D WITH_GTK_2_X=OFF ..

in order to disable Gtk2 from OpenCV.

```sh
# rebuild OpenCV, this might take a while
# this is just a sample of rebuilding OpenCV, modify them to fit your need
$ rm -rf build/
$ mkdir build && cd build/
$ cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local -D WITH_TBB=ON -D BUILD_NEW_PYTHON_EXAMPLES=ON -D BUILD_EXAMPLES=ON -D WITH_QT=ON -D WITH_OPENGL=ON -D WITH_GTK=ON -D WITH_GTK3=ON -D WITH_GTK_2_X=OFF ..
$ make -j6                     # do a $lscpu to see how many cores for CPU
$ sudo make install -j6

# link OpenCV
$ sudo sh -c 'echo "/usr/local/lib" >> /etc/ld.so.conf.d/opencv.conf'
$ sudo ldconfig
```
#### OpenCV CUDA support
To build __OpenCV CUDA__ support, please refer to [INSTALL_OPENCV_CUDA.md](INSTALL_OPENCV_CUDA.md) 

### Build Camera Tool
1. Make
   * Use __make__ when you don't need __OpenCV CUDA__ support
```sh
$ make
```

2. CMake
   * Use __cmake__ when you need __OpenCV CUDA__ support
```sh
$ mkdir build
$ cd build
########################################################################
$ cmake ..                      # USE_OPENCV_CUDA is turned off by default
$ cmake -DUSE_OPENCV_CUDA=ON .. # turn on USE_OPENCV_CUDA for OpenCV CUDA support
########################################################################
$ make

# Add to your project
add_subdirectory(linux_camera_tool)
include_directories(
	linux_camera_tool/src/
)

...

target_link_libraries(<Your Target>
	leopard_tools
)
```
### Install leopard_cam
```sh
$ sudo cp ./leopard_cam /usr/bin
```
### Uninstall leopard_cam
```sh
$ sudo rm -f /usr/bin/leopard_cam 
```
---

## Code Structure
```
$ linux_camera_tool .
├── Makefile                              # for make
├── configure.sh                          
├── CMakeLists.txt                        # for CMake
├── gitlog-to-changelog.md                # automate generating CHANGELOG.md,  version number in software
│
├── README.md                             # THIS FILE
├── INSTALL_OPENCV_CUDA.md
├── CHANGELOG.md
├── LICENSE.md
├── AUTHOR.md
│
├── config.json                           # configuration files for automating
├── BatchCmd.txt                          # camera controls
|
├── includes
│   ├── shortcuts.h                       # common used macro functions  
│   ├── hardware.h                        # for specific hardware feature support
│   ├── gitversion.h                      # auto-generated header file for revision
│   ├── utility.h                         # timer for benchmark
│   ├── cam_property.h
│   ├── extend_cam_ctrl.h
│   ├── core_io.h
|   ├── json_parser.h
│   ├── batch_cmd_parser.h
│   ├── ui_control.h
|   ├── isp_lib.h
│   ├── uvc_extension_unit_ctrl.h
│   └── v4l2_devices.h
│
├── src
│   ├── cam_property.cpp                  # gain, exposure, ptz ctrl etc
│   ├── extend_cam_ctrl.cpp               # camera stream, capture ctrl etc
|   ├── isp_lib.cpp                       # image signal processing using OpenCV
|   ├── core_io.cpp                       # string manipulation for parser
|   ├── json_parser.cpp                   # json parser
|   ├── batch_cmd_parser.cpp              # BatchCmd.txt parser
|   ├── ui_control.cpp                    # control GUI
│   ├── uvc_extension_unit_ctrl.cpp       # all uvc extension unit ctrl
│   └── v4l2_device.cpp                   # udev ctrl
│   
└── test
    └── main.c                              
```
--- 
## Declarations 
### Image Processing Functionalities
Most of the image processing jobs in Linux Camera Tool are done by using OpenCV. Since there are histogram matrix calculations and many other heavy image processing involved, enabling these features will slow down the streaming a lot. If you build Linux Camera Tool with __OpenCV CUDA support__, these feature speed won't be compromised a lot. Detailed benchmark result is listed below.

#### Benchmark Results
Benchmark test is performed on a _12M pixel RAW12 bayer_ camera (IMX477).

| Functions| Non-CUDA OpenCV Support | Latency | CUDA OpenCV Support | Latency |
|----------|-------------------------|---------|---------------------|---------|
| `Auto White Balance`   | Yes                                | 55ms   | Yes                            | 3700us  |
| `CLAHE`                | Yes                                | 40ms    | Yes                            | 7700us  |
| `Gamma Correction`     | Yes                                | 2.7ms   | Yes                            | 1100us  |
| `Sharpness`            | Yes                                | 200ms   | Yes                            | 20ms    |
| `Show Edge`            | Yes                                | 20ms    | Yes                            | 16ms    |
| `Color Correction Matrix` | Yes                             | 18ms    | No                             | N/A     |
| `Perform Shift`        | No                                 | 13ms    | No                             | N/A     |
| `Seperate Display`     | Yes                                | 9000us  | No                             | N/A     |
| `Brightness`           | Yes                                | 7ms   | Yes                            | 400us   |
| `Contrast`             | Yes                                | 7ms   | Yes                            | 400us   |
| `Auto Brightness & Contrast` | Yes                                | 14ms    | Yes                            | 1180us  |

### Datatype Definition for Data Stream
<img src="pic/datatypeExp.jpg" width="7000">

### Auto Exposure
Auto exposure is usually implemented on __sensor/ISP__, which when enabled, won't further slow down the streaming. You need to check with camera driver for auto exposure support. If some sensor don't have AE support built-in, the check button in the first tab of Camera Control (`Enable auto exposure`)won't work. 

There is a software generated auto exposure support in the second tab of Camera Control (`Software AE`), you can enable that if you really need AE support for the camera. Since adjusting gain and exposure under linux will experience split screen issue, same thing will happen to this software AE.


### Exposure Time Calculation
Since _Linux V4L2 API_ doesn't provide a easier way to tell you what exact exposure time in _ms_ like what _Windows_ does, here is the explanation for helping you figuring out your camera's current exposure time in _ms_:

`Exposure time = exposure_time_line_number / (frame_rate * total_line_number)`

, where 
1. __exposure_time_line_number__ is the value that display is linux camera tool exposure control
2. you can get __frame_rate__ from the log message `Get Current Frame Rate =`, don't refer to `Current Fps` that displays in the `Camera View`, that's the frame rate your PC can process, not the physically USB received frame rate from camera.
3. __total_line_number__ is usually around the height of the camera's current resolution, usually slightly larger than that since VD needs blanking. To get the exact __total_line_number__, you might want to read __VTS__ register value for your sensor, which might be a hassle for you so I will only tell you this roughly calculation method...
4. Noticing for exposure time change in Linux Camera Tool Control GUI, the exposure time range will usually be over the camera's resolution height. If you have the exposure value set to be over camera's resolution height, it might affect your frame rate. Since when camera exposure time line number is greater than the total line number, that will slow down your camera's frame rate. This varies from different sensors. Some sensors don't allow that happen, so the exposure time will stick the same when it is over total line number.
   
---
## Run Camera Tool
Run your Linux Camera Tool from terminal after installation 
```leopard_cam```
### Examples
  
#### Test on RAW sensor 12 Megapixel IMX477
__Original streaming after debayer for IMX477:__ 
> image is dark and blue
> <img src="pic/imx477Orig.jpg" width="1000">

__Modified streaming for IMX477:__
> enabled software AWB, gamma correction, black level correction with _non OpenCV CUDA support_
> <img src="pic/imx477.jpg" width="1000">

> enable software awb, auto brightness & contrast, sharpness control with _OpenCV CUDA support_
> <img src="pic/imx477Cuda.jpg" width="1000">

#### Test on RAW sensor 5 Megapixel OS05A20
__Modified streaming for OS05A20 full resolution__
> change bayer pattern to GRBG
> enable software AWB, gamma correction
> increase exposure, gain
> <img src="pic/os05a20.jpg" width="1000">


__Modified OS05A20 resolution to an available one__
> binning mode
> capture raw and bmp, save them in the camera tool directory
<img src="pic/changeResOS05A20.jpg" width="1000">

#### Test on OV4689-OV580 stereo YUV422 2 Megapixel Cam

__Original streaming:__
> Default YUV output
> <img src="pic/stereo.jpg" width="1000">

__CLAHE:__
> Run OpenCV CLAHE(Contrast Limited Adaptive Histogram Equalization)
><img src="pic/stereoClahe.jpg" width="1000">

__Edge detection(Canny):__
> Run OpenCV Canny edge detection
><img src="pic/edge.jpg" width="1000">

---

#### Exit Linux Camera Tool
1. Press __ESC, q, Q__ in your keyboard to exit the whole program
2. Use __EXIT__ button on the grid1 control panel to exit the whole program
3. Use __Help -> Exit__ in the menu bar to exit the whole program
4. User __Ctrl + C__ to exit the control panel after exit from camera streaming GUI
5. Unexpected quit results in defunct process, use 
   ```sh
   $ ps
   ``` 
   to see if the leopard_cam still alive, if it is, use 
   ```sh
   $ killall -9 leopard_cam
   ``` 
   to kill the zombie process

#### Kill Linux Camera Tool Zombie Process
If you forget to exit both windows and tried to run the camera tool again, it will give you the error of 
`VIDIOC_DQBUF: Invalid argument`
Please check your available processes and kill leopard_cam, then restart the camera tool application
```sh
$ ps
$ killall -9 leopard_cam
```

### Test OS Platforms
- `4.18.0-17-generic #18~18.04.1-Ubuntu SMP` 
- `4.15.0-32-generic #35~16.04.1-Ubuntu`
- `4.15.0-20-generic #21~Ubuntu 18.04.1 LTS`

### Test OpenCV Revisions
Due to OpenCV CUDA support hasn't been updated to 4.0.X, it is suggested to install `3.4.3` for OpenCV CUDA support
- `3.4.2`
- `3.4.3` 
- `4.0.1`
- `4.1.1`
  
---
## Known Bugs

### Exposure & Gain Control Momentarily Split Screen
When changing exposure and gain under linux, camera tool will experience split screen issue at the moment change is happened. This issue happens for the USB3 camera that use _manual DMA_ in the FX3 driver. For the camera that utilizes _auto DMA_, the image will be ok when exposure and gain change happens. 

For updating driver from manual DMA to auto DMA, you need to ensure:
1. PTS embedded information is not needed in each frame
2. The camera board has a Crosslink FPGA on the tester board that is able to add uvc header to quality for auto DMA
3. FX3 driver also need to be updated.

### SerDes Camera Experiences First Frame Bad When Uses Trigger
When use triggering mode instead of master free running mode for USB3 SerDes camera, the very first frame received will be bad and should be tossed away. It is recommended to use an external function generator or a dedicated triggering signal for triggering the cameras for the purpose of syncing different SerDes cameras. 

The included "shot 1 trigger" function is only a demonstration on generating one pulse and let camera output one frame after "shot 1 trigger" is clicked once. User should not fully rely on this software generated trigger but use a hardware trigger for sync the camera streaming.

## Reporting Bugs
Report bugs by creating a new issue on this Github page: 
https://github.com/danyu9394/linux_camera_tool/issues
If it is clearly a firmware issue: likely some features hasn't been implemented in the driver, please contact Leopard support: [support@leopardimaging.com](mailto:support@leopardimaging.com)

## Contributing
Fork the project and send pull requests, or send patches by creating a new issue on this Github page:
https://github.com/danyu9394/linux_camera_tool/issues




