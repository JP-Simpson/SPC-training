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
##### Getting started

Open GEE code editor to a new script. **Save** your script as **WallisLakeClassification** or similar. **Remember to save your script regularly so you do not lose any work**.

First, we will add a point to the map representing Wallis Lake, then centre our map on it.
```javascript
var WallisLake = ee.Geometry.Point([152.5, -32.28]);
Map.centerObject(WallisLake, 12);
```
For full explanations of these functions, see the mangrove dieback exercise.

Next, we will load two datasets into GEE - our field observations, and a study area boundary.
