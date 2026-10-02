# Part 69: Maps & Location
## ขั้นตอนที่ 1201-1225

---

## ขั้นตอนที่ 1201: Google Maps SDK

```kotlin
// build.gradle.kts
// implementation("com.google.maps.android:maps-compose:4.x")
// implementation("com.google.android.gms:play-services-maps:19.x")
// implementation("com.google.android.gms:play-services-location:21.x")

// AndroidManifest.xml
// <meta-data
//     android:name="com.google.android.geo.API_KEY"
//     android:value="${MAPS_API_KEY}"/>

// local.properties
// MAPS_API_KEY=your_api_key_here

// build.gradle.kts (app)
// defaultConfig {
//     manifestPlaceholders["MAPS_API_KEY"] = project.findProperty("MAPS_API_KEY") ?: ""
// }

// ============================================
// Google Maps ใน Compose
// ============================================

@Composable
fun MapScreen(
    locations: List<MapLocation>,
    userLocation: LatLng?,
    modifier: Modifier = Modifier
) {
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(
            userLocation ?: LatLng(13.7563, 100.5018),  // Bangkok default
            12f
        )
    }
    
    // Update camera when user location changes
    LaunchedEffect(userLocation) {
        userLocation?.let { location ->
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngZoom(location, 15f),
                durationMs = 1000
            )
        }
    }
    
    GoogleMap(
        modifier = modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(
            isMyLocationEnabled = userLocation != null,
            mapType = MapType.NORMAL
        ),
        uiSettings = MapUiSettings(
            myLocationButtonEnabled = true,
            zoomControlsEnabled = false  // ใช้ custom controls แทน
        ),
        onMapClick = { latLng ->
            // handle map click
        }
    ) {
        // User location marker
        userLocation?.let { location ->
            Marker(
                state = MarkerState(position = location),
                title = "ตำแหน่งของคุณ",
                icon = BitmapDescriptorFactory.defaultMarker(BitmapDescriptorFactory.HUE_AZURE)
            )
        }
        
        // Location markers
        locations.forEach { mapLocation ->
            Marker(
                state = MarkerState(position = LatLng(mapLocation.lat, mapLocation.lng)),
                title = mapLocation.name,
                snippet = mapLocation.description,
                onClick = { marker ->
                    // show info window or navigate
                    true
                }
            )
        }
        
        // Polyline
        if (locations.size >= 2) {
            Polyline(
                points = locations.map { LatLng(it.lat, it.lng) },
                color = Color.Blue,
                width = 5f
            )
        }
    }
}

data class MapLocation(
    val id: Long,
    val name: String,
    val description: String,
    val lat: Double,
    val lng: Double
)
```

---

## ขั้นตอนที่ 1202: Location Services

```kotlin
// ============================================
// FusedLocationProviderClient
// ============================================

class LocationManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val fusedLocationClient = LocationServices.getFusedLocationProviderClient(context)
    
    @SuppressLint("MissingPermission")
    fun getLastLocation(): Flow<Location?> = flow {
        val location = fusedLocationClient.lastLocation.await()
        emit(location)
    }
    
    @SuppressLint("MissingPermission")
    fun observeLocation(
        interval: Long = 5000L,  // 5 seconds
        fastestInterval: Long = 2000L
    ): Flow<Location> = callbackFlow {
        
        val request = LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY,
            interval
        )
            .setMinUpdateIntervalMillis(fastestInterval)
            .build()
        
        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { location ->
                    trySend(location)
                }
            }
        }
        
        fusedLocationClient.requestLocationUpdates(
            request,
            callback,
            Looper.getMainLooper()
        )
        
        awaitClose {
            fusedLocationClient.removeLocationUpdates(callback)
        }
    }
    
    @SuppressLint("MissingPermission")
    suspend fun getCurrentLocation(): Location? {
        return fusedLocationClient.getCurrentLocation(
            Priority.PRIORITY_HIGH_ACCURACY,
            null
        ).await()
    }
    
    fun distanceTo(from: Location, to: LatLng): Float {
        val results = FloatArray(1)
        Location.distanceBetween(
            from.latitude, from.longitude,
            to.latitude, to.longitude,
            results
        )
        return results[0]  // meters
    }
}

// ============================================
// Permission Request ใน Compose
// ============================================

@Composable
fun LocationAwareContent() {
    val locationPermissions = rememberMultiplePermissionsState(
        permissions = listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )
    
    LaunchedEffect(Unit) {
        locationPermissions.launchMultiplePermissionRequest()
    }
    
    when {
        locationPermissions.allPermissionsGranted -> {
            // แสดง map
            LocationMap()
        }
        locationPermissions.shouldShowRationale -> {
            Column(modifier = Modifier.padding(16.dp)) {
                Text("เราต้องการสิทธิ์เข้าถึงตำแหน่งเพื่อแสดงร้านค้าใกล้คุณ")
                Button(onClick = { locationPermissions.launchMultiplePermissionRequest() }) {
                    Text("อนุญาต")
                }
            }
        }
        else -> {
            // ถูก denied permanently
            Column(modifier = Modifier.padding(16.dp)) {
                Text("กรุณาเปิดสิทธิ์ตำแหน่งใน Settings")
                Button(onClick = { /* open settings */ }) {
                    Text("ไปที่ Settings")
                }
            }
        }
    }
}

// Accompanist Permissions library:
// implementation("com.google.accompanist:accompanist-permissions:0.x")
```

---

## ขั้นตอนที่ 1203: Geofencing

