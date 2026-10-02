# Part 22: Activity, Fragment และ Lifecycle
## ขั้นตอนที่ 521-545

---

## ขั้นตอนที่ 521: Activity คืออะไร?

Activity เป็น component หลักของ Android ที่แสดง UI ให้ผู้ใช้

```kotlin
// AndroidManifest.xml
// <activity
//     android:name=".MainActivity"
//     android:exported="true">
//     <intent-filter>
//         <action android:name="android.intent.action.MAIN" />
//         <category android:name="android.intent.category.LAUNCHER" />
//     </intent-filter>
// </activity>

class MainActivity : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // enableEdgeToEdge - ให้ content อยู่ใต้ status/nav bar
        enableEdgeToEdge()
        
        setContent {
            MyAppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    MainScreen()
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 522: Activity Lifecycle ครบสมบูรณ์

```kotlin
class LifecycleExampleActivity : AppCompatActivity() {
    
    private val TAG = "LifecycleDemo"
    
    // onCreate - Activity ถูกสร้าง
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Log.d(TAG, "onCreate: savedInstanceState=${savedInstanceState != null}")
        
        // Restore state
        savedInstanceState?.let {
            val savedText = it.getString("MY_TEXT")
            Log.d(TAG, "Restored: $savedText")
        }
        
        // Setup UI
        setContentView(R.layout.activity_main)
    }
    
    // onStart - Activity กำลังจะแสดง (ยังไม่ interactive)
    override fun onStart() {
        super.onStart()
        Log.d(TAG, "onStart: Activity visible")
        // เหมาะสำหรับ: register receivers, start animations
    }
    
    // onResume - Activity interactive (foreground)
    override fun onResume() {
        super.onResume()
        Log.d(TAG, "onResume: Activity in foreground")
        // เหมาะสำหรับ: start camera, GPS, sensors
    }
    
    // onPause - Activity ถูก interrupt (ยังมองเห็นบางส่วน)
    override fun onPause() {
        super.onPause()
        Log.d(TAG, "onPause: Activity partially hidden")
        // เหมาะสำหรับ: pause animations, save draft data
        // ต้องทำให้เร็ว! ไม่ทำ heavy operations
    }
    
    // onStop - Activity ไม่ visible
    override fun onStop() {
        super.onStop()
        Log.d(TAG, "onStop: Activity hidden")
        // เหมาะสำหรับ: unregister receivers, stop animations
    }
    
    // onDestroy - Activity ถูก destroy
    override fun onDestroy() {
        super.onDestroy()
        Log.d(TAG, "onDestroy: isFinishing=$isFinishing")
        // isFinishing = true: user กด back
        // isFinishing = false: system destroy (rotation, memory)
    }
    
    // onSaveInstanceState - บันทึก state ก่อน destroy
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putString("MY_TEXT", "Hello World")
        outState.putInt("COUNTER", 42)
        Log.d(TAG, "onSaveInstanceState: saved")
    }
    
    // onRestoreInstanceState - คืน state หลัง recreate
    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        val text = savedInstanceState.getString("MY_TEXT")
        val counter = savedInstanceState.getInt("COUNTER")
        Log.d(TAG, "onRestoreInstanceState: text=$text, counter=$counter")
    }
}
```

---

## ขั้นตอนที่ 523: Intent และการส่งข้อมูลระหว่าง Activities

```kotlin
// ============================================
// Explicit Intent - เปิด Activity ที่รู้จัก
// ============================================

class HomeActivity : AppCompatActivity() {
    
    fun openDetailActivity(userId: Long) {
        val intent = Intent(this, DetailActivity::class.java).apply {
            putExtra("USER_ID", userId)
            putExtra("SOURCE", "home")
        }
        startActivity(intent)
    }
    
    fun openDetailForResult(userId: Long) {
        val intent = Intent(this, DetailActivity::class.java).apply {
            putExtra("USER_ID", userId)
        }
        // Modern API ด้วย Activity Result
        detailLauncher.launch(intent)
    }
    
