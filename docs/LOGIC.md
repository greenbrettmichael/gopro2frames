# How it works

The following describes how the script works at each step.

![](/docs/gopro-frame-maker-video-flow.jpg)

[Image source here](https://docs.google.com/drawings/d/1i6givGQnGsu7dW2fLt3qVSWaHDiP0TCciY_DtY5_mc4/edit)

![](/docs/gopro-frame-maker-image-flow.jpg)

[Image source here](https://docs.google.com/drawings/d/1_bgjyKFX_20Ub2uJpnhWTua8xdOuvgiJ0DsocW5Q988/edit?usp=sharing)

# Video input

## 1. User defines video and script settings

Users sets desired settings and input held in config.ini

## 2 Validate mode vs video metadata

In order to process the video in the correct flow, the following logic is checked against mode input.

**Validate GoPro EAC .360 (mode=`video`+`eac`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=428268485

If all validations pass, send to MAX2Sphere frame pipeline.

**Validate dual GoPro Fisheye (mode=`video`+`fisheye`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=2125124048

If all validations pass, send to Fusion2Sphere frame pipeline.

**Validate equirectangular mp4 (mode=`video`+`equirectangular`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=967778760

If all validations pass, send to equirectangular frame pipeline.

**Validate HERO mp4 (mode=`video`+`hero`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=0

If all validations pass, send to HERO frame pipeline.

## 3. Extract metadata from video

Done using exiftool:

```shell
$ exiftool -ee -G3 -api LargeFileSupport=1 -X IN_VIDEO.mp4 > VIDEO_META.xml
```

Note, if dual fisheyes, uses the front image (`GBFR`) for extraction.

## 4. Calculate un-timed GPS points

GoPro reports telemetry at different time intervals, and not every GPS position recorded has a time.

* ~ 18 Hz (up to 18 measurements per second) GPS position (lat/lon/alt/spd)
* 1 Hz (1 measurements per second, but not exactly 1 second apart) GPS timestamps
* 1 Hz (always at same interval as GPS timestamps) GPS accuracy (cm) and fix (2d/3d)

[You can see examples of these values in real GoPro telemetry here](https://github.com/trek-view/gopro-metadata).

Therefore up to 18 points have no time between GPS timestamps being reported. The number of points reported between times also varies based on mode/GPS signal (when GPS signal low, this number might be a lot lower).

To do this we calculate how many points there are between each time interval. Based on the number of points, we then calculate time.

Before doing this we first remove any duplicate GPS points (lat+lon+alt).

e.g.

```xml
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 50&#39; 45.50&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 50&#39; 45.50&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 50&#39; 45.50&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 60&#39; 45.62&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
```

In this case, only these two points would be considered.

```xml
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 50&#39; 45.50&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
<Track4:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track4:GPSLatitude>
<Track4:GPSLongitude>0 deg 60&#39; 45.62&quot; W</Track4:GPSLongitude>
<Track4:GPSAltitude>81.907 m</Track4:GPSAltitude>
```

For example, if `GPSDateTime` 1 = 1:00:00.000 and `GPSDateTime` 2 = 1:00:01.000 and there are 9 (non-dupe) points reported between these two time, we know each point is about .100 second apart so times for points will be 1:00:00.000, 1:00:00.100. 1:00:00.200, ... 1:00:00.900.


**A note on final points reported in GPMF**

At the end of the GoPro telemetry there is not a final time. That is, some GPS points continue beyond final reported time (usually 3 or 4).

e.g here we have 3 points following the final time

```xml
 <Track3:GPSMeasureMode>3-Dimensional Measurement</Track3:GPSMeasureMode>
 <Track3:GPSDateTime>2021:09:04 07:23:17.199</Track3:GPSDateTime>
 <Track3:GPSHPositioningError>1.49</Track3:GPSHPositioningError>
 <Track3:GPSLatitude>51 deg 16&#39; 21.18&quot; N</Track3:GPSLatitude>
 <Track3:GPSLongitude>0 deg 50&#39; 45.54&quot; W</Track3:GPSLongitude>
 <Track3:GPSAltitude>80.771 m</Track3:GPSAltitude>
 <Track3:GPSSpeed>1.152</Track3:GPSSpeed>
 <Track3:GPSSpeed3D>1.18</Track3:GPSSpeed3D>
 <Track3:GPSLatitude>51 deg 16&#39; 21.18&quot; N</Track3:GPSLatitude>
 <Track3:GPSLongitude>0 deg 50&#39; 45.54&quot; W</Track3:GPSLongitude>
 <Track3:GPSAltitude>80.765 m</Track3:GPSAltitude>
 <Track3:GPSSpeed>1.131</Track3:GPSSpeed>
 <Track3:GPSSpeed3D>1.15</Track3:GPSSpeed3D>
 <Track3:GPSLatitude>51 deg 16&#39; 21.18&quot; N</Track3:GPSLatitude>
 <Track3:GPSLongitude>0 deg 50&#39; 45.54&quot; W</Track3:GPSLongitude>
 <Track3:GPSAltitude>80.744 m</Track3:GPSAltitude>
 <Track3:GPSSpeed>1.099</Track3:GPSSpeed>
 <Track3:GPSSpeed3D>1.13</Track3:GPSSpeed3D>
 <Track3:GPSLatitude>51 deg 16&#39; 21.17&quot; N</Track3:GPSLatitude>
 <Track3:GPSLongitude>0 deg 50&#39; 45.54&quot; W</Track3:GPSLongitude>
 <Track3:GPSAltitude>80.748 m</Track3:GPSAltitude>
 <Track3:GPSSpeed>1.11</Track3:GPSSpeed>
 <Track3:GPSSpeed3D>1.1</Track3:GPSSpeed3D>
 <Track3:TimeStamp>20.204954</Track3:TimeStamp>
 <Track3:TimeStamp>20.204954</Track3:TimeStamp>
 <Track3:TimeStamp>20.204954</Track3:TimeStamp>
 <Track3:TimeStamp>20.204954</Track3:TimeStamp>
 <Track3:TimeStamp>20.18495</Track3:TimeStamp>
 <Track3:TimeStamp>20.18495</Track3:TimeStamp>
 <Track3:TimeStamp>20.18495</Track3:TimeStamp>
 <Composite:ImageSize>5376x2688</Composite:ImageSize>
 <Composite:Megapixels>14.5</Composite:Megapixels>
 <Composite:AvgBitrate>117 Mbps</Composite:AvgBitrate>
 <Composite:Rotation>0</Composite:Rotation>
 <Composite:GPSPosition>51 deg 16&#39; 21.21&quot; N, 0 deg 50&#39; 45.54&quot; W</Composite:GPSPosition>
</rdf:Description>
</rdf:RDF>
```

[Full output](https://drive.google.com/file/d/1HGwhK4SH_Wl1eN_mBj1PMFlI8cescto_/view?usp=sharing).

In this case, we can calculate the final time as follows ([I will use examples from the linked file](https://drive.google.com/file/d/1HGwhK4SH_Wl1eN_mBj1PMFlI8cescto_/view?usp=sharing)):

1. Subtract the last reported `GPSDateTime` (end time) from the first reported `GPSDateTime` (start time)
    * `2021:09:04 07:23:17.199` - `2021:09:04 07:22:56.299` = `20.9`s
2. Subtract the video `duration` metadata value from this:
    * `21.2` - `20.9` = `0.3`s 
3. This will give you remaining seconds of video. Add this value to the final reported `GPSDateTime` to create the final time.
    * `2021:09:04 07:23:17.199` + `0.3` = `07:23:17.499`
4. It is then possible to calculate points in-between in exactly the same way as before.

[See also calculations in excel formulae here](https://docs.google.com/spreadsheets/d/1fj-xZsuu7kk11rgTuyeWlvlo7Psiq_Gz/edit?usp=sharing&ouid=111552983205460555579&rtpof=true&sd=true)

## 5. Create video GPX

A GPX file can then be created using all time, elevation (altitude), latitude, and longitude calculated earlier

```xml
<trkseg>
    <trkpt lat="51.27254444444444" lon="-0.8459694444444444">
        <ele>82.008</ele>
        <time>2021-09-04T07:24:07.744000Z</time>
    </trkpt>
    <trkpt...
      ...
</trkseg>

```

## 6. Extract frames at set rate for video type

User can set the ffmpeg `-r` value. Depending on the mode of the video (set by user), we extract like so:

**GoPro EAC .360**

```shell
$ ffmpeg -i INPUT.360 -map 0:0 -r XXX -q:v QQQ trackN/img%d.jpg -map 0:5 -r XXX -q:v QQQ trackN/img%d.jpg
```

Where `XXX` = framerate user passes in CLI. And `QQQ` = quality.

Note: if timelapse mode is used, the track numbers are different:

* Regular video = `-map 0:0` and `-map 0:5`
* Timewarp video = `-map 0:0` and `-map 0:4`

**Dual GoPro Fisheye**

```shell
$ ffmpeg -i INPUT_FR.mp4 -r XXX -q:v QQQ FR/img%d.jpg 
$ ffmpeg -i INPUT_BK.mp4 -r XXX -q:v QQQ BK/img%d.jpg 
```

Where `XXX` = framerate user passes in CLI. And `QQQ` = quality.

**Equirectangular / HERO mp4**

```shell
$ ffmpeg -i INPUT.mp4 -r XXX -q:v QQQ img%d.jpg 
```

## 7. Process to equirectangular for EAC / Fisheyes

Skip step 7 if Equirectangular / HERO mp4 input

## 7A. MAX2Sphere flow (2 GoPro EAX Frames)

**Merge 2 tracks of frames into one**

```shell
$ @SYSTEM_PATH/MAX2spherebatch -w XXXX -n 1 -m YYYY track%d/frame%4d.jpg
```

Note, -w flag variable (`XXXX`), if in XML ImageWidth is:

* 4096, then -w = 4096
* 2272, then -w = 2272

For -m flag variable `YYYY` is equal to number of frames extracted.


## 7B. Fusion2Sphere flow (2 GoPro Fisheye Frames)

```shell
@SYSTEM_PATH/fusion2sphere -b 5 -f FR/img%d.jpg BK/img%d.jpg -o FINAL/img%d.jpg parameter-examples/PPPP.txt
```

Note, `PPPP` variable is determined by frame ImageWidth:

* 3104, then `PPPP` = photo-mode.txt
* 1568, then `PPPP` = video-3k-mode.txt
* 2704, then `PPPP` = video-5_2k-mode.txt

## 8. Add logo

Skip if option not selected by user.

User should be able to pass flag `-n` at upload with path to a logo file.

Logo file must be:

* .png filetype
* square dimensions
* >= 500 px height

### 8B. Add watermark (HERO)

User can set watermark overlay size and logo.

To create the watermark, these steps can be followed:

https://www.trekview.org/blog/2022/adding-a-custom-watermark-to-hero-photo-video/

_Example of watermark image height (non-equirectangular)_

![](/docs/example-watermark-percentage-of-photo.jpeg)

## 9. Insert equirectangular metadata to frames (equirectangular only)

Skip step 7 if HERO mp4 input

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Value type</th><th>Image metadata field injected</th><th>Example injected</th></tr></thead><tbody>
 <tr><td>Fixed</td><td>XMP-GPano:StitchingSoftware</td><td>Spherical Metadata Tool</td></tr>
 <tr><td>Fixed</td><td>XMP-GPano:SourcePhotosCount</td><td>2</td></tr>
 <tr><td>Fixed</td><td>XMP-GPano:UsePanoramaViewer</td><td>TRUE</td></tr>
 <tr><td>Fixed</td><td>XMP-GPano:ProjectionType</td><td>equirectangular</td></tr>
 <tr><td>Is same as ImageHeight value</td><td>XMP-GPano:CroppedAreaImageHeightPixels</td><td>2688</td></tr>
 <tr><td>Is same as ImageWidth value</td><td>XMP-GPano:CroppedAreaImageWidthPixels</td><td>5376</td></tr>
 <tr><td>Is same as ImageHeight value</td><td>XMP-GPano:FullPanoHeightPixels</td><td>2688</td></tr>
 <tr><td>Is same as ImageWidth value</td><td>XMP-GPano:FullPanoWidthPixels</td><td>5376</td></tr>
 <tr><td>Fixed</td><td>XMP-GPano:CroppedAreaLeftPixels</td><td>0</td></tr>
 <tr><td>Fixed</td><td>XMP-GPano:CroppedAreaTopPixels</td><td>0</td></tr>
</tbody></table>

Note, some spatial fields are always fixed (e.g. XMP-GPano:SourcePhotosCount b/c GoPro 360 cameras only have 2 lenses), so values are static.

## 10. Add photo times and add GPS data

### 10A Add photo times

#### First frame (all modes)

To assign first photo time, we use the first GPSDateTime value reported in telemetry and assign it to photo time fields as follows:

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Video metadata field extracted</th><th>Example extracted</th><th>Image metadata field injected</th><th>Example injected</th></tr></thead><tbody>
 <tr><td>TrackN:GPSDateTime</td><td>2020:04:13 15:37:22.444</td><td>DateTimeOriginal</td><td>2020:04:13 15:37:22Z</td></tr>
 <tr><td>TrackN:GPSDateTime</td><td>2020:04:13 15:37:22.444</td><td>SubSecTimeOriginal</td><td>444</td></tr>
 <tr><td>TrackN:GPSDateTime</td><td>2020:04:13 15:37:22.444</td><td>SubSecDateTimeOriginal</td><td>2020:04:13 15:37:22.444Z</td></tr>
</tbody></table>

Example exiftool command to write these values:

```shell
$ exiftool DateTimeOriginal:"2020:04:13 15:37:22Z" SubSecTimeOriginal:"444" SubSecDateTimeOriginal: "2020:04:13 15:37:22.444Z"
```

#### Other frames (normal mode)

Now we need to assign time to other photos. To do this we simply order the photos in ascending numerical order (as we number them sequentially when extracting frames).

We always extract videos at a fixed frame rate based on transport type. Therefore, we really only need to know the video start time, to determine time of first photo. From there we can incrementally add time based on extraction rate (e.g. photo 2 is 0.2 seconds later than photo one where framerate is set at extraction as 5 FPS).

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Extraction Frame rate</th><th>Photo spacing (sec)</th></tr></thead><tbody>
 <tr><td>1</td><td>1</td></tr>
 <tr><td>2</td><td>0.5</td></tr>
 <tr><td>5</td><td>0.2</td></tr>
</tbody></table>

#### Other frames (timewarp mode)

Timewarp is a GoPro mode that speeds up the video (e.g. when set a 5x every second of video is 5 seconds of footage).

We therefore explicitly ask use if video was shot in timewarp mode and the settings used (there is no easy way to determine this automatically).

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Timewarp mode</th><th>Each photo true time (sec) @1 FPS</th><th>Each photo true time (sec) @2 FPS</th><th>Each photo true time (sec) @5 FPS</th></tr></thead><tbody>
 <tr><td>2x</td><td>2</td><td>1</td><td>0.4</td></tr>
 <tr><td>5x</td><td>5</td><td>2.5</td><td>1</td></tr>
 <tr><td>10x</td><td>10</td><td>5</td><td>2</td></tr>
 <tr><td>15x</td><td>15</td><td>7.5</td><td>3</td></tr>
 <tr><td>30x</td><td>30</td><td>15</td><td>6</td></tr>
</tbody></table>

To give an example, lets say first photo gets assigned first GPS time = 00:00:01.000 and we extract photos at 5FPS for timewarp mode 30x. in this case second photo has time 00:00:01.000 +6 secs.

### 10B Add GPS points

Now we can use the photo time and GPS positions / times to geotag the photos:

#### All frames (all modes)

[We can use Exiftool's geotag function to add GPS data (latitude, longitude, altitude)](https://exiftool.org/geotag.html).

```shell
exiftool -Geotag file.xml "-Geotime<SubSecDateTimeOriginal" dir

```

This will write the following fields into the photos

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Image metadata field injected</th><th>Example injected</th></tr></thead><tbody>
 <tr><td>GPS:GPSDateStamp</td><td>2020:04:13</td></tr>
 <tr><td>GPS:GPSTimeStamp</td><td>15:37:22.444</td></tr>
 <tr><td>GPS:GPSLatitude</td><td>51 deg 14' 54.51"</td></tr>
 <tr><td>GPS:GPSLatitudeRef</td><td>North</td></tr>
 <tr><td>GPS:GPSLongitude</td><td>16 deg 33' 55.60"</td></tr>
 <tr><td>GPS:GPSLongitudeRef</td><td>West</td></tr>
 <tr><td>GPS:GPSAltitudeRef</td><td>Above Sea Level</td></tr>
 <tr><td>GPS:GPSAltitude</td><td>157.641 m</td></tr>
</tbody></table>

## 11. Insert final metadata to frames

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Video metadata field extracted</th><th>Example extracted</th><th>Image metadata field injected</th><th>Example injected</th></tr></thead><tbody>
 <tr><td>Trackn:DeviceName</td><td>GoPro Max</td><td>IFD0:Model</td><td>GoPro Max</td></tr>
</tbody></table>

## 12. Create photo GPX

Same as step 5, but now only using GPS points added to photos (not all video points added to photos as more video gps than photos).

## 13. Create sequence json

The sequence JSON holds information about each photo, and its relationship to other photos.

To create this, the following entries are calculated and added in the gpx file:

<table class="tableizer-table">
<thead><tr class="tableizer-firstrow"><th>Name</th><th>Field in telemtry</th><th>Value</th><th>Image metadata (or fixed value)</th></tr></thead><tbody>
 <tr><td>GPS Epoch</td><td>`gps_epoch`</td><td>seconds</td><td>time converted to epoch</td></tr>
 <tr><td>GPS Fix Type</td><td>`gps_fix_typ`e</td><td> </td><td>Either 2 or 3 (reported in GoPro GPSMeasureMode field as NDimensional Measurement)</td></tr>
 <tr><td>Vertical Accuracy</td><td>`gps_vertical_accuracy`</td><td>meters</td><td>reported in GoPro GPSHPositioningError</td></tr>
 <tr><td>Horizontal Accuracy</td><td>`gps_horizontal_accuracy`</td><td>meters</td><td>reported in GoPro GPSHPositioningError</td></tr>
 <tr><td>Velocity East</td><td>`gps_velocity_east_next`</td><td>meters/second</td><td>Calculated using GPS lat,lon position/time between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Velocity North</td><td>`gps_velocity_north_next`</td><td>meters/second</td><td>Calculated using GPS lat,lon position/time between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Velocity Up</td><td>`gps_velocity_up_next`</td><td>meters/second</td><td>Calculated using GPS alt position/time between this and next photo. For last position, is always 0.</td></tr>
 <tr><td>Speed Accuracy</td><td>`gps_speed_accuracy`</td><td>meters/second</td><td>Always '0.1' (fixed)</td></tr>
 <tr><td>Speed</td><td>`gps_speed_next`</td><td>meters/second</td><td>Calculated using GPS lat,lon position/time between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Azimuth (heading)</td><td>`gps_azimuth_next`</td><td>degrees</td><td>Calculated using GPS lat,lon position between this an next photo. For last position, is always 0.</td></tr>
  <tr><td>Pitch</td><td>`gps_pitch_next`</td><td>degrees</td><td>Calculated using GPS alt position between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Distance Meters</td><td>`gps_distance_next`</td><td>meters</td><td>Calculated using GPS lat,lon position between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Distance Time</td><td>`gps_time_next`</td><td>seconds</td><td>Calculated using GPS lat,lon position between this an next photo. For last position, is always 0.</td></tr>
 <tr><td>Elevation change</td><td>`gps_elevation_change_next`</td><td>meters</td><td>Calculated using GPS elevation position between this an next photo. For last position, is always 0.</td></tr>
</tbody></table>

#### gps_time_seconds_next

Is time to next photo.

Uses GPSDateTime values (to subsecond resolution) from each photo. Calculation is:

`Photo Next GPSDateTime - Photo Current GPSDateTime`

Output in seconds

#### gps_distance_meters_next

Is distance to next photo lat/lon.

To calculate distance, you can use the [Haversine formula](https://en.wikipedia.org/wiki/Haversine_formula) to find the distance between two points on a sphere (Earth) given their longitudes and latitudes.

Output is meters

![](/docs/distance.png)

#### gps_elevation_change_meters_next

Is elevation change to next photo.

Uses elevation values from each photo. Calculation is:

`Photo Next Elevation - Photo Elevation`

Output in meters.

#### gps_pitch_degrees_next

Is pitch (or angle) to next photo.

pitch(θ) = tan(θ) = opposite / adjacent

We have the adjacent measurement (gps_distance_meters_next), and the opposite value (gps_elevation_change_meters_next).

Occasionally GPS lat/lon in points is identical, but elevation might change. In this case we default to write pitch as 0. This only happens if distance = 0.

Output is in degrees between -90 and 90

![](/docs/pitch.png)

#### gps_speed_meters_second_next

Is speed to next photo. 

Speed = distance / time

We have both these values from above (`gps_distance_meters_next` and `gps_time_seconds_next`)

Output is meters per second.

#### gps_velocity_east_meters_second_next

Is velocity (east vector) to next photo.

`Velocity = Displacement / Time in a direction`

So `Velocity East = Distance (along latitude) / Time (between source and destination)`

Or using the diagram above `Velocity East = Distance (from A to C) / Time (from A to B)`

Note, this calculation can result in a negative output. North and East are positive directions. If you travel West/South, in terms of an East/North vector, you will be traveling in both a negative East/North velocity.

If I drive from home to work (defining my positive direction), then my velocity is positive if I go to work, but negative when I go home from work. It is all about direction seen from how I defined my positive axis.

![](/docs/velocity-east-north.jpeg)

![](/docs/velocity-east-north-negative-example.jpeg)

Output is meters per second (can be negative)

#### gps_velocity_north_meters_second_next

Is velocity (north vector) to next photo.

Same as above, but...

`Velocity North = Distance from C to B) / Time (from A to B)`

Output is meters per second (can be negative)

#### gps_velocity_up_meters_second_next

Is velocity (up/vertical) to next photo.

Same as above, but...

`Velocity Up = Elevation change (from A to B) / Time (from A to B)`

Output is meters per second (can be negative)

#### gps_heading_degrees_next

Is heading (from North) to next photo.

![](/docs/heading.png)

Output is degrees between 0-360.

### Other sequence json data

Other values included in the json include

* `sequence.uuid`: UUID of the sequence
* `sequence.distance_km`: total distance using all positive distance_mtrs connection values
* `sequence.earliest_time`: earliest time GPSDateTime (if connection method gps / filename) or originalDateTime (if connection originalDateTime) value for photo in sequence
* `sequence.latest_time`: atest time GPSDateTime (if connection method gps / filename) or originalDateTime (if connection originalDateTime) value for photo in sequence
* `sequence.duration_sec`: `sequence.latest_time` - `sequence.earliest_time`
* `sequence.average_speed_kmh`: `sequence_distance_km` / `sequence_duration_sec`
* `sequence.camera_make`:
* `sequence.camera_model`:

The sequence JSON has the following structure:

```json
{
    "sequence" : {
        "uuid": "",
        "distance_km": "",
        "earliest_time": "",
        "latest_time": "",
        "duration_sec": "",
        "average_speed_kmh": "",
        "camera_make": "",
        "camera_model": ""
    },
    "photo" : [
        {
            "uuid": "",
            "gps_latitude": "",
            "gps_longitude": "",
            "gps_epoch": "",
            "gps_fix_type": "",
            "gps_vertical_accuracy": "",
            "gps_horizontal_accuracy": "",
            "gps_velocity_east_next": "", 
            "gps_velocity_north_next": "",
            "gps_velocity_up_next": "",
            "gps_speed_accuracy": "",
            "gps_speed_next": "",
            "gps_pitch_next": "",
            "gps_distance_next": "",
            "gps_time_next": "",
            "gps_elevation_change_next": "",
        },
        {
            "uuid":
            ....
        }
```

### Step 14: Done

You now have:

* a set of geotagged .jpg photos in a directory (`SEQUENCENAME_DATETIME/`)
* a video gpx in the same directory (`SEQUENCENAME_video.gpx`)
* a photo gpx (step 6) in the same directory (`SEQUENCENAME_frames.gpx`)
* a sequence json (`SEQUENCENAME_sequence.json`)

# Image input

## 1. User defines image and script settings

Users sets desired settings and input.

## 2 Validate mode vs photo metadata

In order to process the photo in the correct flow, the following logic is checked against mode input.

**Validate equirectangular jpg (mode=`photo`+`equirectangular`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=459687648

If all validations pass, send to equirectangular frame pipeline.

**Validate HERO jpg (mode=`photo`+`hero`)**

See validations: https://docs.google.com/spreadsheets/d/1x9WnLwPHZIKRyfKkCW2ORVprpsAlA3c2hqVu4S9yIaY/edit#gid=445373652

If all validations pass, send to HERO frame pipeline.

## 3. Extract metadata from photo

Done using exiftool:

```shell
$ exiftool -ee -G3 -X IN_PHOTO.jpg > PHOTO_META.xml
```

## 4. Create photo GPX

A GPX file can then be created using all time, elevation (altitude), latitude.


```xml
<trkseg>
    <trkpt lat="51.27254444444444" lon="-0.8459694444444444">
        <ele>82.008</ele>
        <time>2021-09-04T07:24:07.744000Z</time>
    </trkpt>
    <trkpt...
      ...
</trkseg>

```

## 5 Create sequence JSON

The sequence JSON holds information about each photo, and its relationship to other photos.

(see video flow for how sequence.json is structured)

## 6. Add logo

Skip if option not selected by user.

(see video flow for how logos are added to photos)

### Step 7: Done

You now have:

* a set of geotagged .jpg photos in a directory (`SEQUENCENAME_DATETIME/`)
* a photo gpx in the same directory (`SEQUENCENAME_frames.gpx`)
* a sequence json (`SEQUENCENAME_sequence.json`)