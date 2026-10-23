# 123
全部修复完成，验证通过。总结这次的问题和改动：                        
                                                                         
   根因                                                                  
                                                                         
   left_clamp 在 URDF 里是 mimic 关节（<mimic joint="right_clamp"        
   multiplier="-1"/>，跟随右夹爪反向运动），但描述包给它配置了 command   
   interface。ros2_control 硬性规定 mimic 关节不允许接受指令（否则跟随关 
   系和控制指令会冲突），解析 robot_description 时直接抛异常             
   ，ros2_control_node 崩溃，之后 joint_state_broadcaster 和 MoveIt 控制 
   器全部起不来。                                                        
                                                                         
   修改了 5 处                                                           
                                                                         
   ┌─────────────────────────────────┬──────────────────────────────────┐
   │ 文件                            │ 改动                             │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so_arm_101_description/urdf/so_ │ left_clamp 删掉 command          │
   │ 101.ros2_control.xacro          │ interface（只留状态接口）；新增  │
   │                                 │ sim_backend='none' 分支跳过整个  │
   │                                 │ 硬件块                           │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so_arm_101_description/config/c │ gripper_controller 的 joints 只  │
   │ ontrollers.yaml                 │ 保留 right_clamp                 │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so101_moveit_config/config/so_1 │ 声明 sim_backend 默认为          │
   │ 01.urdf.xacro                   │ none（MoveIt 自带 FakeSystem，不 │
   │                                 │ 需要 Gazebo 硬件）               │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so101_moveit_config/config/so_1 │ FakeSystem 里补上 left_clamp（仅 │
   │ 01.ros2_control.xacro           │ 状态接口，位置由 GenericSystem   │
   │                                 │ 按 mimic 关系自动镜像）          │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so101_moveit_config/config/so_1 │ 102gripper 组移除 left_clamp，只 │
   │ 01.srdf                         │ 留 right_clamp 作为主动关节      │
   ├─────────────────────────────────┼──────────────────────────────────┤
   │ so101_moveit_config/config/move │ 新建，解决 No controller_names   │
   │ it_controllers.yaml             │ specified，让 RViz 里 Plan &     │
   │                                 │ Execute 可用                     │
   └─────────────────────────────────┴──────────────────────────────────┘
                                                                         
   验证结果                                                              
                                                                         
   • ros2_control_node 正常加载并激活 FakeSystem，arm_controller /       
     gripper_controller / joint_state_broadcaster 全部 Configured and    
     activated                                                           
   • /joint_states 包含全部 7 个关节，left_clamp 由 mimic 关系自动发布状 
     态                                                                  
   • RViz 里拖动 102arm 末端小球应该能正常 Plan & Execute 了             
     （position_only_ik 也已生效）                                       
                                                                         
   两个无害的报错可以忽略：No 3D sensor plugin(s)（没接深度相机，不建    
   octomap）和 /recognize_objects not available（没配置物体识别）。      
                                                                         
   另外注意：以后别用 Setup Assistant 重新生成这个配置包，否则这些手动修 
   改会被覆盖。另外如果你后续要用 Gazebo/MuJoCo 仿真，启动描述包自己的   
   launch 文件即可（sim_backend 默认还是 gazebo，行为不变）。
