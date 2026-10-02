# Part 72: Bluetooth & NFC
## ขั้นตอนที่ 1276-1300

---

## ขั้นตอนที่ 1276: Bluetooth Overview

```
Bluetooth ใน Android:

Classic Bluetooth (BT Classic):
- Audio streaming, file transfer
- Serial communication (SPP)
- ต้องมี BLUETOOTH_CONNECT, BLUETOOTH_SCAN permissions

Bluetooth Low Energy (BLE):
- IoT devices, health monitors, beacons
- ประหยัดพลังงานมาก
- Roles: Central (phone) ↔ Peripheral (sensor)

NFC:
- Near Field Communication (<= 10cm)
- Payment, transit cards, tag reading/writing
- Android Beam (ถูก deprecate ใน Android 10)
```

---

## ขั้นตอนที่ 1277: Bluetooth Permissions

```xml
<!-- AndroidManifest.xml -->
<!-- Android 12+ (API 31+) -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation"/>
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE"/>

<!-- Android 11 and below -->
<uses-permission android:name="android.permission.BLUETOOTH"
    android:maxSdkVersion="30"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"
    android:maxSdkVersion="30"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"
    android:maxSdkVersion="30"/>

<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```

```kotlin
// Request permissions at runtime
@Composable
fun BluetoothPermissionsScreen(
    onPermissionsGranted: () -> Unit
) {
    val permissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        listOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT
        )
    } else {
        listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.BLUETOOTH,
            Manifest.permission.BLUETOOTH_ADMIN
        )
    }
    
    val permissionsState = rememberMultiplePermissionsState(permissions)
    
    LaunchedEffect(Unit) {
        permissionsState.launchMultiplePermissionRequest()
    }
    
    if (permissionsState.allPermissionsGranted) {
        LaunchedEffect(Unit) { onPermissionsGranted() }
    } else {
        Column(Modifier.padding(16.dp)) {
            Text("ต้องการสิทธิ์ Bluetooth เพื่อใช้งาน")
            Button(onClick = { permissionsState.launchMultiplePermissionRequest() }) {
                Text("ให้สิทธิ์")
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1278: BLE Scanning

```kotlin
class BleScanner @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val bluetoothManager = context.getSystemService(BluetoothManager::class.java)
    private val bluetoothAdapter = bluetoothManager?.adapter
    private val bleScanner = bluetoothAdapter?.bluetoothLeScanner
    
    data class BleDevice(
        val name: String?,
        val address: String,
        val rssi: Int,
        val serviceUuids: List<ParcelUuid>
    )
    
    @SuppressLint("MissingPermission")
    fun scanDevices(
        serviceUuid: UUID? = null,
        scanDurationMs: Long = 10_000L
    ): Flow<List<BleDevice>> = callbackFlow {
        
        if (bluetoothAdapter?.isEnabled != true) {
            close(IllegalStateException("Bluetooth is not enabled"))
            return@callbackFlow
        }
        
        val devices = mutableMapOf<String, BleDevice>()
        
        val filters = serviceUuid?.let { uuid ->
            listOf(
                ScanFilter.Builder()
                    .setServiceUuid(ParcelUuid(uuid))
                    .build()
            )
        } ?: emptyList()
        
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .build()
        
        val callback = object : ScanCallback() {
            override fun onScanResult(callbackType: Int, result: ScanResult) {
                val device = BleDevice(
                    name = result.device.name,
                    address = result.device.address,
                    rssi = result.rssi,
                    serviceUuids = result.scanRecord?.serviceUuids ?: emptyList()
                )
                devices[device.address] = device
                trySend(devices.values.toList())
            }
            
            override fun onScanFailed(errorCode: Int) {
                close(Exception("BLE scan failed: $errorCode"))
            }
        }
        
        bleScanner?.startScan(filters, settings, callback)
        
        // Stop scan after duration
        delay(scanDurationMs)
        bleScanner?.stopScan(callback)
        
        awaitClose { bleScanner?.stopScan(callback) }
    }
    
    fun isBluetoothEnabled() = bluetoothAdapter?.isEnabled == true
}
```

---

## ขั้นตอนที่ 1279: GATT Connection & Reading Data

```kotlin
@SuppressLint("MissingPermission")
class BleGattManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private var gatt: BluetoothGatt? = null
    
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()
    
    private val _characteristicData = MutableSharedFlow<Pair<UUID, ByteArray>>()
    val characteristicData: SharedFlow<Pair<UUID, ByteArray>> = _characteristicData.asSharedFlow()
    
    fun connect(device: BluetoothDevice) {
        _connectionState.value = ConnectionState.CONNECTING
        
        gatt = device.connectGatt(context, false, object : BluetoothGattCallback() {
            
            override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
                when (newState) {
                    BluetoothProfile.STATE_CONNECTED -> {
                        _connectionState.value = ConnectionState.CONNECTED
                        gatt.discoverServices()
                    }
                    BluetoothProfile.STATE_DISCONNECTED -> {
                        _connectionState.value = ConnectionState.DISCONNECTED
                    }
                }
            }
            
            override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
                if (status == BluetoothGatt.GATT_SUCCESS) {
                    _connectionState.value = ConnectionState.SERVICES_DISCOVERED
                }
            }
            
            override fun onCharacteristicRead(
                gatt: BluetoothGatt,
                characteristic: BluetoothGattCharacteristic,
                value: ByteArray,
                status: Int
            ) {
                if (status == BluetoothGatt.GATT_SUCCESS) {
                    CoroutineScope(Dispatchers.IO).launch {
                        _characteristicData.emit(Pair(characteristic.uuid, value))
                    }
                }
            }
            
            override fun onCharacteristicChanged(
                gatt: BluetoothGatt,
                characteristic: BluetoothGattCharacteristic,
                value: ByteArray
            ) {
                CoroutineScope(Dispatchers.IO).launch {
                    _characteristicData.emit(Pair(characteristic.uuid, value))
                }
            }
        })
    }
    
    fun readCharacteristic(serviceUuid: UUID, characteristicUuid: UUID) {
        val characteristic = gatt?.getService(serviceUuid)
            ?.getCharacteristic(characteristicUuid)
        
        characteristic?.let { gatt?.readCharacteristic(it) }
    }
    
    fun enableNotifications(serviceUuid: UUID, characteristicUuid: UUID) {
        val characteristic = gatt?.getService(serviceUuid)
            ?.getCharacteristic(characteristicUuid) ?: return
        
        gatt?.setCharacteristicNotification(characteristic, true)
        
        // Write to CCCD descriptor to enable notifications
        val descriptor = characteristic.getDescriptor(
            UUID.fromString("00002902-0000-1000-8000-00805f9b34fb")
        )
        gatt?.writeDescriptor(descriptor, BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE)
    }
    
    fun disconnect() {
        gatt?.disconnect()
        gatt?.close()
        gatt = null
    }
    
    enum class ConnectionState {
        DISCONNECTED, CONNECTING, CONNECTED, SERVICES_DISCOVERED
    }
}
```

---

## ขั้นตอนที่ 1280: NFC Tag Reading

```kotlin
class NfcManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    
    private val nfcAdapter = NfcAdapter.getDefaultAdapter(context)
    
    fun isNfcSupported() = nfcAdapter != null
    fun isNfcEnabled() = nfcAdapter?.isEnabled == true
    
    // Enable foreground dispatch เพื่อรับ NFC intent ขณะ app เปิดอยู่
    fun enableForegroundDispatch(activity: Activity) {
        val intent = Intent(activity, activity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP)
        }
        val pendingIntent = PendingIntent.getActivity(
            activity, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )
        
        nfcAdapter?.enableForegroundDispatch(activity, pendingIntent, null, null)
    }
    
    fun disableForegroundDispatch(activity: Activity) {
        nfcAdapter?.disableForegroundDispatch(activity)
    }
    
    // อ่าน NDEF message จาก tag
    fun readNdefMessage(intent: Intent): NdefMessage? {
        if (intent.action != NfcAdapter.ACTION_NDEF_DISCOVERED &&
            intent.action != NfcAdapter.ACTION_TAG_DISCOVERED) return null
        
        return intent.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES)
            ?.firstOrNull() as? NdefMessage
    }
    
    // แปลง NDEF message เป็น string
    fun parseNdefText(message: NdefMessage): String? {
        for (record in message.records) {
            if (record.tnf == NdefRecord.TNF_WELL_KNOWN &&
                Arrays.equals(record.type, NdefRecord.RTD_TEXT)) {
                
                val payload = record.payload
                val encoding = if (payload[0].and(0x80.toByte()) == 0.toByte()) {
                    "UTF-8"
                } else {
                    "UTF-16"
                }
                val languageCodeLength = payload[0].and(0x3F).toInt()
                
                return String(
                    payload,
                    languageCodeLength + 1,
                    payload.size - languageCodeLength - 1,
                    charset(encoding)
                )
            }
        }
        return null
    }
    
    // เขียน NDEF message ลง tag
    fun writeNdefMessage(tag: Tag, text: String): Boolean {
        return try {
            val record = NdefRecord.createTextRecord("en", text)
            val message = NdefMessage(arrayOf(record))
            
            val ndef = Ndef.get(tag)
            ndef.connect()
            ndef.writeNdefMessage(message)
            ndef.close()
            true
        } catch (e: Exception) {
            false
        }
    }
}

