---
title: ego-planner 解析
published: 2025-05-03
updated: 2026-05-26
description: 'ego-planner 路径规划算法源码解析，包括 FSM 状态机、GridMap 地图管理与 trajectory 发布'
image: ''
tags: [ROS, ego-planner, Path Planning, UAV]
category: 'ROS'
draft: false
---

[项目地址](https://github.com/ZJU-FAST-Lab/ego-planner)
# 解析
![ego-planner算法思维导图.png](/images/ego-planner算法思维导图.png)
## advanced_param.xml
首先解释参数xml文件，主要参数如下图
![Pasted image 20250421134655.png](/images/Pasted-image-20250421134655.png)
这四个参数主要配置话题，`/odom_world`以及`/grid_map/odomo`都为里程计话题，且相同
`/grid_map/cloud`为点云话题
`/grid_map/depth`为摄像头的图像话题

下图为pose类型以及地图的帧头frame
![Pasted image 20250421135214.png](/images/Pasted-image-20250421135214.png)
用于改变下图的判断条件![Pasted image 20250421135333.png](/images/Pasted-image-20250421135333.png)
由于使用雷达话题，不使用 `/grid_map_pose`话题，因此只需要将 `pose_type`改为 `ODOMTRY`,具体数字如下图
![Pasted image 20250421135543.png](/images/Pasted-image-20250421135543.png)

## 效果
![Pasted image 20250423163533.png](/images/Pasted-image-20250423163533.png)
给出目标点后会出现上图效果。
绿色线为`global_path`对应话题为`/ego_planner_node/global_list`
蓝色线为`InitTraj`对应话题为`/ego_planner_node/init_list`
红色线为`optimal_traj`对应话题为`/ego_planner_node/optimal_list`

## EGOReplanFSM类
见下图，首先创建PlanningVisualization类变量，以及EGOPlannerManager类变量，具体作用见下文
![Pasted image 20250424121534.png](/images/Pasted-image-20250424121534.png)

在创建两个定时器，`exec_timer`与`safety_timer`，`exec_timer`主要用于状态的改变
```c
    exec_timer_=nh.createTimer(ros::Duration(0.01),&EGOReplanFSM::execFSMCallback, this); //状态定时器
safety_timer_=nh.createTimer(ros::Duration(0.05),&EGOReplanFSM::checkCollisionCallback, this); //检测碰撞定时器
```

```c++
  void EGOReplanFSM::execFSMCallback(const ros::TimerEvent &e)
  {
    static int fsm_num = 0;
    fsm_num++;
    if (fsm_num == 100)
    {
      printFSMExecState();
      if (!have_odom_)
        cout << "no odom." << endl;
      if (!trigger_)
        cout << "wait for goal." << endl;
      fsm_num = 0;
    }
    switch (exec_state_)
    {
    case INIT:
    {
      if (!have_odom_)
      {
        return;
      }
      if (!trigger_)
      {
        return;
      }
      changeFSMExecState(WAIT_TARGET, "FSM");
      break;
    }
    case WAIT_TARGET:
    {
      if (!have_target_)
        return;
      else
      {
        changeFSMExecState(GEN_NEW_TRAJ, "FSM");
      }
      break;
    }
    case GEN_NEW_TRAJ:
    {
      start_pt_ = odom_pos_;
      start_vel_ = odom_vel_;
      start_acc_.setZero();
      // Eigen::Vector3d rot_x = odom_orient_.toRotationMatrix().block(0, 0, 3, 1);
      // start_yaw_(0)         = atan2(rot_x(1), rot_x(0));
      // start_yaw_(1) = start_yaw_(2) = 0.0;
      bool flag_random_poly_init;
      if (timesOfConsecutiveStateCalls().first == 1)
        flag_random_poly_init = false;
      else
        flag_random_poly_init = true;
      bool success = callReboundReplan(true, flag_random_poly_init);
      if (success)
      {
        changeFSMExecState(EXEC_TRAJ, "FSM");
        flag_escape_emergency_ = true;
      }
      else
      {
        changeFSMExecState(GEN_NEW_TRAJ, "FSM");
      }
      break;
    }
    case REPLAN_TRAJ:
    {
      if (planFromCurrentTraj())
      {
        changeFSMExecState(EXEC_TRAJ, "FSM");
      }
      else
      {
        changeFSMExecState(REPLAN_TRAJ, "FSM");
      }
      break;
    }
    case EXEC_TRAJ:
    {
      /* determine if need to replan */
      LocalTrajData *info = &planner_manager_->local_data_;
      ros::Time time_now = ros::Time::now();
      double t_cur = (time_now - info->start_time_).toSec();
      t_cur = min(info->duration_, t_cur);
      Eigen::Vector3d pos = info->position_traj_.evaluateDeBoorT(t_cur);
      /* && (end_pt_ - pos).norm() < 0.5 */
      if (t_cur > info->duration_ - 1e-2)
      {
        have_target_ = false;
        changeFSMExecState(WAIT_TARGET, "FSM");
        return;
      }
      else if ((end_pt_ - pos).norm() < no_replan_thresh_)
      {
        // cout << "near end" << endl;
        return;
      }
      else if ((info->start_pos_ - pos).norm() < replan_thresh_)
      {
        // cout << "near start" << endl;
        return;
      }
      else
      {
        changeFSMExecState(REPLAN_TRAJ, "FSM");
      }
      break;
    }
    case EMERGENCY_STOP:
    {
      if (flag_escape_emergency_) // Avoiding repeated calls
      {
        callEmergencyStop(odom_pos_);
      }
      else
      {
        if (odom_vel_.norm() < 0.1)
          changeFSMExecState(GEN_NEW_TRAJ, "FSM");
      }
      flag_escape_emergency_ = false;
      break;
    }
    }
    data_disp_.header.stamp = ros::Time::now();
    data_disp_pub_.publish(data_disp_);
  }
```
可知，一共给无人机定义了6个状态
```c
    enum FSM_EXEC_STATE
    {
      INIT, //初始化状态
      WAIT_TARGET, //等待目标点状态
      GEN_NEW_TRAJ, //不清楚
      REPLAN_TRAJ, //重规划状态
      EXEC_TRAJ, //执行状态
      EMERGENCY_STOP //紧急停止状态
    };
```

其次订阅odom里程计话题
`odom_sub_ = nh.subscribe("/odom_world", 1, &EGOReplanFSM::odometryCallback, this);`
`EGOReplanFSM::odometryCallback`函数为里程计更新，没啥好说的

然后创建两个发布方
```c
    bspline_pub_ = nh.advertise<ego_planner::Bspline>("/planning/bspline", 10);

    data_disp_pub_ = nh.advertise<ego_planner::DataDisp>("/planning/data_display", 100);
```
`bspline_pub_`发布的数据输入到`traj_server`节点中，而`traj_server`节点发布的数据是`ego-planner`功能包发布的最终数据，我们只需要订阅即可
## PlanningVisualization类
用于rviz可视化
![Pasted image 20250423165229.png](/images/Pasted-image-20250423165229.png)
主要发表上图中的5个话题，作用为可视化路径点等等
如下图
![Pasted image 20250423163533.png](/images/Pasted-image-20250423163533.png)
绿色线为 `global_list_pub`发布
蓝色线为 `init_list_pub`发布
红色线为 `optimal_list_pub`发布

所有发布方均调用下图函数进行发布
![Pasted image 20250423165522.png](/images/Pasted-image-20250423165522.png)
## EGOPlannerManager类
没啥好说的

## GridMap类
改类主要用于地图的更新等等功能

首先需要设置 `grid_map/pose_type`类型，用于如下代码
```c
  if (mp_.pose_type_ == POSE_STAMPED)
  {
    pose_sub_.reset(
        new message_filters::Subscriber<geometry_msgs::PoseStamped>(node_, "/grid_map/pose", 25));
    sync_image_pose_.reset(new message_filters::Synchronizer<SyncPolicyImagePose>(
        SyncPolicyImagePose(100), *depth_sub_, *pose_sub_));
    sync_image_pose_->registerCallback(boost::bind(&GridMap::depthPoseCallback, this, _1, _2));
  }
  else if (mp_.pose_type_ == ODOMETRY)
  {
    odom_sub_.reset(new message_filters::Subscriber<nav_msgs::Odometry>(node_, "/grid_map/odom", 100));
    sync_image_odom_.reset(new message_filters::Synchronizer<SyncPolicyImageOdom>(
        SyncPolicyImageOdom(100), *depth_sub_, *odom_sub_));
    sync_image_odom_->registerCallback(boost::bind(&GridMap::depthOdomCallback, this, _1, _2));
  }
```

可知，`POSE_STAMPED`与`ODOMETRY`为我们可选类型，默认为 `ODOMETRY`类型，对应的数字为如下
```c
enum { POSE_STAMPED = 1, ODOMETRY = 2, INVALID_IDX = -10000 };
```
`/grid_map/odom`为`odom`话题，可用LIO或者VIO算法发布里程计数据



 订阅方，发布方，定时器如下
```c
  depth_sub_.reset(new message_filters::Subscriber<sensor_msgs::Image>(node_, "/grid_map/depth", 50));
  indep_cloud_sub_ =
      node_.subscribe<sensor_msgs::PointCloud2>("/grid_map/cloud", 10, &GridMap::cloudCallback, this);

  indep_odom_sub_ =
      node_.subscribe<nav_msgs::Odometry>("/grid_map/odom", 10, &GridMap::odomCallback, this);

  
  occ_timer_ = node_.createTimer(ros::Duration(0.05), &GridMap::updateOccupancyCallback, this);
  vis_timer_ = node_.createTimer(ros::Duration(0.05), &GridMap::visCallback, this);

  map_pub_ = node_.advertise<sensor_msgs::PointCloud2>("/grid_map/occupancy", 10);
  map_inf_pub_=node_.advertise<sensor_msgs::PointCloud2("/grid_map/occupancy_inflate", 10);
  unknown_pub_ = node_.advertise<sensor_msgs::PointCloud2>("/grid_map/unknown", 10); 
```

`depth_sub_`、`indep_cloud_sub_`、 `indep_odom_sub_`三个主要订阅方，按要求接入话题即可

## 最后
由 `traj_server`节点发布的 `/position_cmd`话题应该为最终的话题，我们控制无人机应该订阅的是这个话题
```c
  cmd.header.stamp = time_now;
  cmd.header.frame_id = "world";
  cmd.trajectory_flag = quadrotor_msgs::PositionCommand::TRAJECTORY_STATUS_READY;
  cmd.trajectory_id = traj_id_;
  
  cmd.position.x = pos(0);
  cmd.position.y = pos(1);
  cmd.position.z = pos(2);

  cmd.velocity.x = vel(0);
  cmd.velocity.y = vel(1);
  cmd.velocity.z = vel(2);

  cmd.acceleration.x = acc(0);
  cmd.acceleration.y = acc(1);
  cmd.acceleration.z = acc(2);

  cmd.yaw = yaw_yawdot.first;
  cmd.yaw_dot = yaw_yawdot.second;

  last_yaw_ = cmd.yaw;
  pos_cmd_pub.publish(cmd);
```

![Pasted image 20250424125945.png](/images/Pasted-image-20250424125945.png)
