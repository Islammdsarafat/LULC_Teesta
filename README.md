# LULC_Teesta
#https://code.earthengine.google.com/f0f010052625cf03aacf60f94d2059ee
// Load Teesta River AOI from your asset
var studyArea = ee.FeatureCollection('projects/ee-islammdsarafatbuaa/assets/Teesta_AOI');
var aoi = studyArea.geometry();

// Date range: March 1–25, 2025
var START = ee.Date('2025-03-01');
var END = ee.Date('2025-03-25');

// Function to mask clouds in Sentinel-2 using Scene Classification Layer (SCL)
function maskS2clouds(image) {
  var scl = image.select('SCL');
  var cloudMask = scl.neq(9).and(scl.neq(8)).and(scl.neq(7)); // Removes clouds, cirrus, and shadows
  return image.updateMask(cloudMask).divide(10000);
}

// Get cloud-masked Sentinel-2 images
var s2Col = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(aoi)
  .filterDate(START, END)
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10))
  .map(maskS2clouds);

// Create a median composite of Sentinel-2 images for better visualization
var s2Median = s2Col.median();

// Get corresponding Dynamic World images
var dwCol = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
  .filterBounds(aoi)
  .filterDate(START, END);

// Link collections by finding the closest timestamp within a time window
var linkedCol = dwCol.map(function(dwImage) {
  var dwTime = ee.Date(dwImage.get('system:time_start'));
  var closest = s2Col.filter(ee.Filter.and(
    ee.Filter.gte('system:time_start', dwTime.advance(-1, 'day').millis()),
    ee.Filter.lte('system:time_start', dwTime.advance(1, 'day').millis())
  )).sort('system:time_start').first();

  // Ensure closest image is not null before selecting bands
  var s2Image = ee.Algorithms.If(
    closest,
    closest.select(['B4', 'B3', 'B2']).rename(['S2_R', 'S2_G', 'S2_B']),
    ee.Image.constant([0, 0, 0]).rename(['S2_R', 'S2_G', 'S2_B']).toByte()
  );

  return ee.Image.cat([
    ee.Image(s2Image),
    dwImage
  ]).copyProperties(dwImage, ['system:time_start', 'system:index']);
});

// Get the first linked image for display
var linkedImg = ee.Image(linkedCol.first());
print('Linked Image Bands:', linkedImg.bandNames());

// Visualization parameters
var CLASS_NAMES = [
  'water', 'trees', 'grass', 'flooded_vegetation', 'crops',
  'shrub_and_scrub', 'built', 'bare', 'snow_and_ice'
];

var VIS_PALETTE = [
  '419bdf', '397d49', '88b053', '7a87c6', 'e49635', 'dfc35a', 'c4281b',
  'a59b8f', 'b39fe1'
];

// Create Dynamic World visualization
var dwRgb = linkedImg
  .select('label')
  .visualize({min: 0, max: 8, palette: VIS_PALETTE})
  .divide(255);

var top1Prob = linkedImg.select(CLASS_NAMES).reduce(ee.Reducer.max());
var top1ProbHillshade = ee.Terrain.hillshade(top1Prob.multiply(100)).divide(255);
var dwRgbHillshade = dwRgb.multiply(top1ProbHillshade);

// Display layers
Map.centerObject(aoi, 10);
Map.addLayer(s2Median.select(['B4', 'B3', 'B2']), {min: 0, max: 0.3}, 'Sentinel-2 Median Composite');
Map.addLayer(dwRgbHillshade, {min: 0, max: 0.65}, 'Dynamic World - Label Hillshade');
Map.addLayer(aoi, {color: 'FF0000'}, 'Teesta AOI', false);
// Define the export task for the linked image
Export.image.toDrive({
  image: linkedImg.select(['S2_R', 'S2_G', 'S2_B']),  // Selecting RGB bands
  description: 'linked_image_export',
  scale: 10,  // Set scale based on Sentinel-2 resolution (10 meters for RGB bands)
  region: aoi,  // Set the region of interest
  fileFormat: 'GeoTIFF',  // Export as GeoTIFF
  maxPixels: 1e8  // Set maximum pixels to avoid exporting too much data
});
