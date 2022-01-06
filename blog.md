# Code it: creating hillshade effects using ArcGIS Runtime #

*This series of blog posts are intended for GIS professionals who have an interest in learning how to code geospatial workflows already familiar to them, using Esri's ArcGIS Runtime APIs*

![Image of hillshade renderer in Arran](HillshadeRendererOnArran.png)

The first cartographic technique I came across during my early career in geology was that of applying hillshades to digital elevation models, in order to make the dull, flat, greyscale raster **pop** into life as a realistic landscape with depth and dimension. Hillshade rendering uses highlights and shadows to create this 3D landscape, and is an invaluable tool in picking out subtle landscape features during the desk study phase of any geological investigation. It was one of my favourite tools when using ArcGIS Pro (well, ArcMap at the time!), with many an hour disappearing tweaking the settings in order to maximise landscape features that would later become the focus of a field survey: is that long sinuous feature an esker (marking the site of a river which once flowed beneath a glacier) and therefore worthy of a field study, or is it human-made and can therefore not of interest in a glacial superficial deposit field survey? When I changed careers and joined the ArcGIS Runtime for Java API team as a product engineer, I was delighted to discover much of the functionality I loved from Esri's desktop software was present in their Runtime offerings.

There are so many amazing blogs/tutorials out there already for hillshade rendering in Esri's flagship ArcGIS Pro, so I won't be stepping into detail about what a hillshade is, or the algorithms behind it in this blog. Instead, I want to share an alternative, coded approach to using ArcGIS Pro which achieves similar visually stunning hillshaded results using ArcGIS Runtime, and inspire GIS professionals interested in coding to give it a go.  I'll show how easy it is to load raster data into a desktop application using the ArcGIS Runtime for Java API, and how to apply a hillshade renderer to it and create a 3D digital visually realistic landscape. The results are as you would expect from ArcGIS Pro, but with the added fun of you get to code it up yourself and customise your application how ever you want.

## Data ##