    // Activity Result API (แนะนำ)
    private val detailLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (result.resultCode == RESULT_OK) {
            val data = result.data?.getStringExtra("RESULT")
            Toast.makeText(this, "Got result: $data", Toast.LENGTH_SHORT).show()
        }
    }
}

class DetailActivity : AppCompatActivity() {
    
    private val userId by lazy {
        intent.getLongExtra("USER_ID", -1)
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val userId = intent.getLongExtra("USER_ID", -1)
        val source = intent.getStringExtra("SOURCE")
        
        // Setup UI with userId
    }
    
    fun returnResult(data: String) {
        val resultIntent = Intent().apply {
            putExtra("RESULT", data)
        }
        setResult(RESULT_OK, resultIntent)
        finish()
    }
}

// ============================================
// Implicit Intent - ให้ OS เลือก app
// ============================================

fun shareText(text: String) {
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "text/plain"
        putExtra(Intent.EXTRA_TEXT, text)
    }
    startActivity(Intent.createChooser(intent, "Share via"))
}

fun openUrl(url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    if (intent.resolveActivity(packageManager) != null) {
        startActivity(intent)
    }
}

fun callPhone(phone: String) {
    val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:$phone"))
    startActivity(intent)
}

fun openEmail(to: String, subject: String = "", body: String = "") {
    val intent = Intent(Intent.ACTION_SENDTO).apply {
        data = Uri.parse("mailto:")
        putExtra(Intent.EXTRA_EMAIL, arrayOf(to))
        putExtra(Intent.EXTRA_SUBJECT, subject)
        putExtra(Intent.EXTRA_TEXT, body)
    }
    if (intent.resolveActivity(packageManager) != null) {
        startActivity(intent)
    }
}
```

---

## ขั้นตอนที่ 524: Fragment

```kotlin
// Fragment เป็น reusable UI component ที่ต้องอยู่ใน Activity

class UserListFragment : Fragment(R.layout.fragment_user_list) {
    
    // ViewModel ที่ share กับ Activity
    private val viewModel: UsersViewModel by activityViewModels()
    
    // หรือ ViewModel เฉพาะ Fragment
    private val localViewModel: LocalViewModel by viewModels()
    
    private lateinit var adapter: UserAdapter
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        setupRecyclerView(view)
        observeViewModel()
    }
    
    private fun setupRecyclerView(view: View) {
        adapter = UserAdapter { userId ->
            // Navigate to detail
            findNavController().navigate(
                UserListFragmentDirections.actionToDetail(userId)
            )
        }
        
        view.findViewById<RecyclerView>(R.id.recyclerView).apply {
            layoutManager = LinearLayoutManager(requireContext())
            adapter = this@UserListFragment.adapter
        }
    }
    
    private fun observeViewModel() {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.users.collect { users ->
                    adapter.submitList(users)
                }
            }
        }
    }
}

// Fragment Lifecycle
class LifecycleFragment : Fragment() {
    
    override fun onAttach(context: Context) {
        super.onAttach(context)
        // Fragment ถูก attach กับ Activity
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Fragment ถูกสร้าง (ยังไม่มี view)
    }
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        // สร้าง View
        return inflater.inflate(R.layout.fragment_lifecycle, container, false)
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // View พร้อมแล้ว - setup UI ที่นี่
    }
    
    override fun onStart() { super.onStart() }
    override fun onResume() { super.onResume() }
    override fun onPause() { super.onPause() }
    override fun onStop() { super.onStop() }
    
    override fun onDestroyView() {
        super.onDestroyView()
        // View ถูก destroy (Fragment ยังอยู่)
        // ต้อง clear binding ที่นี่!
    }
    
    override fun onDestroy() { super.onDestroy() }
    override fun onDetach() { super.onDetach() }
}
```

---

## ขั้นตอนที่ 525: Fragment Communication

```kotlin
// ============================================
// 1. ViewModel ที่ share กัน (แนะนำ)
// ============================================

