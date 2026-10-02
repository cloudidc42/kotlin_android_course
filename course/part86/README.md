# Part 86: Real-Time Features (WebSocket & SSE)
## ขั้นตอนที่ 1626-1650

---

## ขั้นตอนที่ 1626: Real-Time Communication Options

```
Real-Time Options:

1. WebSocket - full-duplex, persistent connection
   - Chat apps, live trading, gaming
   - OkHttp WebSocket, Socket.IO

2. Server-Sent Events (SSE) - server → client only
   - Live feeds, notifications, progress updates
   - Simpler than WebSocket

3. Long Polling - HTTP request ที่ server hold จนมีข้อมูล
   - Fallback เมื่อ WebSocket ไม่รองรับ

4. Firebase Realtime Database / Firestore
   - Managed real-time sync

5. gRPC Streaming - high performance
   - Internal services
```

---

## ขั้นตอนที่ 1627: OkHttp WebSocket

```kotlin
class WebSocketManager @Inject constructor(
    private val okHttpClient: OkHttpClient,
    private val json: Json
) {
    
    private var webSocket: WebSocket? = null
    
    private val _messages = MutableSharedFlow<ChatMessage>()
    val messages: SharedFlow<ChatMessage> = _messages.asSharedFlow()
    
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState.asStateFlow()
    
    fun connect(roomId: String, token: String) {
        val request = Request.Builder()
            .url("wss://api.myapp.com/ws/chat/$roomId")
            .addHeader("Authorization", "Bearer $token")
            .build()
        
        webSocket = okHttpClient.newWebSocket(request, object : WebSocketListener() {
            
            override fun onOpen(webSocket: WebSocket, response: Response) {
                _connectionState.value = ConnectionState.CONNECTED
                // Send authentication message
                webSocket.send(json.encodeToString(AuthMessage(token)))
            }
            
            override fun onMessage(webSocket: WebSocket, text: String) {
                val message = json.decodeFromString<WebSocketMessage>(text)
                
                when (message.type) {
                    "chat" -> {
                        val chatMessage = json.decodeFromString<ChatMessage>(message.payload)
                        CoroutineScope(Dispatchers.IO).launch {
                            _messages.emit(chatMessage)
                        }
                    }
                    "typing" -> { /* handle typing indicator */ }
                    "read" -> { /* handle read receipt */ }
                }
            }
            
            override fun onClosing(webSocket: WebSocket, code: Int, reason: String) {
                webSocket.close(1000, null)
            }
            
            override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
                _connectionState.value = ConnectionState.DISCONNECTED
            }
            
            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                _connectionState.value = ConnectionState.ERROR
                // Auto-reconnect with exponential backoff
                scheduleReconnect()
            }
        })
    }
    
    fun sendMessage(roomId: String, content: String) {
        val message = SendMessageRequest(
            type = "chat",
            roomId = roomId,
            content = content,
            timestamp = System.currentTimeMillis()
        )
        webSocket?.send(json.encodeToString(message))
    }
    
    fun sendTyping(roomId: String) {
        webSocket?.send(json.encodeToString(TypingRequest(roomId)))
    }
    
    fun disconnect() {
        webSocket?.close(1000, "User disconnected")
        webSocket = null
        _connectionState.value = ConnectionState.DISCONNECTED
    }
    
    private var reconnectJob: Job? = null
    private var reconnectAttempt = 0
    
    private fun scheduleReconnect() {
        reconnectJob?.cancel()
        reconnectJob = CoroutineScope(Dispatchers.IO).launch {
            val delayMs = minOf(1000L * (1 shl reconnectAttempt), 30_000L)
            delay(delayMs)
            reconnectAttempt++
            // reconnect with saved credentials
        }
    }
    
    enum class ConnectionState {
        DISCONNECTED, CONNECTING, CONNECTED, ERROR
    }
}

@Serializable
data class WebSocketMessage(val type: String, val payload: String)

@Serializable
data class ChatMessage(
    val id: String,
    val senderId: Long,
    val senderName: String,
    val content: String,
    val timestamp: Long,
    val roomId: String
)
```

---

## ขั้นตอนที่ 1628: Chat UI ด้วย Compose

