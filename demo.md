<h1 align="center">Demo</h1>

## Grid Set-up

---
Before starting the following three commands should be run to install shapely, geopandas, and gmaps. Shapely and geopandas allows for easier manipulation of our data; while gmaps allows for code integrations with the Google maps API.
```python
pip install shapely
```

```python
pip install geopandas
```

```python
pip install gmaps
```
The following imports will be needed to create the grid and scrape the data from Google maps.
```python
import matplotlib.pyplot as plt
import pandas as pd
import geopandas as gpd
import shapely
from shapely.geometry import Point, Polygon
import numpy as np
import pickle
import gmaps
```
Any shapefile can be used depending on the scope of your project. This project is using the nation shapefile from the [U.S. Census Bureau](https://www.census.gov/geographies/mapping-files/time-series/geo/carto-boundary-file.html). Note: When working with shapefiles you will have multiple other files besides the actual shapefile. Please keep these files together, the shapefile will <ins>NOT</ins> work without the other files.
```python
states = gpd.read_file('/path/to/cb_2018_us_nation_5m.shp')
```

```python
states
```

![output](/images/2.png)

The following code will output the graphical representation of the shapefile, with the center of the mainland U.S. marked.
```python
%%capture --no-display
mainland_center = Point(-98.35,39.50)
for i,j in enumerate(states['geometry'][0]):
    x,y = j.exterior.xy
    plt.plot(x,y)
    if mainland_center.within(j):
        mainland = np.array([(yj,xj) for xj,yj in zip(x,y)])
plt.scatter([-98.35], [39.50], c='r', marker='x', s=200)
plt.xlim([-190,-60])
plt.ylim([15,75])
plt.show()
```
![output](/images/3.png)<br><br>
Now the mainland can be isolated.
```python
plt.plot(mainland[:,1],mainland[:,0])
plt.show()
```
![output](/images/4.png)<br><br>
Pickle will be used to save this now map of the mainland. Pickle allows for saving of data and loading that same data without the risk of losing progress. The code below shows that the map is saved as [mainland.pickle](geoguessr/data/pickled_data/mainland.pickle) and that data can then be loaded back unchanged whenever needed.
```python
pickle.dump(mainland,open("/path/to/mainland.pickle","wb"))
mainland = pickle.load(open("/path/to/mainland.pickle", "rb"))
mainland = Polygon(np.flip(mainland))
x,y = mainland.exterior.xy
plt.plot(x,y)
```
![output](/images/4.png)<br><br>
Next a grid is overlayed on the map. The number of grids can be adjusted by changing the value of 'base'. Increasing 'base' will decrease the number of grids, while decreasing 'base' will increase the number of grids.
```python
value = mainland.bounds
base = 4

min_x = int(value[0]//base)
min_y = int(value[1]//base)
max_x = int(value[2]//base)
max_y = int(value[3]//base)

for i in range(min_x, max_x + 1):
for j in range(min_y, max_y + 1):
y = shapely.geometry.box(i*base, j*base, (i+1)*base, (j+1)*base)
r = mainland.intersection(y)
x,y = y.exterior.xy
plt.plot(x,y,c='y')
if r.is_empty:
continue
elif type(r) == shapely.geometry.multipolygon.MultiPolygon:
for gems in r.geoms:
x,y = gems.exterior.xy
plt.plot(x,y,c='g')
else:
x,y = r.exterior.xy
plt.plot(x,y,c='g')
plt.show()
```
![output](/images/5.png)<br><br>
The following chunk of code will take the map and grid overlay and perform two functions. One function performed will be the combining of the grid and map to prevent and overflow of the grid. The other function will merge sections that are too small with adjacent grids. The magnatude of effect by mergeFactor can be controlled by the value passed to mergeFactor. A larger mergeFactor will result in larger grid sections; while a smaller mergeFactor will result in smaller sections.
```python
def partition(mainland, base, mergeFactor):
    '''
    polygon: Unsplit polygon of mainland US
    dim: The dimensions of each grid to split the map into
    mergeFactor: threshold of smallest grid. 
    Any grid smaller will be combined with neighbouring grids
    '''
    value = mainland.bounds
    min_x = int(value[0]//base)
    min_y = int(value[1]//base)
    max_x = int(value[2]//base)
    max_y = int(value[3]//base)
    grid = 0
    res = []
    for i in range(min_x, max_x+1):
        for j in range(min_y, max_y+1):
            y = shapely.geometry.box(i*base, j*base, (i+1)*base, (j+1)*base)
            r = mainland.intersection(y)
            if r.is_empty:
                continue
            if type(r)==shapely.geometry.multipolygon.MultiPolygon:
                for gems in r.geoms:
                    res.append(gems)
                    grid += 1
            else:
                res.append(r)
                grid += 1
    return merge(res, mergeFactor)

def merge(polyList, mergeFactor):
    '''
    polyList: list of polygon grids the map is split into
    mergeFactor: threshold of smallest grid. 
    Any grid smaller will be combined with neighbouring grids
    '''
    def combine(pidx, polyL):
        p = polyL[pidx]
        del polyL[pidx]
        for idx,i in enumerate(polyL):
            u = p.union(i)
            if p.intersects(i) and type(u)!=shapely.geometry.multipolygon.MultiPolygon:
                polyL[idx] = u
                break
        return polyL
    
    mnLimit = max(polyList, key=lambda x:x.area).area * mergeFactor
    mnPoly = min(polyList, key=lambda x:x.area)
    while(mnPoly.area<=mnLimit):
        polyList = combine(polyList.index(mnPoly), polyList)
        mnPoly = min(polyList, key=lambda x:x.area)
        
    result = {}
    for idx,i in enumerate(polyList):
        x,y = i.exterior.xy
        result[idx] = np.array([(y,x) for x,y in zip(x,y)])
    return result

def plotMap(mainlandGrid):
    gPoly = []
    gMarkLoc = []
    gMarkInf = []
    info_box_template = """
    <dl>
    <dd>{}</dd>
    </dl>
    """
    for k,v in mainlandGrid.items():
        gPoly.append(gmaps.Polygon(
                        list(v),
                        stroke_color='red',
                        fill_color='blue'
                        ))
        gMarkLoc.append((v[0][0],v[0][1]))
        gMarkInf.append(info_box_template.format(k))
    fig = gmaps.figure(center=(39.50,-98.35), zoom_level=4, map_type='TERRAIN')
    fig.add_layer(gmaps.drawing_layer(features=gPoly))
    fig.add_layer(gmaps.marker_layer(gMarkLoc, info_box_content=gMarkInf))
    return fig
```
Call the above code and print the number of sections in the final map. This particualar example will produce 65 sections indexed 0-64.
```python
mainlandGrid = partition(mainland, base, mergeFactor=0.2)
len(mainlandGrid)
```
65
```python
for i in mainlandGrid.values():
    plt.plot(i[:,1],i[:,0])
plt.show()
```
![output](/images/6.png)<br><br>
The final grid can be seen above. This map is now saved into a pickle file for use later.
```python
pickle.dump(mainlandGrid,open("/to/path/mainlandGrid.pickle","wb"))
```

## Data Scraping

---
### Set-up

To get started with the data scraping, a Google Cloud account is required. We have provided a link below to get you started.

[Google Cloud Console](https://console.cloud.google.com)

The next step is to set up a Google Street View API set up on your Google Cloud Console. The link below provides details on how you are able to scrape images using your API key. Tutorials can be found on the Google Cloud Console to get your account billing set up, but a free $300 credit was provided by Google at the time of this project.

[Google Street View API Overview](https://developers.google.com/maps/documentation/streetview/overview)

Now that we have an API key, we can get started with scraping. The key has been left out below, but the variable is still set up for you to paste your new key into. A file directory to store both the training and testing images should be created, and the dataDir variable should store this file path.

```python
# Insert API key
key = ''
import requests
import json, os
import urllib.request
import random
dataDir = "/path/to/training_data"
```
The code below is utilizing the mainlandGrid.keys to create the list of grids to search for images in. An example is shown to scrape for images in both all grids and for only the first three grids.

```python
# Data is scraped from all grids
# searchGrids = mainlandGrid.keys()

# Data is scraped for first 3 grids
searchGrids = list(range(0,3))
print("Search in Grids: {}".format("All" if searchGrids==mainlandGrid.keys() else searchGrids))
```
![output](/images/7.png)<br><br>
We have specified that each image scraped will be 400x200 pixels. The location will be randomly generated using poly.bounds as the range for these numbers. The count variable allows you to change the number of locations to search for within each grid section. The trial variable allows you to attempt scraping an image from (or near) that coordinate a specificed amount of times before moving on. The ignum variable specifies the image number for that grid. For our project, we used 10 locations per grid section (8 for training 2 for testing) 3 images per location, so this means we pulled 24 images per grid section for training and 6 images for testing.

### Training data
For training data ignum iterates from 0 to 23 for the training locations becuase we are scraping images with a heading of 0, 90, and 180 degrees. This number is attached to the end of the image file name to ensure you scrape the same number of images per location. If the scraping does not work for a specified grid, you can add a conditional in the while loop to look like grid==gridMissing and it will only scrape images for that one grid location. You will also need to update the while loop to reflect the number of images you are missing  (i.e. count<3 would mean you need 3 more image locations).

```python
base = 'https://maps.googleapis.com/maps/api/streetview'
ext = '?size=400x200&location={}&fov=100&heading={}&radius={}&pitch=10&key={}'
print("Seacrchin Grids: {}".format("All" if searchGrids==mainlandGrid.keys() else searchGrids))
for grid,coor in mainlandGrid.items():        
    poly = Polygon(np.flip(coor))
    minx, miny, maxx, maxy = poly.bounds
    count = 0
    trials = 0
    locList = []
    if grid in searchGrids:
        saveFolder = dataDir
        if os.path.exists(saveFolder)==False:
            os.mkdir(saveFolder)
        locList = os.listdir(saveFolder)
        print("################## Searching grid {} ###################".format(grid))
        imgnum = 0
        
        # Only scraping 8 locations for training data (3 pictues from each location)
        while count<8 and trials<4:
            pnt = Point(random.uniform(minx, maxx), random.uniform(miny, maxy))
            location = str(pnt.y)+','+str(pnt.x)
            if (poly.contains(pnt)) and (location not in locList):
                metaUrl = base + '/metadata' + ext.format(location, 0, 10000, key)
                r = requests.get(metaUrl).json()
                trials += 1
                print("Trial: {}, count: {}".format(trials,count))
                if r['status']=='OK' and poly.contains(Point(r['location']['lng'],r['location']['lat'])):
                    location = str(r['location']['lat'])+','+str(r['location']['lng'])
                    if (location not in locList):
                        print("Valid location found: {}".format(location))
                        locList.append(location)
                        saveFile = saveFolder
                        if os.path.exists(saveFile)==False:
                            os.mkdir(saveFile)

                        for heading in [0,90,180]:
                            imgUrl = base + ext.format(location, heading, 10000, key)
                            urllib.request.urlretrieve(imgUrl,saveFile+'/{}_{}.jpg'.format(grid,imgnum))
                            imgnum += 1
                        count += 1
                        trials = 0
                    else:
                        print("Failed trial {} location exists".format(trials))
                        print("Location {}".format(location))
                else:
                    print("Failed trial {} status or contains".format(trials))
                    print("Location {}".format(location))
        print("No duplicates: {}".format(len(locList)==len(set(locList))))
```
### Testing data

The same strategy used above is used to generate images for the testing data. The dataDir variable should be changed to the path for your testing data folder.

```python
# Change to save images to testing data folder
dataDir = "/path/to/testing_data"
```
The count variable is limited to 2, so only two locations for images will be used in the testing data. This will be a total of 6 images, 20% of the total of images per grid in the training data set.

```python
base = 'https://maps.googleapis.com/maps/api/streetview'
ext = '?size=400x200&location={}&fov=100&heading={}&radius={}&pitch=10&key={}'
print("Seacrchin Grids: {}".format("All" if searchGrids==mainlandGrid.keys() else searchGrids))
for grid,coor in mainlandGrid.items():        
    poly = Polygon(np.flip(coor))
    minx, miny, maxx, maxy = poly.bounds
    count = 0
    trials = 0
    locList = []
    if grid in searchGrids:
        saveFolder = dataDir
        if os.path.exists(saveFolder)==False:
            os.mkdir(saveFolder)
        locList = os.listdir(saveFolder)
        print("################## Searching grid {} ###################".format(grid))
        imgnum = 0
        
        # Only scraping 2 locations for testing data (3 pictues from each location)
        while count<2 and trials<4:
            pnt = Point(random.uniform(minx, maxx), random.uniform(miny, maxy))
            location = str(pnt.y)+','+str(pnt.x)
            if (poly.contains(pnt)) and (location not in locList):
                metaUrl = base + '/metadata' + ext.format(location, 0, 10000, key)
                r = requests.get(metaUrl).json()
                trials += 1
                print("Trial: {}, count: {}".format(trials,count))
                if r['status']=='OK' and poly.contains(Point(r['location']['lng'],r['location']['lat'])):
                    location = str(r['location']['lat'])+','+str(r['location']['lng'])
                    if (location not in locList):
                        print("Valid location found: {}".format(location))
                        locList.append(location)
                        saveFile = saveFolder
                        if os.path.exists(saveFile)==False:
                            os.mkdir(saveFile)

                        for heading in [0,90,180]:
                            imgUrl = base + ext.format(location, heading, 10000, key)
                            urllib.request.urlretrieve(imgUrl,saveFile+'/{}_{}.jpg'.format(grid,imgnum))
                            imgnum += 1
                        count += 1
                        trials = 0
                    else:
                        print("Failed trial {} location exists".format(trials))
                        print("Location {}".format(location))
                else:
                    print("Failed trial {} status or contains".format(trials))
                    print("Location {}".format(location))
        print("No duplicates: {}".format(len(locList)==len(set(locList))))
```

## Convert to numpy array

Currently two folders containing the training images and testing images are available. Each image name is formated (grid no.) _ (0-23).jpg for training and (grid no.) _ (0-5).jpg for testing.<br><br> These pictures need to be converted into numpy arrays to be used by the model.<br> The structure for the numpy arrays are modeled after the tuple of numpy arrays found in OLA8.

Creating your numpy arrays is encuraged but premade arrays can be found on ~~[Google Drive](https://drive.google.com/drive/folders/1RZKKJQbPvG1PoUpcjkIwt54VZd9Z83LG?usp=sharing)~~ while available. (Issues occur with linked files - Making your own is recommended)

* x_train - holds all of the training images.
    * Expected shape (# of images, width, height, 3)
* y_train - holds the corresponding grid number for each element in x_train.
    * Expected shape (# of images)
* x_test - holds all of the testing images.
    * Expected shape (# of images, width, height, 3)
* y_test - holds the corresponding grid number for each element in x_test.
    * Expected shape (# of images)

### Create x_train and x_test
```python
from PIL import Image
import glob

# Load training data images
filelist = glob.glob('/path/to/training_data/*.jpg')
```

```python
x_train = np.array([np.array(Image.open(fname)) for fname in filelist])
print(x_train.shape)
```
![output](/images/8.png)
```python
plt.imshow(x_train[6])
```
![output](/images/9.png)
```python
# Load testing data images
filelist = glob.glob('/path/to/testing_data/*.jpg')
```

```python
x_test = np.array([np.array(Image.open(fname)) for fname in sorted(filelist)])
print(x_test.shape)
```
![output](/images/10.png)
```python
plt.imshow(x_test[0])
```
![output](/images/11.png)
### Create y_train and y_test

```python
y_train = []
for i in range(1560):
    y_train.append(i // 24)
y_train = np.array(y_train)
```

```python
print(y_train)
```
![output](/images/12.png)
```python
y_test = []
for j in range(390):
    y_test.append(j // 6)
y_test = np.array(y_test)
```

```python
print(y_test)
```
![output](/images/13.png)

## Create Network Architecture

For this project a Convolution Neural Network was used with keras.Sequential(). The use of Sequential allowed for the model to train sucessfully without running into memory issues. Implementing checkpoints was considered but not used, adding checkpoints would be recommended if attempting to train a more robust version the model.

### Network

```python
import tensorflow.keras as keras
from tensorflow.keras import backend as K
from IPython.display import display
from mpl_toolkits.mplot3d import axes3d
%matplotlib inline
from tensorflow.keras import layers
```

```python
model = keras.Sequential()
model.add(layers.Input(x_train.shape[1:]))
model.add(layers.Conv2D(8, kernel_size=(16,16), activation='relu'))
model.add(layers.Conv2D(16, kernel_size=(32,32), activation='relu'))
model.add(layers.MaxPool2D(pool_size=(16,16)))
model.add(layers.Dropout(0.25))
model.add(layers.Flatten())
model.add(layers.Dense(32, activation='relu'))
model.add(layers.Dropout(0.5))
# Output Logits (64)
model.add(layers.Dense(len(np.unique(y_train))))
model.compile(loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True),
optimizer=keras.optimizers.Adam(),
metrics=[keras.metrics.SparseCategoricalAccuracy()])

model.summary()
```
![output](/images/14.png)
```python
keras.utils.plot_model(model,show_shapes=True,expand_nested=True)
```
![output](/images/15.png)

### Train model

Adding a data generator
```python
data_generator = keras.preprocessing.image.ImageDataGenerator(
    width_shift_range=0.1,
    height_shift_range=0.1,
    rotation_range=10,
    zoom_range=0.1,
    horizontal_flip=False)
dg_trainer = data_generator.flow(x_train,y_train,
                                 batch_size=256)
```
Training can begin with the training data
```python
epochs = 1
history = model.fit(dg_trainer,
                    epochs=epochs,
                    verbose=1,
                    validation_data = (x_train,y_train))
```
After sucessfully training the model, it can be saved using the save function. Doing so saves the parameters and the weights of the model. Allowing it to be reused.
```python
model.save('model.h5')
```

### Load model

Use the load_model command to load the saved trained model in the '.h5' file. 
```python
from keras.models import load_model
```

```python
model = load_model('/path/to/model.h5')
```

### Run test

Run the trained model using the test data.
```python
results = model1.evaluate(x_test, y_test, verbose=1)
print('Test loss:', results[0])
print('Test accuracy:', results[1])
```

Print out the probabilites the model created. Specific images can be singled out, any index number between 0 and 389 is valid. 65 propabilites are produced for each test image, one for each grid section.
```python
# Predict the grid for image at index 0
av = model1.predict(x_test[0:1,:,:,:])
print(av)
```
[[-9.7438577e-05 -3.5351203e-03  3.4451382e-03  4.9460265e-03
   3.9203633e-03  4.7128745e-03  4.2857816e-03  6.4281514e-04
   2.7480620e-04  3.4770833e-03 -3.8042972e-03 -2.9051316e-03
  -3.5399429e-03  1.4375881e-03  2.2989979e-03  4.1965935e-03
   1.5280354e-03  1.3383749e-03  2.9648589e-03  3.7478625e-03
   4.2069498e-03  2.2267867e-03 -3.4646064e-03  1.5764928e-03
   9.0027624e-04 -4.1558030e-03  2.8071324e-03  1.4901732e-03
   3.6471479e-03 -3.0805981e-03 -3.4800523e-03  3.0209175e-03
  -1.9643460e-03 -3.7715195e-03 -3.1339342e-04 -3.9459565e-03
  -4.6856711e-03 -3.5233803e-03 -2.3624091e-03  3.3202167e-03
   3.4473673e-03  2.7110346e-03  3.1323442e-03  3.8978946e-03
   4.4271760e-03  4.2961910e-03 -1.3119633e-03  3.3928377e-03
   2.0747068e-03 -1.4319129e-03  2.8503067e-03 -3.2617716e-04
   4.2725494e-03  4.4618594e-03  4.4642398e-03  3.7621376e-03
   3.7214982e-03  4.8789396e-03  2.9642789e-03  2.9696927e-03
   4.4229962e-03  3.2635415e-03 -2.8053141e-04  5.3079412e-03
   1.6730761e-03]]
   
Using the argmax function we can find the index location with the highest probability. This index location represents the section the model believe to be correct.
```python
mx = np.argmax(av)
mx
```
63

## Output

The index location must now be converted into the correct grid number using the following code. This step is required with how the training and testing images are sorted. Instead of being sorted in proper numerical order i.e.(0,1,2,3,4,5,6,...,64) the images were sorted according to the first number this caused some images to be out of order i.e.(59,6,60,61,62,63,64,7,8,9).

The following code converts the index number to the correct grid number.
```python
# Convert prediction to correct grid number
fa = 0

if mx <= 1:
    fa = mx
elif mx >= 2 and mx <= 11:
    fa = mx+8
elif mx == 12:
    fa = 2
elif mx >= 13 and mx <= 22:
    fa = mx+7
elif mx == 23:
    fa = 3
elif mx >= 24 and mx <= 33:
    fa = mx+6
elif mx == 34:
    fa = 4
elif mx >= 35 and mx <= 44:
    fa = mx+5
elif mx == 45:
    fa = 5
elif mx >= 46 and mx <= 55:
    fa = mx+4
elif mx == 56:
    fa = 6
elif mx >= 57 and mx <= 61:
    fa = mx+3
else:
    fa = mx-55
```
The output from this cell will give you the actual grid number the model guessed. Looking at same index location in the list of test images will tell you the correct grid number.
```python
# Model's predicted grid location
fa
```
8

The numbered map below shows the correspnding location to the grid number.

In our example the image from index 0 is from grid 0, this can be compared to the models best guess grid 8.

![Numbered Grid](images/numberedGrid.png)

The following code can be used to view the image the model was given. Just replace the 0 with the index number inserted into the predict function above.
```python
# Actual image
plt.imshow(x_test[0])
```

![Output](images/out1.png)

The following blocks of code will show the test images available in the grid section the model guessed.
```python
# A picture from the predicted region the picture is from (6 photos for each grid... mx*6 (increment by 1 to get the remaining 5 pictures))
plt.imshow(x_test[mx*6])
```
![Output](images/out2.png)
```python
plt.imshow(x_test[mx*6+1])
```
![Output](images/out3.png)
```python
plt.imshow(x_test[mx*6+2])
```
![Output](images/out4.png)
```python
plt.imshow(x_test[mx*6+3])
```
![Output](images/out5.png)
```python
plt.imshow(x_test[mx*6+4])
```
![Output](images/out6.png)
```python
plt.imshow(x_test[mx*6+5])
```
![Output](images/out7.png)