// Shared ViewModel
class SharedViewModel : ViewModel() {
    private val _selectedUser = MutableStateFlow<User?>(null)
    val selectedUser: StateFlow<User?> = _selectedUser.asStateFlow()
    
    fun selectUser(user: User) {
        _selectedUser.value = user
    }
}

// Master Fragment
class MasterFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    fun onUserClicked(user: User) {
        sharedViewModel.selectUser(user)
    }
}

// Detail Fragment
class DetailFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                sharedViewModel.selectedUser.collect { user ->
                    user?.let { showUserDetail(it) }
                }
            }
        }
    }
    
    private fun showUserDetail(user: User) { /* update UI */ }
}

// ============================================
// 2. Fragment Result API
// ============================================

// Sender Fragment
class PickerFragment : Fragment() {
    fun onItemPicked(item: String) {
        val bundle = Bundle().apply {
            putString("picked_item", item)
        }
        setFragmentResult("picker_request", bundle)
        findNavController().popBackStack()
    }
}

// Receiver Fragment
class MainFragment : Fragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Register result listener
        setFragmentResultListener("picker_request") { key, bundle ->
            val item = bundle.getString("picked_item")
            println("Got result: $item")
        }
    }
}
```

---

## ขั้นตอนที่ 526: Permission Handling

```kotlin
class PermissionActivity : AppCompatActivity() {
    
    // ============================================
    // ด้วย Activity Result API (Modern)
    // ============================================
    
    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            accessLocation()
        } else {
            showPermissionDeniedMessage()
        }
    }
    
    private val requestMultiplePermissionsLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val allGranted = permissions.values.all { it }
        if (allGranted) {
            startCamera()
        } else {
            val denied = permissions.filterValues { !it }.keys
            showDeniedPermissions(denied)
        }
    }
    
    fun checkLocationPermission() {
        when {
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.ACCESS_FINE_LOCATION
            ) == PackageManager.PERMISSION_GRANTED -> {
                accessLocation()
            }
            
            shouldShowRequestPermissionRationale(
                Manifest.permission.ACCESS_FINE_LOCATION
            ) -> {
                showRationaleDialog {
                    requestPermissionLauncher.launch(
                        Manifest.permission.ACCESS_FINE_LOCATION
                    )
                }
            }
            
            else -> {
                requestPermissionLauncher.launch(
                    Manifest.permission.ACCESS_FINE_LOCATION
                )
            }
        }
    }
    
    fun checkCameraPermissions() {
        val permissions = arrayOf(
            Manifest.permission.CAMERA,
            Manifest.permission.RECORD_AUDIO
        )
        
        val notGranted = permissions.filter {
            ContextCompat.checkSelfPermission(this, it) != PackageManager.PERMISSION_GRANTED
        }
        
        if (notGranted.isEmpty()) {
            startCamera()
        } else {
            requestMultiplePermissionsLauncher.launch(notGranted.toTypedArray())
        }
    }
    
    private fun accessLocation() { /* use location */ }
    private fun startCamera() { /* open camera */ }
    private fun showPermissionDeniedMessage() { /* show UI */ }
    private fun showDeniedPermissions(denied: Set<String>) { /* show UI */ }
    
    private fun showRationaleDialog(onConfirm: () -> Unit) {
        AlertDialog.Builder(this)
            .setTitle("ต้องการ Permission")
            .setMessage("แอปต้องการ Location permission เพื่อแสดงสถานที่ใกล้เคียง")
            .setPositiveButton("อนุญาต") { _, _ -> onConfirm() }
            .setNegativeButton("ปฏิเสธ", null)
            .show()
    }
}
```

---

## ขั้นตอนที่ 527: Lifecycle ใน Jetpack Compose

```kotlin
// ใน Compose ใช้ LocalLifecycleOwner
@Composable
fun LifecycleAwareComponent() {
    val lifecycleOwner = LocalLifecycleOwner.current
    
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> println("Resumed")
                Lifecycle.Event.ON_PAUSE -> println("Paused")
                Lifecycle.Event.ON_STOP -> println("Stopped")
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
    
    // UI content
}

