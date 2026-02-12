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
- I am confused about how to implement this, everything is out of the box setup as per instructions but none share the under the hood setup making it hard to derive what I need to do to set up multi stereo on ISSAC ROS 3.2
- my suspcision is I may need to make my own custom nodes to do this. With a yaml file.
- At the end of the day I have come to the opinion that I need to make a customer launch file and since it is multi stereo 
