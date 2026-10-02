# Part 47: Android - Permissions Handling
## ขั้นตอนที่ 951-975

---

## ขั้นตอนที่ 951: ประเภทของ Permissions ใน Android

Android มี permission 3 ระดับ: Normal, Dangerous, Special
- Normal: อนุมัติอัตโนมัติ เช่น INTERNET
- Dangerous: ต้องขอ runtime เช่น CAMERA, LOCATION
- Special: ขอผ่าน Settings เช่น MANAGE_EXTERNAL_STORAGE

```kotlin
// AndroidManifest.xml
// Normal permission (ไม่ต้องขอ runtime)
// <uses-permission android:name="android.permission.INTERNET" />
// <uses-permission android:name="android.permission.VIBRATE" />

// Dangerous permissions (ต้องขอ runtime)
// <uses-permission android:name="android.permission.CAMERA" />
// <uses-permission android:name="android.permission.READ_CONTACTS" />
// <uses-permission android:name="android.permission.RECORD_AUDIO" />
// <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
// <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />

// Accompanist Permissions library (ง่ายกว่า)
// build.gradle.kts
// implementation("com.google.accompanist:accompanist-permissions:0.32.0")
```

---

## ขั้นตอนที่ 952: ขอ Permission ด้วย ActivityResultLauncher

```kotlin
import android.Manifest
import android.content.pm.PackageManager
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.runtime.*
import androidx.core.content.ContextCompat

@Composable
fun CameraPermissionExample() {
    val context = LocalContext.current
    var hasCameraPermission by remember {
        mutableStateOf(
            ContextCompat.checkSelfPermission(
                context,
                Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED
        )
    }
    var showRationale by remember { mutableStateOf(false) }

    // Launcher สำหรับขอ permission เดียว
    val cameraPermissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        hasCameraPermission = isGranted
        if (!isGranted) {
            showRationale = true
        }
    }

    Column(modifier = Modifier.padding(16.dp)) {
        if (hasCameraPermission) {
            // มี permission - แสดง camera UI
            Text("กล้องพร้อมใช้งาน ✓", color = Color.Green)
            Button(onClick = { /* เปิดกล้อง */ }) {
                Icon(Icons.Default.CameraAlt, null)
                Spacer(Modifier.width(8.dp))
                Text("ถ่ายรูป")
            }
        } else {
            // ไม่มี permission - ขอก่อน
            Button(onClick = {
                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
            }) {
                Text("ขออนุญาตใช้กล้อง")
            }
        }
    }

    if (showRationale) {
        AlertDialog(
            onDismissRequest = { showRationale = false },
            title = { Text("ต้องการสิทธิ์กล้อง") },
            text = { Text("แอปนี้ต้องการสิทธิ์กล้องเพื่อถ่ายรูป กรุณาเปิดสิทธิ์ในการตั้งค่า") },
            confirmButton = {
                TextButton(onClick = {
                    showRationale = false
                    // เปิด App Settings
                    val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                        data = Uri.fromParts("package", context.packageName, null)
                    }
                    context.startActivity(intent)
                }) { Text("ไปที่ตั้งค่า") }
            },
            dismissButton = {
                TextButton(onClick = { showRationale = false }) { Text("ยกเลิก") }
            }
        )
    }
}
```

---

## ขั้นตอนที่ 953: ขอ Multiple Permissions

```kotlin
import android.Manifest
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts

@Composable
fun MultiplePermissionsExample() {
    val context = LocalContext.current

    val permissions = arrayOf(
        Manifest.permission.CAMERA,
        Manifest.permission.RECORD_AUDIO,
        Manifest.permission.READ_EXTERNAL_STORAGE
    )

    var permissionState by remember {
        mutableStateOf(
            permissions.associateWith { permission ->
                ContextCompat.checkSelfPermission(context, permission) ==
                    PackageManager.PERMISSION_GRANTED
            }
        )
    }

    // Launcher สำหรับขอหลาย permission
    val multiplePermissionsLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestMultiplePermissions()
    ) { results ->
        permissionState = results.mapValues { it.value }
    }

    Column(modifier = Modifier.padding(16.dp)) {
        Text("สิทธิ์ที่จำเป็น:", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))

        permissionState.forEach { (permission, isGranted) ->
            Row(
                modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(permission.substringAfterLast("."))
                Icon(
                    if (isGranted) Icons.Default.CheckCircle else Icons.Default.Cancel,
                    contentDescription = null,
                    tint = if (isGranted) Color.Green else Color.Red
                )
            }
        }

        Spacer(Modifier.height(16.dp))

        val allGranted = permissionState.values.all { it }

        if (!allGranted) {
            Button(
                onClick = {
                    multiplePermissionsLauncher.launch(
                        permissionState.filter { !it.value }.keys.toTypedArray()
                    )
                },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("ขออนุญาตที่ยังไม่ได้รับ")
            }
        } else {
            Text("ได้รับสิทธิ์ทั้งหมดแล้ว!", color = Color.Green)
        }
    }
}
```

