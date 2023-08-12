# Quick Start Guide for USG

Hello Universities at Shady Grove faculty! If you are looking to wire and start up the Nerf Turret in its current configuration, this is the correct file.

If this software is being used on another system that is not the USG Nerf Turret, or you want to run in non-headless mode, or you want docs about the extended functionality of this software, this is not the right file.

### Hardware
1. Attach a Raspberry Pi 4 to the standoffs or whatever mount was on the acrylic base of the Turret. 
2. Hook up the webcam to a USB on the Pi and the USB C that's coming off of the 12VDC to 5VDC converters to the USB C power port on the Pi.
3. Connect the servos as such:

| Component  | BCM numbering | Board Number |
| --- | --- | -- |
| Left Pitch Servo |GPIO18| 12 |
| Right Pitch Servo  |  GPIO13| 33 |
|  Yaw (left-right) servo|  GPIO12| 32 |
| Fire relay |  GPIO23| 16 |
| Rev relay |  GPIO24| 18|

* The rev relay should have a sticker on it with the letter "R"
* For the L, R and Y servos, just trace the wires up to the gimbal system.


### Software

1. Power on the Pi. 
2. Connect a display to the HDMI outputs on the Pi. This is the only time headless operation of the turret is not possible.
3. In the user's home directory, extract the file named `server.tar.gz` we gave to Dr. Tiglao on Google Classroom into a directory of any name, or alternatively, run:


```
pi@raspberrypi:~ $ git clone https://github.com/yearbook-milk/usgnerfwars -b "usg-configuration"
```
4. Use any text editor to open `config.py`. You should have downloaded the right version which has all the settings already set up for our system, but if not, here are the important settings to verify:

* `TCP_PORT` should be set to a port (where port > `1024`). This is the one you will use to initiate a connection with the Pi in wireless mode.
* `UDP_PORT` should be set to a port (where port > `1024` and is not the same as TCP_PORT). This will be the port video data will be sent over. On initiating a connection, the Pi will propose this UDP port # to the client, and, if it accepts, the Pi will send data over that port.
* `NETWORK_IMAGE_COMPRESSION` should be set to `0.15` or lower. This controls how much the image is compressed before being sent over UDP, and if the resultant compressed image (+encoded metadata) exceeds the packet size limit for UDP, wireless mode won't work.
* `SHOW_LOCAL_OUTPUT` should be `False`.
* `ENABLE_NETWORKING` should be `True` for now.
* `ENABLE_HSI` should be `True`. This exposes the hardware software interface to `main.py` so that it can actually move the servos and trip the relays.
* `DEFAULT_CAMERA` should be `0`, and the USB webcam should be the camera attached to the Pi.
* `CENTERING_METHOD` should be `"RATIO"`
* `CETNERING_TOLERANCE` should be `50`
* `YAW_TO_EDGE_OF_FRAME_DEG` should be `3.25`
* `PITCH_TO_EDGE_OF_FRAME_DEG` should be `-2`

5. If you didn't install dependencies yet (i.e. you're using another Pi that our team never touched), you'll have to run the following:

```
pi@raspberrypi:~ $ pip install numpy
pi@raspberrypi:~ $ pip install opencv-contrib-python
```
* You NEED opencv-contrib, ideally 4.8.0. 


### How to actually use
In order to use this system in headless mode, you'll need another computer with the client software installed. If you haven't already, you also need to start the `pigpio` daemon on the Pi by using `sudo pigpiod`.

For more information about the client we wrote during the project, go to (PUT GH LINK HERE).

To just test it out:
1. Get another computer.
2. Run
```
C:\Users\C.Hu> git clone https://github.com/yearbook-milk/usgnerfwars-remotecontroller/ -b "usg-configuration"
```
3. We never released any OS specific executables for our client. Because of this, your other computer needs Python, as well as dependencies:
```
C:\Users\C.Hu> py -m pip install numpy
C:\Users\C.Hu> py -m pip install opencv-python
```
4. On the RPi, start `main.py` and wait until it says something along the lines of `[net] Listening on TCP port 42069`. This means that startup was successful and that the RPi will now accept incoming TCP connections. 
5. Start `controltrackpad.py` (the client python file), type `r` for remote mode, then enter the IP address and TCP port number of the Turret. 
6. The client will then start its GUI. 
* To control the servos manually, use the trackpad in the top left. If 
* To lock onto a target, look at their bounding box and note what # is next to the box.
* Hit that corresponding yellow button on the right to start tracking the target (if your target was highlighted in blue and it said `detection #1`, then hit the button labeled `1`. The servos should automatically start working to track them.
	* If you're only seeing one detection when there should be multiple, hit the LargestPolygonOnly button once.
* To rev and fire, use Rev and Fire Auto. Allow the blaster to rev up for 1-2 seconds before firing, and shut both off as soon as you're done. Do NOT let the pusher in the gun run for more than 3-4 seconds and do NOT rev the gun for more than 4-5 seconds.
	* Note that shutting the flywheel motors off also disables the pusher motor.
* To stop tracking, use Stop Tracking.
* To disconnect, use Graceful D/C.