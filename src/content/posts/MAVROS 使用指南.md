---
title: MAVROS 使用指南
published: 2025-05-18
updated: 2026-05-26
description: 'MAVROS 安装、配置与无人机控制指南，包括 OFFBOARD 模式设置与位置/速度控制'
image: ''
tags: [ROS, MAVROS, PX4, MAVLink, UAV]
category: 'ROS'
draft: false
---

[[PX4]]
[项目地址](https://github.com/mavlink/mavros)
[[ROS]]
`MAVROS`说白了就是 `mavlink`协议在ros环境下的实现，`MAVROS`中集成了`mavlink`。 `mavros`官方没有给出详细的使用方法，目前可以通过 `readme`文件以及阅读源码的方式进行学习。

使用下面命令即可安装`mavros`
```sh
sudo apt install ros-noetic-mavros
```
当然他还有许多拓展功能包，可以选择性安装。

关于使用只需要输入
```bash
roslaunch mavros px4.launch
```
即可

```yaml
<launch>
    <!-- vim: set ft=xml noet : -->
    <!-- example launch script for PX4 based FCU's -->

    <arg name="fcu_url" default="/dev/ttyACM0:57600" />
    <arg name="gcs_url" default="" />
    <arg name="tgt_system" default="1" />
    <arg name="tgt_component" default="1" />
    <arg name="log_output" default="screen" />
    <arg name="fcu_protocol" default="v2.0" />
    <arg name="respawn_mavros" default="false" />

    <include file="$(find mavros)/launch/node.launch">
        <arg name="pluginlists_yaml" value="$(find mavros)/launch/px4_pluginlists.yaml" />
        <arg name="config_yaml" value="$(find mavros)/launch/px4_config.yaml" />

        <arg name="fcu_url" value="$(arg fcu_url)" />
        <arg name="gcs_url" value="$(arg gcs_url)" />
        <arg name="tgt_system" value="$(arg tgt_system)" />
        <arg name="tgt_component" value="$(arg tgt_component)" />
        <arg name="log_output" value="$(arg log_output)" />
        <arg name="fcu_protocol" value="$(arg fcu_protocol)" />
        <arg name="respawn_mavros" default="$(arg respawn_mavros)" />
    </include>
</launch>
```
这是 `px4.launch`文件内容，可见调用了 `node.launch`文件，而重要的是 `fcu_url`参数，通信只需要修改该参数，如果是使用机载电脑，则通信方式应该是使用带有 `ch340`芯片的usb转ttl模块进行通信，而参数就需要改成usb号。

而如果在仿真情况或者无线通信情况下，需要将 `fcu_url`参数改成ip地址，例如 `udp://:14540@192.168.1.36:14557`

 使用mavros控制无人机时，需要将无人机设置为 `OFFBOARD`模式
```c++
    ros::ServiceClient set_mode_client = n.serviceClient<mavros_msgs::SetMode>("/" + robot + "_" + std::to_string(ID)+ "/mavros/set_mode");
    for(int i=0; i<100 && ros::ok(); i++){
        PosTarget_pub.publish(PosTarget);
        r.sleep();
    }
     if(set_mode_client.call(offb_set_mode) && offb_set_mode.response.mode_sent){
        ROS_INFO("set OFFBOARD mode");
    }
```

但在设置模式之前需要先对无人机进行控制，换句话说，先对控制话题输出一些信息，如果没有进行该步骤，则无法设置 `OFFBOARD`模式，因此如下代码进行的便是该操作。
```c++
    for(int i=0; i<100 && ros::ok(); i++){
        PosTarget_pub.publish(PosTarget);
        r.sleep();
    }
```
在官方教程中也提到了这个步骤的重要性。见下图。
![Pasted image 20250506140357.png](/images/Pasted-image-20250506140357.png)
如果还是无法设置`OFFBOARD`模式，可能是 `COM_RCL_EXCEPT`参数的问题，需要将该参数设置成4，见以下代码。
```c++
    mavros_msgs::ParamSet Param_S;
    mavros_msgs::ParamValue Param_S_Value;
    Param_S_Value.integer=4;
    Param_S_Value.real=0.0;
    Param_S.request.param_id = std::string("COM_RCL_EXCEPT");
    Param_S.request.value = Param_S_Value;

    Param_G.request.param_id=std::string("COM_RCL_EXCEPT");
    if(Set_Param_client.call(Param_S) && Param_S.response.success){
        ROS_INFO("Set Param COM_RCL_EXCEPT 4");
    }else{
        ROS_WARN("worning: can't set Param COM_RCL_EXCEPT 4");
    }
```
设置之后应该就没有问题了。
之后需要对飞机解锁，即==设置为arm模式==，见如下代码。
```c++
    mavros_msgs::CommandBool arm_cmd;
    ros::ServiceClient arming_client = n.serviceClient<mavros_msgs::CommandBool>("/" + robot + "_" + std::to_string(ID)+ "/mavros/cmd/arming");
    arm_cmd.request.value = true;
    if(arming_client.call(arm_cmd) && arm_cmd.response.success){
        ROS_INFO("state: arm");
    }
```

控制无人机时我们需要往 `/mavros/setpoint_raw/local`话题发送数据，这个话题的数据类型是 `mavros_msgs::PositionTarget`类型，以下是数据的结构。
```c++
uint8 FRAME_LOCAL_NED=1
uint8 FRAME_LOCAL_OFFSET_NED=7
uint8 FRAME_BODY_NED=8
uint8 FRAME_BODY_OFFSET_NED=9
uint16 IGNORE_PX=1
uint16 IGNORE_PY=2
uint16 IGNORE_PZ=4
uint16 IGNORE_VX=8
uint16 IGNORE_VY=16
uint16 IGNORE_VZ=32
uint16 IGNORE_AFX=64
uint16 IGNORE_AFY=128
uint16 IGNORE_AFZ=256
uint16 FORCE=512
uint16 IGNORE_YAW=1024
uint16 IGNORE_YAW_RATE=2048
std_msgs/Header header
  uint32 seq
  time stamp
  string frame_id
uint8 coordinate_frame
uint16 type_mask
geometry_msgs/Point position
  float64 x
  float64 y
  float64 z
geometry_msgs/Vector3 velocity
  float64 x
  float64 y
  float64 z
geometry_msgs/Vector3 acceleration_or_force
  float64 x
  float64 y
  float64 z
float32 yaw
float32 yaw_rate
```
可见有三种控制方式，分别是 `position`、`velocity`、`acceleration_or_force`，代表的是位置控制，速度控制，加速度控制。我们选择哪一中控制方式，就填充信息在这个控制方式下的成员，同时需要设置 `type_mask`。以下是例子。

速度控制
```c++
        PosTarget.velocity.x = 0;
        PosTarget.velocity.y = 0;
        PosTarget.velocity.z = 0;
        PosTarget.yaw_rate = 0;
        PosTarget.type_mask = PosTarget.IGNORE_PX + PosTarget.IGNORE_PY + PosTarget.IGNORE_PZ  + PosTarget.IGNORE_AFX + PosTarget.IGNORE_AFY + PosTarget.IGNORE_AFZ + PosTarget.IGNORE_YAW;
        PosTarget.coordinate_frame = 1;
```

位置控制
```c++
        PosTarget.position.x=0;
        PosTarget.position.y=0;
        PosTarget.position.z=0;
        PosTarget.yaw = 0;
        PosTarget.type_mask = PosTarget.IGNORE_VX + PosTarget.IGNORE_VY + PosTarget.IGNORE_VZ + PosTarget.IGNORE_AFX + PosTarget.IGNORE_AFY + PosTarget.IGNORE_AFZ + PosTarget.IGNORE_YAW_RATE;
        PosTarget.coordinate_frame = 1;
```
可见，`type_mask`只要把不需要的信息相加在赋值给自己，`IGNORE_VX`字面意思也就是忽视速度，举一反三，便可知道其他的意思。

```bash
rosrun mavros mavcmd long 511 31 10000 0 0 0 0 0

rostopic hz /mavros/imu/data
```
