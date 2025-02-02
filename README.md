# Introduction 

This repository contains a .tif file (computer vision) model executer which means that it iterates and implements a (computer vision) model on every pixel and/or extracted image kernel in a .tif file.
As long as a model has a .predict function in python any model can be used in this executer from simple models to deep learning models.

For the models that use image processing kernels, code functionality is written here which should makes kernel extraction from .tif files easy to do. 

For more on what image processing kernels are: [Here](https://en.wikipedia.org/wiki/Kernel_(image_processing))

The iterative loop that loops over every pixel and/or extracted image kernel in a .tif, because of the long processing time, is done in a multi processing loop.
Which means that it can't be run in a (jupyter) notebook interface, it has to be run from a terminal and it freezes your computer.

In addition this repository contains all the models  which we have used for the remote sensing project by the Province of South Holland.

Through out this document we will be referring to image object detection and image segmentation as segmentation.

# Dependencies.
If you are a Windows user you have to install the dependencies via wheels. The wheels for the following dependencies should be downloaded from https://www.lfd.uci.edu/~gohlke/pythonlibs/:

- [![GDAL>=3.0.4 ](https://img.shields.io/badge/GDAL-%3E%3D3.0.4-blue)](https://gdal.org/)
- [![Fiona>=1.8.13 ](https://img.shields.io/badge/Fiona-%3E%3D1.8.13-green)](https://pypi.org/project/Fiona/)
- [![rasterio>=1.1.3 ](https://img.shields.io/badge/rasterio-%3E%3D1.1.3-blue)](https://rasterio.readthedocs.io/en/latest/)
- [![Shapely>=1.7.0 ](https://img.shields.io/badge/Shapely-%3E%3D1.7.0-green)](https://shapely.readthedocs.io/en/stable/manual.html)
- [![geopandas>=0.9.0](https://img.shields.io/badge/geopandas-%3E%3D0.9.0-blue)](https://geopandas.org/en/stable/)
- [![scikit-learn==1.0.2](https://img.shields.io/badge/scikit--learn-%3D%3D1.0.2-brightgreen)](https://scikit-learn.org/stable/)

These should be installed in de following order: first GDAL, then Fiona and then rasterio. After these you can install the rest.

Download the wheels according to your system settings. For instance, wheel rasterio 1.2.10 cp39 cp39 win_amd64.whl is used with the 64-bit version of Windows and a 3.9 version of python. Install the wheel with pip install XXX.XX.XX.whl.

Or else check out this stack overflow post:
https://gis.stackexchange.com/questions/2276/installing-gdal-with-python-on-windows 


# Model input.
 
The following figure represents the input we have for a model:

![Alt text](basic_model_input.png?raw=true "Title")

This input data is generated in .tif files which is done in the other satellite images nso github repository [Here](https://github.com/Provincie-Zuid-Holland/satellite_images_nso/)

And for the height data here: [Here]( https://github.com/Provincie-Zuid-Holland/vdwh_ahn_processing )

# (Image Processing) Kernels.
The main functionality of this repository is to extract image kernels and the multiprocessing for loop for looping over all the pixels and/or image kernels in a given satellite .tif file to make predictions on them.

The following picture gives a illustration about how this extracting  of kernels is done:
![Alt text](kernel_extract.png?raw=true "Title")


Here below we will have a code example about how this work. In this example we will use a Euclidean distance model to segment all the pixels in a .tif file into segments that are specified in a annotations file.

```python

import nso_ds_classes.nso_tif_kernel_iterator as nso_tif_kernel_iterator
import nso_ds_classes.nso_ds_models as nso_ds_models
path_to_tif_file = "<PATH_TO_NSO_SAT_IMG.TIF>"
out_path = "<PATH_TO_OUTPUT_FILE.shp"

# The kernel size will be 32 by 32
x_size_kernel = 32
y_size_kernel = 32

# Extract the x row and y column.
x_row = 16
y_row = 4757

# Setup up a kernel generator for this .tif file.
tif_kernel_generator = nso_tif_kernel_iterator. .nso_tif_kernel_iterator_generator(path_to_tif_file, x_size_kernel, y_size_kernel)

kernel = tif_kernel_generator.get_kernel_for_x_y(x_row,y_row )
kernel.shape
#output: (4, 32, 32)
# This .tif file contains 4 dimensions in RGBI 

# Set a fade kernel which gives more weight to the centre pixel in the kernel.
tif_kernel_generator.set_fade_kernel()

# Make a euclidean distance model.
euclidean_distance_model = nso_ds_models.euclidean_distance_model(tif_kernel_generator)

# Set annotations for the model to predict on.
euclidean_distance_model.set_ec_distance_custom_annotations(path_to_tif_file.split("/")[-1], fade=True)

# Iterates and predicts all the pixels in a .tif file with a particular model and stores the dissolved results in the out_path file in a multiprocessing way. So this has to be run from a terminal.  
tif_kernel_generator.predict_all_output(euclidean_distance_model, out_path , parts = 3, fade=True)
```

# Models

[comment]: <> ( Since the satellite files contain a lot of pixels or extracted kernels and since we predict per pixel or extracted kernel, sometimes 3 billion pixels in total, we have used different models in order to increase performance for larger satellite images.)

We have tried different models all of which are still stored in nso_ds_classes /nso_ds_models.py file.

After a certain amount of testing we have decided that the (scaled) contrast model is the final model we will use. This is due to it's capability to adjust for variances in the atmosphere which causes different color interferences in the satellite images. And it’s better computing performance versus conventional best performance models like deep learning models, which was a couple of minutes in contrast to days.

## (Main) Scaled Contrast Model
### Predetermined contrast classes
First Predetermined classes centers are made which classify classes based on there contrasts. These classes were made with domain expertise. 

These are the current cluster centers we are using:

![Alt text](cluster_centers_contrast_model.PNG?raw=true "Title")

Were band 3 is the color blue, band 5 is NDVI more about what NDVI is [Here](https://www.sciencedirect.com/topics/earth-and-planetary-sciences/normalized-difference-vegetation-index#:~:text=The%20NDVI%20is%20a%20dimensionless,From%3A%20Environmental%20Research%2C%202018) and band 6 Lidar height data more information about which Lidar used  [Here](https://www.ahn.nl)

These values are already scaled more about this in the next section.

### Value scaler
This model first calculates contrast for every different input values using scaler values between 0 en 1. This has to be done for each satellite image and for each input/band.
We use sklearn to make these min max scalers which have to be stored in a .save file which in turn is stored in the scalers folder.

For more about how min max scalers, read the sklearn page [Here](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.MinMaxScaler.html)

### Input filtering

For the input we discovered that the atmosphere had a collinearity effect on the three red, green and blue values. As such we only pick the blue band in order to reduce the relative color weight in the prediction compared to NDVI and height data.
A other advantage of using the color blue is that it’s the best color to  detect sand with. Detecting sand is particularly interesting for ecologists. 

The following illustration exemplifies the eventual and final input we have for the contrast model:

![Alt text](contrast_model_input.png?raw=true "Title")

### Predicting

The following picture gives a easy illustration about how the prediction works:

![Alt text](Contrast_model_example.png?raw=true "Title")

For a particular pixel or kernel, in which blue, NDVI and height is used and scaled between 0 en 1, Euclidean distance is calculated for blue, NDVI and height to each cluster center. The shortest Euclidean distance to a class, is then the predicted class.

This can easily be interpreted as if for a given certain pixel or kernel the scaled blue color is between 1 and 0.7 this would mean that for this certain pixel/kernel sand for the input value blue. The same goes for the other input values NDVI and Height. The combinations of different input values gives the final predicted class.


### Example code
Below here we will give a code example of how to use this contrast model and added comments.

```python

import nso_ds_classes.nso_tif_kernel_iterator as nso_tif_kernel_iterator
import nso_ds_classes.nso_ds_models as nso_ds_models
import glob
from nso_ds_classes.nso_ds_models import cluster_scaler_BNDVIH_model
from nso_ds_classes.nso_ds_normalize_scaler import scaler_class_BNDVIH
import nso_ds_classes.nso_ds_cluster as nso_ds_cluster 
from os.path import exists

""" 
This is now the default model.

"""

if __name__ == '__main__':

    # Set a kernel generator, in this case we will only use a kernel size of 1.
    x_kernel_width = 1
    y_kernel_height = 1

    # Loop through a directory which contains .tif files to be predicted.
    for file in glob.glob("<PATH>/<TO>/<TIF_DIRECTORY>/*blue_ndvi_height.tif"):

        # Setup a tif kernel iterator.
        path_to_tif_file = file.replace("\\","/")
        print(path_to_tif_file)
        out_path = "E:/output/Coepelduynen_segmentations/"+path_to_tif_file.split("/")[-1].replace(".tif","_normalised_cluster_model.shp")      
        tif_kernel_generator = nso_tif_kernel_iterator.nso_tif_kernel_iterator_generator(path_to_tif_file, x_kernel_width , y_kernel_height)

        
        # Path to the predetermined classes.
        cluster_centers_file = "./cluster_centers/normalized_5_BHNDVI_cluster_centers.csv"

               
        # Load the class cluster centers into a euclidean distance model.   
        a_cluster_annotations_stats_model = cluster_scaler_BNDVIH_model(cluster_centers_file)

        # This model needs scalers in order to be useful for a specific .tif file, check if they already exist. Make them if they don't already exist.
        if exists("./scalers/"+path_to_tif_file.split("/")[-1]+"_band3.save") is False:
                print("No scalers found making scalers")
                a_nso_cluster_break = nso_ds_cluster.nso_cluster_break(tif_kernel_generator)
                a_nso_cluster_break.make_scaler_stepped_pixel_df(output_name = path_to_tif_file.split("/")[-1], parts=2, begin_part=1, multiprocessing= True)

        # Initialize a scaler model for the iterater to be used.
        a_normalize_scaler_class_BNDVIH = scaler_class_BNDVIH( "./scalers/"+path_to_tif_file.split("/")[-1]+"_band3.save", \
                                                                            scaler_file_band5 = "./scalers/"+path_to_tif_file.split("/")[-1]+"_band5.save", \
                                                                            scaler_file_band6 = "./scalers/ahn4.save")

        # The main iterator loop, if no multiprocessing variable is giving, it automatically will do multiprocessing and thus has to be run from a kernel. 
        tif_kernel_generator.predict_all_output(a_cluster_annotations_stats_model, out_path , parts = 3,  normalize_scaler= a_normalize_scaler_class_BNDVIH )

```

## Euclidean distance Model.

This model simply calculates the Euclidean distance to pre annotated pixel kernels and then basis it's predictions on the shortest distance.

The following image demonstrates this model:

![Alt text](simple_model.png?raw=true "Title")

Predicting with Euclidean distance model is implement in the following way:

```python
import nso_ds_classes.nso_tif_kernel as nso_tif_kernel
import nso_ds_classes.nso_ds_models as nso_ds_models

# Set up a kernel generator.
x_kernel_width = 32
y_kernel_height = 32

path_to_tif_file = "<PATH_TO_NSO_SAT_IMG.TIF>"
out_path = "<PATH_TO_OUTPUT_FILE.shp"

tif_kernel_generator = nso_tif_kernel.nso_tif_kernel_generator(path_to_tif_file, x_kernel_width, y_kernel_height)

# Setup a euclidean distance model based on the tif kernel generator.
euclidean_distance_model = nso_ds_models.euclidean_distance_model(tif_kernel_generator)

# Load the model with annotations wich have been presupplied in a .csv file.
euclidean_distance_model.set_ec_distance_annotations()

# Get a random kernel based on a row and a column position.
x_row = 50
y_row = 4757
kernel = tif_kernel_generator.get_kernel_for_x_y(x_row,y_row)

# Make a prediction based on this kernel.
euclidean_distance_model.predict_kernel(kernel)
#output: "bos"
```

## Deep learning Model.

We use Keras in python and made a deep learning with following architecture:

![Alt text](deep_learning_architecture.png?raw=true "Title")

Which is used in the following way:

```python
    import nso_ds_classes.nso_tif_kernel as nso_tif_kernel
    import nso_ds_classes.nso_ds_models as nso_ds_models

    x_kernel_width = 32
    y_kernel_height = 32

    path_to_tif_file = "<PATH_TO_TIF_FILE>"
    out_path = "<PATH_TO_OUTPUT_FILE"
    tif_kernel_generator = nso_tif_kernel.nso_tif_kernel_generator(path_to_tif_file, x_kernel_width , y_kernel_height)

    # Set a model on a .tif generator. 
    model = nso_ds_models.deep_learning_model(tif_kernel_generator)
    # Get annotations for a particular .tif file.
    model.get_annotations(path_to_tif_file.split("/")[-1])
    # Set a type of deep learning network.
    model.set_standard_convolutional_network()
    # Train the specificed model on the annotations.
    model.train_model_on_sat_anno(path_to_tif_file.split("/")[-1])

    # Use the model to predict all the pixels in a .tif in a multiprocessing way.    
    tif_kernel_generator.predict_all_output(model, out_path , parts = 3)

```

# Author
Michael de Winter

Jeroen Esseveld.
# Contact

Contact us at vdwh@pzh.nl







