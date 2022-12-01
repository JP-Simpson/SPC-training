# Classifying seagrass using Sentinel-2 data, field observations and Google Earth Engine

This exercise will demonstrate another capability of Google Earth Engine (GEE): image classification. This will be introduced in the context of mapping two critical species of seagrass in Wallis Lake, NSW, Australia. This will use similar techniques to the previous GEE exercise, as well as demonstrating how limited field observation data can be extended across a large area using remote sensing analysis.

---

### About classification
Image classification is the process of placing remote sensing image pixels into categories based on spectral information, and in some cases other information. There are a wide array of different classification approaches and algorithms, but they can broadly be divided into three categories:
1. **Unsupervised classification** - Unsupervised classification sorts pixels into classes based only on spectral variation between pixels. Unsupervised classification approaches are not trained (i.e. they do not use field observations to relate spectral signatures to features). They are often used in initial analysis to identify spectral variation, or for analysis of very large areas where ground truth data is unavailable.
2. **Supervised classification** - Supervised classification uses pixels, areas or features identified by a user as training data to tell the classification algorithm what those features are expected to look like. This allows the algorithm to sort image pixels into categories based on the users' identification of a limited number of training areas.
3. **Object-Based Image Analysis** - Object-based Image Analysis (OBIA) separates an image into clusters of similar pixels, before placing each cluster into a class based on its spectral properties, and in some cases, other characteristics such as shape. This is useful as it can provide a more realistic representation of the real world, which is not divided into pixels like a satellite image.

We are going to apply approach 2, supervised classification. This allows us, in simple terms, to use limited field observations to map a much larger area. GEE can also be used for unsupervised or OBIA classification.

---

### About the field site
Wallis Lake is a barrier estuary in NSW, approximately 200 km north-northeast of Sydney. It covers a total area of about 100 km<sup>2</sup> and is about 25 km long. Two towns, Forster and Tuncurry, are at the mouth of the estuary where it meets the Pacific Ocean. Much of the rest of the lake is bordered by rural and bushland areas, including two national parks: Wallingat National Park to the west, and Booti Booti National Park to the east.

As well as being popular for recreation, camping, and fishing, Wallis Lake is home to some of NSW's most extensive seagrass beds. Seagrasses are flowering plants which can grow submerged in marine water. They play a significant ecological role in coastal areas, acting as habitat and shelter for marine fauna, preventing erosion, sequestering CO<sub>2</sub> from the atmosphere, cleaning water, and acting as a fertile substrate for aquaculture.

Two seagrass species are widespread within Wallis Lake: *Zostera capricorni* and *Posidonia australis*.

*Z. capricorni* is a small, common, bright green, fast-growing species which grows in extensive beds all along the NSW coast. *P. australis* is larger, darker in colour, and grows in smaller patches. Various species of algae also grow in the estuary.

We will use supervised image classification to distinguish areas of each of these species, as well as areas where the estuary floor is bare of vegetation, and areas with algae present.

---

### Exercise instructions
#### Getting started

Open GEE code editor and select **New>File** in the **Scripts** tab. **Save** your new script as **WallisLakeClassification** or similar. **Remember to save your script regularly so you do not lose any work**.

First, we will add a point to the map representing Wallis Lake, then centre our map on it.
```javascript
var WallisLake = ee.Geometry.Point([152.5, -32.28]);
Map.centerObject(WallisLake, 12);
```
For full explanations of these functions, see the mangrove dieback exercise.

Next, we will load two datasets into GEE - our field observations, and a study area boundary.