For this blog, open source Scottish Public Sector LiDAR data from the Scottish Remote Sensing Portal (https://remotesensingdata.gov.scot/) was used, with the data made available under the Open Government Licence v3. These are incredibly detailed data sets covering parts of Scotland, including complete 1m coverage of the Isle of Arran: a geological playground with varied topography, that I couldn't wait to load it up in Runtime and see the results. I was really impressed this caliber of data was freely and openly available: check it out if you haven't already, there's a handy webmap interface where you can see the coverage, including 25cm resolution in parts of the Outer Hebrides.

## The Code ##

*If you're new to the ArcGIS Runtime APIs suite, you can find out more information here https://developers.arcgis.com/documentation/mapping-apis-and-services/apis-and-sdks/#native-apis, or to follow along using the ArcGIS Runtime for Java API, get started here https://developers.arcgis.com/java/*

After setting up the JavaFX UI component of a new Java application in your IDE of choice (read here for more details https://developers.arcgis.com/java/maps-2d/tutorials/display-a-map/#add-a-ui-for-the-map-view), create a `ArcGISScene` class along with a new `SceneView`. The `ArcGISScene` is the equivalent of a 3D scene in ArcGIS Pro, and hosts layers that can be displayed in 3D. The `SceneView` is the user interface that displays scene layers and graphics in 3D, and is the equivalent of the view pane in ArcGIS Pro.

```java
// create a scene and scene view
var scene = new ArcGISScene();
var sceneView = new SceneView(); // member variable
```

Next, bring the data (in geotiff format) into the application (you can either download your own data, or access the LiDAR tiles for Arran via a hosted open item on ArcGIS Online(https://arcgisruntime.maps.arcgis.com/home/item.html?id=ce99a45b9e664b4ebe3cb1cedf552b1d)).

```java
// get the GeoTiffs that contain 1m lidar digital terrain model (data copyright Scottish Government and SEPA (2014)).
List<String> tifFiles = new ArrayList<>(Arrays.asList(
  new File("./data/arran-lidar-data/NS02_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NS03_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NS04_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR82_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR83_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR84_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR93_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR92_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR94_1M_DTM_PHASE2.tif").getAbsolutePath(),
  new File("./data/arran-lidar-data/NR95_1M_DTM_PHASE2.tif").getAbsolutePath()
));
```

Now for the fun part! The hillshade renderer parameters are set up via the constructor for a new `HillshadeRenderer` class. If you are familiar with ArcGIS Pro's hillshade toolset, you'll recognise the same parameters here (https://pro.arcgis.com/en/pro-app/latest/tool-reference/3d-analyst/how-hillshade-works.htm).
- `altitude` - light's angle of elevation above the horizon, in degrees
- `azimuth` - light's relative angle along the horizon, in degrees; measured clockwise, 0 is north
- `zFactor` - factor to convert z unit to x,y units

I went for parameters which would mimic sunset conditions in November, to create a visually striking hillshade effect and emphasise the deepening shadows in the valleys of the Isle of Arran before the sun falls below the horizon.

```java
// create a new hillshade renderer that is representative of sunset conditions early November 2021 over Scotland
var hillshadeRenderer = new HillshadeRenderer(10, 225, 1); // altitude, azimuth, zFactor
```

This hillshade renderer instance can be applied to multiple raster layers, which we have to programatically make from the data. Loop over each geotiff, and for each one of them, create a new `Raster` from the geotiff, and then create a `RasterLayer` from the raster. A `Raster` class represents raster data that can be rendered using a `RasterLayer`, and in this demo's case can be created from a raster file on device, using `Raster(String)`. A `RasterLayer` is required to display raster data in the scene.

```java
// loop through the geotiffs
for (String tifFile : tifFiles) {

  // create a raster from every GeoTIFF
  var raster = new Raster(tifFile);
  // create a raster layer from the raster
  var rasterLayer = new RasterLayer(raster);
}
```

Now we can apply the hillshade renderer to the raster layers, and add them to the ArcGIS Scene's collection of operational layers in order to display them in the scene.

```java
// set a hillshade renderer to the raster layer
rasterLayer.setRasterRenderer(hillshadeRenderer);
// add the raster layer to the scene's operational layers
arcGISScene.getOperationalLayers().add(rasterLayer);
```

The hillshade renderer is applied, and the raster data is displayed on the scene: and looks great, just like you'd expect from the equivalent ArcGIS Pro hillshade toolset. Already features in the landscape are **popping** into life in the same exciting way I remember from my ArcGIS Pro days.

![Image of hillshade renderer over Goatfell, Arran](2DHillshadeRendererGoatFell.png)

The final twist in this bitesized coding tale is to make full use of the geotiff data and not only make the landscape *look* 3D, but actually **make** it 3D, by creating a 3D surface. We do this in Runtime by instantiating a new `RasterElevationSource` with the list of geotiffs provided as a parameter. Create a new `Surface`, get its list of elevation sources and add the `rasterElevationSource` to it. Finally, set the surface as the base surface of the ArcGIS Scene.

```java
// create an elevation source from the GeoTIFF (raster) collection
var rasterElevationSource = new RasterElevationSource(tifFiles);

// create a surface, get its elevation sources, and add the raster elevation source to the collection
var surface = new Surface();
surface.getElevationSources().add(rasterElevationSource);
// set the surface to the scene
arcGISScene.setBaseSurface(surface);
```

And tada, that's it! In only a few lines of code, a hillshade effect is rendered quickly and easily using the open source LiDAR data, and a 3D base surface created to provide a simple, high resolution 3D digital twin of the Isle of Arran as it would appear at sunset on a November evening. As when using Esri's desktop offerings, visual analysis of this data running in a native application is invaluable for desktop studies. With ArcGIS Runtime, you can customise this application even further and continue exploring the functionality the native APIs from Esri give you or bundle this up with your other favourite libraries.
