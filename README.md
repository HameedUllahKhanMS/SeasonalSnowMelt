var roi = table; // Replace 'table' with your actual ROI
var startDate = '2018-06-01'; // Start date
var endDate = '2018-07-31'; // End date
var sentinel2 = ee.ImageCollection('COPERNICUS/S2')
                  .filterDate(startDate, endDate)
                  .filterBounds(roi)
                  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20));

function addNDSI(image) {
  var ndsi = image.normalizedDifference(['B3', 'B11']).rename('NDSI');
  return image.addBands(ndsi);
}

var sentinel2WithNDSI = sentinel2.map(addNDSI);
var medianImage = sentinel2WithNDSI.median().clip(roi);
Map.centerObject(roi, 10);
Map.addLayer(medianImage.select('NDSI'), {min: 0, max: 1, palette: ['blue', 'white']}, 'Median NDSI');
var snowThreshold = 0.4; // Adjust this threshold based on your analysis
var snowIce = medianImage.select('NDSI').gt(snowThreshold).selfMask();
Map.addLayer(snowIce, {palette: 'cyan'}, 'Snow/Ice');
var pixelArea = snowIce.multiply(ee.Image.pixelArea());
var glacierArea = pixelArea.reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: roi,
  scale: 10,
  maxPixels: 1e9
});
glacierArea.get('NDSI').evaluate(function(area) {
  print('Glacier Area (square meters):', area);
});

Export.image.toDrive({
  image: medianImage.select('NDSI'),
  description: 'Median_NDSI',
  scale: 10,
  region: roi,
  fileFormat: 'GeoTIFF',
  maxPixels: 1e9
});

// Export the classified snow/ice layer to Google Drive
Export.image.toDrive({
  image: snowIce,
  description: 'Snow_Ice',
  scale: 10,
  region: roi,
  fileFormat: 'GeoTIFF',
  maxPixels: 1e9
});
