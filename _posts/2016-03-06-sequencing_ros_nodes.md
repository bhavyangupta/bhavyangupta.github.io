---
layout: post
title: "Design pattern for sequencing ROS nodes"
---

In this post I want to present a "design pattern" for event-based sequencing
of ROS nodes. I will first describe the scenario in which the solution may be
applicable, followed by a description of the solution I chose.

## Design Problem

We have 3 ROS nodes, lets call them *low_level_node*, *mid_level_node* and 
*high_level_node* respectively. The levels in the node denote the relative 
positions of the nodes in the control hierarchy - i.e. the *low_level_node*
subscribes to messages published by the *mid_level_node* and uses them to 
generate its output. The *mid_level_node* subscribes to the messages published 
by *high_level_node* and uses them to generate its output.

An example scenario is that
of trajectory control of robots - here the *high_level_node* is the node that
generates the desired trajectory, *mid_level_node* can be a trajectory controller
that subscribes to the trajectory generator and vicon messages and publishes 
the control command. Since the output of this controller (generally force and/or
moments) may not be directly usable by the robot, a *low_level_node* can be used 
that subscribes to the control messages and maps them to actuator commands. In 
fact, this is the case that motivates this post.

If all we wanted were three nodes connected in a cascaded manner and we didn't 
care when the nodes started, then the problem was trivial. Let's setup some 
stricter requirements which will make this post worthwhile.

**Requirements**

1. The *high_level_node* starts publishing as soon as its started and its output
   is some function of the time elapsed and initial state. Consequently, the
   time and state at which this node is started should be well-defined.

2. In an experimental setting, we would like to setup the lower-level nodes 
   first to make sure that the setup is functioning as expected and only proceed
   to start the higher level node if its safe to actually send commands to the
   robot. Therefore **the *high_level_node* must be started at a later point than
   the lower level nodes. The state initialization of the *higher_level_node*
   must be taken care of.**

3. We want to be able to use different kinds of *high_level_nodes* with the 
   same low-level nodes. (Eg: be able to follow different kinds of trajectories
   using the same trajectory controller and low level controller).

## Possible Solutions

The first obvious choice to look into is to use a single launch file and start
all nodes simultaneously. This clearly violates requirement 2 since roslaunch
will launch all nodes together and start the experiment, without allowing
time to check the setup.

If the three nodes were implemented as classes, then we could create an 
explicit dependency between them by creating a node that declares the objects
of the three classes and monitors and manipulates their state. This would 
not satisfy requirement 3, since changing the trajectory would require changing
the object used to generate the trajectory. Moreover, this defeats the purpose
of using different ROS nodes for the three levels in the first place.

## Working Solution

Now that we've looked at solutions that don't work, lets arrive at something
that will work.

Firstly, we need to ensure that the initialization of the *high_level_node* is
independent of the mechanism used to invoke it. In other words, we don't want 
the initial conditions to be passed as rosparams or be set manually in the code.
To solve this, the high level node must use `ros::Time::now()` and store it as
time `t0` when it starts. All time elapsed must be measured relative to this 
initial time using `ros::Time` arithmetic [1]. 

The state initialization would depend on what kind of state we are trying to 
initialize, but in most cases it can be done by capturing some messages on a 
topic when the node starts and storing them. In the trajectory following case,
the generator must know the initial pose of the vehicle to give correct commands
and therefore, the generator can subscribe to the vicon topic and record the
first message it gets as the initial pose. Since the generator won't need the
vicon data after this, it can shut down the vicon subscriber.

Now that we have ensured that the *high_level_node* would be in the **right state**
when it starts, we need a way to invoke it at the **right time**. This is where
the concept of "event" is important. There are two ways to do this, which are
very similar:

1. **Use two separate launch files with manual invocation:** one for the 
   lower-level nodes and another for the higher-level nodes. Start the
   lower-level one first, perform the checks that need to be performed and then 
   launch the higher-level nodes. The launching of both files is done by the 
   person conducting the experiment.