```kotlin
// ============================================
// Geofencing - trigger เมื่อเข้า/ออก พื้นที่
// ============================================

class GeofenceManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val geofencingClient = LocationServices.getGeofencingClient(context)
    
    @SuppressLint("MissingPermission")
    fun addGeofence(
        id: String,
        latLng: LatLng,
        radiusMeters: Float = 200f,
        transitionTypes: Int = Geofence.GEOFENCE_TRANSITION_ENTER or Geofence.GEOFENCE_TRANSITION_EXIT
    ): Task<Void> {
        val geofence = Geofence.Builder()
            .setRequestId(id)
            .setCircularRegion(latLng.latitude, latLng.longitude, radiusMeters)
            .setExpirationDuration(Geofence.NEVER_EXPIRE)
            .setTransitionTypes(transitionTypes)
            .build()
        
        val request = GeofencingRequest.Builder()
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .addGeofence(geofence)
            .build()
        
        val intent = PendingIntent.getBroadcast(
            context, 0,
            Intent(context, GeofenceBroadcastReceiver::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )
        
        return geofencingClient.addGeofences(request, intent)
    }
    
    fun removeGeofence(id: String): Task<Void> {
        return geofencingClient.removeGeofences(listOf(id))
    }
}

class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return
        
        if (event.hasError()) {
            Log.e("Geofence", "Error: ${event.errorCode}")
            return
        }
        
        val transition = event.geofenceTransition
        val triggeredIds = event.triggeringGeofences?.map { it.requestId } ?: emptyList()
        
        when (transition) {
            Geofence.GEOFENCE_TRANSITION_ENTER -> {
                // User entered geofence
                showStoreNotification(context, triggeredIds)
            }
            Geofence.GEOFENCE_TRANSITION_EXIT -> {
                // User left geofence
            }
        }
    }
    
    private fun showStoreNotification(context: Context, storeIds: List<String>) {
        // Show "You're near our store!" notification
    }
}
```

---

## ขั้นตอนที่ 1204: Route Drawing

```kotlin
// ============================================
// Google Directions API
// ============================================

data class DirectionsRequest(
    val origin: LatLng,
    val destination: LatLng,
    val mode: TravelMode = TravelMode.DRIVING
)

enum class TravelMode { DRIVING, WALKING, BICYCLING, TRANSIT }

data class Route(
    val polyline: List<LatLng>,
    val distanceMeters: Int,
    val durationSeconds: Int,
    val summary: String
)

class DirectionsService(private val apiKey: String) {
    
    private val client = OkHttpClient()
    
    suspend fun getDirections(request: DirectionsRequest): Route? {
        val url = buildUrl(request)
        
        val response = withContext(Dispatchers.IO) {
            client.newCall(Request.Builder().url(url).build()).execute()
        }
        
        return response.body?.string()?.let { parseResponse(it) }
    }
    
    private fun buildUrl(request: DirectionsRequest): String {
        return "https://maps.googleapis.com/maps/api/directions/json" +
            "?origin=${request.origin.latitude},${request.origin.longitude}" +
            "&destination=${request.destination.latitude},${request.destination.longitude}" +
            "&mode=${request.mode.name.lowercase()}" +
            "&key=$apiKey"
    }
    
    private fun parseResponse(json: String): Route? {
        val gson = Gson()
        val response = gson.fromJson(json, DirectionsResponse::class.java)
        
        val route = response.routes?.firstOrNull() ?: return null
        val leg = route.legs?.firstOrNull() ?: return null
        
        val polyline = decodePolyline(route.overviewPolyline?.points ?: return null)
        
        return Route(
            polyline = polyline,
            distanceMeters = leg.distance?.value ?: 0,
            durationSeconds = leg.duration?.value ?: 0,
            summary = route.summary ?: ""
        )
    }
    
    private fun decodePolyline(encoded: String): List<LatLng> {
        // Decode Google Polyline Encoding
        val points = mutableListOf<LatLng>()
        var index = 0
        var lat = 0
        var lng = 0
        
        while (index < encoded.length) {
            var shift = 0
            var result = 0
            var b: Int
            do {
                b = encoded[index++].code - 63
                result = result or (b and 0x1f shl shift)
                shift += 5
            } while (b >= 0x20)
            val dlat = if (result and 1 != 0) (result shr 1).inv() else result shr 1
            lat += dlat
            
            shift = 0
            result = 0
            do {
                b = encoded[index++].code - 63
                result = result or (b and 0x1f shl shift)
                shift += 5
            } while (b >= 0x20)
            val dlng = if (result and 1 != 0) (result shr 1).inv() else result shr 1
            lng += dlng
            
            points.add(LatLng(lat.toDouble() / 1E5, lng.toDouble() / 1E5))
        }
        
        return points
    }
}
```

---

## แบบฝึกหัด Part 69

```kotlin
// แบบฝึกหัด: Delivery Tracking Screen

// TODO: สร้าง DeliveryTrackingScreen ที่:
// 1. แสดง Google Map
// 2. Marker สำหรับ pickup point และ delivery destination
// 3. Route ระหว่าง 2 จุด (polyline)
// 4. Courier location ที่ update ทุก 5 วินาที
// 5. Estimated arrival time
// 6. Zoom in ให้เห็นทั้ง route

data class DeliveryInfo(
    val orderId: String,
    val pickupLocation: LatLng,
    val deliveryLocation: LatLng,
    val courierLocation: LatLng,
    val estimatedMinutes: Int,
    val route: List<LatLng>
)

@Composable
fun DeliveryTrackingScreen(
    orderId: String,
    viewModel: DeliveryViewModel = hiltViewModel()
) {
    val delivery by viewModel.delivery.collectAsStateWithLifecycle()
    
    // TODO: implement map + courier tracking
}
```

---

*Part 69 จบแล้ว | ก่อนหน้า: [Part 68](../part68/README.md) | ถัดไป: [Part 70](../part70/README.md)*
