# 📚 Osmand Usage Documentation

## ⚙️ Configuring Osmand

With the **Osmand SDK**, it is possible to easily customize the navigation experience by accessing the application's settings.

**Setting up the Osmand SDK**

**First step:** Retrieve all dependencies related to the Osmand SDK and place them in a Gradle module. This Gradle module will then be used inside the main module of IsmartNav.

**Second step:** Add the Java libraries required for the proper functioning of the OpenGL rendering engine. Place them in the root folder of IsmartNavigation, create a "libs" folder, and put the 4 Java libraries inside it.

**Third step:** Add the map to our XML layout via the code below:

```xml
<net.osmand.plus.views.MapViewWithLayers
    android:id="@+id/map_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:layout_editor_absoluteX="0dp"
    tools:layout_editor_absoluteY="0dp" />
```

**Example: Change the navigation icon**

```java
osmandApplication.getSettings().NAVIGATION_ICON.set(LocationIcon.MOVEMENT_CAR.toString());
```

**Example: Disable automatic centering when outside `followMode`**

```java
osmandApplication.getSettings().AUTO_FOLLOW_ROUTE.set(0);
```

---

## 📍 Display a marker on the map

We use an **`MapMarkerBuilder`** provided by Osmand to create a custom marker.

```java
MapMarkerBuilder mapMarkerBuilder = new MapMarkerBuilder();
mapMarkerBuilder
    .setMarkerId(id)
    .setPosition(point)
    .setIsHidden(false)
    .setBaseOrder(baseOrder)
    .setPinIcon(NativeUtilities.createSkImageFromBitmap(bitmap))
    .setPinIconVerticalAlignment(MapMarker.PinIconVerticalAlignment.CenterVertical)
    .setPinIconHorisontalAlignment(MapMarker.PinIconHorisontalAlignment.CenterHorizontal);

mapMarkerBuilder.buildAndAddToCollection(markersCollection);
```

**Example: Remove a marker based on its `baseOrder`**

```java
QListMapMarker markers = markersCollection.getMarkers();
List<MapMarker> toRemove = new ArrayList<>();

for (int i = 0; i < markers.size(); i++) {
    MapMarker marker = markers.get(i);
    if (marker != null && marker.getBaseOrder() == baseOrder) {
        toRemove.add(marker);
    } else {
        Log.i("MapMarkerManager", "removeMarker: marker with id " + marker.getMarkerId() + " has base order " + marker.getBaseOrder() + ", not removing");
    }
}

for (MapMarker marker : toRemove) {
    markersCollection.removeMarker(marker);
}
```

---

## 📏 Display a vector line (example: green line)

We convert a list of **`LatLon`** positions into **`PointI`** so that Osmand can display dynamic 2D/3D paths.

```java
QVectorPointI points = new QVectorPointI();
for (Position position : realTracePositions) {
    points.add(new PointI(
        MapUtils.get31TileNumberX(position.getLatLon().getLongitude()),
        MapUtils.get31TileNumberY(position.getLatLon().getLatitude())
    ));
}
```

Then, we use a **builder** to customize the display.

```java
trackLineBuilder
    .setBaseOrder(300000)
    .setIsHidden(false)
    .setLineWidth(30)
    .setPoints(points)
    .setEndCapStyle(VectorLine.EndCapStyle.ROUND.ordinal())
    .setFillColor(NativeUtilities.createFColorARGB(GREEN));
```

For performance reasons, vector lines displayed on the map are cleared each time we redraw them; otherwise, the application accumulates lines and eventually becomes too slow:

```java
  public void removeRealTraceLinesCollection () {
        if (realTraceLinesCollection != null && mapTileView != null && mapTileView.getMapRenderer() != null) {
            mapTileView.getMapRenderer().removeSymbolsProvider(realTraceLinesCollection);
            realTraceLinesCollection.removeAllLines();
        }
    }
```

💡 A higher `baseOrder` allows displaying the line **above** others (example: green line at 300100 above the blue line).

---

## 🧭 Start navigation

### Navigation with intermediate points