Download the two shapefiles [here](https://github.com/JP-Simpson/SPC-training/tree/main/Seagrass%20Classification%20Data). Ensure you download all six related files for each shapefile.

Now we will add these shapefiles as assets to GEE. Click on the **Assets** tab in the top left, then click **New > Shape files**

![Assets tab in GEE](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/AssetsTab.png)

In the **Upload a New Shape File Asset** window, click **Select**, then select the six associated files for the **GroundTruthData** shapefile. Click **Upload**, and the shapefile will appear in your **Assets** tab. Repeat this with the **WallisLakeBoundary** shapefile.

![Shapefile upload](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ShapefileUpload.png)

**GroundTruthData** represents a series of points from transects, recording the bottom cover identified at each location. This is represented by the **label** field, as such:

| Label | Cover class |
|-------|-------------|
|   0   | Unvegetated |
|   1   |   Zostera   |
|   2   |  Posidonia  |
|   3   |    Algae    |

**WallisLakeBoundary** is a shapefile with the approximate boundary of the study area. We will use this for clipping the data later.

Now, add both the **GroundTruthData** and **WallisLakeBoundary** assets to your script by clicking the **Import** arrow, as shown below.

![Import assets into script](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ImportIntoScript.png)

After importing, your shapefile assets will appear like this in the **Imports** section of your script:

![Import section default names](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ImportedTables.PNG)

Click where the text says **table** and **table2** to change the variables to the correct names:

![Correct imported table names](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ImportedTablesNamesChanged.PNG)

---

#### Accessing Sentinel-2 data
Sentinel-2 refers to a European Space Agency (ESA) Earth Observation mission involving sensors mounted on two satellites, Sentinel-2A and Sentinel-2B. It provides remote sensing data in the visible, near-infrared, and shortwave infrared parts of the spectrum, at a spatial resolution ranging from 10m to 60m depending upon spectral band. You can read more about the Sentinel-2 mission [here](https://sentinel.esa.int/web/sentinel/missions/sentinel-2).

For this analysis, we are going to access the Sentinel-2 image collection, filter it to images over our site, and then focus on images with little cloud cover captured in the summer of 2018-19.

First access the image collection:
```javascript
var S2 = ee.ImageCollection("COPERNICUS/S2_SR");
```
Then apply filters to the imagecollection:
```javascript
S2 = S2.filterBounds(WallisLake).filterDate('2018-12-01','2019-03-01').filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10));
```

This command is **chaining** together multiple filters. These filters have the following effects:
| Filter | Effect |
|--------|--------|
|.filterBounds(WallisLake)|Includes only images which intersect with our Wallis Lake point|
|.filterDate('2018-12-01','2019-03-01')|Includes only images captured on or after the first date, and before the second date|
|.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10))| Uses the Sentinel-2 metadata to include only images with fewer than 10% of pixels obstructed by cloud|

The resulting variable **S2** is an **imagecollection** containing seven images which fit all these criteria.

---

#### Creating a median image

Satellite imagery over water is frequently very noisy. This is caused by several factors. For instance, water is a dark target - compared to most land cover types, it reflects relatively little light. Additionally, there tends to be more significant atmospheric interference over water areas, and the water's surface leads to specular reflectance effects which can produce noise in data.

Mapping seagrass is also complicated because of the effect of the water. Water absorbs and reflects light, and so do the contents of water, such as suspended and dissolved material. This means that, even with atmospherically corrected imagery, the signal we receive over water is made up of reflected light from both the water and, if it shallow enough, features on the bottom of the water. Further complicating this, the water level changes over time due to tides, and the composition of the water changes due to rain, wind, and other environmental factors.

There are many approaches to denoising remote sensing data over water and accounting for the effects on the signal of the water column, but these are advanced. For this exercise, we will use a relatively simple approach of calculating median pixel values over our image collection. This does not reduce the effect of the water, but it does reduce the amount of noise over water areas.

Using our filtered image collection, calculate a single median image with this script:
```javascript
var S2median = S2.median()
    .setDefaultProjection('EPSG:4326', null, 10)
    .select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8','B11'])
```

Again, this code is chaining together functions:

| Function | Effect |
|----------|--------|
|.median()|Calculates the median pixel value across the seven images in the image collection|
|.setDefaultProjection('EPSG:4326', null, 10)|Sets a projection for our median image. This is important as it will allow GEE to handle the image properly later|
|.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8','B11'])|Select only relevant bands from the Sentinel-2 data|

#### Masking out land areas

Next, we want to remove the land areas from our image. These are not relevant as we are only interested in seagrass and algae in Lake Wallis.

First, we will calculate the Normalized Difference Water Index using the green and shortwave infrared (SWIR) bands of the Sentinel-2 image:
```javascript
var NDWI = S2median.normalizedDifference(['B3','B11']);
```
Because water reflects green light and not SWIR light, this index produces high values (over 0) in water areas, and low values (below 0) in land areas. Now, we will create an image mask using this index:
```javascript
var water = NDWI.gte(0);
```
This is a binary layer. All areas that NDWI indicates to be water have the value of 1, while land areas have a value of 0.

