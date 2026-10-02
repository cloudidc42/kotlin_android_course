# Part 34: Compose - LazyColumn, LazyRow (เทียบเท่า RecyclerView)
## ขั้นตอนที่ 626-650

---

## ขั้นตอนที่ 626: LazyColumn พื้นฐาน

LazyColumn เทียบเท่า RecyclerView แบบ vertical ใน XML - render เฉพาะ item ที่มองเห็น ประหยัด memory

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.itemsIndexed
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

data class Product(
    val id: Int,
    val name: String,
    val price: Double,
    val category: String
)

val sampleProducts = (1..50).map { i ->
    Product(
        id = i,
        name = "สินค้า #$i",
        price = (100..5000).random().toDouble(),
        category = listOf("อาหาร", "เครื่องดื่ม", "อิเล็กทรอนิกส์").random()
    )
}

@Composable
fun BasicLazyColumn() {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        // item เดี่ยว (header)
        item {
            Text(
                text = "รายการสินค้า",
                style = MaterialTheme.typography.headlineMedium
            )
        }

        // items จาก List
        items(sampleProducts) { product ->
            ProductCard(product = product)
        }

        // item เดี่ยว (footer)
        item {
            Text(
                text = "ทั้งหมด ${sampleProducts.size} รายการ",
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}

@Composable
fun ProductCard(product: Product) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier = Modifier.padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Column {
                Text(text = product.name, style = MaterialTheme.typography.titleMedium)
                Text(text = product.category, style = MaterialTheme.typography.bodySmall)
            }
            Text(
                text = "฿${product.price}",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}
```

---

## ขั้นตอนที่ 627: LazyColumn พร้อม Index และ Key

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.itemsIndexed
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun LazyColumnWithIndexAndKey() {
    var products by remember { mutableStateOf(sampleProducts) }

    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // itemsIndexed - ได้ทั้ง index และ item
        // key - ช่วยให้ Compose track item ได้ถูกต้องเมื่อ list เปลี่ยน
        itemsIndexed(
            items = products,
            key = { _, product -> product.id } // key ต้องไม่ซ้ำกัน
        ) { index, product ->
            ListItem(
                headlineContent = { Text("${index + 1}. ${product.name}") },
                supportingContent = { Text(product.category) },
                trailingContent = {
                    Text(
                        "฿${product.price}",
                        color = MaterialTheme.colorScheme.primary
                    )
                },
                tonalElevation = if (index % 2 == 0) 2.dp else 0.dp
            )
            if (index < products.lastIndex) {
                Divider()
            }
        }
    }
}

// LazyColumn แบบแบ่ง section
@Composable
fun GroupedLazyColumn() {
    val grouped = sampleProducts.groupBy { it.category }

    LazyColumn(contentPadding = PaddingValues(16.dp)) {
        grouped.forEach { (category, items) ->
            // stickyHeader - หัวข้อ sticky
            stickyHeader {
                Surface(color = MaterialTheme.colorScheme.primaryContainer) {
                    Text(
                        text = category,
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(8.dp),
                        style = MaterialTheme.typography.titleSmall
                    )
                }
            }

            items(items, key = { it.id }) { product ->
                ProductCard(product = product)
                Spacer(Modifier.height(4.dp))
            }
        }
    }
}
```

---

## ขั้นตอนที่ 628: LazyRow พื้นฐาน

LazyRow คือ RecyclerView แบบ horizontal เหมาะสำหรับ stories, chips, categories

```kotlin
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

data class Category(val id: Int, val name: String, val icon: String)

val categories = listOf(
    Category(1, "ทั้งหมด", "🏠"),
    Category(2, "อาหาร", "🍔"),
    Category(3, "เครื่องดื่ม", "🍹"),
    Category(4, "อิเล็กทรอนิกส์", "💻"),
    Category(5, "เสื้อผ้า", "👔"),
    Category(6, "กีฬา", "⚽"),
    Category(7, "หนังสือ", "📚")
)

@Composable
fun CategoryRow() {
    var selectedCategory by remember { mutableStateOf(1) }

    LazyRow(
        contentPadding = PaddingValues(horizontal = 16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(categories, key = { it.id }) { category ->
            FilterChip(
                selected = selectedCategory == category.id,
                onClick = { selectedCategory = category.id },
                label = { Text("${category.icon} ${category.name}") }
            )
        }
    }
}

// Horizontal Card List
@Composable
fun HorizontalProductList() {
    LazyRow(
        contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(sampleProducts.take(10), key = { it.id }) { product ->
            Card(
                modifier = Modifier.width(160.dp),
                shape = RoundedCornerShape(12.dp)
            ) {
                Column(modifier = Modifier.padding(12.dp)) {
                    Box(
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(100.dp)
                            .background(MaterialTheme.colorScheme.primaryContainer)
                    )
                    Spacer(Modifier.height(8.dp))
                    Text(
                        text = product.name,
                        style = MaterialTheme.typography.bodyMedium,
                        maxLines = 2,
                        overflow = TextOverflow.Ellipsis
                    )
                    Text(
                        text = "฿${product.price}",
                        style = MaterialTheme.typography.titleSmall,
                        color = MaterialTheme.colorScheme.primary
                    )
                }
            }
        }
    }
}
```

---

## ขั้นตอนที่ 629: LazyVerticalGrid และ LazyHorizontalGrid

```kotlin
import androidx.compose.foundation.lazy.grid.*
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun ProductGrid() {
    // Grid แบบ fixed columns
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.fillMaxSize()
    ) {
        item(span = { GridItemSpan(maxLineSpan) }) {
            // item ที่กว้างเต็ม 2 คอลัมน์
            Text(
                text = "สินค้าทั้งหมด",
                style = MaterialTheme.typography.headlineSmall,
                modifier = Modifier.padding(bottom = 8.dp)
            )
        }

        items(sampleProducts, key = { it.id }) { product ->
            GridProductCard(product = product)
        }
    }
}

@Composable
fun GridProductCard(product: Product) {
    Card(modifier = Modifier.fillMaxWidth()) {
        Column(modifier = Modifier.padding(12.dp)) {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(120.dp)
                    .background(
                        MaterialTheme.colorScheme.surfaceVariant,
                        shape = RoundedCornerShape(8.dp)
                    )
            )
            Spacer(Modifier.height(8.dp))
            Text(text = product.name, style = MaterialTheme.typography.bodyMedium)
            Text(
                text = "฿${product.price}",
                style = MaterialTheme.typography.titleSmall,
                color = MaterialTheme.colorScheme.primary
            )
        }
    }
}

// Adaptive Grid - ปรับ column อัตโนมัติตามความกว้างหน้าจอ
@Composable
fun AdaptiveGrid() {
    LazyVerticalGrid(
        columns = GridCells.Adaptive(minSize = 150.dp), // อย่างน้อย 150dp ต่อ column
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(sampleProducts) { product ->
            GridProductCard(product = product)
        }
    }
}
```

---

## ขั้นตอนที่ 630: LazyColumn พร้อม Swipe to Delete และ Pull to Refresh

```kotlin
import androidx.compose.animation.animateColorAsState
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material3.*
import androidx.compose.material3.pulltorefresh.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.scale
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeToDeleteList() {
    var items by remember {
        mutableStateOf((1..20).map { "รายการที่ $it" }.toMutableList())
    }
    val scope = rememberCoroutineScope()
    var isRefreshing by remember { mutableStateOf(false) }
    val pullRefreshState = rememberPullToRefreshState()

    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn(
            modifier = Modifier.fillMaxSize(),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            items(
                items = items,
                key = { it }
            ) { item ->
                val dismissState = rememberSwipeToDismissBoxState(
                    confirmValueChange = { value ->
                        if (value == SwipeToDismissBoxValue.EndToStart) {
                            scope.launch {
                                delay(300)
                                items = items.toMutableList().also { it.remove(item) }
                            }
                            true
                        } else false
                    }
                )

                SwipeToDismissBox(
                    state = dismissState,
                    backgroundContent = {
                        val color by animateColorAsState(
                            when (dismissState.targetValue) {
                                SwipeToDismissBoxValue.EndToStart -> Color.Red
                                else -> Color.Transparent
                            }
                        )
                        Box(
                            modifier = Modifier
                                .fillMaxSize()
                                .background(color)
                                .padding(end = 16.dp),
                            contentAlignment = Alignment.CenterEnd
                        ) {
                            Icon(
                                Icons.Default.Delete,
                                contentDescription = "ลบ",
                                tint = Color.White
                            )
                        }
                    }
                ) {
                    Card(modifier = Modifier.fillMaxWidth()) {
                        Text(
                            text = item,
                            modifier = Modifier.padding(16.dp)
                        )
                    }
                }
            }
        }
    }
}
```

---

*Part 34 จบแล้ว | ก่อนหน้า: [Part 33](../part33/README.md) | ถัดไป: [Part 35](../part35/README.md)*