2. **Invoke the higher-level node launch file from the lower-level controller:**
   Use the `execve` system call to execute the higher level launch file when 
   some conditions are satisfied.

There is a subtle difference between the two cases: in the first case, the lower
level node does not know when or if the higher level node has been started. On
the other hand, in the second case, the the lower level node is responsible for
invoking the higher level node and knows when the higher level node starts. In
other words, **the second approach allows the lower level node to start the
higher level node based on some event**. This difference is important if you
want the lower level node to behave differently until the higher level node is
started.

In the trajectory following application, I chose the second option because of
that specific reason. The robot's constraints required that no messages be 
published until you want to actually move the vehicle. This required that the 
lower level control nodes wait until the higher level trajectory generator
had started and then proceed to publish piloting commands. The "event" I used
was a joystick button press that a user gives when he/she is sure that the 
experimental setup works as expected and its safe to proceed with trajectory
following.

Here's the low-level controller code that implements approach 2:

```
#include <ros/ros.h>
#include <string>
#include <unistd.h>
#include <signal.h>
#include <iostream>
#include <trajectory_controller/ViconControl.h>
#include "uno_controller.hpp"

using std::cout;
using std::cerr;
using std::endl;
using std::string;

void WaitForJoyStartCmd(UnoController&);

const int kPollingRateHz = 30;

int main(int argc, char** argv) {
  ros::init(argc, argv, "uno_bootstrap");
  ros::NodeHandle nh;
  string node_name;
  string pkg_name;
  string launch("roslaunch");
  nh.param<string>("traj_node_name", node_name, "circle_trajectory.launch");
  nh.param<string>("traj_pkg_name", pkg_name, "trajectory_generator");

  // Start the vehicle controller. This sets up the subscribers and publishers
  // but doesn't publish anything.
  UnoController uno_controller(nh);
 
  // Wait for the explicit command to start the experiment. Blocking call.
  WaitForJoyStartCmd(uno_controller); 
  /*** Complete setup checks till this point ****/ 

  // Spawn a new process and start the trajectory generator in it. The generator
  // fetches the inital pose on its own by subscribing to vicon.
  int child_pid = fork();
    if (child_pid == 0) {
    ROS_INFO_STREAM("Starting Node: " << launch << " " << pkg_name << " " \ 
                    << node_name);
    int retval = execlp(launch.c_str(),launch.c_str(), pkg_name.c_str(), \
                        node_name.c_str(), (char*) NULL);
    if (retval == -1) {
      ROS_ERROR("execl Failed");
    }
  } 
  // This is the normal control loop of the vicon controller.
  ros::Rate loop_rate(30);
  while(nh.ok()) {
    uno_controller.PublishPilotCmd();
    ros::spinOnce();
    loop_rate.sleep();
  }
  // Kill child to avoid orphan process.
  ROS_INFO_STREAM("Killing trajectory_generator node...");
  kill(child_pid, NULL); 
  ROS_INFO_STREAM("...Killed");
  exit(0);
}

void WaitForJoyStartCmd(UnoController& uno_handle) {
  Joy joy_msg = uno_handle.GetLastJoyMsg();
  ros::Rate polling_rate(kPollingRateHz);
  ROS_INFO_STREAM("Waiting for joystick start cmd...");
  while (1) {
    joy_msg = uno_handle.GetLastJoyMsg();
    if (!joy_msg.buttons.empty() && joy_msg.buttons[1]) {
      break;
    }
    ros::spinOnce();
    polling_rate.sleep();
  }
  ROS_INFO_STREAM("Starting Experiment...");
  return;
}

```

Here's a video that shows how the `rqt_graph` changes as this code executes:
<iframe width="560" height="315" src="https://www.youtube.com/embed/Lo3GrzQBPz8" frameborder="0" allowfullscreen> 
</iframe>

## Reference

1. http://wiki.ros.org/roscpp/Overview/Time

