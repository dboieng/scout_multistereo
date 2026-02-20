# scout_multistereo
This repo runs multistereo on the Jetson Orin Nano on ROS ISAAC 3.2 using 2 Intel realsense Camera's.

# Multi-Stereo Setup (Plan)
- Both guides are very out of the box guides and assume you have the parts as required. Need to do some more Digging.


# Stereo Log Book
## Stereo Log 04.02.26
- Found out that Jetson Orin Nano is Release 36 Version 4.4 which runs ROS ISSAC 3.2 and not the most recent ROS ISSAC 4.1
- I Asked on the NIVIDA forums [here](https://forums.developer.nvidia.com/t/does-ros-issac-4-1-0-work-on-the-jetson-orin-nano/359612) if the Jetson Orin Nano will work on ROS ISSAC 4.1 and if ISSAC ROS 3.2 will have continued support.

## Stereo Log 05.02.26
- Based on the forum response I will go through the process of install ROS ISSAC 3.2
- I am a  bit annoyed they have not answered the question about continued surpport


## Stereo Log 06.02.26
- Had to take day off - largest stock loss in Australia since April 2025

## Stereo Log 10.02.26
- Set up Single stereo on the Jetson Orin Nano following the insturctions [here](https://github.com/dboieng/Thesis)
- This takes approximately 5 hours to complete isntallation of ROS ISSAC 3.2 on the first upload. Note I am using the realsense D435i and it needs to be plugged in during setup.

## Stereo Log 11.02.26
- I want to develop a theorical steps to mutli stereo before implementation.
- There is a tutorial for multiple realsense cameras [here](https://nvidia-isaac-ros.github.io/concepts/visual_slam/cuvslam/tutorial_multi_realsense.html) for ROS ISAAC 4.1 which I need to adapt for ROS ISSAC 3.2
- There is also a tutorial uisng multiple HAWK cameras [here](https://nvidia-isaac-ros.github.io/v/release-3.1/concepts/visual_slam/cuvslam/tutorial_multi_hawk.html) for ROS ISAAC 3.1 that may be useful.
- Both guides are very out of the box guides and assume you have the parts as required. Need to do some more Digging.

## Stereo Log 12.02.26
- I am confused about how to implement this, everything is out of the box setup as per instructions but none share the under the hood setup making it hard to derive what I need to do to set up multi stereo on ISSAC ROS 3.2.
- my suspcision is I may need to make my own custom nodes to do this. With a yaml file.
- At the end of the day I have come to the opinion that I need to make a customer launch file and since it is multi stereo.

## Stereo Log 13.02.26
- I researched more how to implement and found this hidden example with no documentation (here)[https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_examples/blob/release-3.2/isaac_ros_multicamera_vo/launch/isaac_ros_visual_slam_multirealsense.launch.py]
- I have decided with the current time frame to run and implement this hiiden package without instructions.

## Stereo Log 16.02.26
- I have today implemented the above package and found it somewhat works using the following get up commands

```bash
  cd ${ISAAC_ROS_WS}/src

  # Your command (this is fine)
  git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_examples.git isaac_ros_examples
```

```bash
  colcon build --symlink-install   --packages-up-to isaac_ros_multicamera_vo   --packages-skip isaac_ros_ess_models_install isaac_ros_peoplesemseg_models_install isaac_ros_peoplenet_models_install   --allow-overriding isaac_ros_stereo_image_proc isaac_ros_visual_slam
```

```bash
  ros2 launch isaac_ros_multicamera_vo isaac_ros_visual_slam_multirealsense.launch.py
```

- At end of the day I got the launch file to run! but it would tell me there are errors.
- I keep getting a sync error so tomorrow I am hardware sync the cameras and see what happens and then start the debugging process


## Stereo Log 17.02.26
- Today was a disaster when I came into the lab all the hardware for the Multi-Stereo Setup went missing. I couldn't find it anywhere.
- I was so annoyed and upset, I went ahead and bought the parts from digiKey again but the delivery says arrival on Friday wheich is when the presentation is.
- I am going to go ahead and try anf jerry rig something up instead but the connection will be flimsy without the right parts.
- They are in a box named scout parts which has gone missing

## Stereo Log 18.02.26
- I had another look through the lab and found the box hidden behind a whole heap of other boxes within the lab! Sucess
- I went to work and started the hardware setup as per (this)[https://github.com/NVlabs/PyCuVSLAM/blob/main/examples/realsense/multicamera_hardware_assembly.md] guide, which matches the realsaense guide.

## Stereo Log 19.02.26
- I spent hours trying to get multi stereo to work only to find that the IntelRealsense 435 cameras do not work in ISAAC ROS, only 455 and 435i works.
- I found this out by going back and trying to get single stereo to work first and found it did work so i switched to the 435i which worked!
- I took an image of the node diagram (below) my suspcision is that an extra camera node should exist publishing into the vslam node.
<img width="1098" height="705" alt="Screenshot from 2026-02-12 17-24-01" src="https://github.com/user-attachments/assets/7f16e101-2ed7-452b-9f9f-5a227f55f687" />
- I swapped to 2 Intel Realsense 435i, and then things looking cleaner however, I faced an issue where the camera nodes where silently failing and not publishing to the resapective topics.
- I ended up with the Node diagram that looked like this after I relased the urdf and yaml files must have matching camera names:

## Stereo Log 20.02.26
- We present today I have 4 hours to fix this, to solve the problem I begin dubugging through everything and found out that I needed to upload a specfic .xml file to foxglove.
- Once I fixed this issue, I found there was a comma missing at the very end of the launch file causing the realsense camera node to not start up and silently fail.
- Once I fixed this issue one camera node was connecting to vslam however the second node was still failing silently. I found that there was a naming clash and rewrote the realsense node code.
- sucess! problem solved! it works halleuia!
- Setup Guide to follow.   