Now update the mask on our Sentinel-2 image:
```javascript
var S2masked = S2median.updateMask(water);
```
The result of this will include only water areas. However, we have also included a large amount irrelevant water in our image, as Wallis Lake is located near the open ocean. We can fix this using the study area dataset we added earlier:
```javascript
S2masked = S2masked.clip(WallisLakeBoundary);
```
Add your layer to your map:
```javascript
Map.addLayer(S2masked);
```
Set visualisation parameters by clicking on **Layers** in the top right of the map window, then selecting the options symbol:

![Visualisation parameters options symbol](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/VisualisationOptions.PNG)

Try using the following parameters:

![Visualisation parameters](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/VisParam.png)

Don't forget to click **import** so you can import your visualisation parameters as a variable and use them again in the future.

Your image should look something like this:

![Masked image](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/S2Masked.png)

---

#### Dividing ground truth data into training and validation datasets

Next, we want to divide our ground truth data into **training** and **validation** datasets. **Training** data will be used to train the classification algorithm, while **validation** data will be used to measure how accurate the classification was.

*A note about ground truth data - In this instance, we are using field data which has not been captured randomly, but has been captured opportunistically. For more scientifically rigorous studies, we would prefer to use a sampling method such as stratified random sampling to gather our field data. In this case, we are making do with the data we have access to. Ask your instructor if you'd like to know more about this.*

We will begin by associated a random number (between 0 and 1) with each record in our ground truth dataset:
```javascript
var groundTruth = GroundTruthData.randomColumn();
```

Then, based on these random numbers, we will divide the groundtruth dataset into 60% **training** data and 40% **validation** data:
```javascript
var training = groundTruth.filter(ee.Filter.gt('random', 0.4));
var validation = groundTruth.filter(ee.Filter.lte('random', 0.4));
```

At this stage, your code should look something like this:
```javascript
//Setup
var WallisLake = ee.Geometry.Point([152.5, -32.28]);
Map.centerObject(WallisLake, 12);

//Access imagecollection
var S2 = ee.ImageCollection("COPERNICUS/S2_SR");

//Filter imagecollection
S2 = S2.filterBounds(WallisLake).filterDate('2018-12-01','2019-03-01').filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10));

//Calculate median image
var S2median = S2.median()
    .setDefaultProjection('EPSG:4326', null, 10)
    .select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8','B11']);

//Calculate NDWI and create mask
var NDWI = S2median.normalizedDifference(['B3','B11']);
var water = NDWI.gte(0);

//Update median image mask
var S2masked = S2median.updateMask(water);
S2masked = S2masked.clip(WallisLakeBoundary);

//Divide ground truth data into training and validation
var groundTruth = GroundTruthData.randomColumn();
var training = groundTruth.filter(ee.Filter.gt('random', 0.4));
var validation = groundTruth.filter(ee.Filter.lte('random', 0.4));
```

---

#### Training the classifier and classifying the image

Now it is finally time to classify our image. First, we must train the classifier, then classify the image like this:
```javascript
var trainingRegions = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).sampleRegions(training);
var RF = ee.Classifier.smileRandomForest(100).train(trainingRegions, 'label');
var classified = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).classify(RF);
```
The first line of code creates training regions based on our point dataset. These contain the relevant spectral information needed for the classification algorithm to function. The second line creates a **classifier** object based on an algorithm called **Random Forest**. This classifier object is the algorithm we need to produce classification results. The third line classifies the image, producing an output called **classified**.

Note that we are again filtering bands from the image, this time to remove the SWIR Band 11. This is important because there is very low reflectance across all water areas in this band, so it can interfere with the classification process.

We can now add the classified image to our map. Try visualising it with the following parameters:

![Classification layer visualisation parameters](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ClassVisParam.png)

Don't forget to click **import** so you can import your visualisation parameters as a variable and use them again in the future.

Your layer should look something like this:

![Initial classification output](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ClassInitialOutput.png)