// repeatOnLifecycle ใน Compose
@Composable
fun CollectWithLifecycle() {
    val viewModel: MyViewModel = hiltViewModel()
    
    // collectAsStateWithLifecycle - pause collection เมื่อ app ไป background
    val state by viewModel.state.collectAsStateWithLifecycle()
    
    // ทั้งสองวิธีนี้ทำงานเหมือนกัน
    // val state by viewModel.state.collectAsState()  // ไม่ pause
    
    Text(text = state.toString())
}

// Lifecycle-aware side effects
@Composable
fun LifecycleSideEffects() {
    val context = LocalContext.current
    
    // LaunchedEffect ทำงาน 1 ครั้งตอน composition
    LaunchedEffect(Unit) {
        println("Composed!")
    }
    
    // DisposableEffect - cleanup เมื่อ leave composition
    DisposableEffect(Unit) {
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                println("Received broadcast")
            }
        }
        context.registerReceiver(receiver, IntentFilter("MY_ACTION"))
        
        onDispose {
            context.unregisterReceiver(receiver)
        }
    }
}
```

---

## ขั้นตอนที่ 528: Back Stack และ Task Management

```kotlin
// Task = stack ของ Activities
// Back Stack = ลำดับของ Activities ที่เปิด

class TaskManagementActivity : AppCompatActivity() {
    
    fun openActivityClearTop(targetClass: Class<*>) {
        val intent = Intent(this, targetClass).apply {
            // ถ้า Activity มีอยู่แล้วใน stack, clear activities ที่อยู่เหนือมัน
            flags = Intent.FLAG_ACTIVITY_CLEAR_TOP or Intent.FLAG_ACTIVITY_SINGLE_TOP
        }
        startActivity(intent)
    }
    
    fun openActivityNewTask(targetClass: Class<*>) {
        val intent = Intent(this, targetClass).apply {
            // เปิดใน task ใหม่
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        startActivity(intent)
    }
    
    fun returnToHome() {
        // ปิด Activity ทั้งหมด กลับ home
        val intent = Intent(this, HomeActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_CLEAR_TASK or Intent.FLAG_ACTIVITY_NEW_TASK
        }
        startActivity(intent)
    }
}

// Activity launchMode ใน AndroidManifest
// android:launchMode="standard"      - default, สร้าง instance ใหม่เสมอ
// android:launchMode="singleTop"     - ถ้า top แล้ว ใช้ instance เดิม
// android:launchMode="singleTask"    - มีแค่ 1 instance ต่อ task
// android:launchMode="singleInstance"- มีแค่ 1 instance ทั้ง device
```

---

## แบบฝึกหัด Part 22

```kotlin
// แบบฝึกหัด: สร้าง Activity ที่จัดการ state ได้ถูกต้อง
// เมื่อ rotate screen, counter ต้องไม่ reset

class CounterActivity : ComponentActivity() {
    
    // TODO: ใช้ ViewModel เพื่อเก็บ state
    // private val viewModel: CounterViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        setContent {
            // TODO: แสดง counter และปุ่ม increment/decrement
            // ใช้ ViewModel state
        }
    }
}

class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()
    
    fun increment() { _count.update { it + 1 } }
    fun decrement() { _count.update { it - 1 } }
    fun reset() { _count.value = 0 }
}
```

---

*Part 22 จบแล้ว | ก่อนหน้า: [Part 21](../part21/README.md) | ถัดไป: [Part 23](../part23/README.md)*
