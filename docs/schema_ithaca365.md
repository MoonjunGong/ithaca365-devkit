Ithaca365 schema
==========
This document describes the database schema used in Ithaca365.
All annotations and meta data (including calibration, maps, vehicle coordinates etc.) are covered in a relational database.
The database tables are listed below.
Every row can be identified by its unique primary key `token`.
Foreign keys such as `sample_token` may be used to link to the `token` of the table `sample`.
Please refer to the [tutorials] for an introduction to the most important database tables.

![](Ithaca365Schema.png)

attribute
---------
An attribute is a property of an instance that can change while the category remains the same.

```
attribute {
   "token":                   <str> -- Unique record identifier.
   "name":                    <str> -- Attribute name.
   "description":             <str> -- Attribute description.
}
```

calibrated_sensor
---------
Definition of a particular sensor (lidar/camera) as calibrated on a particular vehicle.
All extrinsic parameters are given with respect to the ego vehicle body frame.
All camera images come undistorted and rectified.
```
calibrated_sensor {
   "token":                   <str> -- Unique record identifier.
   "sensor_token":            <str> -- Foreign key pointing to the sensor type.
   "translation":             <float> [3] -- Coordinate system origin in meters: x, y, z.
   "rotation":                <float> [4] -- Coordinate system orientation as quaternion: w, x, y, z.
   "camera_intrinsic":        <float> [3, 3] -- Intrinsic camera calibration. Empty for sensors that are not cameras.
}
```

category
---------
Taxonomy of object categories (e.g. vehicle, pedestrian). 
```
category {
   "token":                   <str> -- Unique record identifier.
   "name":                    <str> -- Category name. Subcategories indicated by period.
   "description":             <str> -- Category description.
   "index":                   <int> -- The index of the label used for efficiency reasons in the .bin label files of Ithaca365-lidarseg. This field did not exist previously.
}
```

ego_pose
---------
Ego vehicle pose at a particular timestamp. Given with respect to global coordinate system of the log's map.
The esgcalization algorithm described in our paper.
The localization is 2-dimensional in the x-y plane.
```
ego_pose {
   "token":                   <str> -- Unique record identifier.
   "translation":             <float> [3] -- Coordinate system origin in meters: x, y, z. Note that z is always 0.
   "rotation":                <float> [4] -- Coordinate system orientation as quaternion: w, x, y, z.
   "timestamp":               <int> -- Unix time stamp.
}
```

instance
---------
An object instance, e.g. particular vehicle.
This table is an enumeration of all object instances we observed.
Note that instances are not tracked across scenes.
```
instance {
   "token":                   <str> -- Unique record identifier.
   "category_token":          <str> -- Foreign key pointing to the object category.
   "nbr_annotations":         <int> -- Number of annotations of this instance.
   "first_annotation_token":  <str> -- Foreign key. Points to the first annotation of this instance.
   "last_annotation_token":   <str> -- Foreign key. Points to the last annotation of this instance.
}
```
logs
---------
```
logs e record identifier.
   "logfile":                 <str> -- Log file name.
   "vehicle":                 <str> -- Vehicle name.
   "date_captured":           <str> -- Date (YYYY-MM-DD).
   "location":                <str> -- Area where log was captured, e.g. singapore-onenorth.
}
```
sample
---------
A sample is an annotated keyframe at 2 Hz.
The data is collected at (approximately) the same timestamp as part of a single LIDAR sweep.
```
sample {
   "token":                   <str> -- Unique record identifier.
   "timestamp":               <int> -- Unix time stamp.
   "scene_token":             <str> -- Foreign key pointing to the scene.
   "next":                    <str> -- Foreign key. Sample that follows this in time. Empty if end of scene.
   "prev":                    <str> -- Foreign key. Sample that precedes this in time. Empty if start of scene.
}
```

