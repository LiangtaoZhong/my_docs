# ROS

## Change the brightness of the backlight in Ubuntu

1. `$ sudo add-apt-repository ppa:apandada1/brightness-controller` 
2. `$ sudo apt-get update` 
3. `$ sudo apt-get install brightness-controller` 
4. `$ brightness-controler` 

## Install all dependence for a ROS workplace

```shell
rosdep install --from-paths src --ignore-src --rosdistro=melodic -y
rosdep install --from-paths src --ignore-src --rosdistro=kinetic -y
```

## Filter out topics from a bag

```shell
rosbag filter in.bag out.bag "topic == '/tf'"
```

## Static_transform_publisher

```shell
static_transform_publisher x y z yaw pitch roll frame_id child_frame_id period_in_ms
```

- Publish a static coordinate transform to tf using an x/y/z offset in meters and yaw/pitch/roll in radians. (yaw is rotation about Z, pitch is rotation about Y, and roll is rotation about X). The period, in milliseconds, specifies how often to send a transform. 100ms (10hz) is a good value.

```shell
static_transform_publisher x y z qx qy qz qw frame_id child_frame_id period_in_ms
```

- Publish a static coordinate transform to tf using an x/y/z offset in meters and quaternion. The period, in milliseconds, specifies how often to send a transform. 100ms (10hz) is a good value.

`static_transform_publisher` is designed both as a command-line tool for manual use, as well as for use within [roslaunch](http://wiki.ros.org/roslaunch) files for setting static transforms. For example:

```shell
   1 <launch>
   2 <node pkg="tf" type="static_transform_publisher" name="link1_broadcaster" args="1 0 0 0 0 0 1 link1_parent link1 100" />
   3 </launch>
```

## sudo rosdep init error

```shell
sudo vim /etc/hosts
151.101.84.133  raw.githubusercontent.com
```

## HDL-GRPHA-SLAM

To save map.pcd, please use absolute path like `/home/user/map.pcd`, not `~/map.pcd`， or you will get errors.

## cannot find python node

it is caused by **no execute permission**:

so do this:

```shell
chmod 777 node.py
```

## gazebo: symbol lookup error: /usr/lib/x86_64-linux-gnu/libgazebo_common.so.9

```shell
sudo apt upgrade libignition-math2
```