```java
targetPointsHelper.navigateToPoint(intermediate.getLatLon(), true, 1, pointDescription);
targetPointsHelper.navigateToPoint(destinationPosition.getLatLon(), true, -1, pointDescription);
launchNavigation(startPosition, pointDescription, true);
```

The **`-1`** means it is **not** an intermediate point.

**Example with a second intermediate point:**

```java
targetPointsHelper.navigateToPoint(intermediate2.getLatLon(), true, 2, pointDescription);
```

### Initialize navigation

Before starting, we adjust some settings:

```java
osmandSettings.FOLLOW_THE_ROUTE.set(true);
osmandSettings.AUTO_ZOOM_MAP.set(false);
routingHelper.setFollowingMode(true);
routingHelper.setRoutePlanningMode(false);
```

Start with **`mapActions`**:

```java
app.getOsmandMap().getMapActions().enterRoutePlanningModeGivenGpx(
    null,
    startPosition.getLatLon(),
    pointDescription,
    hasIntermediate,
    false
);
app.getOsmandMap().getMapActions().startNavigation();
```

### Follow the moving vehicle

```java
mapViewTrackingUtilities.backToLocationImpl(20, true);
```

---

### Launch the tile download activity in Offline mode

```java
Intent intent = new Intent(this, osmandApplication.getAppCustomization().getDownloadActivity());
```

#### In the manifest, declare the activity
```xml
<activity android:name="net.osmand.plus.download.DownloadActivity" />
```

---

## 🗺️ Map event handling

### Switch to 2D/3D mode

To manage 3D and 2D mode, everything is based on the map's elevation. To modify this value, simply use the method provided by the SDK:

```java
  osmandMapTileView.setElevationAngle(25.0f); // switch to 3D mode
  osmandMapTileView.setElevationAngle(90.0f); // switch to 2D mode, top view of the vehicle
```

### Detect a single tap on the map

```java
mapTileView.addTouchListener(event -> {
    if (event.getAction() == MotionEvent.ACTION_DOWN) {
        onTouchOsmMap();
    }
});
```

---

## 🚚 Advanced navigation

### Display remaining time, distance, and current street name

This is handled in the BasicMapActivity in the onLocationChanged method provided by LocationListener.
In this section, we will mainly use a RoutingHelper instance, which allows us to retrieve a lot of information about the current navigation.

**Get the remaining time before arrival:**

```java
if (navigationController.isNavigationInProgress() && routingHelper.getLeftTime() != 0 && navStatus.getCircuit().isCollectable()) {
            navigationInfoBanner.setTimeToArrival(routingHelper.getLeftTime());
    }
```

**Get the current street name (several steps):**

```java
List<RouteDirectionInfo> routeDirections = routingHelper.getRouteDirections();
RouteDirectionInfo currentDirection = routingHelper.getRoute() != null ? routingHelper.getRoute().getCurrentDirection() : null;
String currentStreetName = null;
```

```java
if (currentDirection != null) {
   currentStreetName = currentDirection.getStreetName();
}

if (currentStreetName != null && !currentStreetName.equals(lastStreetName)) {
    navigationInfoBanner.setManeuver(null, currentStreetName);
    lastStreetName = currentStreetName;
}
```

**Get the type of the next intersection:**

```java 
if (routingHelper.getRoute() != null && routingHelper.getRoute().getCurrentDirection() != null) {
    RouteCalculationResult route = routingHelper.getRoute();
    List<RouteDirectionInfo> directions = route.getRouteDirections();
    List<net.osmand.Location> locations = route.getImmutableAllLocations();

    if (directions != null && !directions.isEmpty() && locations != null) {
        for (RouteDirectionInfo dir : directions) {

            TurnType turnType = dir.getTurnType();
            int offset = dir.routePointOffset;

            if (turnType != null && offset > 0 && offset < locations.size()) {
                net.osmand.Location target = locations.get(offset);
                double distance = target.distanceTo(new net.osmand.Location("", location.getLatitude(), location.getLongitude()));
                navigationInfoBanner.setManeuver(turnType, route.getCurrentDirection().getStreetName());
                navigationInfoBanner.setDistanceToNextManeuver((long) distance);
                break;
            }
        }
    }
}
```