Remember what the classes 0-3 represent in this image:

| Label | Cover class |
|-------|-------------|
|   0   | Unvegetated |
|   1   |   Zostera   |
|   2   |  Posidonia  |
|   3   |    Algae    |

Add your results and your original image to your map and symbolise them so that the data can be compared.

---

#### Cleaning up and analysing the classification results

These results look generally quite good, but there is some clear "speckling" effects across the image. We can correct this by applying a filter to the output data:
```javascript
var classifiedFiltered = classified.focal_mode();
```
This function offers a relatively simple way of simplifying classification results. More complex methods are available, but they are not necessary in this case.

Now, we can perform some analysis on our results. First, we should calculate an **error matrix**. This is a representation of the accuracy of our results, based on the **validation** data we set aside earlier.

We do this by running the validation data through the classifier, and then using the errorMatrix function:
```javascript
validation = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).sampleRegions(validation);
validation = validation.classify(RF);
var errorMatrix = validation.errorMatrix('label', 'classification');
print(errorMatrix,'Error Matrix');
```
This will print an error matrix in our console. Additionally, we can calculate overall accuracy and kappa coefficient for our data:
```javascript
var accuracy = errorMatrix.accuracy();
var kappa = errorMatrix.kappa();
print(accuracy,'Accuracy');
print(kappa,'Kappa coefficient');
```

Finally, we can calculate the overall area of each class in our study area. This script produces a simplke chart showing the total area (in square metres) of all the classes in our results:
```javascript
var areaChart = ui.Chart.image.byClass({
  image: ee.Image.pixelArea().addBands(classifiedFiltered),
  classBand: 'classification', 
  region: WallisLakeBoundary,
  scale: 10,
  reducer: ee.Reducer.sum()
});
print(areaChart,'Band sum values by class');
```

---

At the end, your imports should look like this:

![Final imports](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/FinalImports.png)

The remained of your script should look like this:
```javascript
//Setup
var WallisLake = ee.Geometry.Point([152.5, -32.28]);
Map.centerObject(WallisLake, 12);

//Access imagecollection
var S2 = ee.ImageCollection("COPERNICUS/S2_SR");

//Filter imagecollection
S2 = S2.filterBounds(WallisLake).filterDate('2018-12-01','2019-03-01').filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',10));

//Calculate median image
var S2median = S2.median()
    .setDefaultProjection('EPSG:4326', null, 10)
    .select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8','B11']);

//Calculate NDWI and create mask
var NDWI = S2median.normalizedDifference(['B3','B11']);
var water = NDWI.gte(0);

//Update median image mask
var S2masked = S2median.updateMask(water);
S2masked = S2masked.clip(WallisLakeBoundary);

//Divide ground truth data into training and validation
var groundTruth = GroundTruthData.randomColumn();
var training = groundTruth.filter(ee.Filter.gt('random', 0.4));
var validation = groundTruth.filter(ee.Filter.lte('random', 0.4));

//Classify image
var trainingRegions = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).sampleRegions(training);
var RF = ee.Classifier.smileRandomForest(100).train(trainingRegions, 'label');
var classified = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).classify(RF);

//Create simplified results for viewing
var classifiedFiltered = classified.focal_mode();

//Validate classification results
validation = S2masked.select(['B2', 'B3', 'B4', 'B5', 'B6', 'B7','B8']).sampleRegions(validation);
validation = validation.classify(RF);
var errorMatrix = validation.errorMatrix('label', 'classification');
print(errorMatrix,'Error Matrix');

//Calculate accuracy and kappa coefficient
var accuracy = errorMatrix.accuracy();
var kappa = errorMatrix.kappa();
print(accuracy,'Accuracy');
print(kappa,'Kappa coefficient');

//Calculate area of classes
var areaChart = ui.Chart.image.byClass({
  image: ee.Image.pixelArea().addBands(classified),
  classBand: 'classification', 
  region: WallisLakeBoundary,
  scale: 10,
  reducer: ee.Reducer.sum()
});
print(areaChart,'Band sum values by class');

//Add layers to map
Map.addLayer(S2masked,S2VisParam);
Map.addLayer(classified,classVisParam);
Map.addLayer(classifiedFiltered,classVisParam);
```
