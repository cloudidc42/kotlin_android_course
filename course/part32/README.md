# Part 32: Compose - Text, Button, Image พื้นฐาน
## ขั้นตอนที่ 576-600

---

## ขั้นตอนที่ 576: Text Component ใน Compose

Text คือ component พื้นฐานที่สุดใน Compose ใช้แสดงข้อความ รองรับ styling หลากหลาย

```kotlin
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.sp

@Composable
fun TextExamples() {
    // Text ธรรมดา
    Text(text = "สวัสดีโลก")

    // Text พร้อม styling
    Text(
        text = "หัวข้อสำคัญ",
        fontSize = 24.sp,
        fontWeight = FontWeight.Bold,
        color = Color.Blue
    )

    // Text จัดกึ่งกลาง
    Text(
        text = "ข้อความกึ่งกลาง",
        textAlign = TextAlign.Center
    )

    // Text ที่ตัดเมื่อยาวเกิน
    Text(
        text = "ข้อความที่ยาวมากๆ ซึ่งอาจจะยาวเกินหน้าจอและต้องถูกตัด",
        maxLines = 1,
        overflow = TextOverflow.Ellipsis
    )

    // Text หลายบรรทัด
    Text(
        text = "บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3",
        lineHeight = 24.sp
    )
}
```

---

## ขั้นตอนที่ 577: AnnotatedString สำหรับ Text หลายสไตล์

AnnotatedString ช่วยให้ข้อความเดียวกันมีหลาย style เช่น บางคำหนา บางคำมีสี

```kotlin
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.SpanStyle
import androidx.compose.ui.text.buildAnnotatedString
import androidx.compose.ui.text.font.FontStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.withStyle

@Composable
fun StyledTextExample() {
    val annotatedString = buildAnnotatedString {
        append("ราคา: ")

        withStyle(style = SpanStyle(
            color = Color.Red,
            fontWeight = FontWeight.Bold,
            fontSize = 20.sp
        )) {
            append("฿299")
        }

        append(" (")

        withStyle(style = SpanStyle(
            color = Color.Gray,
            fontStyle = FontStyle.Italic
        )) {
            append("ลด 30%")
        }

        append(")")
    }

    Text(text = annotatedString)
}

// ใช้ HyperlinkText
@Composable
fun HyperlinkText() {
    val annotatedString = buildAnnotatedString {
        append("อ่านเพิ่มเติมที่ ")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com")
        withStyle(style = SpanStyle(color = Color.Blue)) {
            append("คลิกที่นี่")
        }
        pop()
    }

    Text(text = annotatedString)
}
```

---

## ขั้นตอนที่ 578: Button Component ใน Compose

Button ใน Compose มีหลายแบบ - FilledButton, OutlinedButton, TextButton, ElevatedButton

```kotlin
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun ButtonExamples() {
    var clickCount by remember { mutableStateOf(0) }

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // Filled Button (ค่าเริ่มต้น)
        Button(onClick = { clickCount++ }) {
            Text("กดแล้ว $clickCount ครั้ง")
        }

        // Outlined Button
        OutlinedButton(onClick = { clickCount = 0 }) {
            Text("รีเซ็ต")
        }

        // Text Button (ไม่มีกรอบ)
        TextButton(onClick = { }) {
            Text("ยกเลิก")
        }

        // Elevated Button
        ElevatedButton(onClick = { }) {
            Text("Elevated")
        }

        // Filled Tonal Button
        FilledTonalButton(onClick = { }) {
            Text("Tonal")
        }

        // Disabled Button
        Button(
            onClick = { },
            enabled = false
        ) {
            Text("ปิดใช้งาน")
        }
    }
}
```

---

