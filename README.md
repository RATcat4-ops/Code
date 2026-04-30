Regualting Service- Carbon Stock 
// TADOBA ANDHARI TIGER RESERVE - FINAL BALANCED SCRIPT 
// One more precision adjustment to hit ISFR targets 
// 1. LOAD STUDY AREA  
var tatrBoundary = ee.FeatureCollection('projects/ndvi-464315/assets/TATR_buffer'); 
var geometry = tatrBoundary.geometry(); 
Map.centerObject(geometry, 11); 
Map.addLayer(tatrBoundary, {color: 'yellow', fillColor: '00000000'}, 'TATR Boundary'); 
print('=== TADOBA ANDHARI TIGER RESERVE ==='); 
print('Total Area (sq km):', geometry.area().divide(1e6)); 
print('Total Area (ha):', geometry.area().divide(10000)); 
// ISFR 2021 Targets 
var ISFR_TARGETS = { 
VDF: 655.47, 
MDF: 444.18, 
OF: 162.00, 
TOTAL: 1261.65 
}; 
print('ISFR 2021 Targets:'); 
print('  VDF:', ISFR_TARGETS.VDF, 'sq km'); 
print('  MDF:', ISFR_TARGETS.MDF, 'sq km'); 
print('  OF:', ISFR_TARGETS.OF, 'sq km'); 
// 2. CARBON PARAMETERS 
var params = { 
vdfCarbon: 112.19, 
mdfCarbon: 85.00, 
ofCarbon: 45.00, 
scrubCarbon: 15.00, 
litterPercent: 1.48, 
deadwoodPercent: 0.78, 
socPercent: 56.18, 
rotationPeriod: 75, 
co2Conversion: 3.6667, 
sccMin: 37.17, 
sccMax: 86.00, 
}; 
// 3. LOAD SENTINEL-2 DATA 
print('=== Loading Sentinel-2 Data (Oct-Dec 2021) ==='); 
var sentinel = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED') 
.filterBounds(geometry) 
.filterDate('2021-10-01', '2021-12-31') 
.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 20)) 
.map(function(image) { 
return image.divide(10000).clip(geometry); 
}) 
.median(); 
var ndvi = sentinel.normalizedDifference(['B8', 'B4']).rename('NDVI'); 
var ndviStats = ndvi.reduceRegion({ 
reducer: ee.Reducer.mean().combine(ee.Reducer.minMax(), null, true), 
geometry: geometry, 
scale: 100, 
bestEffort: true 
}); 
ndviStats.evaluate(function(stats) { 
print('NDVI Mean:', stats.NDVI_mean); 
print('NDVI Max:', stats.NDVI_max); 
print('NDVI Min:', stats.NDVI_min); 
}); 
Map.addLayer(ndvi, {min: 0, max: 1, palette: ['red', 'yellow', 'green']}, 'NDVI'); 
// 4. FINAL BALANCED THRESHOLDS 
// Based on your results (VDF=450, MDF=844, OF=80, Total=1,373): 
// - Lower VDF threshold to capture more VDF (from 0.72 to 0.70) 
// - Raise MDF lower bound to reduce MDF (from 0.55 to 0.60) 
// - Lower OF upper bound to allow more OF (from 0.55 to 0.60, but OF needs more area) 
//   Actually OF needs to increase from 80 to 162, so we need to expand OF range 
var vdfThreshold = 0.70;    // Lowered from 0.72 to capture more VDF 
var mdfLowThreshold = 0.60; // Raised from 0.55 to reduce MDF 
var mdfHighThreshold = 0.70; // Upper bound for MDF 
var ofLowThreshold = 0.48;  // Lowered from 0.50 to capture more OF 
var ofHighThreshold = 0.60; // Raised from 0.55 to capture more OF 
var scrubLowThreshold = 0.25; 
var scrubHighThreshold = 0.48; 
// Apply masks 
var vdfMask = ndvi.gt(vdfThreshold); 
var mdfMask = ndvi.gt(mdfLowThreshold).and(ndvi.lte(mdfHighThreshold)); 
var ofMask = ndvi.gt(ofLowThreshold).and(ndvi.lte(ofHighThreshold)); 
var scrubMask = ndvi.gt(scrubLowThreshold).and(ndvi.lte(scrubHighThreshold)); 
var nonForest = ndvi.lte(scrubLowThreshold); 
// Create density class 
var densityClass = ee.Image(0) 
.where(vdfMask, 1) 
.where(mdfMask, 2) 
.where(ofMask, 3) 
.where(scrubMask, 4) 
.where(nonForest, 5); 
// Assign carbon density 
var carbonDensity = ee.Image(0) 
.where(vdfMask, params.vdfCarbon) 
.where(mdfMask, params.mdfCarbon) 
.where(ofMask, params.ofCarbon) 
.where(scrubMask, params.scrubCarbon) 
.where(nonForest, 0) 
.rename('Carbon_Density_MgC_ha'); 
// 5. EDGE EFFECTS 
var forestMask = ndvi.gt(ofLowThreshold); 
var distanceToEdge = forestMask.fastDistanceTransform().sqrt().multiply(20); 
var edgeBuffer = distanceToEdge.lte(100); 
var edgeReduction = edgeBuffer.multiply(0.85).add(edgeBuffer.not()); 
var carbonWithEdge = carbonDensity.multiply(edgeReduction); 
// 6. DOM AND SOIL CARBON 
var vegetationCarbon = carbonWithEdge; 
var litterCarbon = vegetationCarbon.multiply(params.litterPercent / 100); 
var deadwoodCarbon = vegetationCarbon.multiply(params.deadwoodPercent / 100); 
var soilCarbon = vegetationCarbon.multiply(params.socPercent / 100); 
var totalEcosystemCarbon = vegetationCarbon 
.add(litterCarbon) 
.add(deadwoodCarbon) 
.add(soilCarbon) 
.rename('Total_Ecosystem_Carbon_MgC_ha'); 
// 7. AREA CALCULATIONS 
var areaImage = ee.Image.pixelArea().divide(10000); 
var vdfArea = vdfMask.multiply(areaImage).reduceRegion({ 
reducer: ee.Reducer.sum(), 
geometry: geometry, 
scale: 30, 
maxPixels: 1e11, 
bestEffort: true 
}); 
var mdfArea = mdfMask.multiply(areaImage).reduceRegion({ 
reducer: ee.Reducer.sum(), 
geometry: geometry, 
scale: 30, 
maxPixels: 1e11, 
bestEffort: true 
}); 
var ofArea = ofMask.multiply(areaImage).reduceRegion({ 
reducer: ee.Reducer.sum(), 
geometry: geometry, 
scale: 30, 
maxPixels: 1e11, 
bestEffort: true 
}); 
var totalCarbonImage = totalEcosystemCarbon.multiply(areaImage); 
var totalCarbonSum = totalCarbonImage.reduceRegion({ 
reducer: ee.Reducer.sum(), 
geometry: geometry, 
scale: 30, 
maxPixels: 1e11, 
bestEffort: true 
}); 
// 8. DISPLAY 
var densityPalette = ['#004d00', '#228B22', '#90EE90', '#CD853F', '#4682B4', '#FF6347']; 
Map.addLayer(densityClass, {min: 1, max: 5, palette: densityPalette}, 'Balanced Forest Density'); 
Map.addLayer(totalEcosystemCarbon, {min: 0, max: 150, palette: ['brown', 'yellow', 'green']}, 'Total Carbon'); 
var vdfHa = ee.Number(vdfArea.get('NDVI')); 
var mdfHa = ee.Number(mdfArea.get('NDVI')); 
var ofHa = ee.Number(ofArea.get('NDVI')); 
vdfHa.evaluate(function(vdf) { 
mdfHa.evaluate(function(mdf) { 
ofHa.evaluate(function(of) { 
var vdfSqKm = vdf / 100; 
var mdfSqKm = mdf / 100; 
var ofSqKm = of / 100; 
var totalSqKm = vdfSqKm + mdfSqKm + ofSqKm; 
totalCarbonSum.evaluate(function(result) { 
var totalCarbon = result.Total_Ecosystem_Carbon_MgC_ha || 0; 
// 10. EXPORT 
Export.image.toDrive({ 
image: totalEcosystemCarbon, 
description: 'TATR_Balanced_Carbon_2021', 
folder: 'GEE_Exports', 
region: geometry, 
scale: 10, 
maxPixels: 1e10, 
crs: 'EPSG:4326' 
}); 
Export.image.toDrive({ 
image: densityClass, 
description: 'TATR_Balanced_Density_2021', 
folder: 'GEE_Exports', 
region: geometry, 
scale: 10, 
maxPixels: 1e10, 
crs: 'EPSG:4326' 
}); 