```kotlin
@HiltViewModel
class ChatViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val webSocketManager: WebSocketManager,
    private val chatRepository: ChatRepository,
    private val userPreferences: UserPreferences
) : ViewModel() {
    
    private val roomId = savedStateHandle.get<String>("roomId") ?: ""
    
    data class UiState(
        val messages: List<ChatMessage> = emptyList(),
        val typingUsers: Set<String> = emptySet(),
        val connectionState: WebSocketManager.ConnectionState = WebSocketManager.ConnectionState.DISCONNECTED,
        val inputText: String = "",
        val isLoadingHistory: Boolean = false
    )
    
    private val _uiState = MutableStateFlow(UiState())
    val uiState = _uiState.asStateFlow()
    
    init {
        loadHistory()
        observeMessages()
        connectWebSocket()
    }
    
    private fun loadHistory() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoadingHistory = true) }
            val history = chatRepository.getMessageHistory(roomId)
            _uiState.update { it.copy(messages = history, isLoadingHistory = false) }
        }
    }
    
    private fun observeMessages() {
        viewModelScope.launch {
            webSocketManager.messages.collect { message ->
                if (message.roomId == roomId) {
                    _uiState.update { state ->
                        state.copy(messages = state.messages + message)
                    }
                }
            }
        }
        
        viewModelScope.launch {
            webSocketManager.connectionState.collect { state ->
                _uiState.update { it.copy(connectionState = state) }
            }
        }
    }
    
    private fun connectWebSocket() {
        viewModelScope.launch {
            val token = userPreferences.getAuthToken() ?: return@launch
            webSocketManager.connect(roomId, token)
        }
    }
    
    fun onInputChange(text: String) {
        _uiState.update { it.copy(inputText = text) }
        webSocketManager.sendTyping(roomId)
    }
    
    fun sendMessage() {
        val text = _uiState.value.inputText.trim()
        if (text.isBlank()) return
        
        _uiState.update { it.copy(inputText = "") }
        webSocketManager.sendMessage(roomId, text)
    }
    
    override fun onCleared() {
        webSocketManager.disconnect()
        super.onCleared()
    }
}

@Composable
fun ChatScreen(
    roomId: String,
    currentUserId: Long,
    viewModel: ChatViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()
    
    // Auto-scroll to latest message
    LaunchedEffect(uiState.messages.size) {
        if (uiState.messages.isNotEmpty()) {
            listState.animateScrollToItem(uiState.messages.size - 1)
        }
    }
    
    Scaffold(
        topBar = {
            ChatTopBar(
                title = "ห้องสนทนา",
                connectionState = uiState.connectionState
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
        ) {
            // Messages
            LazyColumn(
                state = listState,
                modifier = Modifier.weight(1f),
                contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(uiState.messages, key = { it.id }) { message ->
                    MessageBubble(
                        message = message,
                        isFromCurrentUser = message.senderId == currentUserId
                    )
                }
                
                // Typing indicator
                if (uiState.typingUsers.isNotEmpty()) {
                    item {
                        TypingIndicator(users = uiState.typingUsers.toList())
                    }
                }
            }
            
            // Input bar
            ChatInputBar(
                text = uiState.inputText,
                onTextChange = viewModel::onInputChange,
                onSend = viewModel::sendMessage
            )
        }
    }
}

@Composable
fun MessageBubble(
    message: ChatMessage,
    isFromCurrentUser: Boolean
) {
    val bubbleColor = if (isFromCurrentUser) {
        MaterialTheme.colorScheme.primary
    } else {
        MaterialTheme.colorScheme.surfaceVariant
    }
    
    val textColor = if (isFromCurrentUser) {
        MaterialTheme.colorScheme.onPrimary
    } else {
        MaterialTheme.colorScheme.onSurfaceVariant
    }
    
    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = if (isFromCurrentUser) Arrangement.End else Arrangement.Start
    ) {
        if (!isFromCurrentUser) {
            AsyncImage(
                model = message.senderName,
                contentDescription = null,
                modifier = Modifier.size(32.dp).clip(CircleShape)
            )
            Spacer(Modifier.width(8.dp))
        }
        
        Column(horizontalAlignment = if (isFromCurrentUser) Alignment.End else Alignment.Start) {
            if (!isFromCurrentUser) {
                Text(
                    message.senderName,
                    style = MaterialTheme.typography.labelSmall,
                    color = MaterialTheme.colorScheme.primary
                )
            }
            
            Box(
                modifier = Modifier
                    .background(bubbleColor, RoundedCornerShape(16.dp))
                    .padding(horizontal = 12.dp, vertical = 8.dp)
            ) {
                Text(message.content, color = textColor)
            }
            
            Text(
                text = formatTime(message.timestamp),
                style = MaterialTheme.typography.labelSmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## แบบฝึกหัด Part 86

```kotlin
// แบบฝึกหัด: Live Stock Price Feed ด้วย SSE

// Server-Sent Events สำหรับ live stock prices
class StockPriceFeed @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    
    fun observePrices(symbols: List<String>): Flow<StockPrice> = flow {
        val symbolList = symbols.joinToString(",")
        val request = Request.Builder()
            .url("https://api.example.com/stocks/live?symbols=$symbolList")
            .addHeader("Accept", "text/event-stream")
            .build()
        
        okHttpClient.newCall(request).execute().use { response ->
            val reader = response.body?.charStream()?.buffered() ?: return@flow
            
            var dataBuffer = StringBuilder()
            
            for (line in reader.lineSequence()) {
                when {
                    line.startsWith("data: ") -> {
                        dataBuffer.append(line.removePrefix("data: "))
                    }
                    line.isEmpty() && dataBuffer.isNotEmpty() -> {
                        // End of event - parse and emit
                        val price = Json.decodeFromString<StockPrice>(dataBuffer.toString())
                        emit(price)
                        dataBuffer.clear()
                    }
                }
            }
        }
    }.flowOn(Dispatchers.IO)
        .retry(3) { delay(5000); true }
}

@Serializable
data class StockPrice(
    val symbol: String,
    val price: Double,
    val change: Double,
    val changePercent: Double,
    val timestamp: Long
)

// TODO: สร้าง StockDashboardScreen ที่:
// 1. แสดงรายการหุ้นพร้อมราคา live
// 2. สีเขียว/แดง ตาม change direction
// 3. Mini sparkline chart
// 4. Connection status indicator
```

---

*Part 86 จบแล้ว | ก่อนหน้า: [Part 85](../part85/README.md) | ถัดไป: [Part 87](../part87/README.md)*
