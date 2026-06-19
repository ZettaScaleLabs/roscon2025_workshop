#
# Copyright (c) 2025 ZettaScale Technology
#
# This program and the accompanying materials are made available under
# the terms of the Apache License, Version 2.0 which is available at
# https://www.apache.org/licenses/LICENSE-2.0.
#
# SPDX-License-Identifier: Apache-2.0
#
# Contributors:
#   ZettaScale Zenoh Team, <zenoh@zettascale.tech>
#

ARG ROS_DISTRO=jazzy

ARG BASE_IMAGE=ghcr.io/tiryoh/ros2-desktop-vnc:${ROS_DISTRO}

FROM ${BASE_IMAGE}

# Install usefull tools
RUN apt-get update && apt upgrade -y && apt-get install -y --no-install-recommends \
        git wget curl gpg vim nano emacs iputils-ping iproute2 net-tools just iftop

# Install Nav2 Turtlebot3 demo
RUN apt-get install -y --no-install-recommends \
        ros-${ROS_DISTRO}-turtlebot3-simulations ros-${ROS_DISTRO}-turtlebot3-teleop \
        ros-${ROS_DISTRO}-nav2-bringup

# Build rmw_zenoh from sources for latest version
RUN mkdir -p /home/ubuntu/rmw_zenoh/src \
    && cd /home/ubuntu/rmw_zenoh/src \
    && git clone --branch $ROS_DISTRO https://github.com/ros2/rmw_zenoh.git \
    && cd .. \
    && rosdep install --from-paths src --ignore-src --rosdistro $ROS_DISTRO -y \
    && source /opt/ros/$ROS_DISTRO/setup.bash \
    && colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release \
    && rm -fr build/ log/

# Build Neobotix ROX simulation packages
RUN apt-get install -y --no-install-recommends \
        xterm python3-colcon-common-extensions \
        ros-${ROS_DISTRO}-robotiq-description \
        ros-${ROS_DISTRO}-realsense2-description \
    && mkdir -p /home/ubuntu/rox_ws/src \
    && cd /home/ubuntu/rox_ws/src \
    && git clone --branch $ROS_DISTRO     https://github.com/neobotix/rox.git \
    && git clone --branch $ROS_DISTRO     https://github.com/neobotix/neo_local_planner2.git \
    && git clone --branch $ROS_DISTRO     https://github.com/neobotix/neo_localization2.git \
    && git clone --branch $ROS_DISTRO     https://github.com/neobotix/neo_rox_moveit2.git \
    && git clone --branch $ROS_DISTRO     https://github.com/neobotix/neo_common2 \
    && git clone --branch master          https://github.com/neobotix/neo_msgs2 \
    && git clone --branch master          https://github.com/neobotix/neo_srvs2 \
    && git clone --branch main            https://github.com/neobotix/neo_gz_worlds.git \
    && cd .. \
    && rosdep install --from-paths ./src --ignore-src --rosdistro $ROS_DISTRO -r --skip-keys "$skip_depend" -y \
    && source /opt/ros/$ROS_DISTRO/setup.bash \
    && colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release \
    && rm -fr build/ log/

# Install other ROS usefull packages
RUN apt-get install -y --no-install-recommends \
        ros-${ROS_DISTRO}-rclpy \
        ros-${ROS_DISTRO}-turtlebot3-simulations \
        ros-${ROS_DISTRO}-turtlebot3-teleop \
        ros-${ROS_DISTRO}-nav2-bringup

# Cleanup
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy environment and other files
COPY files_to_copy/ /home/ubuntu/

# Make ~/.bashrc to load ~/workshop_env.bash and just completions
RUN echo 'source /home/ubuntu/workshop_env.bash' >> /home/ubuntu/.bashrc \
    && echo 'eval "$(just --completions bash)"' >> /home/ubuntu/.bashrc
