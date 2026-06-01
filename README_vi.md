# VietMap Navigation Android SDK — Hướng dẫn tích hợp

Tài liệu này hướng dẫn bạn tích hợp **VietMap Navigation SDK** vào ứng dụng Android để xây dựng tính năng **dẫn đường turn-by-turn** (chỉ đường theo thời gian thực, thông báo rẽ bằng giọng nói, tự tìm đường lại khi đi sai…).

---

## Mục lục

1. [Bạn làm được gì với SDK này](#1-bạn-làm-được-gì-với-sdk-này)
2. [Yêu cầu & Cài đặt](#2-yêu-cầu--cài-đặt)
3. [Luồng tích hợp tổng quan](#3-luồng-tích-hợp-tổng-quan)
4. [Bước 1 — Hiển thị bản đồ](#4-bước-1--hiển-thị-bản-đồ)
5. [Bước 2 — Khởi tạo bộ điều khiển dẫn đường](#5-bước-2--khởi-tạo-bộ-điều-khiển-dẫn-đường)
6. [Bước 3 — Tìm một tuyến đường](#6-bước-3--tìm-một-tuyến-đường)
7. [Bước 4 — Vẽ tuyến lên bản đồ](#7-bước-4--vẽ-tuyến-lên-bản-đồ)
8. [Bước 5 — Bắt đầu dẫn đường](#8-bước-5--bắt-đầu-dẫn-đường)
9. [Bước 6 — Lắng nghe sự kiện hành trình](#9-bước-6--lắng-nghe-sự-kiện-hành-trình)
10. [Bước 7 — Kết thúc & giải phóng](#10-bước-7--kết-thúc--giải-phóng)
11. [Giải thích các hàm cơ bản](#11-giải-thích-các-hàm-cơ-bản)
12. [Tùy chỉnh hành vi dẫn đường](#12-tùy-chỉnh-hành-vi-dẫn-đường)
13. [Quản lý vòng đời màn hình](#13-quản-lý-vòng-đời-màn-hình)
14. [Câu hỏi thường gặp](#14-câu-hỏi-thường-gặp)
15. [Liên hệ](#15-liên-hệ)

---

## 1. Bạn làm được gì với SDK này

| Tính năng | Mô tả |
| --- | --- |
| Tìm tuyến đường | Tìm đường giữa hai điểm (hoặc nhiều điểm dừng), kèm các tuyến thay thế. |
| Vẽ tuyến lên bản đồ | Hiển thị đường đi, tuyến thay thế và mũi tên báo rẽ. |
| Dẫn đường turn-by-turn | Bám vị trí người dùng theo thời gian thực, hiển thị khoảng cách/thời gian còn lại. |
| Thông báo bằng giọng nói | Đọc chỉ dẫn rẽ, banner hướng dẫn. |
| Tự tìm đường lại | Khi người dùng đi chệch tuyến, tự động tìm tuyến mới phù hợp. |
| Mô phỏng hành trình | Chạy thử dẫn đường mà không cần di chuyển thật (rất tiện khi test). |

Việc tích hợp xoay quanh **hai đối tượng** bạn sẽ làm việc trực tiếp:

- **Bộ điều khiển dẫn đường** — quản lý phiên dẫn đường, vị trí, sự kiện hành trình (tiến trình, đi sai đường, đến nơi…).
- **Bộ vẽ tuyến** — chịu trách nhiệm hiển thị đường đi lên bản đồ.

Bạn lấy tuyến đường, đưa cho *bộ vẽ tuyến* để hiển thị và đưa cho *bộ điều khiển* để bắt đầu dẫn đường.

---

## 2. Yêu cầu & Cài đặt

**Yêu cầu tối thiểu**
- `compileSdk` / `targetSdk` ≥ **33**
- `minSdk` ≥ **21**
- Một **API key VietMap** và một **style URL** bản đồ (do [VietMap](https://maps.vietmap.vn/) cung cấp).

### 2.1. Thêm repository

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

### 2.2. Thêm dependency

Tải qua JitPack (`build.gradle` của module app):
```gradle
dependencies {
    // SDK navigation của VietMap (logic + UI)
    implementation 'com.github.vietmap-company:maps-sdk-navigation-android:<version>'
    implementation 'com.github.vietmap-company:maps-sdk-navigation-ui-android:<version>'

    // Map SDK & tiện ích
    implementation 'org.maplibre.gl:android-sdk-services:5.9.0'
    implementation 'org.maplibre.gl:android-sdk-turf:5.9.0'

    // Phụ thuộc thường dùng
    implementation 'com.google.android.gms:play-services-location:21.0.1'
    implementation 'androidx.recyclerview:recyclerview:1.2.1'
    implementation 'androidx.cardview:cardview:1.0.0'
    implementation 'androidx.lifecycle:lifecycle-extensions:2.2.0'
    implementation 'androidx.core:core-ktx:1.7.0'
    implementation 'com.squareup.picasso:picasso:2.8'
}
```

### 2.3. Khai báo quyền

`AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.VIBRATE" />
```

> Dẫn đường chạy ở chế độ nền nên cần quyền `FOREGROUND_SERVICE`. Bạn phải **xin quyền vị trí lúc chạy (runtime permission)** trước khi bắt đầu dẫn đường, nếu không SDK sẽ không nhận được vị trí.

---

## 3. Luồng tích hợp tổng quan

```
 ┌──────────────────────────────────────────────────────────────┐
 │                     Ứng dụng của bạn                           │
 │                                                                │
 │  (1) Tìm tuyến          (2) Vẽ tuyến          (3) Dẫn đường     │
 │  ───────────────►       ───────────────►      ───────────────► │
 │   Tìm đường giữa         Hiển thị đường         Bắt đầu phiên    │
 │   2 điểm, nhận về        đi lên bản đồ          dẫn đường, bám   │
 │   danh sách tuyến                               vị trí real-time │
 │        │                      │                       │         │
 │        ▼                      ▼                       ▼         │
 │  [ Bộ tìm tuyến ]      [ Bộ vẽ tuyến ]       [ Bộ điều khiển ]   │
 │                                                       │         │
 │                                                       ▼         │
 │   (4) Nhận sự kiện: tiến trình • đi sai đường • milestone •     │
 │       đến nơi  ──►  cập nhật UI / camera / giọng nói            │
 └──────────────────────────────────────────────────────────────┘
```

Vòng đời một phiên dẫn đường:

```
  Tìm tuyến ──► Vẽ tuyến ──► Đăng ký listener ──► startNavigation()
                                                        │
                          ┌─────────────────────────────┤
                          ▼                             ▼
                  onProgressChange()             userOffRoute()
                  (liên tục cập nhật)            (đi sai đường)
                          │                             │
                          │                             ▼
                          │                     Tìm tuyến mới ──► startNavigation()
                          ▼
                     onArrival() (đến nơi)
                          │
                          ▼
                  stopNavigation() ──► onDestroy()
```

---

## 4. Bước 1 — Hiển thị bản đồ

Thêm `MapView` vào layout của Activity:
```xml
<vn.vietmap.vietmapsdk.maps.MapView
    android:id="@+id/mapView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Khởi tạo và load style trong Activity:
```java
mapView = findViewById(R.id.mapView);
mapView.onCreate(savedInstanceState);
mapView.getMapAsync(vietmapGL -> {
    this.vietmapGL = vietmapGL;
    vietmapGL.setStyle(new Style.Builder().fromUri("YOUR_STYLE_URL"), style -> {
        enableLocationComponent(style); // bật hiển thị chấm vị trí người dùng
        // sẵn sàng để tạo bộ vẽ tuyến (Bước 4)
    });
});
```

> Thay `YOUR_STYLE_URL` bằng style URL bản đồ [VietMap](https://maps.vietmap.vn/) cấp cho bạn.

---

## 5. Bước 2 — Khởi tạo bộ điều khiển dẫn đường

```java
import vn.vietmap.services.android.navigation.v5.navigation.VietmapNavigation;

// Cách đơn giản nhất — dùng cấu hình mặc định
VietmapNavigation navigation = new VietmapNavigation(this);
```

Bộ điều khiển này quản lý toàn bộ logic của phiên dẫn đường. Bạn sẽ dùng nó để bắt đầu/kết thúc dẫn đường và đăng ký các listener nhận sự kiện hành trình (xem [Bước 6](#9-bước-6--lắng-nghe-sự-kiện-hành-trình)).

> Muốn tùy chỉnh hành vi (ngưỡng đi sai đường, bật/tắt thông báo, mô phỏng…)? Xem [mục 12](#12-tùy-chỉnh-hành-vi-dẫn-đường).

---

## 6. Bước 3 — Tìm một tuyến đường

Cung cấp điểm đi (`origin`) và điểm đến (`destination`):

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
        .profile("driving-traffic")   // xe hơi có traffic; còn: driving, walking, cycling, motorcycle
        .origin(origin)
        .destination(destination)
        .alternatives(true)           // lấy thêm các tuyến thay thế
        .build()
        .getRoute(new Callback<DirectionsResponse>() {
            @Override
            public void onResponse(Call<DirectionsResponse> call, Response<DirectionsResponse> response) {
                if (response.body() == null || response.body().routes().isEmpty()) return;

                currentRoute = response.body().routes().get(0);   // tuyến chính
                drawRoutes(response.body().routes());             // Bước 4
            }

            @Override
            public void onFailure(Call<DirectionsResponse> call, Throwable t) {
                // xử lý lỗi mạng / không tìm được tuyến
            }
        });
```

Các tham số hay dùng khi tìm tuyến:

| Tham số | Ý nghĩa |
| --- | --- |
| `apikey(...)` | API key VietMap (**bắt buộc**). |
| `profile(...)` | Loại phương tiện: `driving-traffic`, `driving`, `walking`, `cycling`, `motorcycle`. |
| `origin(...)` / `destination(...)` | Điểm đi / điểm đến. |
| `addWaypoint(...)` | Thêm điểm dừng trung gian (gọi nhiều lần cho nhiều điểm). |
| `alternatives(true/false)` | Có trả về tuyến thay thế hay không. |
| `language(...)` | Ngôn ngữ chỉ dẫn. |

---

## 7. Bước 4 — Vẽ tuyến lên bản đồ

```java
import vn.vietmap.services.android.navigation.ui.v5.route.NavigationMapRoute;
```

> **Lưu ý quan trọng khi import:** hãy chọn đúng `NavigationMapRoute` thuộc package kết thúc bằng **`...ui.v5.route`**. Nếu IDE gợi ý nhầm, code sẽ không vẽ được tuyến. Khi không chắc, chọn gói có chữ **`ui`**.

Tạo bộ vẽ tuyến (làm **một lần** sau khi bản đồ sẵn sàng):
```java
NavigationMapRoute mapRoute = new NavigationMapRoute(navigation, mapView, vietmapGL);

// Lắng nghe khi người dùng chạm chọn một tuyến thay thế làm tuyến chính
mapRoute.setOnRouteSelectionChangeListener(selectedRoute -> {
    currentRoute = selectedRoute;
});
```

Vẽ tuyến (gọi từ callback ở Bước 3):
```java
private void drawRoutes(List<DirectionsRoute> routes) {
    mapRoute.addRoutes(routes);   // phần tử đầu là tuyến chính, còn lại là tuyến thay thế
}
```

Nếu chỉ muốn vẽ đúng một tuyến:
```java
mapRoute.addRoute(currentRoute);
```

---

## 8. Bước 5 — Bắt đầu dẫn đường

```java
private void startNavigation() {
    if (currentRoute == null) return;

    // Đăng ký nhận sự kiện hành trình
    navigation.addProgressChangeListener(this);
    navigation.addOffRouteListener(this);
    navigation.addMilestoneEventListener(this);
    navigation.addNavigationEventListener(this);

    // Hiển thị tuyến + mũi tên rẽ; ẩn các tuyến thay thế khi đã đi
    mapRoute.updateRouteVisibilityTo(true);
    mapRoute.updateRouteArrowVisibilityTo(true);
    mapRoute.showAlternativeRoutes(false);

    // Bắt đầu!
    navigation.startNavigation(currentRoute);
}
```

Từ lúc này, vị trí người dùng được bám theo thời gian thực và các listener bắt đầu nhận sự kiện.

---

## 9. Bước 6 — Lắng nghe sự kiện hành trình

Cho Activity của bạn implement các listener cần dùng:

```java
public class NavigationActivity extends AppCompatActivity implements
        ProgressChangeListener,    // tiến trình hành trình
        OffRouteListener,          // đi sai đường
        MilestoneEventListener,    // chỉ dẫn voice/banner
        NavigationEventListener {  // trạng thái chạy/dừng
}
```

**Tiến trình hành trình** — gọi liên tục mỗi khi có vị trí mới:
```java
@Override
public void onProgressChange(Location location, RouteProgress routeProgress) {
    double distanceRemaining = routeProgress.distanceRemaining(); // mét còn lại
    double durationRemaining = routeProgress.durationRemaining(); // giây còn lại
    // Cập nhật UI: khoảng cách, thời gian, hướng rẽ tiếp theo...
}
```

**Đi sai đường** — kích hoạt tìm tuyến mới:
```java
@Override
public void userOffRoute(Location location) {
    Point newOrigin = Point.fromLngLat(location.getLongitude(), location.getLatitude());
    fetchRoute(newOrigin, destination); // tìm lại tuyến rồi gọi startNavigation() với tuyến mới
}
```

**Chỉ dẫn (voice/banner):**
```java
@Override
public void onMilestoneEvent(RouteProgress routeProgress, String instruction, Milestone milestone) {
    // instruction chứa nội dung chỉ dẫn của bước hiện tại
}
```

**Trạng thái dẫn đường:**
```java
@Override
public void onRunning(boolean running) {
    // running = true: phiên dẫn đường đã chạy; false: đã dừng
}
```

> Để biết khi người dùng **đến nơi**, kiểm tra trong `onProgressChange` xem `distanceRemaining` đã đủ nhỏ chưa, hoặc dựa vào ngưỡng `metersRemainingTillArrival` (xem [mục 12](#12-tùy-chỉnh-hành-vi-dẫn-đường)).

---

## 10. Bước 7 — Kết thúc & giải phóng

Khi người dùng dừng dẫn đường hoặc thoát màn hình:
```java
private void stopNavigation() {
    navigation.stopNavigation();
    navigation.removeProgressChangeListener(this);
    navigation.removeOffRouteListener(this);
    navigation.removeMilestoneEventListener(this);
    navigation.removeNavigationEventListener(this);
}
```

Trong `onDestroy()` của Activity, dọn dẹp triệt để để tránh rò rỉ bộ nhớ:
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    navigation.onDestroy(); // dừng dẫn đường + gỡ toàn bộ listener còn lại
    mapView.onDestroy();
}
```

> `navigation.onDestroy()` tự gỡ mọi listener nên bạn không bắt buộc gỡ thủ công khi hủy màn hình.

---

## 11. Giải thích các hàm cơ bản

### Tìm & dẫn đường

| Hàm | Tác dụng |
| --- | --- |
| `NavigationRoute.builder(context)...build().getRoute(callback)` | Tìm tuyến giữa các điểm và trả kết quả qua callback. |
| `startNavigation(route)` | Bắt đầu phiên dẫn đường. Gọi lại hàm này với tuyến mới để **cập nhật tuyến** khi tìm đường lại. |
| `stopNavigation()` | Dừng phiên dẫn đường hiện tại. |
| `onDestroy()` | Giải phóng tài nguyên và gỡ mọi listener. Gọi khi hủy màn hình. |

### Nhận sự kiện

| Hàm | Tác dụng |
| --- | --- |
| `addProgressChangeListener(...)` | Nhận cập nhật tiến trình (khoảng cách/thời gian còn lại, bước rẽ tiếp theo). |
| `addOffRouteListener(...)` | Nhận thông báo khi người dùng đi chệch tuyến. |
| `addMilestoneEventListener(...)` | Nhận chỉ dẫn voice/banner theo mốc. |
| `addNavigationEventListener(...)` | Nhận trạng thái phiên đang chạy/đã dừng. |
| `removeXxxListener(...)` | Gỡ listener tương ứng (truyền `null` để gỡ tất cả listener loại đó). |

### Hiển thị tuyến trên bản đồ

| Hàm | Tác dụng |
| --- | --- |
| `addRoute(route)` | Vẽ **một** tuyến lên bản đồ. |
| `addRoutes(routes)` | Vẽ danh sách tuyến — tuyến đầu là tuyến chính, còn lại là tuyến thay thế. |
| `updateRouteVisibilityTo(boolean)` | Ẩn/hiện đường đi. |
| `updateRouteArrowVisibilityTo(boolean)` | Ẩn/hiện mũi tên báo rẽ. |
| `showAlternativeRoutes(boolean)` | Ẩn/hiện các tuyến thay thế (thường ẩn khi đã bắt đầu đi). |
| `setOnRouteSelectionChangeListener(...)` | Nhận sự kiện khi người dùng chạm chọn tuyến thay thế. |

---

## 12. Tùy chỉnh hành vi dẫn đường

Truyền cấu hình khi khởi tạo bộ điều khiển:

```java
import vn.vietmap.services.android.navigation.v5.navigation.VietmapNavigationOptions;

VietmapNavigationOptions options = VietmapNavigationOptions.builder()
        .maximumDistanceOffRoute(50.0)          // lệch quá 50m mới coi là đi sai đường
        .metersRemainingTillArrival(20.0)       // còn 20m thì coi như đã đến nơi
        .enableOffRouteDetection(true)          // bật tự phát hiện đi sai đường
        .enableFasterRouteDetection(false)      // tắt tìm tuyến nhanh hơn
        .defaultMilestonesEnabled(true)         // bật chỉ dẫn voice/banner mặc định
        .build();

VietmapNavigation navigation = new VietmapNavigation(this, options);
```

Một số tùy chọn thường dùng:

| Tùy chọn | Ý nghĩa |
| --- | --- |
| `maximumDistanceOffRoute(double)` | Ngưỡng (mét) coi là đi sai đường. |
| `metersRemainingTillArrival(double)` | Ngưỡng (mét) coi là đã đến nơi. |
| `secondsBeforeReroute(int)` | Thời gian chờ trước khi tìm tuyến lại. |
| `enableOffRouteDetection(boolean)` | Bật/tắt tự phát hiện đi sai đường. |
| `enableFasterRouteDetection(boolean)` | Bật/tắt gợi ý tuyến nhanh hơn. |
| `defaultMilestonesEnabled(boolean)` | Bật/tắt chỉ dẫn voice/banner mặc định. |
| `snapToRoute(boolean)` | Có "ghim" vị trí hiển thị vào tuyến hay không. |

### Mô phỏng hành trình (để test)

Để chạy thử mà không cần di chuyển thật, dùng location engine mô phỏng:
```java
import vn.vietmap.services.android.navigation.v5.location.replay.ReplayRouteLocationEngine;

ReplayRouteLocationEngine mockEngine = new ReplayRouteLocationEngine();
mockEngine.assign(currentRoute);
navigation.setLocationEngine(mockEngine);   // gọi trước startNavigation()
```

---

## 13. Quản lý vòng đời màn hình

`MapView` và bộ điều khiển dẫn đường cần được đồng bộ với vòng đời của Activity/Fragment để hoạt động đúng và không rò rỉ bộ nhớ:

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

## 14. Liên hệ

- 🌐 Website: [maps.vietmap.vn](https://maps.vietmap.vn/)
- 💬 Hỗ trợ qua Zalo: [Zalo OA](https://zalo.me/3189066936017422854)
</content>
