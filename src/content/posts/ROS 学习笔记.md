---
title: ROS 学习笔记
published: 2025-02-07
updated: 2026-05-26
description: 'ROS 基础概念、话题通信、服务通信、定时器与参数使用笔记'
image: ''
tags: [ROS, Tutorial, C++]
category: 'ROS'
draft: false
---

`ros`玩起来有点像在玩嵌入式的实时操作系统，例如 `FreeRTOS`。
在 `ros`中我不喜欢在`while`里面写东西，更喜欢定义多个定时器，实现多线程方法，这样才优雅。:-)
这就像是`FreeRTOS`，单片机单线程工作太多局限，使用实时操作系统更好。
同时我们要故意的去使用C++中的面对对象编程，不要还使用着c语言中的编程思想，有点老套了。
赵虚左老师的ros1教程中没有使用面对对象，但我们自己要基于他的教程上使用面对对象编程方法。
# 教程
[赵虚左老师的ros教程](http://www.autolabor.com.cn/book/ROSTutorials/index.html)
[鱼香ros一键安装](https://fishros.org.cn/forum/topic/20/%E5%B0%8F%E9%B1%BC%E7%9A%84%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85%E7%B3%BB%E5%88%97)
[官网教程](http://wiki.ros.org/)
# 踩坑
1. 在anaconda环境下运行ros会有报错，见[csdn](https://blog.csdn.net/weixin_40324045/article/details/111478774)
```bash
pip install rospkg rospy catkin_tools scipy

conda install -c conda-forge catkin_make

conda install -c conda-forge empy
```

2. ros需要python=3.8环境 被装错了，关于建python环境见[[Conda]]
3. 关于在anaconda环境下使用ros见[csdn](https://blog.csdn.net/Hide_on_Stream/article/details/127363246)
4. fuck!!! ROS的功能报如果是python语言，最好别用conda的虚拟环境，会卡死

# packages
[[ego-planner]]
[[FAST_LIO]]
[[Fast_Planner]]
[[MAVROS]]
[[Kalibr]]
[[realsense-ros]]
[[VINS-Mono]]
[[VINS-Fusion]]
# CLI
- `catkin_make -DCATKIN_WHITELIST_PACKAGES="name"` 选择编译特定功能包
- `rosrun`
- `roslaunch`
- `rosmsg`
- `rossrv`
# 基础
ros老容易忘，因此把最基础的东西做一下笔记，其他的主要是围绕着这些基础的东西。
## 话题
```c++
    ros::Publisher PosTarget_pub = n.advertise<mavros_msgs::PositionTarget>("/" + robot + "_" + std::to_string(ID)+ "/mavros/setpoint_raw/local", 10);//发布方
    ros::Subscriber sub = n.subscribe<mavros_msgs::State>("/" + robot + "_" + std::to_string(ID)+ "/mavros/state", 10, &my_mavros::state_callback, this);//订阅方
```
`my_mavros::state_callback`为回调函数,下面代码是回调函数定义，==注意参数部分==
```c++
void my_mavros::state_callback(const mavros_msgs::State::ConstPtr& msg){
    state = *msg;
    return;
}
```
使用 `PosTarget_pub.punlish(msg)`即可发布话题信息
## 服务
```c++
    ros::ServiceClient arming_client = n.serviceClient<mavros_msgs::CommandBool>("/" + robot + "_" + std::to_string(ID)+ "/mavros/cmd/arming");

    ros::ServiceClient set_mode_client = n.serviceClient<mavros_msgs::SetMode>("/" + robot + "_" + std::to_string(ID)+ "/mavros/set_mode");

    ros::ServiceClient Set_Param_client = n.serviceClient<mavros_msgs::ParamSet>("/" + robot + "_" + std::to_string(ID)+ "/mavros/param/set");

    ros::ServiceClient Get_Param_client = n.serviceClient<mavros_msgs::ParamGet>("/" + robot + "_" + std::to_string(ID)+ "/mavros/param/get");
```
上面代码均为服务通信客户端的定义，服务端就不介绍了。客户端与服务器通信只需要执行 `client.call(msg)`即可通信，而返回来的信息是 `msg.response`下面的数据。
## 定时器
```c++
    exec_timer_ = nh.createTimer(ros::Duration(0.01), &EGOReplanFSM::execFSMCallback, this);

    safety_timer_ = nh.createTimer(ros::Duration(0.05), &EGOReplanFSM::checkCollisionCallback, this);
```
与话题通信的订阅放类似，也需要定义一个回调函数，`ros::Duration(0.01)`是0.01秒一次，即频率为100hz。
```c++
void my_mavros::DroneControl_callback(const ros::TimerEvent &e){
}
```
上面代码是定时器的回调函数定义，==要注意参数==。
## 参数
```xml
<launch>
    <arg name="robot" value="iris"/>
    <arg name="ID" value="0"/>
    <arg name="Mode" value="OFFBOARD" />
    <node pkg="my_mavros" name="my_mavros_node" type="my_mavros_node" output="screen">
        <param name="robot" value="$(arg robot)" type="string"/>
        <param name="ID" value="$(arg ID)" type="int"/>"
        <param name="Mode" value="$(arg Mode)" type="string"/>
    </node>
</launch>
```
 上面的代码使用的是 `arg`传参，主要还是
```xml
        <param name="robot" value="$(arg robot)" type="string"/>
        <param name="ID" value="$(arg ID)" type="int"/>"
        <param name="Mode" value="$(arg Mode)" type="string"/>
```
参数 `name`是自己定义的参数， `type`是数据类型，以上是`launch`文件的传参语法

```c++
    std::string robot, Mode;
    int ID;
    n.param<std::string>("robot", robot, "iris");
    n.param<std::string>("Mode", Mode, "OFFBOARD");
    n.param("ID", ID, 0);
```
以上代码是c++文件中读取参数的语法，==要注意接受参数的变量需要定义==，同时接受字符串类型时需要加上 `<std::string>`，`n.param()`的第三个参数是默认值，当launch文件中没有给出该参数的值，则为默认值。