## ขั้นตอนที่ 579: Button พร้อม Icon

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun ButtonWithIconExamples() {
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // Button พร้อม Icon ด้านซ้าย
        Button(onClick = { }) {
            Icon(
                imageVector = Icons.Default.Add,
                contentDescription = "เพิ่ม",
                modifier = Modifier.size(ButtonDefaults.IconSize)
            )
            Spacer(modifier = Modifier.size(ButtonDefaults.IconSpacing))
            Text("เพิ่มรายการ")
        }

        // Button พร้อม Icon ด้านขวา
        OutlinedButton(onClick = { }) {
            Text("แชร์")
            Spacer(modifier = Modifier.size(ButtonDefaults.IconSpacing))
            Icon(
                imageVector = Icons.Default.Share,
                contentDescription = "แชร์",
                modifier = Modifier.size(ButtonDefaults.IconSize)
            )
        }

        // IconButton (ปุ่มที่มีแค่ไอคอน)
        IconButton(onClick = { }) {
            Icon(
                imageVector = Icons.Default.Favorite,
                contentDescription = "ชื่นชอบ"
            )
        }

        // FAB (Floating Action Button)
        FloatingActionButton(onClick = { }) {
            Icon(Icons.Default.Add, contentDescription = "เพิ่ม")
        }

        // Small FAB
        SmallFloatingActionButton(onClick = { }) {
            Icon(Icons.Default.Edit, contentDescription = "แก้ไข")
        }
    }
}
```

---

## ขั้นตอนที่ 580: Image Component ใน Compose

Image ใน Compose รองรับ Vector, Bitmap และ Network image (ผ่าน Coil)

```kotlin
import androidx.compose.foundation.Image
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.ColorFilter
import androidx.compose.ui.graphics.ColorMatrix
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.res.painterResource
import androidx.compose.ui.unit.dp

@Composable
fun ImageExamples() {
    Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // รูปจาก Resource
        Image(
            painter = painterResource(id = R.drawable.ic_launcher_foreground),
            contentDescription = "App Icon",
            modifier = Modifier.size(100.dp)
        )

        // รูปทรงกลม (Avatar)
        Image(
            painter = painterResource(id = R.drawable.ic_launcher_foreground),
            contentDescription = "Avatar",
            modifier = Modifier
                .size(80.dp)
                .clip(CircleShape),
            contentScale = ContentScale.Crop
        )

        // รูปมุมมน
        Image(
            painter = painterResource(id = R.drawable.ic_launcher_foreground),
            contentDescription = "Rounded Image",
            modifier = Modifier
                .size(120.dp)
                .clip(RoundedCornerShape(16.dp)),
            contentScale = ContentScale.Fit
        )

        // รูปขาวดำ (Grayscale)
        Image(
            painter = painterResource(id = R.drawable.ic_launcher_foreground),
            contentDescription = "Grayscale",
            modifier = Modifier.size(80.dp),
            colorFilter = ColorFilter.colorMatrix(ColorMatrix().apply { setToSaturation(0f) })
        )
    }
}

// ContentScale options:
// ContentScale.Crop - ครอบตามขนาด (ตัดบางส่วน)
// ContentScale.Fit - พอดีทั้งรูป (อาจมีพื้นที่ว่าง)
// ContentScale.FillBounds - ยืดเต็มพื้นที่ (อาจบิดเบี้ยว)
// ContentScale.Inside - เหมือน Fit แต่ไม่ขยายเกินขนาดจริง
```

---

## ขั้นตอนที่ 581: Icon Component และ Vector Icons

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.unit.dp

@Composable
fun IconExamples() {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // Icon พื้นฐาน
        Icon(
            imageVector = Icons.Default.Home,
            contentDescription = "หน้าแรก"
        )

        // Icon มีสี
        Icon(
            imageVector = Icons.Default.Favorite,
            contentDescription = "ชื่นชอบ",
            tint = Color.Red
        )

        // Icon ขนาดใหญ่
        Icon(
            imageVector = Icons.Default.Star,
            contentDescription = "ดาว",
            modifier = Modifier.size(48.dp),
            tint = Color(0xFFFFC107)
        )

        // Outlined Icons (เส้นขอบ ไม่เติม)
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Icon(Icons.Filled.Email, contentDescription = null) // เติม
            Icon(Icons.Outlined.Email, contentDescription = null) // เส้นขอบ
        }

        // Icon ใน Badge
        BadgedBox(badge = {
            Badge { Text("3") }
        }) {
            Icon(Icons.Default.Notifications, contentDescription = "การแจ้งเตือน")
        }
    }
}
```

---

*Part 32 จบแล้ว | ก่อนหน้า: [Part 31](../part31/README.md) | ถัดไป: [Part 33](../part33/README.md)*
