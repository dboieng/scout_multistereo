# scout_multistereo
This repo runs multistereo on the Jetson Orin Nano using 3 Intel realsense Camera's

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

