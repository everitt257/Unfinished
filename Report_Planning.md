# Planning

The `waypoint_updater` node will publish future waypoints from current position. It would also modify the speed given the traffic light condition. If no red light has been detected, this node will set the future waypoints' speed to be a constant number. As one will see in the simulator, the speed would be around 15 MPH when no traffic is detected. However when traffic light is detected and the right stopping region is entered, the future waypoints' speed would be reduced in a linear fashion.

## Setup
The `waypoint_updater` node is subscribed to three topics. Namely, they each are the `/current_pose`, `/base_waypoints`, `/traffic_waypoint`. 

```python
	rospy.Subscriber('/current_pose', PoseStamped, self.pose_cb)
	rospy.Subscriber('/base_waypoints', Lane, self.waypoints_cb)
	rospy.Subscriber('/traffic_waypoint', Int32, self.traffic_cb)
```

And it is publishing the `\finaly_waypoints` topic to be send to the controller. 
```python
	self.final_waypoints_pub = rospy.Publisher('final_waypoints', Lane, queue_size=1)
```

From the subscribers' side it will fetch the entire track, the vehicle's current position and the location of the traffic light.

The publisher would then make use of this information to compose future waypoints that are sent to the controller.

### User defined parameters

Beside the default `LOOKAHEAD_WPS`, the user can set a region where the vehicle is suppose to stop when red light is seen. They may adjust this with the `MIN_D` and `MAX_D`.

The `RefSpeed` defines each target waypoint's speed without the inteference of traffic. The `STOP_LINE` acts as a cutoff region where the vehicle must be absolutely stopped when seen red light.

The `BufferTime` defines *something that needs to addressed later*.

## Localize and Follow Waypoints

In side the `loop`, the waypoint_update node will continusly fetch an index from the `/base_waypoints` topic that is closely related the current position of the vehicle. 

```python
    def nearest_wp(self, last_position, waypoints):
        """find nearest waypoint index to the current location"""
        dl = lambda a, b: math.sqrt((a.x-b.x)**2 + (a.y-b.y)**2  + (a.z-b.z)**2)
        nearest_distance = 9999;
        nearest_index = -1;
        for index, waypoint in enumerate(waypoints):
            waypoint_pos = waypoint.pose.pose.position
            distance = dl(last_position, waypoint_pos)
            if distance < nearest_distance:
                nearest_index = index
                nearest_distance = distance
        return nearest_index
```
This is then used to fetch next number of `LOOKAHEAD_WPS` and store them in a `lookAheadWpts` list, which will be published to the final topic `/final_waypoints`. 
```python
	message_to_sent = Lane()
	message_to_sent.header.stamp = rospy.Time.now()
	message_to_sent.header.frame_id = self.frame_id
	message_to_sent.waypoints = lookAheadWpts
	self.final_waypoints_pub.publish(message_to_sent)
```

## Incorporating Traffic light

To decide when to slow down, there are two condis