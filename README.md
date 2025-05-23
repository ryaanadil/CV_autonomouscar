<h1 align="center">Team 6 Final Project - Drone Following with Computer Vision</h1>
---

[![Video of final project](https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/68714078/c1dc3038-7345-4b93-b0b6-4b096942d60a)](http://www.youtube.com/watch?v=H7b-YXtxtiM&ab_)

### Abstract 
Our team built an autonomous car that could identify, track, and follow a flying drone using machine learning, openCV, and ROS2

**Objectives**
- Use machine learning and OpenCV to recognize drone
- Use ROS2 running on the Jetson Nano to connect vision and car control
- Follow the drone as it flies around
- Arrest motion when drone leaves the frame or moves to land on the moving base


## Hardware 

  <p align="center"><img src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/164306890/d3f91601-d6c6-426f-b8aa-4733af1c4f2f" width="700" height="400"></p>


One of the main focus was to design a camera mount with an adjustable angle of attack for us to optimize the cars field of view and take in account that latency would result in the drone being lost intermittently. To account for this we tried to optimize our field of view

  <p align="center"><img src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/164306890/fec69d6d-94ca-4cc7-9c39-1b82783bdb7e" width="700" height="400"></p>



## The Roboflow Trained Model


Using Roboflow we were able to train the model and push it on to our UCSD docker container.
importing the roboflow model was one thing, using code to located the centroid and filter our confidence level to an optimal value such that our Oak'd would not mistake any other foriegn object was another hassle. As can be seen in the images below:

  <p align="center"><img src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/164306890/3cf85766-3ce2-4085-9c80-a0b58e8be5d9"width="400" height="400"></p>



The image below illustrates the optimized version and the code utilized in attaining centriod coordinates with respective error bars. Now the object with the highest confidence interval is selected for centroid placement and this holds valid for our objective that is to track the drone.

            predictions = result["predictions"]

            # log the x y coords of the drone
            self.prev_highest_confidence = 0
            for p in predictions:
                
                self.confidence = float(p.confidence)
                if self.confidence > self.prev_highest_confidence:
                    self.prev_highest_confidence = self.confidence
                    self._x = int(p.x)
                    self._y = int(p.y)

                self.get_logger().info(f'x: {self._x}, y: {self._y}, Confidence: {self.confidence}, Highest confidence: {self.prev_highest_confidence}')

            # Get the center coordinates of the frame
            #if 
            frame_height, frame_width, num_channels = frame.shape
            center_x = frame_width // 2
            center_y = frame_height // 2


            ### CALCULATE THE ERROR FOR STEERING ###
            self.error = self._x - center_x
            self.get_logger().info(f'Error: {self.error}')

  
   <p align="center"><img src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/164306890/4158c6bd-3b4d-49e4-834d-327ca3fca48a"width="400" height="400"></p>


## Using ROS2 To Configure The Respective Nodes

Once we trained our model the next step was to create a Python Package in ROS2 to configure our nodes to publish and subrcribe to the displayed information by the OAK-D camera.

**Our Nodes Can Be Divided into the following Categories:**
- The Vision Node (drone_detection2.py)
- The PID Car Control Node (pid.py)
- The VESC Node (vesc_twist_node_custom.py)

**Drone Detection Using OAK-D Vision Node**

For vision testing we imported the following libraries to our node and dfined a class to process the coordinate points for our captured drone images
        
    import rclpy
    from rclpy.node import Node
    from roboflowoak import RoboflowOak
    import cv2
    from cv_bridge import CvBridge
    import time
    import numpy as np
    #from pub_drone_detection2_pkg.msg import CameraError
    from std_msgs.msg import Int32

     NODE_NAME = "topic_erreurFrame"

     class RoboflowOakNode(Node):

    error = 0
    prev_error = 0
    _x = 0
    _y = 0
    confidence = 0
    repeat_counter = 0

    def __init__(self):
        super().__init__(NODE_NAME)

        # Instantiate an object (rf) with the RoboflowOak module
        self.rf = RoboflowOak(model="drone-tracking-fztzm", confidence=0.05, overlap=0.5,
                               version="1", api_key="0RmqstHKjwDcunOH9wus", rgb=True,
                               depth=True, device=None, blocking=True)
        
        self.publisher_ = self.create_publisher(Int32, NODE_NAME, 10)

        self.run()

Once the Oak'd node detects the drone it publishes on a common topic subscribed by the PID node and the Vesc node. This streamlines the detection capability and allows realtime communication and allows for smoother following as opposed to exclusively using Python instead of ROS2.


**The PID Node**

After subscribing to drone_detection2.py the car is then able to adjust its steering and maneuverability using the set PID values. For this case we found P = 0.6 I = 0.25 and D = 0.2 as optimal.

    def __init__(self):
        super().__init__(NODE_NAME) # We are calling the constructor (Initilizes Object state, setting initial values for attributes) of a super class (Node) which passes the "NODE_NAME"           as the arguement 
        self.twist_publisher = self.create_publisher(Twist, ACTUATOR_TOPIC_NAME, 10)
        self.twist_cmd = Twist()
        self.error_subscriber = self.create_subscription(Int32, ERROR_TOPIC_NAME, self.controller, 10)

        # # Default actuator values
        # self.declare_parameters(
        #     namespace='',
        #     parameters=[
        #         ('Kp_steering', 1),
        #         ('Ki_steering', 0),
        #         ('Kd_steering', 0),
        #         ('error_threshold', 0.15),
        #         ('zero_throttle',0.0),
        #         ('max_throttle', 0.2),
        #         ('min_throttle', 0.1),
        #         ('max_right_steering', 1.0),
        #         ('max_left_steering', -1.0)
        #     ])

        self.Kp = 0.60 # between [0,1]
        self.Ki = 0.25 # between [0,1]
        self.Kd = 0.20 # between [0,1]
        
        self.zero_throttle = 0 # between [-1,1] but should be around 0
        self.max_throttle = 0.2 # between [-1,1]
        self.min_throttle = 0 # between [-1,1]
        self.max_right_steering = 0.5 # between [-1,1]
        self.max_left_steering = -0.5 # between [-1,1]
        self.error_threshold = 0.15

        # initializing PID control
        self.Ts = float(1/20)
        self.ek = 0 # current error
        self.ek_1 = 0 # previous error
        self.proportional_error = 0 # proportional error term for steering
        self.derivative_error = 0 # derivative error term for steering
        self.integral_error = 0 # integral error term for steering
        self.integral_max = 1E-8
        
Similarly the max and min throttle are multiples and hence we set them to slightly lower values for better maneuverabilty and operation at lower rpm.

**The Vesc Node**

Once subscribeed to the PID node the vesc is able to control the motors and steering accordingly to follow the drone. By default the car will stop once the drone is no longer detected. To account for temporary loss of sight the car is programed to drive and steer with a lag before coming to a halt when the drone is no longer sighted.



    def callback(self, msg):
        self.get_logger().info(f'vesc: got into callback')
        # # Steering map from [-1,1] --> [0,1]
        steering_angle = float(self.steering_offset + self.remap(msg.angular.z))
        
        # RPM map from [-1,1] --> [-max_rpm,max_rpm]
        rpm = int(self.max_rpm * msg.linear.x)

        # self.get_logger().info(f'rpm: {rpm}, steering_angle: {steering_angle}')
        
        self.vesc.send_rpm(int(self.throttle_polarity * rpm))
        self.vesc.send_servo_angle(float(self.steering_polarity * steering_angle))

    def remap(self, value):
        input_start = -1
        input_end = 1
        output_start = 0
        output_end = 1
        normalized_output = float(output_start + (value - input_start) * ((output_end - output_start) / (input_end - input_start)))
        return normalized_output
    
    def clamp(self, data, upper_bound, lower_bound=None):
            if lower_bound==None:
                lower_bound = -upper_bound # making lower bound symmetric about zero
            if data < lower_bound:
                data_c = lower_bound
            elif data > upper_bound:
                data_c = upper_bound
            else:
                data_c = data
            return data_c 
            def main(args=None):
    rclpy.init(args=args)
    try:
        vesc_twist = VescTwist()
        rclpy.spin(vesc_twist)
        vesc_twist.destroy_node()
        rclpy.shutdown()
    except:
        vesc_twist.get_logger().info(f'Could not connect to VESC, Shutting down {NODE_NAME}...')
        vesc_twist.destroy_node()
        rclpy.shutdown()
        vesc_twist.get_logger().info(f'{NODE_NAME} shut down successfully.')


    if __name__ == '__main__':
        main()

## Issues With X-11 Forwarding - X Authority

As a result of the jetson crashing multiple times, we faced multiple hinderances with launching our nodes mainly due to X-authority files, for which we found a simple fix that is to remove the X-authority files, re-install the X server using the following command, and set all permissions for .Xauthority file 

          sudo apt-get install --reinstall xserver-xorg

and to set permissions:

          Sudo chmod 777 .Xauthority

## The Schematics
<img width="562" alt="image" src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/68714078/345b6fe9-c39f-4d32-8d54-741b34929343">


## What We Could Have Done If Had More Time.
- Incorporate a second camera to land the drone perfectly on the landing base
- Incorporate obstruction avoidance using Lidar or RGB depth
- add a solid landing base for the drone to land on the car


## The Team

<p align="center"><img src="https://github.com/UCSD-ECEMAE-148/winter-2024-final-project-team-6/assets/164306890/66c37c50-6c7b-4a25-b513-662b73621d7b"width="500" height="400"></p>

**Contact**
- Clement BONOMELLI Clement.BONOMELLI@estaca.eu
- Ryaan Adil Rashid ryaanadil@outlook.com
- Ben Garofalo bgarofalo@ucsd.edu [LinkedIn](https://www.linkedin.com/in/bengarofalo/)
- Jeffery Keppler jkeppler@ucsd.edu

## Acknowledgments
Thank you to UCSD Jacobs School of Engineering, Triton AI, Professor Jack Silberman, Arjun Naageshwaran, Alexander Haken, Winston Chou, John Liu (especially thank you lending us drone), and Daniel Vega Zepeda for taking the time to support us along the way!

