# Networking Protocol Specification

### Overview
The Nerf Turret's onboard software uses a custom networking protocol built on top of TCP/IP and UDP. 

To expose the networking API and thus allow `main.py` to access it, the setting `enable_networking` in `config.py` on the server must be set to `True`. 


### Connection Lifecycle
1. The connection lifecycle begins when the Nerf Turret's or other device's onboard computer starts listening on a TCP port for incoming connections (which will block execution of other code). 

2. Once both sides have established a TCP connection, the client should then start listening for a single TCP packet. This is because the server is about to send a string representation of the port number the UDP channel should be made over (the payload will literally be the ASCII bytes that represent a number, such as `b"10007"`). 

3. The client, once it receives this number, should either close the connection if it cannot listen over the proposed UDP port; or save this number for later. The server should do the same and have that number saved somewhere.

4. The initialization phase has ended, and the two peers can now start exchanging data and listening to each other. Both the client and the server (the Nerf Turret) should be running main loops (the client runs a main loop to update the GUI, and the server runs a main loop to pull frames from a webcam and run computer vision code). At some point in such a main loop, each side should pull new packets from the buffer for both UDP and TCP.
	* Note that reading from buffer should be a non-blocking operation, that is, the program will not block if there is no data to be received. Both the server and client have other things to be doing and thus cannot hang waiting for a packet to come in (for instance, the server still needs to be pulling in frames from webcam and analyzing them while there are no packets coming from the client.
	* Writing over UDP should also not block, although TCP might have to block in order to ensure reliable delivery of the packet. 

### Packet Contents
Server to Client:
* Every time the server runs image data through the computer vision pipeline, it will transmit the image, along with some other encoded elements and metadata, to the client's IP address, and it will come in through the UDP socket on the correct port on the client.
	* The parts of the packet are divided by the four bytes representing `::::`, and are as follows:
		* Python-pickled image data (represented as a numpy array) 
		* Python-pickled text data (represented as an array of arrays)
			* Each array is in the format `[int x, int y, tuple color, float scale, string content]`. It is then up to the client to decide whether or not to draw this text on the image it just received, or even to simply not display the image at all.
		* Python-pickled image shape (represented as a tuple with 3 elements, in OpenCV's (height, width, channelno) dimensions tuple format)
	* To parse, split the binary data with the delimiter `b"::::"` , and pass each part into the `pickle.loads()` (if you're not using a language that has a library for working with the Python serialization protocol, best of luck, because we didn't think about this before Demo Day).
* The server, within 3 seconds of forming a good connection, will, over TCP, send a payload with the bytes representing the string `casconfig 8080` (with 8080 being replaced by the Java HTTP-based Reconfig Server's port number).  See that section for more details.

### Client to Server
Packets sent from the client to the server (the Nerf Turret or other device) are mostly commands such as "rev", "turn servo" and "choose target". These commands can be sent over UDP or TCP; as the server will be listening for these commands on both the TCP and UDP sockets. Usually, important control commands like fire, rev, and stop/start tracking are sent over TCP to ensure reliable delivery, whereas commands such as servo position are fired and forgotten over UDP to ensure quick delivery (and because the client often constantly has a servo command to send, there's always more of those packets where they came from).

The list of all commands that the client can send to the server over either socket are as follows: (to send these commands you send the bytes representing the literal words over):
* `abspitch 90` (absolute pitch): send a pitch command to the server, which the remote should act on by turning the pitch servos (assuming that it can do so while also not exceeding the servo pitch limits set forth in `config.py`. The angles run from -90 to 90. If it receives a pitch command that it cannot act on, it will simply ignore the command.
	* Note that this sends the 90 deg pitch command to the servo, and does not make the pitch go up by 90. It does not "add 90", it "sets it to 90".
* `absyaw -90` (absolute yaw): does the same thing that `abspitch` does, but for the yaw servo.
* `toggle_lpo` (toggle LargestPolygonOnly): **toggles** LPO to the state that it is not in. If turned on, detection (cyan boxes) will only present the largest target by area to the user as a valid target, when off, the detection will present all detections to the user as valid targets.
* `select 2` (select target 2): if you have three targets on screen labeled 0, 1 and 2, and you want to start tracking and following the target with a 2 next to their bounding box, you would send this packet over to tell the server that it should extract key features from and start tracking target #2. Replace 2 with the target numbet. 
* `forget` (forget target): to stop tracking a target and return to scanning and highlighting all potential targets, send this packet over.
* `stop` (disconnect): tells the server to close the sockets, file handles, access to webcam, etc, then stop. 
* `dtoggle rev` and `dtoggle fire` (digital toggle rev, fire): send this to tell the Nerf Turret or other device to **toggle** the GPIO pins set as the "rev" and "fire" relays

#### Notes:
* If the server is unable to execute a command that the client sends it, it should simply ignore it. 
* There is currently no specified reconnection logic for either the client or the server. 
* A default wrapper implementing this protocol is documented in (LINK HERE) and is in use here and here (LINKS HERE)