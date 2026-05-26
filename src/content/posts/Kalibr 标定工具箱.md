---
title: Kalibr 标定工具箱
published: 2025-05-11
updated: 2026-05-26
description: 'Kalibr 相机与 IMU 联合标定工具箱使用指南，包括依赖安装、编译与校准命令'
image: ''
tags: [ROS, Kalibr, Camera Calibration, IMU]
category: 'ROS'
draft: false
---

[github项目地址](https://github.com/ethz-asl/kalibr)
`Kalibr is a toolbox that solves the following calibration problems`
kalibr是一个用来解决校准问题的工具箱。

# 依赖安装
```bash
sudo apt-get install -y \
    git wget autoconf automake nano \
    libeigen3-dev libboost-all-dev libsuitesparse-dev \
    doxygen libopencv-dev \
    libpoco-dev libtbb-dev libblas-dev liblapack-dev libv4l-dev
# Ubuntu 16.04
sudo apt-get install -y python2.7-dev python-pip python-scipy \
    python-matplotlib ipython python-wxgtk3.0 python-tk python-igraph python-pyx
# Ubuntu 18.04
sudo apt-get install -y python3-dev python-pip python-scipy \
    python-matplotlib ipython python-wxgtk4.0 python-tk python-igraph python-pyx
# Ubuntu 20.04
sudo apt-get install -y python3-dev python3-pip python3-scipy \
    python3-matplotlib ipython3 python3-wxgtk4.0 python3-tk python3-igraph python3-pyx
```

# 编译
```sh
mkdir -p ~/kalibr_workspace/src
cd ~/kalibr_workspace
export ROS1_DISTRO=noetic # kinetic=16.04, melodic=18.04, noetic=20.04
source /opt/ros/$ROS1_DISTRO/setup.bash
catkin init
catkin config --extend /opt/ros/$ROS1_DISTRO
catkin config --merge-devel # Necessary for catkin_tools >= 0.4.
catkin config --cmake-args -DCMAKE_BUILD_TYPE=Release
```
```sh
cd ~/kalibr_workspace/src
git clone https://github.com/ethz-asl/kalibr.git
```
```sh
cd ~/kalibr_workspace/
catkin build -DCMAKE_BUILD_TYPE=Release -j4
```

# yaml文件配置，april_6x6.yaml
```yaml
target_type: 'aprilgrid' #gridtype
tagCols: 6               #number of apriltags
tagRows: 6               #number of apriltags
tagSize: 0.088           #size of apriltag, edge to edge [m]
tagSpacing: 0.3          #ratio of space between tags to tagSize
                         #example: tagSize=2m, spacing=0.5m --> tagSpacing=0.25[-]
```
tagSpacing是比例，固定为0.3
我们需要修改的是tagSize参数，0.088米对应的是8.8cm即a0纸大小，使用a4纸的话需要更改成0.022。
tagSpacing如下图
![Pasted image 20250510150603.png](/images/Pasted-image-20250510150603.png)
# bag format
```sh
kalibr_bagcreater --folder dataset-dir/. --output-bag awsome.bag #使用图片生成bag文件

kalibr_bagextractor --image-topics /cam0/image_raw /cam1/image_raw --imu-topics /imu0 --output-folder dataset-dir --bag awsome.bag #使用bag文件生成图片文件
```

# Calibration validator  校准验证器

验证工具在 ROS 图像流中提取校准目标，并显示与提取角点的重投影叠加的图像。此外，还计算并显示单目和跨摄像头重投影误差的统计信息。
该工具必须提供相机系统校准文件和校准目标的配置。多相机校准器的输出 YAML 可用作相机系统配置。
```sh
rosrun kalibr kalibr_camera_validator --cam camchain.yaml --target target.yaml
```

# Camera focus  相机对焦
此 [ROS](https://github.com/ethz-asl/kalibr/wiki/www.ros.org) 节点可帮助您以可复现的方式设置相机的焦距。该工具订阅相机的图像主题，并显示实时图像以及焦距测量值。将相机对准包含高频分量的场景（例如西门子星、棋盘格），并调整焦距，同时尽量最小化计算出的焦距测量值。
```sh
rosrun kalibr kalibr_camera_focus --topic [IMAGE_TOPIC]  
rosrun kalibr kalibr_camera_focus --topic [图像主题]
```


# 相机校准
可能需要先输入 `export KALIBR_MANUAL_FOCAL_LENGTH_INIT=1`即手动输入焦距

```sh
rosrun kalibr kalibr_calibrate_cameras \
    --target /data/april_6x6.yaml \  # 标定板配置文件
    --bag /data/cam_calib.bag \      # 采集的数据包
    --models pinhole-equi \          # 相机模型(pinhole-equi/radtan等)
    --topics /cam0/image_raw \       # 图像话题
    --show-extraction                # 显示特征点提取
```

# 相机-imu联合校准
```sh
rosrun kalibr kalibr_calibrate_imu_camera \
    --target april_6x6.yaml \
    --imu imu_adis16448.yaml \
    --imu-models calibrated \
    --cam cam_april-camchain.yaml \
    --bag imu_april.bag
```
`imu_adis16448.yaml`为imu内参文件
`cam_april-camchain.yaml`为相机内参文件
`imu_april.bag`为用于校准的bag包
