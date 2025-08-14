# 📚 Documentation Utilisation Osmand

## ⚙️ Paramétrer Osmand

Grâce au **SDK Osmand**, il est possible de personnaliser facilement l’expérience de navigation en accédant aux paramètres de l’application.

**Mettre en place le SDK d'Osmand**

**Première étape :** Récupérer toutes les dépendances liés au SDK osmand et les mettres dans un module gradle. On utilisera ensuite ce module gradle pour le placer dans le module principal de IsmartNav.

**Deuxième étape :** Ajouter les librairies Java au bon fonctionnement du moteur de rendu OpenGL. Les placer dans le dossier racine de IsmartNavigation et créer un dossier "libs" et y placer les 4 librairies Java.

**Troisième étape :** Ajouter la carte à notre rendu xml via le code montré ci dessous : 

```xml
<net.osmand.plus.views.MapViewWithLayers
    android:id="@+id/map_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:layout_editor_absoluteX="0dp"
    tools:layout_editor_absoluteY="0dp" />
```

**Exemple : changer l’icône de navigation**

```java
osmandApplication.getSettings().NAVIGATION_ICON.set(LocationIcon.MOVEMENT_CAR.toString());
```

**Exemple : désactiver le centrage automatique hors `followMode`**

```java
osmandApplication.getSettings().AUTO_FOLLOW_ROUTE.set(0);
```

Toutes les personnalisations de carte et de navigation sont principalement gérées dans la **classe `MapController`**.

---

## 📍 Afficher un marker sur la carte

On utilise un **`MapMarkerBuilder`** fourni par Osmand pour créer un marker personnalisé.

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

**Exemple : retirer un marker selon son `baseOrder`**

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

## 📏 Afficher une ligne vectorielle (exemple : ligne verte)

On convertit une liste de positions **`LatLon`** en **`PointI`** afin qu’Osmand puisse afficher des tracés dynamiques en 2D/3D.

```java
QVectorPointI points = new QVectorPointI();
for (Position position : realTracePositions) {
    points.add(new PointI(
        MapUtils.get31TileNumberX(position.getLatLon().getLongitude()),
        MapUtils.get31TileNumberY(position.getLatLon().getLatitude())
    ));
}
```

Ensuite, on utilise un **builder** pour personnaliser l’affichage.

```java
trackLineBuilder
    .setBaseOrder(300000)
    .setIsHidden(false)
    .setLineWidth(30)
    .setPoints(points)
    .setEndCapStyle(VectorLine.EndCapStyle.ROUND.ordinal())
    .setFillColor(NativeUtilities.createFColorARGB(GREEN));
```

Pour des questions de performances les lignes vectorielles affichés sur la carte sont effacés à chaque fois qu'on en redessine sinon l'application cumule les lignes et finit pas être trop lente : 

```java
  public void removeRealTraceLinesCollection () {
        if (realTraceLinesCollection != null && mapTileView != null && mapTileView.getMapRenderer() != null) {
            mapTileView.getMapRenderer().removeSymbolsProvider(realTraceLinesCollection);
            realTraceLinesCollection.removeAllLines();
        }
    }
```

💡 Un `baseOrder` plus élevé permet d’afficher la ligne **au-dessus** des autres (exemple : ligne verte à 300100 au-dessus de la ligne bleue).

---

## 🧭 Lancer une navigation

### Navigation avec points intermédiaires

```java
targetPointsHelper.navigateToPoint(intermediate.getLatLon(), true, 1, pointDescription);
targetPointsHelper.navigateToPoint(destinationPosition.getLatLon(), true, -1, pointDescription);
launchNavigation(startPosition, pointDescription, true);
```

Le **`-1`** signifie que ce n’est **pas** un point intermédiaire.

**Exemple avec un deuxième point intermédiaire :**

```java
targetPointsHelper.navigateToPoint(intermediate2.getLatLon(), true, 2, pointDescription);
```

### Initialiser la navigation

Avant de démarrer, on ajuste certains paramètres :

```java
osmandSettings.FOLLOW_THE_ROUTE.set(true);
osmandSettings.AUTO_ZOOM_MAP.set(false);
routingHelper.setFollowingMode(true);
routingHelper.setRoutePlanningMode(false);
```

Démarrage avec les **`mapActions`** :

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

### Suivre le véhicule en mouvement

```java
mapViewTrackingUtilities.backToLocationImpl(20, true);
```

---

### Lancer l'activité de téléchargements de tuiles en mode Offline

```java
Intent intent = new Intent(this, osmandApplication.getAppCustomization().getDownloadActivity());
```

#### Au sein du manifest on déclare l'activité en question 
```xml
<activity android:name="net.osmand.plus.download.DownloadActivity" />
```

---

## 🗺️ Gestion des événements de carte 

### Passage en mode 2D/3D 

Pour gérer le mode 3D et 2D tout va se passer par rapport à l'élévation de la carte. Et pour modifier cette valeur on va simplement utiliser la méthode fournit par le SDK pour cette effet la : 

```java
  osmandMapTileView.setElevationAngle(25.0f); // passage en mode 3D
  osmandMapTileView.setElevationAngle(90.0f); // passage en mode 2D, vue au dessus du véhicule
```

### Détection d'un appui simple sur la carte

```java
mapTileView.addTouchListener(event -> {
    if (event.getAction() == MotionEvent.ACTION_DOWN) {
        onTouchOsmMap();
    }
});
```

---

## 🚚 Navigation avancée

### Affichage du temps, distance restant et nom rue actuelle

Ceci se passe dans la BasicMapActivity dans la méthode onLocationChanged fournit par LocationListener.
On utilisera principalement dans cette partie une instance de RoutingHelper qui nous permet de récupérer beaucoup d'information sur la navigation en cours.

**Récupérer le temps restant avant arrivé :**

```java
if (navigationController.isNavigationInProgress() && routingHelper.getLeftTime() != 0 && navStatus.getCircuit().isCollectable()) {
            navigationInfoBanner.setTimeToArrival(routingHelper.getLeftTime());
    }
```

**Récupérer le nom de la rue actuelle (plusieurs étapes):**

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

**Récupérer le type de la prochaine intersection :**

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