---

## ขั้นตอนที่ 954: Accompanist Permissions (วิธีที่ง่ายกว่า)

```kotlin
// ต้องเพิ่ม dependency:
// implementation("com.google.accompanist:accompanist-permissions:0.32.0")

import com.google.accompanist.permissions.*

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun AccompanistPermissionExample() {
    // Single permission
    val cameraPermissionState = rememberPermissionState(
        Manifest.permission.CAMERA
    )

    Column(modifier = Modifier.padding(16.dp)) {
        when {
            cameraPermissionState.status.isGranted -> {
                Text("กล้องพร้อมใช้งาน")
                CameraPreview()
            }
            cameraPermissionState.status.shouldShowRationale -> {
                // ผู้ใช้เคยปฏิเสธ - แสดงเหตุผล
                Column {
                    Text("แอปนี้ต้องการกล้องเพื่อสแกน QR Code")
                    Button(onClick = { cameraPermissionState.launchPermissionRequest() }) {
                        Text("อนุญาต")
                    }
                }
            }
            !cameraPermissionState.status.isGranted -> {
                Button(onClick = { cameraPermissionState.launchPermissionRequest() }) {
                    Text("ขอสิทธิ์กล้อง")
                }
            }
        }
    }
}

// Multiple permissions
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MultiplePermissionsAccompanist() {
    val multiplePermissionsState = rememberMultiplePermissionsState(
        listOf(
            Manifest.permission.CAMERA,
            Manifest.permission.RECORD_AUDIO
        )
    )

    Column {
        when {
            multiplePermissionsState.allPermissionsGranted -> {
                Text("ได้รับทุก permission!")
            }
            multiplePermissionsState.shouldShowRationale -> {
                Column {
                    Text("กล้องและไมโครโฟนจำเป็นสำหรับการวิดีโอคอล")
                    Button(onClick = { multiplePermissionsState.launchMultiplePermissionRequest() }) {
                        Text("อนุญาตทั้งหมด")
                    }
                }
            }
            else -> {
                Button(onClick = { multiplePermissionsState.launchMultiplePermissionRequest() }) {
                    Text("ขอสิทธิ์")
                }
            }
        }

        // แสดงสถานะแต่ละ permission
        multiplePermissionsState.permissions.forEach { permState ->
            Text(
                "${permState.permission.substringAfterLast(".")}: " +
                    if (permState.status.isGranted) "✓" else "✗"
            )
        }
    }
}
```

---

## ขั้นตอนที่ 955: Location Permission

```kotlin
import android.Manifest
import android.annotation.SuppressLint
import android.location.Location
import com.google.android.gms.location.*

@SuppressLint("MissingPermission")
@Composable
fun LocationPermissionExample() {
    val context = LocalContext.current
    var location by remember { mutableStateOf<Location?>(null) }
    var permissionGranted by remember { mutableStateOf(false) }

    // FusedLocationProvider
    val fusedLocationClient = remember {
        LocationServices.getFusedLocationProviderClient(context)
    }

    val locationPermissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        permissionGranted = permissions.getOrDefault(
            Manifest.permission.ACCESS_FINE_LOCATION, false
        ) || permissions.getOrDefault(
            Manifest.permission.ACCESS_COARSE_LOCATION, false
        )

        if (permissionGranted) {
            // ดึง location ล่าสุด
            fusedLocationClient.lastLocation.addOnSuccessListener { loc ->
                location = loc
            }
        }
    }

    // ตรวจสอบสิทธิ์เมื่อเข้าหน้า
    LaunchedEffect(Unit) {
        val fineGranted = ContextCompat.checkSelfPermission(
            context, Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED

        if (fineGranted) {
            permissionGranted = true
            fusedLocationClient.lastLocation.addOnSuccessListener { loc ->
                location = loc
            }
        }
    }

    Column(modifier = Modifier.padding(16.dp)) {
        if (permissionGranted) {
            location?.let { loc ->
                Text("ละติจูด: ${loc.latitude}")
                Text("ลองจิจูด: ${loc.longitude}")
                Text("ความแม่นยำ: ${loc.accuracy} เมตร")
            } ?: Text("กำลังหาตำแหน่ง...")
        } else {
            Button(onClick = {
                locationPermissionLauncher.launch(
                    arrayOf(
                        Manifest.permission.ACCESS_FINE_LOCATION,
                        Manifest.permission.ACCESS_COARSE_LOCATION
                    )
                )
            }) {
                Icon(Icons.Default.LocationOn, null)
                Spacer(Modifier.width(8.dp))
                Text("ขอสิทธิ์ตำแหน่ง")
            }
        }
    }
}
```

---

*Part 47 จบแล้ว | ก่อนหน้า: [Part 46](../part46/README.md) | ถัดไป: [Part 48](../part48/README.md)*
