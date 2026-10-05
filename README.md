# Optris ROS 2 Driver

ROS 2 driver for Optris thermal cameras based on the Optris **IR Direct SDK (`libirimager`)**.

This version has been tested with:

* **Ubuntu 22.04 LTS**
* **ROS 2 Humble**
* **Optris PI640**
* USB connection
* 640 × 480 thermal image
* 32 Hz acquisition rate

> This repository is based on the original `Optris/optris_drivers2` package and includes the configuration and setup required to run it on ROS 2 Humble.

---

## Requirements

Install the required ROS 2 and Linux packages:

```bash
sudo apt update

sudo apt install -y \
    ros-humble-image-transport \
    ros-humble-image-transport-plugins \
    ros-humble-camera-info-manager \
    ros-humble-image-tools \
    v4l-utils \
    usbutils \
    libudev-dev \
    libusb-1.0-0-dev
```

The **Optris IR Direct SDK (`libirimager`)** must also be installed.

The SDK can be downloaded from:

https://github.com/Optris/irdirectsdk_downloads

Install the Debian package corresponding to Ubuntu 22.04 and your architecture.

Then install the downloaded package:

```bash
sudo apt install ./libirimager*.deb
```

Connect camera and verify the installation:

```bash
which ir_find_serial
which ir_generate_configuration
which ir_download_calibration
```

---

# USB and Video Permissions

The Optris camera is accessed through the Linux Video4Linux2 (`V4L2`) interface.

The user running the ROS 2 node must have permission to access `/dev/video*`.

Add the current user to the `video` group:

```bash
sudo usermod -aG video $USER
```

After running this command, **log out and log back in**, or reboot the computer.

Verify that the user belongs to the `video` group:

```bash
groups
```

The output should contain:

```text
video
```

Check the available video devices:

```bash
v4l2-ctl --list-devices
```

---

# UVC Video Configuration

For high-rate streaming with some Optris cameras, the Linux `uvcvideo` driver may require the `nodrop=1` option.

Check the current value:

```bash
cat /sys/module/uvcvideo/parameters/nodrop
```

To enable it temporarily:

```bash
sudo sh -c 'echo -n 1 > /sys/module/uvcvideo/parameters/nodrop'
```

Alternatively, reload the module with:

```bash
sudo rmmod uvcvideo
sudo modprobe uvcvideo nodrop=1
```

> If another USB webcam is currently using `uvcvideo`, close any camera application before reloading the module.

To make this configuration persistent:

```bash
echo "options uvcvideo nodrop=1" | \
sudo tee /etc/modprobe.d/uvcvideo.conf
```

Reboot the computer after creating the persistent configuration.

Verify:

```bash
cat /sys/module/uvcvideo/parameters/nodrop
```

---

# Camera Calibration

Find the camera serial number:

```bash
ir_find_serial
```

Download the corresponding Optris calibration files:

```bash
sudo ir_download_calibration
```

The calibration files are normally stored in:

```text
/usr/share/libirimager/cali
```

---

# Camera Configuration

An example configuration for the **Optris PI640** is provided in:

```text
config/pi640.example.xml
```

Copy the example before editing it:

```bash
cp config/pi640.example.xml config/pi640.xml
```

Edit:

```xml
<serial>YOUR_SERIAL_NUMBER</serial>
```

using the serial number returned by:

```bash
ir_find_serial
```

Alternatively, a configuration can be generated directly by the Optris SDK:

```bash
ir_generate_configuration > config/pi640.xml
```

---

# Build with ROS 2 Humble

Create or use a ROS 2 workspace:

```bash
mkdir -p ~/optris_ws/src
cd ~/optris_ws/src
```

Clone the repository:

```bash
git clone <REPOSITORY_URL>
```

Install ROS dependencies:

```bash
cd ~/optris_ws

source /opt/ros/humble/setup.bash

rosdep install \
    --from-paths src \
    --ignore-src \
    -r -y
```

Compile:

```bash
colcon build \
    --packages-select optris_drivers2 \
    --symlink-install
```

Source the workspace:

```bash
source ~/optris_ws/install/setup.bash
```

---

# Run the Optris PI640

Make sure that:

* No other application is using the camera.

Run the camera node:

```bash
ros2 run optris_drivers2 optris_imager_node \
    /absolute/path/to/pi640.xml
```

For example:

```bash
ros2 run optris_drivers2 optris_imager_node \
    ~/optris_ws/src/optris_drivers2/config/pi640.xml
```

---

# Verify the ROS 2 Stream

In another terminal:

```bash
source /opt/ros/humble/setup.bash
source ~/optris_ws/install/setup.bash
```

List the topics:

```bash
ros2 topic list
```

Check the thermal image frequency:

```bash
ros2 topic hz /thermal_image
```

The main image topic is:

```text
/thermal_image
```

---

# False-Color Thermal Image

Start the color conversion node:

```bash
ros2 run optris_drivers2 optris_colorconvert_node
```

The converted thermal image is published on:

```text
/thermal_image_view
```

It can be visualized using:

```bash
ros2 run image_tools showimage -t /thermal_image_view
```

---

# Troubleshooting

If the camera is detected but no thermal images are published, check the system layer by layer.

### 1. Check USB

```bash
lsusb
```

### 2. Check the V4L2 device

```bash
v4l2-ctl --list-devices
```

### 3. Check user permissions

```bash
groups
ls -l /dev/video*
```

The user should belong to the `video` group.

---

## Credits

This repository is based on the original Optris ROS 2 driver:

https://github.com/Optris/optris_drivers2

which was originally derived from:

https://github.com/evocortex/optris_drivers2

The original license and copyright notices are preserved.
