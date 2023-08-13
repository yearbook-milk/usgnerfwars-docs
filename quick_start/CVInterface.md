# Writing your own detection and tracking algorithms / connecting pre-existing ones to this software
In this tutorial, we will write our own detection "algorithm", and have it be used to "detect" targets, as well as learn about how tracking algorithms are connected to the main program.


## Writing our own detection alg.

### Basics
Let's first start with detection: the point of a detector here is to look at a frame, and return an array containing bounding boxes of ALL potential targets in frame. In this case however, let's just generate a bunch of random detections without actually detecting anything in the image. If we put this code in a file named `random_detect.py`:

```
import cv2
from random import randint

def ri(f, c):
	return randint(f, c)

def _init(other_modules):
	# here, other_modules provides access to all other modules in the same
	# cwd as random_detect.py
	# this means you can do things like call helpers.line(), etc.
    print("Random target detection started!")

def _attempt_detection(image, filterdata):
	# every time there is a new frame, main.py will pass 
	# a frame as well as the contents of detectorfilterdata.txt as a dict
	# to this method
	potential_targets = []
	random_bbox = (ri(20, 100), ri(20,100), ri(20, 100), ri(20, 100))
	potential_targets.append(random_bbox)
	return (potential_targets, image)
	# we actually return a tuple that contains our array of bounding boxes
	# and our image for backwards compatibility reasons
	# that's why we don't just "return potential_targets"

# yes, the functions MUST be named _init() and _attempt_detection(), 
# and they MUST accept those arguments only 
```
### Enabling this new detector
If you then go to `detectorpipeline.txt` and set the contents to `[random_detect]`, then restart our `main.py`, you should see the cyan boxes spazzing about (because its not actually detecting anything, its just spitting out a random ROI as a potential target).

### Reading from a config file
Now let's say our pretend detection algorithm needs to pull detection parameters from a config file, in this case how many random boxes to spit out. We would first modify our config file by adding the section at the bottom (random_detect):

```
{
    "color_detect": {
        "colormasks": color_detect.colorMasksGenerator("red orange yellow"),
        "other_parameters": {
            "lower_hue_tolerance": 10,
            "upper_hue_tolerance": 10,
            "min_hsv_saturation": 10,
            "min_hsv_value": 150,
            "blur_level": 25,
            "minPolygonWidth": 70,
            "minPolygonHeight": 70,
            "maxPolygonWidth": 1200,
            "maxPolygonHeight": 2400,
        },
    },

    "aruco_marker_detect": {
        
    },

    "random_detect": {
        "generations": 15
    },

}
```


Then, we would modify our `random_detect.py` code like this:
```
from random import randint

# set a globalvar to hold a copy of the settings passed to _init()
numberOfRandomBoxes = 1

def ri(f, c):
    return randint(f, c)

def _init(other_modules): #here, other_modules provides access to all other modules in the same# cwd as random_detect.py# this means you can do things like call helpers.line(), etc.
    print("Random target detection started!")

    #let 's call helpers.file_get_contents(), and convert that into a Python dict, and then extract config data from there
    f = eval(other_modules["helpers"].file_get_contents("detectorfilterdata.txt"), other_modules)

    global numberOfRandomBoxes
    numberOfRandomBoxes = f["random_detect"]["generations"]
    print(f"Will now generate {numberOfRandomBoxes} random bounding boxes!")

def _attempt_detection(image, filterdata): #every time there is a new frame, main.py will pass# a frame as well as the contents of detectorfilterdata.txt as a dict# to this method
  potential_targets = []
  global numberOfRandomBoxes

  # actually generate X number of random bboxes
  for i in range(numberOfRandomBoxes):
      random_bbox = (ri(20, 100), ri(20, 100), ri(20, 100), ri(20, 100))
      potential_targets.append(random_bbox)
  return (potential_targets, image)
  
# yes, the functions MUST be named _init() and _attempt_detection(), #and they MUST accept those arguments only
```
We:
1. Added a global variable to store a setting,
2. Made init extract config data using `f = eval(other_modules["helpers"].file_get_contents("detectorfilterdata.txt"), other_modules)`
3. Made _attempt_detection actually generate that many bounding boxes with `for i in range(numberOfRandomBoxes):`


## Connecting tracking algs. to the main prog.
Now that we have successfully created and integrated our own "detection" algorithm, and used it to generate crap detections, we can now move onto seeing how the tracking algs. are connected to the main program. It works mostly the same way; see this already existing simple code that just works as a layer between OpenCV's already existing tracker_KCF and our `main.py`:

```
import cv2
import numpy as np

the_tracker = None

def _init(frame, ROI):
	# when main.py wants us to track something around, it will, on the 
	# time around, pass us the initial frame and the bounding box
	# that contains what we're supposed to follow
	
    global the_tracker
	
	# so, we just create a TrackerKCF object...
    the_tracker = cv2.TrackerKCF_create()

	# and tell it to follow whatever was in the boundaries of ROI
    the_tracker.init(frame, ROI)

	
    return None

def _update(frame):
	# every frame after the initial, main.py will then
	# pass us the next frame, and the tracking algorithm
	# is supposed to return a bounding box with the target in it
	# (or return (False, None) if it can't see the target anymore)
	
    success, box = the_tracker.update(frame)

	# yes, you do have to return a tuple in this format
    return (success, box)
```

Let's see how an alg. tracking a particular ID Aruco marker was implemented:

```
import cv2
import numpy as np
import helpers

# this variable keeps track of the selected aruco marker id 
# (so if other markers show up in frame, it will ignore them and only follow the one it was initially following
aruco_id = None

def _init(frame, ROI, dicti = cv2.aruco.DICT_4X4_250):
    # we are passed in the initiating frame and region of interest to track
    global aruco_id, aruco_detector
    dictionary = cv2.aruco.getPredefinedDictionary(dicti)
    parameters =  cv2.aruco.DetectorParameters()
    aruco_detector = cv2.aruco.ArucoDetector(dictionary, parameters)
    global aruco_id
    
    # read the initial frame passed by main.py and look in the ROI passed in
    newROI = helpers.resizeBox(ROI, 1.20)
    corners, ids, _ = aruco_detector.detectMarkers(frame)
    
    # note down what ID the marker is (so that way, we only track that marker and not
    # any others that may appear subsequently)
    if len(ids) > 0:
        aruco_id = ids[0][0]
        print(f"Just locked onto an aruco marker with ID {ids[0][0]}")
    
    
def _update(frame):
	# this is run every time main.py wants to know where the targeted Aruco marker is now 
    global aruco_id, aruco_detector
    frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    corners, ids, _ = aruco_detector.detectMarkers(frame)
    output = None
    
    # for every marker found in frame
    for i in range(0, len(corners)):
        # if it was the same one we were following
        if ids[i] == aruco_id:
            four_corners = corners[i][0]
            minX = 10000
            minY = 10000
            maxX = 0
            maxY = 0
            # figure out which of the corners returned is the top left 
            # we do this to convert from (tl, tr, bl, br) or whatever format to (x, y, w, h)
            for point in four_corners:
                if   point[0] > maxX: maxX = point[0]
                elif point[0] < minX: minX = point[0]
                if   point[1] > maxY: maxY = point[1]
                elif point[1] < minY: minY = point[1]

            return (True, (int(minX), int(minY), int(maxX - minX), int(maxY - minY)))
            # if success, return (True, (bbox))
        
    return (False, None)
    # if failure, return (False, None)
```            
            
To turn on Aruco marker track, all you would have to do is open `tracker.txt` and set it to `[aruco_marker_track]`.