# s22-team10-project
<h1 align='center'> GEOGUESSR PROJECT</h1>
<p align='center'> <strong>Team Members:</strong><br> Andrew Preston<br> Matt Fadler<br> Riley Welch<br> Yassaman Nassiri<br> Jansen Long<p>
<br>
  
For a step-by-step guide to the project please see:
>[DEMO](./demo.md)
  
All files can be found under the folder [geoguessr](./geoguessr)

### Overview
---

##### Grid Making
We loaded a cartographic boundary shapefile of the U.S. available through the US Census Bureau ([Source](https://www.census.gov/geographies/mapping-files/time-series/geo/carto-boundary-file.html)) into [mainlandGrid.ipynb](geoguessr/mainlandGrid.ipynb). <p align= 'center'>![Unedited Map](/images/us_all.png)</p> Next we isolated the mainland U.S. <p align= 'center'>![Isolated U.S.](/images/mainland.png)</p> We overlayed it with a grid and combined the two to cut off the grids using the boundry of the U.S. A function called MergeFactor was used to combined sections that were too small after the previous process. <p align= 'center'>![Grid Overlay](/images/mainland+grid.png)</p> This left us with a file called [mainlandGrid.pickle](geoguessr/data/pickled_data/mainlandGrid.pickle) that was a map of the mainland U.S. with 65 grids across it. Each section of the grid was associated with a number 0-64. <p align= 'center'>![Mainland w/grid](/images/mainlandGrid.png)</p>

##### Data Scraping
Using this map data and [datascraper.ipynb](geoguessr/datascraper.ipynb) we scraped images from Google maps using the Google maps API. For each grid section we scrapped 10 locations (650 location total). For each location we got 3 pictures (1950 pictures total), since Google map images are 360-degree images we used the API to grab an image of size 400px by 200px at 0, 90, and 180-degrees. For Training/Testing we used an 80%/20% split; 8 locations or 24 images per section for training and 2 locations or 6 images per section for testing. The format for labeling each image is [grid no.]_[unique no.].jpg (ex. 0_5.jpg).

>[Source](https://github.com/Nirvan66/geoguessrLSTM): Used this as a reference for the grid maker and datascraper.

##### Prepping Data
After scrapping the data we were left with two folders <ins>training_data</ins> with 1560 unique images and <ins>testing_data</ins> with 390 unique images. In-order to be usable by the model we needed to convert the images into numpy arrays. Using the [load_data.ipynb](geoguessr/load_data.ipynb) file we converted <ins>training_data</ins> images into nparray x_train and <ins>testing_data</ins> images into nparray x_test. We also created nparray y_train and y_test that held the grid section number for each image in its corresponding nparray. This tuple of nparrays were inspired and modeled after the MNIST nparrays used in OLA8 from class.

##### Model Training
Using the x_train and y_train arrays we trained up a Convolution Neural Network (CNN). After sucessfully training, the trained model was saved in a '.h5' file. Saving this model this way allows for the reuse of the trained model without losing the weights of the model.

##### Output
Using the trained model, the x_test data can be loaded into the model and ran. Using the predict() function on the model we can isolate a grid section to see the weights the model predicts it to be. Using the argmax() function on the array of weights gives the index location on the array with the highest weight. This highest weight represtents the best guess given by the model. This index number is then translated into the corresponding grid section the model is guessing. This predicted grid number can be compared to the grid number entered by the user in the predict() function to see if they match. If they do not match you can see how far away the sections are using the Numbered Grid below. We can also output the pictures from the actual grid and predicted grid to compare them visually.

![Numbered Grid](images/numberedGrid.png)