sample_annotation
---------
For LiDAR Data: A bounding box defining the position of an object seen in a sample.
All location data is given with respect to the global coordinate system.
```
sample_annotation {
   "token":                   <str> -- Unique record identifier.
   "sample_token":            <str> -- Foreign key. NOTE: this points to a sample NOT a sample_data since annotations are done on the sample level taking all relevant sample_data into account.
   "instance_token":          <str> -- Foreign key. Which object instance is this annotating. An instance can have multiple annotations over time.
   "attribute_tokens":        <str> [n] -- Foreign keys. List of attributes for this annotation. Attributes can change over time, so they belong here, not in the instance table.
   "visibility_token":        <str> -- Foreign key. Visibility may also change over time. If no visibility is annotated, the token is an empty string.
   "translation":             <float> [3] -- Bounding box location in meters as center_x, center_y, center_z.
   "size":                    <float> [3] -- Bounding box size in meters as width, length, height.
   "rotation":                <float> [4] -- Bounding box orientation as quaternion: w, x, y, z.
   "num_lidar_pts":           <int> -- Number of lidar points in this box. Points are counted during the lidar sweep identified with this sample.
   "num_radar_pts":           <int> -- Number of radar points in this box. Points are counted during the radar sweep identified with this sample. This number is summed across all radar sensors without any invalid point filtering.
   "next":                    <str> -- Foreign key. Sample annotation from the same object instance that follows this in time. Empty if this is the last annotation for this object.
   "prev":                    <str> -- Foreign key. Sample annotation from the same object instance that precedes this in time. Empty if this is the first annotation for this object.
}
```

object_ann
---------
The annotation of a foreground object (car, bike, pedestrian) in an image.
Each foreground object is annotated with a 2d box, a 2d instance mask and category-specific attributes.
```
object_ann {
    "token":                  <str> -- Unique record identifier.
    "sample_data_token":      <str> -- Foreign key pointing to the sample data, which must be a keyframe image.
    "category_token":         <str> -- Foreign key pointing to the object category.
    "attribute_tokens":       <str> [n] -- Foreign keys. List of attributes for this annotation.
    "bbox":                   <int> [4] -- Annotated amodal bounding box. Given as [xmin, ymin, xmax, ymax].
    "mask":                   <RLE> -- Run length encoding of instance mask using the pycocotools package.
}
```

surface_ann
---------
The amodal annotation of a background object (road) in an image.
Each background object is annotated with a 2d amodal semantic segmentation mask.
```
surface_ann {
   "token":                   <str> -- Unique record identifier.
    "sample_data_token":      <str> -- Foreign key pointing to the sample data, which must be a keyframe image.
    "category_token":         <str> -- Foreign key pointing to the surface category.
    "mask":                   <RLE> -- Run length encoding of segmentation mask using the pycocotools package.
}
```

sample_data
---------
A sensor data e.g. image, point cloud or radar return. 
For sample_data with is_key_frame=True, the time-stamps should be very close to the sample it points to.
For non key-frames the sample_data points to the sample that follows closest in time.
```
sample_data {
   "token":                   <str> -- Unique record identifier.
   "sample_token":            <str> -- Foreign key. Sample to which this sample_data is associated.
   "ego_pose_token":          <str> -- Foreign key.
   "calibrated_sensor_token": <str> -- Foreign key.
   "filename":                <str> -- Relative path to data-blob on disk.
   "fileformat":              <str> -- Data file format.
   "width":                   <int> -- If the sample data is an image, this is the image width in pixels.
   "height":                  <int> -- If the sample data is an image, this is the image height in pixels.
   "timestamp":               <int> -- Unix time stamp.
   "is_key_frame":            <bool> -- True if sample_data is part of key_frame, else False.
   "next":                    <str> -- Foreign key. Sample data from the same sensor that follows this in time. Empty if end of scene.
   "prev":                    <str> -- Foreign key. Sample data from the same sensor that precedes this in time. Empty if start of scene.
}
```

scene
---------
A scene is a traversal long sequence of consecutive frames extracted from a log.  
Note that object identities (instance tokens) are not preserved across scenes.
```
scene {
   "token":                   <str> -- Unique record identifier.
   "name":                    <str> -- Short string identifier.
   "description":             <str> -- Longer description of the scene.
   "log_token":               <str> -- Foreign key. Points to log from where the data was extracted.
   "nbr_samples":             <int> -- Number of samples in this scene.
   "first_sample_token":      <str> -- Foreign key. Points to the first sample in scene.
   "last_sample_token":       <str> -- Foreign key. Points to the last sample in scene.
}
```

sensor
---------
A specific sensor type.
```
sensor {
   "token":                   <str> -- Unique record identifier.
   "channel":                 <str> -- Sensor channel name.
   "modality":                <str> {camera, lidar, radar} -- Sensor modality. Supports category(ies) in brackets.
}
```

