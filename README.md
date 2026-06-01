# VietMap Navigation Android SDK — Integration Guide

This guide walks you through integrating the **VietMap Navigation SDK** into your Android app to build **turn-by-turn navigation** (real-time guidance, spoken turn announcements, automatic rerouting when the user goes off course, etc.).

By the end, you will be able to build a complete navigation screen on your own: fetch a route → draw it on the map → start navigation → receive trip events → finish.

---

## Table of Contents

1. [What you can build with this SDK](#1-what-you-can-build-with-this-sdk)
2. [Requirements & Installation](#2-requirements--installation)
3. [Integration flow overview](#3-integration-flow-overview)
4. [Step 1 — Show the map](#4-step-1--show-the-map)
5. [Step 2 — Create the navigation controller](#5-step-2--create-the-navigation-controller)
6. [Step 3 — Fetch a route](#6-step-3--fetch-a-route)
7. [Step 4 — Draw the route on the map](#7-step-4--draw-the-route-on-the-map)
8. [Step 5 — Start navigation](#8-step-5--start-navigation)
9. [Step 6 — Listen to trip events](#9-step-6--listen-to-trip-events)
10. [Step 7 — Stop & release](#10-step-7--stop--release)
11. [Core functions explained](#11-core-functions-explained)
12. [Customizing navigation behavior](#12-customizing-navigation-behavior)
13. [Managing the screen lifecycle](#13-managing-the-screen-lifecycle)
14. [Contact](#14-contact)

---

## 1. What you can build with this SDK

| Feature | Description |
| --- | --- |
| Route fetching | Find a route between two points (or multiple stops), with alternative routes. |
| Route rendering | Display the route, alternative routes, and turn arrows on the map. |
| Turn-by-turn navigation | Track the user's location in real time and show remaining distance/time. |
| Voice guidance | Read out turn instructions and guidance banners. |
| Automatic rerouting | When the user drifts off course, automatically find a better route. |
| Trip simulation | Run navigation without moving in real life (very handy for testing). |

Integration revolves around **two objects** you work with directly:

- **The navigation controller** — manages the navigation session, location, and trip events (progress, off-route, arrival, etc.).
- **The route renderer** — responsible for displaying the route on the map.

You fetch a route, hand it to the *route renderer* to display it, and hand it to the *controller* to start navigation.

---

## 2. Requirements & Installation

**Minimum requirements**
- `compileSdk` / `targetSdk` ≥ **33**
- `minSdk` ≥ **21**
- A **VietMap API key** and a map **style URL** (provided by [VietMap](https://maps.vietmap.vn/)).

### 2.1. Add the repository

`settings.gradle`:
```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### 2.2. Add the dependency

Pull from JitPack (in your app module's `build.gradle`):
```gradle
dependencies {
    // VietMap navigation SDK (logic + UI)
    implementation 'com.github.vietmap-company:maps-sdk-navigation-android:<version>'
    implementation 'com.github.vietmap-company:maps-sdk-navigation-ui-android:<version>'

    // Map SDK & utilities
    implementation 'org.maplibre.gl:android-sdk-services:5.9.0'
    implementation 'org.maplibre.gl:android-sdk-turf:5.9.0'

    // Common dependencies
    implementation 'com.google.android.gms:play-services-location:21.0.1'
    implementation 'androidx.recyclerview:recyclerview:1.2.1'
    implementation 'androidx.cardview:cardview:1.0.0'
    implementation 'androidx.lifecycle:lifecycle-extensions:2.2.0'
    implementation 'androidx.core:core-ktx:1.7.0'
    implementation 'com.squareup.picasso:picasso:2.8'
}
```

### 2.3. Declare permissions

`AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.VIBRATE" />
```

> Navigation runs in the background, so the `FOREGROUND_SERVICE` permission is required. You must **request location permission at runtime** before starting navigation; otherwise the SDK won't receive location updates.

---

## 3. Integration flow overview

```
 ┌──────────────────────────────────────────────────────────────┐
 │                          Your app                              │
 │                                                                │
 │  (1) Fetch route        (2) Draw route        (3) Navigate     │
 │  ───────────────►       ───────────────►      ───────────────► │
 │   Find a route           Display the           Start the       │
 │   between 2 points,      route on the          session, track  │
 │   get a list of routes   map                   location live   │
 │        │                      │                       │         │
 │        ▼                      ▼                       ▼         │
 │  [ Route fetcher ]    [ Route renderer ]    [ Nav controller ]  │
 │                                                       │         │
 │                                                       ▼         │
 │   (4) Receive events: progress • off-route • milestone •       │
 │       arrival  ──►  update UI / camera / voice                 │
 └──────────────────────────────────────────────────────────────┘
```

Lifecycle of a navigation session:

```
  Fetch route ──► Draw route ──► Register listeners ──► startNavigation()
                                                              │
                            ┌─────────────────────────────────┤
                            ▼                                 ▼
                    onProgressChange()                 userOffRoute()
                    (continuous updates)               (off-route)
                            │                                 │
                            │                                 ▼
                            │                       Fetch new route ──► startNavigation()
                            ▼
                       onArrival() (arrived)
                            │
                            ▼
                    stopNavigation() ──► onDestroy()
```

---

## 4. Step 1 — Show the map

Add a `MapView` to your Activity layout:
```xml
<vn.vietmap.vietmapsdk.maps.MapView
    android:id="@+id/mapView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Initialize and load a style in your Activity:
```java
mapView = findViewById(R.id.mapView);
mapView.onCreate(savedInstanceState);
mapView.getMapAsync(vietmapGL -> {
    this.vietmapGL = vietmapGL;
    vietmapGL.setStyle(new Style.Builder().fromUri("YOUR_STYLE_URL"), style -> {
        enableLocationComponent(style); // show the user's location dot
        // ready to create the route renderer (Step 4)
    });
});
```

> Replace `YOUR_STYLE_URL` with the map style URL provided by [VietMap](https://maps.vietmap.vn/).

---

## 5. Step 2 — Create the navigation controller

```java
import vn.vietmap.services.android.navigation.v5.navigation.VietmapNavigation;

// Simplest form — use the default configuration
VietmapNavigation navigation = new VietmapNavigation(this);
```

This controller manages all the logic of a navigation session. You use it to start/stop navigation and register listeners that receive trip events (see [Step 6](#9-step-6--listen-to-trip-events)).

> Want to customize behavior (off-route threshold, toggling announcements, simulation, etc.)? See [section 12](#12-customizing-navigation-behavior).

---

## 6. Step 3 — Fetch a route

Provide an `origin` and a `destination`:

```java
import vn.vietmap.services.android.navigation.v5.navigation.NavigationRoute;
import com.mapbox.geojson.Point;
import com.mapbox.api.directions.v5.models.DirectionsResponse;
import com.mapbox.api.directions.v5.models.DirectionsRoute;
import retrofit2.Call;
import retrofit2.Callback;
import retrofit2.Response;

Point origin      = Point.fromLngLat(106.675884, 10.759197);
Point destination = Point.fromLngLat(106.686777, 10.775056);

NavigationRoute.builder(this)
        .apikey("YOUR_VIETMAP_API_KEY")
        .profile("driving-traffic")   // car with traffic; also: driving, walking, cycling, motorcycle
        .origin(origin)
        .destination(destination)
        .alternatives(true)           // also fetch alternative routes
        .build()
        .getRoute(new Callback<DirectionsResponse>() {
            @Override
            public void onResponse(Call<DirectionsResponse> call, Response<DirectionsResponse> response) {
                if (response.body() == null || response.body().routes().isEmpty()) return;

                currentRoute = response.body().routes().get(0);   // primary route
                drawRoutes(response.body().routes());             // Step 4
            }

            @Override
            public void onFailure(Call<DirectionsResponse> call, Throwable t) {
                // handle network error / no route found
            }
        });
```

Commonly used route request parameters:

| Parameter | Meaning |
| --- | --- |
| `apikey(...)` | VietMap API key (**required**). |
| `profile(...)` | Vehicle type: `driving-traffic`, `driving`, `walking`, `cycling`, `motorcycle`. |
| `origin(...)` / `destination(...)` | Start point / end point. |
| `addWaypoint(...)` | Add an intermediate stop (call multiple times for multiple stops). |
| `alternatives(true/false)` | Whether to return alternative routes. |
| `language(...)` | Instruction language. |

---

## 7. Step 4 — Draw the route on the map

```java
import vn.vietmap.services.android.navigation.ui.v5.route.NavigationMapRoute;
```

> **Important import note:** make sure you pick the `NavigationMapRoute` from the package ending in **`...ui.v5.route`**. If the IDE suggests the wrong one, the route won't render. When in doubt, choose the package that contains **`ui`**.

Create the route renderer (do this **once**, after the map is ready):
```java
NavigationMapRoute mapRoute = new NavigationMapRoute(navigation, mapView, vietmapGL);

// Listen for when the user taps an alternative route to make it the primary
mapRoute.setOnRouteSelectionChangeListener(selectedRoute -> {
    currentRoute = selectedRoute;
});
```

Draw the route (called from the Step 3 callback):
```java
private void drawRoutes(List<DirectionsRoute> routes) {
    mapRoute.addRoutes(routes);   // the first element is the primary, the rest are alternatives
}
```

If you only want to draw a single route:
```java
mapRoute.addRoute(currentRoute);
```

---

## 8. Step 5 — Start navigation

```java
private void startNavigation() {
    if (currentRoute == null) return;

    // Register to receive trip events
    navigation.addProgressChangeListener(this);
    navigation.addOffRouteListener(this);
    navigation.addMilestoneEventListener(this);
    navigation.addNavigationEventListener(this);

    // Show the route + turn arrows; hide alternatives once moving
    mapRoute.updateRouteVisibilityTo(true);
    mapRoute.updateRouteArrowVisibilityTo(true);
    mapRoute.showAlternativeRoutes(false);

    // Go!
    navigation.startNavigation(currentRoute);
}
```

From this point on, the user's location is tracked in real time and your listeners start receiving events.

---

## 9. Step 6 — Listen to trip events

Have your Activity implement the listeners you need:

```java
public class NavigationActivity extends AppCompatActivity implements
        ProgressChangeListener,    // trip progress
        OffRouteListener,          // off-route
        MilestoneEventListener,    // voice/banner instructions
        NavigationEventListener {  // running/stopped state
}
```

**Trip progress** — called continuously whenever a new location arrives:
```java
@Override
public void onProgressChange(Location location, RouteProgress routeProgress) {
    double distanceRemaining = routeProgress.distanceRemaining(); // meters remaining
    double durationRemaining = routeProgress.durationRemaining(); // seconds remaining
    // Update the UI: distance, time, next turn, etc.
}
```

**Off-route** — triggers a new route fetch:
```java
@Override
public void userOffRoute(Location location) {
    Point newOrigin = Point.fromLngLat(location.getLongitude(), location.getLatitude());
    fetchRoute(newOrigin, destination); // re-fetch the route, then call startNavigation() with the new route
}
```

**Instructions (voice/banner):**
```java
@Override
public void onMilestoneEvent(RouteProgress routeProgress, String instruction, Milestone milestone) {
    // instruction holds the guidance text for the current step
}
```

**Navigation state:**
```java
@Override
public void onRunning(boolean running) {
    // running = true: the session has started; false: it has stopped
}
```

> To detect **arrival**, check in `onProgressChange` whether `distanceRemaining` is small enough, or rely on the `metersRemainingTillArrival` threshold (see [section 12](#12-customizing-navigation-behavior)).

---

## 10. Step 7 — Stop & release

When the user stops navigation or leaves the screen:
```java
private void stopNavigation() {
    navigation.stopNavigation();
    navigation.removeProgressChangeListener(this);
    navigation.removeOffRouteListener(this);
    navigation.removeMilestoneEventListener(this);
    navigation.removeNavigationEventListener(this);
}
```

In the Activity's `onDestroy()`, clean up thoroughly to avoid memory leaks:
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    navigation.onDestroy(); // stops navigation + removes all remaining listeners
    mapView.onDestroy();
}
```

> `navigation.onDestroy()` removes all listeners automatically, so you don't strictly need to remove them manually when destroying the screen.

---

## 11. Core functions explained

### Routing & navigation

| Function | Purpose |
| --- | --- |
| `NavigationRoute.builder(context)...build().getRoute(callback)` | Fetch a route between points and return the result via callback. |
| `startNavigation(route)` | Start a navigation session. Call it again with a new route to **update the route** when rerouting. |
| `stopNavigation()` | Stop the current navigation session. |
| `onDestroy()` | Release resources and remove all listeners. Call it when destroying the screen. |

### Receiving events

| Function | Purpose |
| --- | --- |
| `addProgressChangeListener(...)` | Receive progress updates (remaining distance/time, next step). |
| `addOffRouteListener(...)` | Get notified when the user drifts off course. |
| `addMilestoneEventListener(...)` | Receive voice/banner instructions at milestones. |
| `addNavigationEventListener(...)` | Receive the running/stopped state of the session. |
| `removeXxxListener(...)` | Remove the corresponding listener (pass `null` to remove all listeners of that type). |

### Displaying the route on the map

| Function | Purpose |
| --- | --- |
| `addRoute(route)` | Draw **one** route on the map. |
| `addRoutes(routes)` | Draw a list of routes — the first is primary, the rest are alternatives. |
| `updateRouteVisibilityTo(boolean)` | Show/hide the route. |
| `updateRouteArrowVisibilityTo(boolean)` | Show/hide turn arrows. |
| `showAlternativeRoutes(boolean)` | Show/hide alternative routes (usually hidden once moving). |
| `setOnRouteSelectionChangeListener(...)` | Get notified when the user taps an alternative route. |

---

## 12. Customizing navigation behavior

Pass a configuration when creating the controller:

```java
import vn.vietmap.services.android.navigation.v5.navigation.VietmapNavigationOptions;

VietmapNavigationOptions options = VietmapNavigationOptions.builder()
        .maximumDistanceOffRoute(50.0)          // off-route only beyond 50m
        .metersRemainingTillArrival(20.0)       // consider arrived within 20m
        .enableOffRouteDetection(true)          // enable automatic off-route detection
        .enableFasterRouteDetection(false)      // disable faster-route detection
        .defaultMilestonesEnabled(true)         // enable default voice/banner instructions
        .build();

VietmapNavigation navigation = new VietmapNavigation(this, options);
```

Commonly used options:

| Option | Meaning |
| --- | --- |
| `maximumDistanceOffRoute(double)` | Threshold (meters) for considering the user off-route. |
| `metersRemainingTillArrival(double)` | Threshold (meters) for considering the user arrived. |
| `secondsBeforeReroute(int)` | Wait time before rerouting. |
| `enableOffRouteDetection(boolean)` | Enable/disable automatic off-route detection. |
| `enableFasterRouteDetection(boolean)` | Enable/disable faster-route suggestions. |
| `defaultMilestonesEnabled(boolean)` | Enable/disable default voice/banner instructions. |
| `snapToRoute(boolean)` | Whether to "snap" the displayed location onto the route. |

### Simulating a trip (for testing)

To run a trip without moving in real life, use the mock location engine:
```java
import vn.vietmap.services.android.navigation.v5.location.replay.ReplayRouteLocationEngine;

ReplayRouteLocationEngine mockEngine = new ReplayRouteLocationEngine();
mockEngine.assign(currentRoute);
navigation.setLocationEngine(mockEngine);   // call before startNavigation()
```

---

## 13. Managing the screen lifecycle

The `MapView` and the navigation controller must be synchronized with the Activity/Fragment lifecycle to work correctly and avoid memory leaks:

```java
@Override protected void onStart()      { super.onStart();      mapView.onStart(); }
@Override protected void onResume()     { super.onResume();     mapView.onResume(); }
@Override protected void onPause()      { super.onPause();      mapView.onPause(); }
@Override protected void onStop()       { super.onStop();       mapView.onStop(); }
@Override public    void onLowMemory()  { super.onLowMemory();  mapView.onLowMemory(); }
@Override protected void onDestroy()    { super.onDestroy();    navigation.onDestroy(); mapView.onDestroy(); }

@Override
protected void onSaveInstanceState(Bundle outState) {
    super.onSaveInstanceState(outState);
    mapView.onSaveInstanceState(outState);
}
```

---

## 14. Contact

- 🌐 Website: [maps.vietmap.vn](https://maps.vietmap.vn/)
- 💬 Support via Zalo: [Zalo OA](https://zalo.me/3189066936017422854)
</content>
