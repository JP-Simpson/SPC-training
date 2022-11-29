# Classifying seagrass using Sentinel-2 data, field observations and Google Earth Engine

This exercise will demonstrate another capability of Google Earth Engine (GEE): image classification. This will be introduced in the context of mapping two critical species of seagrass in Wallis Lake, NSW, Australia. This will use similar techniques to the previous GEE exercise, as well as demonstrating how limited field observation data can be extended across a large area using remote sensing analysis.

### About classification
Image classification is the process of placing remote sensing image pixels into categories based on spectral information, and in some cases other information. There are a wide array of different classification approaches and algorithms, but they can broadly be divided into three categories:
1. **Unsupervised classification** - Unsupervised classification sorts pixels into classes based only on spectral variation between pixels. Unsupervised classification approaches are not trained (i.e. they do not use field observations to relate spectral signatures to features). They are often used in initial analysis to identify spectral variation, or for analysis of very large areas where ground truth data is unavailable.
2. **Supervised classification** - Supervised classification uses pixels, areas or features identified by a user as training data to tell the classification algorithm what those features are expected to look like. This allows the algorithm to sort image pixels into categories based on the users' identification of a limited number of training areas.
3. **Object-Based Image Analysis** - Object-based Image Analysis (OBIA) separates an image into clusters of similar pixels, before placing each cluster into a class based on its spectral properties, and in some cases, other characteristics such as shape. This is useful as it can provide a more realistic representation of the real world, which is not divided into pixels like a satellite image.

We are going to apply approach 2, supervised classification. This allows us, in simple terms, to use limited field observations to map a much larger area. GEE can also be used for unsupervised or OBIA classification.

### About the field site
Wallis Lake is a barrier estuary in NSW, approximately 200 km north-northeast of Sydney. It covers a total area of about 100 km<sup>2</sup> and is about 25 km long. Two towns, Forster and Tuncurry, are at the mouth of the estuary where it meets the Pacific Ocean. Much of the rest of the lake is bordered by rural and bushland areas, including two national parks: Wallingat National Park to the west, and Booti Booti National Park to the east.

As well as being popular for recreation, camping, and fishing, Wallis Lake is home to some of NSW's most extensive seagrass beds. Seagrasses are flower plants which can grow submerged in marine water. They play a significant ecological role in coastal areas, acting as habitat and shelter for marine fauna, preventing erosion, sequestering CO<sub>2</sub> from the atmosphere, cleaning water, and acting as a fertile substrate for aquaculture.

Two seagrass species are widespread within Wallis Lake: *Zostera capricorni* and *Posidonia australis*.

*Z. capricorni* is a small, common, bright green, fast-growing species which grows in extensive beds all along the NSW coast. *P. australis* is larger, darker in colour, and grows in smaller patches. Various species of algae also grow in the estuary.

We will use supervised image classification to distinguish areas of each of these species, as well as areas where the estuary floor is bare of vegetation, and areas with algae present.

### Exercise instructions
#### Getting started

Open GEE code editor to a new script. **Save** your script as **WallisLakeClassification** or similar. **Remember to save your script regularly so you do not lose any work**.

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

These assets can now be added into your script by simply clicking the **Import** arrow.

![Import assets into script](https://github.com/JP-Simpson/SPC-training/blob/main/tutorial%20assets/ImportIntoScript.png)

#### Accessing Sentinel-2 data
Sentinel-2 refers to a European Space Agency (ESA) Earth Observation mission involving sensors mounted on two satellites, called Sentinel-2A MSI and Sentinel-2B MSI. It provides remote sensing data in the visible, near-infrared, and shortwave infrared parts of the spectrum, at a spatial resolution ranging from 10m to 60m depending upon spectral band. You can read more about the Sentinel-2 mission [here](https://sentinel.esa.int/web/sentinel/missions/sentinel-2).

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

The resulting variable **S2** contains seven images which fit all these criteria.

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
var water = NDWI.gte(0.05);
```
This is a binary layer, with all areas with an NDWI greater than or equal to 0.05 equal to 1, and all areas with an NDWI less than 0.05 equal to 0.

Now update the mask on our Sentinel-2 image:
```javascript
var S2masked = S2median.updateMask(water);
```
The result of this will include only water areas. However, we have also included a large amount irrelevant water in our image, as Wallis Lake is located near the open ocean. We can fix this using the study area dataset we added earlier:
```javascript
S2masked = S2masked.clip(WallisLakeBoundary);
```


##### Dividing ground truth data into training and validation datasets

##### Training the classifier and classifying the image

##### Cleaning up and analysing the classification results
