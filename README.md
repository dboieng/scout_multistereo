# scout_multistereo
This repo runs multistereo on the Jetson Orin Nano on ROS ISAAC 3.2 using 2 Intel Realsense 435i Camera's.
Start from **Step 8** if you have setup this up before.

[Screencast from 02-20-2026 01:32:36 PM.webm](https://github.com/user-attachments/assets/f73d2423-a015-455b-881f-f4c10373a887)


## Multi-Stereo Setup (Guide)
- This guide is designed for ROS ISAAC 3.2 on the Jetson Orin Nano fixing hidden pre-existing examples. I assume you have ROS ISAAC container setup and 2 IntelRealsense 435i's.
- It is an amalgmation of different guides and uses a Nivida Example that exists in the ISAAC ROS 3.2 Git repo with no setup guide or reference in NIVIDA's 3.1 or 3.2 documentation it is ported over to visual slam folder from ROS ISAAC 4.0.

### Prerequisites - I assume you have Hardware setup on the Jetson Orin Nano
- If not follow my guide [here](https://github.com/dboieng/Thesis/edit/main/README.md) steps 1 to 7.

### Step 0 - Setup Hardware on Intel Realsense
- Follow the parts guide [here](https://github.com/NVlabs/PyCuVSLAM/blob/main/examples/realsense/multicamera_hardware_assembly.md) and find attached image of my setup:
![9X0eVoPh](https://github.com/user-attachments/assets/a1efe1c2-36e4-44a0-866a-4e8e82c805dc)

### Step 1 - Create the Isaac ROS workspace and set environment variable

```bash
  mkdir -p ~/isaac_ros_ws/src

  # Add ISAAC_ROS_WS to your shell configuration (bash example)
  echo "export ISAAC_ROS_WS=$HOME/isaac_ros_ws" >> ~/.bashrc
  source ~/.bashrc

  cd "${ISAAC_ROS_WS}/src"
```

### Step 2. Clone Isaac ROS repositories (release 3.2)

```bash
cd "${ISAAC_ROS_WS}/src"

# Isaac ROS common (container tooling)
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common.git isaac_ros_common

# Image pipeline (includes stereo image processing)
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_image_pipeline.git isaac_ros_image_pipeline

# Visual SLAM (cuVSLAM)
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_visual_slam.git isaac_ros_visual_slam

# GPU accleration (important)
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nitros.git

# ⭐ REQUIRED for multi-camera VO
git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_examples.git isaac_ros_examples

```

### Step 3. Configure Isaac ROS Container (RealSense Image)

Create the Isaac ROS common config file and set the CONFIG_IMAGE_KEY to the RealSense-enabled image (includes librealsense SDK and realsense-ros):
```bash
cd "${ISAAC_ROS_WS}/src/isaac_ros_common/scripts"

touch .isaac_ros_common-config
echo "CONFIG_IMAGE_KEY=ros2_humble.realsense" > .isaac_ros_common-config
```

### Step 4. Fit Jetson CDI Issue (Important)
From the host:
This rebuilds the container image using Dockerfile.realsense in one of its layered stages. Rebuilding can take several minutes.
```bash
cd ${ISAAC_ROS_WS}/src/isaac_ros_common && \
./scripts/run_dev.sh -d ${ISAAC_ROS_WS}
```
- if you get this: docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: could not apply required modification to OCI specification: error modifying OCI spec: failed to inject CDI devices: unresolvable CDI devices nvidia.com/gpu=all: unknown
- open ~/isaac_ros_ws/src/isaac_ros_common/scripts/run_dev.sh and find the line: DOCKER_ARGS+=("-e NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=all,nvidia.com/pva=all") and replace with DOCKER_ARGS+=("-e NVIDIA_VISIBLE_DEVICES=all")
- I Found the above solution to not be very good and instead tried this off the forums which works:
- to these install on host
```bash
sudo nvidia-ctk cdi generate --mode=csv --output=/etc/cdi/nvidia.yaml

# Add Jetson public APT repository
sudo apt-get update
sudo apt-get install software-properties-common
sudo apt-key adv --fetch-key https://repo.download.nvidia.com/jetson/jetson-ota-public.asc
sudo add-apt-repository 'deb https://repo.download.nvidia.com/jetson/common r36.4 main'
sudo apt-get update
sudo apt-get install -y pva-allow-2
```

- Verify Realsense Setup with
```bash
  realsense-viewer
```
- If you have mutiple cameras
```bash
  rs-enumerate-devices -S
```

### Step 5. Fix the launch file
- The launch file has two key errors there is a missing comma at the end of the file that leads to the node setup hanging and there setup of the camera nodes fails because it does not iteratet he setup.

```bash
  cd ${ISAAC_ROS_WS}/src/isaac_ros_examples/isaac_ros_multicamera_vo/launch
  vim isaac_ros_slam_multirealsense.launch.py
```

- please copy and paste the following launch file into the file named 'isaac_ros_slam_multirealsense.launch.py'

```bash
  # SPDX-FileCopyrightText: NVIDIA CORPORATION & AFFILIATES
# Copyright (c) 2021-2024 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# SPDX-License-Identifier: Apache-2.0
# flake8: noqa: F403,F405

import os
import yaml

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.substitutions import LaunchConfiguration
from launch.conditions import LaunchConfigurationEquals
from launch_ros.actions import ComposableNodeContainer, LoadComposableNodes, Node
from launch_ros.descriptions import ComposableNode
from launch_xml.launch_description_sources import XMLLaunchDescriptionSource


def generate_launch_description():

    use_rosbag_arg = DeclareLaunchArgument('use_rosbag', default_value='False',
                                           description='Whether to execute on rosbag')
    use_rosbag = LaunchConfiguration('use_rosbag')

    config_directory = get_package_share_directory('isaac_ros_multicamera_vo')
    foxglove_xml_config = os.path.join(config_directory, 'config', 'foxglove_bridge_launch.xml')
    foxglove_bridge_launch = IncludeLaunchDescription(
        XMLLaunchDescriptionSource([foxglove_xml_config])
    )

    urdf_file = os.path.join(config_directory, 'urdf', '2_realsense_calibration.urdf.xacro')
    with open(urdf_file, 'r') as f:
        robot_description = f.read()

    rs_config_path = os.path.join(config_directory, 'config', 'multi_realsense.yaml')
    with open(rs_config_path) as rs_config_file:
        rs_config = yaml.safe_load(rs_config_file)

    remapping_list, optical_frames = [], []
    num_cameras = 2*len(rs_config['cameras'])

    for idx in range(num_cameras):
        infra_cnt = idx % 2+1
        camera_cnt = rs_config['cameras'][idx//2]['camera_name']
        optical_frames += [f'{camera_cnt}_infra{infra_cnt}_optical_frame']
        remapping_list += [(f'visual_slam/image_{idx}',
                            f'/{camera_cnt}/infra{infra_cnt}/image_rect_raw'),
                           (f'visual_slam/camera_info_{idx}',
                            f'/{camera_cnt}/infra{infra_cnt}/camera_info')]

    def realsense_capture(common_params, camera_params):
        return ComposableNode(
            name='realsense',  # keep node name same inside each namespace
            namespace=camera_params['camera_name'],  # <-- IMPORTANT
            package='realsense2_camera',
            plugin='realsense2_camera::RealSenseNodeFactory',
            parameters=[{**common_params, **camera_params}],
        )

    visual_slam_node = ComposableNode(
        name='visual_slam_node',
        package='isaac_ros_visual_slam',
        plugin='nvidia::isaac_ros::visual_slam::VisualSlamNode',
        parameters=[{
            'enable_image_denoising': False,
            'rectified_images': True,
            'enable_imu_fusion': False,
            'image_jitter_threshold_ms': 34.00,
            'base_frame': 'base_link',
            'enable_slam_visualization': True,
            'enable_landmarks_view': True,
            'enable_observations_view': True,
            'enable_ground_constraint_in_odometry': False,
            'enable_ground_constraint_in_slam': False,
            'enable_localization_n_mapping': True,
            'enable_debug_mode': False,
            'num_cameras': num_cameras,
            'min_num_images': num_cameras,
            'camera_optical_frames': optical_frames,
        }],
        remappings=remapping_list,
    )

    visual_slam_launch_container = ComposableNodeContainer(
        name='visual_slam_launch_container',
        namespace='',
        package='rclcpp_components',
        executable='component_container_mt',
        composable_node_descriptions=([visual_slam_node]),
        output='screen',
    )

    realsense_image_capture = LoadComposableNodes(
        target_container='visual_slam_launch_container',
        condition=LaunchConfigurationEquals('use_rosbag', 'False'),
        composable_node_descriptions=([realsense_capture(rs_config['common_params'], camera_config)
                                       for camera_config in rs_config['cameras']]),
    )

    state_publisher = Node(package='robot_state_publisher',
                           executable='robot_state_publisher',
                           output='both',
                           parameters=[{'robot_description': robot_description}])

    return LaunchDescription([use_rosbag_arg,
                              foxglove_bridge_launch,
                              state_publisher,
                              visual_slam_launch_container,
                              realsense_image_capture,])
```

### Step 6. Find out your Realsense Camera Serial numbers 
- They have to be the same camera class either (X2 435i or X2 455) for the hardware sync to work. Other cameras including 435 and 415 don't work.
```bash
  rs-enumerate-devices -S
```

- And place in the file 'multi_realsense.yaml' located at:
```bash
  cd ${ISAAC_ROS_WS}/src/isaac_ros_examples/isaac_ros_multicamera_vo/config
```

### Step 7. Update the urdf file
- There is a Naming mismatch please copy the following urdf file into 2_realsense_calibration.urdf.xacro
```bash
  cd ${ISAAC_ROS_WS}\${ISAAC_ROS_WS}/src/isaac_ros_examples/isaac_ros_multicamera_vo/urdf
  vim 2_realsense_calibration.urdf.xacro
```
- Copy the following:
```bash
  <?xml version="1.0"?>
<!--
// Copyright 2024 Stereolabs
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<robot name="4_realsenses">
  <link name="base_link"/>

  <joint name="camera1" type="fixed">
    <parent link="base_link"/>
    <child link="camera1_link"/>
    <origin xyz="0.0 0.0 0.0" rpy="0 0 0"/>
  </joint>

  <link name="camera1_link"/>

  <joint name="camera2" type="fixed">
    <parent link="base_link"/>
    <child link="camera2_link"/>
    <origin xyz="-0.1 -0.0475 0.0" rpy="0 0 3.14159"/>
  </joint>

  <link name="camera2_link"/>

</robot>
```

### 8. In the container: Install NGC tooling and download cuVSLAM assets

```bash
sudo apt-get update
sudo apt-get install -y curl jq tar
```
Then, run these commands to download the asset from NGC:
```bash
NGC_ORG="nvidia"
NGC_TEAM="isaac"
PACKAGE_NAME="isaac_ros_visual_slam"
NGC_RESOURCE="isaac_ros_visual_slam_assets"
NGC_FILENAME="quickstart.tar.gz"
MAJOR_VERSION=3
MINOR_VERSION=2

VERSION_REQ_URL="https://catalog.ngc.nvidia.com/api/resources/versions?orgName=$NGC_ORG&teamName=$NGC_TEAM&name=$NGC_RESOURCE&isPublic=true&pageNumber=0&pageSize=100&sortOrder=CREATED_DATE_DESC"

AVAILABLE_VERSIONS=$(curl -s \
    -H "Accept: application/json" "$VERSION_REQ_URL")

LATEST_VERSION_ID=$(echo "$AVAILABLE_VERSIONS" | jq -r "
    .recipeVersions[]
    | .versionId as \$v
    | \$v | select(test(\"^\\\\d+\\\\.\\\\d+\\\\.\\\\d+$\"))
    | split(\".\") | {major: .[0]|tonumber, minor: .[1]|tonumber, patch: .[2]|tonumber}
    | select(.major == $MAJOR_VERSION and .minor <= $MINOR_VERSION)
    | \$v
    " | sort -V | tail -n 1
)

if [ -z "$LATEST_VERSION_ID" ]; then
    echo "No corresponding version found for Isaac ROS $MAJOR_VERSION.$MINOR_VERSION"
    echo "Found versions:"
    echo "$AVAILABLE_VERSIONS" | jq -r '.recipeVersions[].versionId'
else
    mkdir -p "${ISAAC_ROS_WS}/isaac_ros_assets"
    FILE_REQ_URL="https://api.ngc.nvidia.com/v2/resources/$NGC_ORG/$NGC_TEAM/$NGC_RESOURCE/versions/$LATEST_VERSION_ID/files/$NGC_FILENAME"

    curl -LO --request GET "${FILE_REQ_URL}"
    tar -xf "${NGC_FILENAME}" -C "${ISAAC_ROS_WS}/isaac_ros_assets"
    rm "${NGC_FILENAME}"
fi
```
This will populate ${ISAAC_ROS_WS}/isaac_ros_assets with the required cuVSLAM model and configuration assets.

### Step 9. Install package dependencies via rosdep
```bash
cd "${ISAAC_ROS_WS}"

rosdep update

# Stereo pipeline
rosdep install --from-paths \
  ${ISAAC_ROS_WS}/src/isaac_ros_image_pipeline/isaac_ros_stereo_image_proc \
  --ignore-src -y

# Visual SLAM
rosdep install --from-paths \
  ${ISAAC_ROS_WS}/src/isaac_ros_visual_slam/isaac_ros_visual_slam \
  --ignore-src -y

# ⭐ Multi-camera VO dependencies
rosdep install --from-paths \
  ${ISAAC_ROS_WS}/src/isaac_ros_examples/isaac_ros_multicamera_vo \
  --ignore-src -y
```

### Step 10. Build Packages in the correct order
```bash
cd ${ISAAC_ROS_WS}

colcon build --symlink-install \
  --packages-up-to isaac_ros_stereo_image_proc \
  --base-paths ${ISAAC_ROS_WS}/src/isaac_ros_image_pipeline/isaac_ros_stereo_image_proc

colcon build --symlink-install \
  --packages-up-to isaac_ros_visual_slam \
  --base-paths ${ISAAC_ROS_WS}/src/isaac_ros_visual_slam/isaac_ros_visual_slam


colcon build --symlink-install \
  --packages-up-to isaac_ros_multicamera_vo \
  --packages-skip \
    isaac_ros_ess_models_install \
    isaac_ros_peoplesemseg_models_install \
    isaac_ros_peoplenet_models_install \
  --allow-overriding \
    isaac_ros_stereo_image_proc \
    isaac_ros_visual_slam

source install/setup.bash

```

### Step 11. Run Multi-stereo cuVSLAM
```bash
ros2 launch isaac_ros_multicamera_vo \
  isaac_ros_visual_slam_multirealsense.launch.py
````

- ros2 topic list should output:
```bash
ros2 topic list
/camera1/extrinsics/depth_to_infra1
/camera1/extrinsics/depth_to_infra2
/camera1/imu
/camera1/infra1/camera_info
/camera1/infra1/image_rect_raw
/camera1/infra1/image_rect_raw/compressed
/camera1/infra1/image_rect_raw/compressedDepth
/camera1/infra1/image_rect_raw/theora
/camera1/infra1/metadata
/camera1/infra2/camera_info
/camera1/infra2/image_rect_raw
/camera1/infra2/image_rect_raw/compressed
/camera1/infra2/image_rect_raw/compressedDepth
/camera1/infra2/image_rect_raw/theora
/camera1/infra2/metadata
/camera2/extrinsics/depth_to_infra1
/camera2/extrinsics/depth_to_infra2
/camera2/imu
/camera2/infra1/camera_info
/camera2/infra1/image_rect_raw
/camera2/infra1/image_rect_raw/compressed
/camera2/infra1/image_rect_raw/compressedDepth
/camera2/infra1/image_rect_raw/theora
/camera2/infra1/metadata
/camera2/infra2/camera_info
/camera2/infra2/image_rect_raw
/camera2/infra2/image_rect_raw/compressed
/camera2/infra2/image_rect_raw/compressedDepth
/camera2/infra2/image_rect_raw/theora
/camera2/infra2/metadata
/diagnostics
/joint_states
/parameter_events
/robot_description
/rosout
/tf
/tf_static
/visual_slam/status
/visual_slam/tracking/odometry
/visual_slam/tracking/slam_path
/visual_slam/tracking/vo_path
/visual_slam/tracking/vo_pose
/visual_slam/tracking/vo_pose_covariance
/visual_slam/trigger_hint
/visual_slam/vis/gravity
/visual_slam/vis/landmarks_cloud
/visual_slam/vis/localizer
/visual_slam/vis/localizer_loop_closure_cloud
/visual_slam/vis/localizer_map_cloud
/visual_slam/vis/localizer_observations_cloud
/visual_slam/vis/loop_closure_cloud
/visual_slam/vis/observations_cloud
/visual_slam/vis/pose_graph_edges
/visual_slam/vis/pose_graph_edges2
/visual_slam/vis/pose_graph_nodes
/visual_slam/vis/slam_odometry
/visual_slam/vis/velocity
```
- Note tf should exist if it doesn't there is a major problem.


### Step 12. Foxglove visualisation
```bash
  foxglove-studio
```
- The jetson Orin Nano does not have the computational space to support foxglove and cuVSLAM please use wifi or ethernet to connect via foxglove
- on jetson get the hostname:
```bash
  hostname -I
```
- Open connection -> WebSocket: ws://JETSON_IP:8765
- Import file from foxglove layouts click: 'import from file' and select 'foxglove_layout_realsense.json'
- Should Look like this:
<img width="2680" height="1660" alt="image" src="https://github.com/user-attachments/assets/88f2c82b-495a-4385-9ad5-37c13ccf8260" />


## Helpful Debugging commands
- rs-enumerate-devices -S shows all cameras
- ros2 node list shows camera nodes
```bash
  ros2 node list | grep camera
  ros2 node info /camera1/realsense | grep -A3 Publishers
  ros2 node info /camera2/realsense | grep -A3 Publishers
```
- ros2 topic hz shows ~30 Hz
```bash
  ros2 topic hz /camera1/infra1/image_rect_raw
  ros2 topic hz /camera2/infra1/image_rect_raw
```
- confirm TF tree exists for both
```bash
ros2 run tf2_ros tf2_echo camera1_link camera1_infra1_optical_frame
ros2 run tf2_ros tf2_echo camera1_link camera1_infra2_optical_frame
ros2 run tf2_ros tf2_echo camera2_link camera2_infra1_optical_frame
ros2 run tf2_ros tf2_echo camera2_link camera2_infra2_optical_frame
```

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

<img width="1178" height="618" alt="Screenshot from 2026-02-20 11-23-56" src="https://github.com/user-attachments/assets/4ed3fc6c-f3e5-4a54-9326-d218be1d1a83" />


## Stereo Log 20.02.26
- We present today I have 4 hours to fix this, to solve the problem I begin dubugging through everything and found out that I needed to upload a specfic .xml file to foxglove.
- Once I fixed this issue, I found there was a comma missing at the very end of the launch file causing the realsense camera node to not start up and silently fail.

<img width="1172" height="795" alt="Screenshot from 2026-02-19 18-00-45" src="https://github.com/user-attachments/assets/7fd577e2-a8d9-4f62-a310-76ab50328172" />

- Once I fixed this issue one camera node was connecting to vslam however the second node was still failing silently. I found that there was a naming clash and rewrote the realsense node code.
- sucess! problem solved! it works halleuia!

<img width="1099" height="620" alt="Screenshot from 2026-02-20 15-51-54" src="https://github.com/user-attachments/assets/6fdf101b-5a97-4482-a2a9-d714b398eaae" />

- Setup Guide to follow.   