// ใน Activity
class NfcActivity : ComponentActivity() {
    
    @Inject lateinit var nfcManager: NfcManager
    
    override fun onResume() {
        super.onResume()
        nfcManager.enableForegroundDispatch(this)
    }
    
    override fun onPause() {
        super.onPause()
        nfcManager.disableForegroundDispatch(this)
    }
    
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        
        val message = nfcManager.readNdefMessage(intent)
        val text = message?.let { nfcManager.parseNdefText(it) }
        
        text?.let {
            // Handle scanned NFC tag text
            viewModel.onNfcTagScanned(it)
        }
    }
}
```

---

## แบบฝึกหัด Part 72

```kotlin
// แบบฝึกหัด: BLE Heart Rate Monitor

// สร้าง HeartRateMonitorScreen ที่:
// 1. Scan BLE devices ที่มี Heart Rate Service (UUID: 0x180D)
// 2. เชื่อมต่อกับ device ที่เลือก
// 3. อ่านค่า Heart Rate ต่อเนื่อง (Notify characteristic UUID: 0x2A37)
// 4. แสดงกราฟ heart rate ย้อนหลัง 60 วินาที
// 5. Alert เมื่อ BPM > 150

// Heart Rate Measurement parsing
fun parseHeartRate(value: ByteArray): Int {
    val flag = value[0].toInt()
    return if (flag and 0x01 != 0) {
        // 16-bit format
        (value[1].toInt() and 0xFF) or (value[2].toInt() shl 8)
    } else {
        // 8-bit format
        value[1].toInt() and 0xFF
    }
}

// Standard BLE UUIDs
object HeartRateService {
    val SERVICE_UUID = UUID.fromString("0000180D-0000-1000-8000-00805f9b34fb")
    val CHARACTERISTIC_UUID = UUID.fromString("00002A37-0000-1000-8000-00805f9b34fb")
}
```

---

*Part 72 จบแล้ว | ก่อนหน้า: [Part 71](../part71/README.md) | ถัดไป: [Part 73](../part73/README.md)*
