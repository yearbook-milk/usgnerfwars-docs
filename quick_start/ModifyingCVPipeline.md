# Changing what CV algorithms are in use

There are 3 files in the /computervision directory that control the computer vision algorithms in use: `detectorfilterdata.txt`, `detectorpipeline.txt`, and `tracker.txt`.

If you look in the directory you pulled out of the .tar.gz we gave to Dr. Tiglao, or pulled off the usg-configuration branch off of Github, you'll see these .py files:

```powershell
C:\Users\C.Hu\usgnerfwars\computervision>dir
08/12/2023  01:46 PM    <DIR>          .
08/12/2023  01:46 PM    <DIR>          ..
08/11/2023  05:40 PM             1,570 aruco_marker_detect.py
08/11/2023  05:34 PM             2,016 aruco_marker_track.py
08/11/2023  05:38 PM             5,684 color_detect.py
08/11/2023  05:34 PM               352 cvbuiltin_csrt_tracker.py
08/11/2023  05:38 PM               862 cvbuiltin_csrt_tracker_75.py
08/11/2023  05:38 PM               296 cvbuiltin_kcf_tracker.py
08/11/2023  05:38 PM               804 cvbuiltin_kcf_tracker_75.py
08/12/2023  01:44 PM               544 detectorfilterdata.txt
08/11/2023  05:07 PM                23 detectorpipeline.txt
08/11/2023  05:30 PM             3,420 helpers.py
08/11/2023  05:08 PM                22 tracker.txt
```
Each of the `_detect` files contains code for finding and highlighting all potential targets on screen, and each of the `_tracker` files contains code that takes in a frame and predicts where an initially selected ROI has moved to.

To configure what *detection* algorithms are in use, open `detectorpipeline.txt`. You'll find something that looks like this:

```python
[color_detect]
```
Now, for instance, if you wanted to highlight all Aruco markers in frame as well as track a certain color, you would simply add the filename of the file you want, minus the file extension:

```python
[color_detect,aruco_marker_detect]
```
Then, on next the restart or when you apply settings, both Aruco markers and targets with the specified colors will show up as potential targets.

Some of the detectors (namely color_detect) need other config data. That is stored in a file named `detectorfilterdata.txt`, and it looks like this:

```python
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
        
    }

}
```

Note that this is in Python's dictionary syntax. 

Lastly, to switch which tracker algorithm you use, open `tracker.txt`:
```python
[cvbuiltin_KCF_tracker]
```
If you want to exclusively track Aruco markers, you can do this:
```python
[aruco_marker_track]
```
Note that it is not advised to run multiple tracking algorithms at the same time (i.e. `[cvbuiltin_KCF_track,cvbuiltin_CSRT_track]`; as that can severely limit framerate.

See `CVInterface.md` for information about programming in your own detection and tracking algorithms. 