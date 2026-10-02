# Part 33: Compose - TextField, Checkbox, Switch, Slider
## ขั้นตอนที่ 601-625

---

## ขั้นตอนที่ 601: TextField พื้นฐาน

TextField ใช้รับข้อมูลจากผู้ใช้ ใน Compose ต้องจัดการ state เอง

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp

@Composable
fun TextFieldExamples() {
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    var phone by remember { mutableStateOf("") }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // TextField ธรรมดา
        TextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("ชื่อ") },
            placeholder = { Text("กรอกชื่อของคุณ") }
        )

        // OutlinedTextField (สวยกว่า)
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("อีเมล") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction = ImeAction.Next
            )
        )

        // TextField สำหรับเบอร์โทร
        OutlinedTextField(
            value = phone,
            onValueChange = { phone = it.filter { c -> c.isDigit() } },
            label = { Text("เบอร์โทร") },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Phone
            ),
            prefix = { Text("+66 ") }
        )

        Text("ชื่อ: $name | อีเมล: $email | โทร: $phone")
    }
}
```

---

## ขั้นตอนที่ 602: TextField พร้อม Validation

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Clear
import androidx.compose.material.icons.filled.Visibility
import androidx.compose.material.icons.filled.VisibilityOff
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.text.input.VisualTransformation
import androidx.compose.ui.unit.dp

@Composable
fun ValidatedTextField() {
    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var passwordVisible by remember { mutableStateOf(false) }
    var usernameError by remember { mutableStateOf("") }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // Username พร้อม validation
        OutlinedTextField(
            value = username,
            onValueChange = {
                username = it
                usernameError = when {
                    it.isEmpty() -> "กรุณากรอกชื่อผู้ใช้"
                    it.length < 3 -> "ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัวอักษร"
                    else -> ""
                }
            },
            label = { Text("ชื่อผู้ใช้") },
            isError = usernameError.isNotEmpty(),
            supportingText = {
                if (usernameError.isNotEmpty()) {
                    Text(usernameError, color = MaterialTheme.colorScheme.error)
                }
            },
            trailingIcon = {
                if (username.isNotEmpty()) {
                    IconButton(onClick = { username = "" }) {
                        Icon(Icons.Default.Clear, "ล้างข้อความ")
                    }
                }
            }
        )

        // Password Field
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("รหัสผ่าน") },
            visualTransformation = if (passwordVisible)
                VisualTransformation.None
            else
                PasswordVisualTransformation(),
            trailingIcon = {
                IconButton(onClick = { passwordVisible = !passwordVisible }) {
                    Icon(
                        if (passwordVisible) Icons.Default.Visibility
                        else Icons.Default.VisibilityOff,
                        contentDescription = if (passwordVisible) "ซ่อนรหัสผ่าน" else "แสดงรหัสผ่าน"
                    )
                }
            }
        )
    }
}
```

---

## ขั้นตอนที่ 603: Checkbox และ RadioButton

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun CheckboxExamples() {
    var isChecked by remember { mutableStateOf(false) }
    var termsAccepted by remember { mutableStateOf(false) }
    var selectedOption by remember { mutableStateOf("option1") }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // Checkbox ธรรมดา
        Row(verticalAlignment = Alignment.CenterVertically) {
            Checkbox(
                checked = isChecked,
                onCheckedChange = { isChecked = it }
            )
            Text("ฉันยอมรับเงื่อนไข")
        }

        // Checkbox ที่คลิกทั้ง Row ได้
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.clickable { termsAccepted = !termsAccepted }
        ) {
            Checkbox(
                checked = termsAccepted,
                onCheckedChange = { termsAccepted = it }
            )
            Spacer(Modifier.width(4.dp))
            Text("ยอมรับข้อกำหนดและนโยบายความเป็นส่วนตัว")
        }

        Divider()

        // RadioButton Group
        Text("เลือกขนาด:", style = MaterialTheme.typography.labelLarge)
        val options = listOf("เล็ก (S)", "กลาง (M)", "ใหญ่ (L)", "ใหญ่พิเศษ (XL)")
        options.forEach { option ->
            Row(
                verticalAlignment = Alignment.CenterVertically,
                modifier = Modifier.clickable { selectedOption = option }
            ) {
                RadioButton(
                    selected = selectedOption == option,
                    onClick = { selectedOption = option }
                )
                Spacer(Modifier.width(4.dp))
                Text(option)
            }
        }

        Text("เลือก: $selectedOption")
    }
}
```

---

## ขั้นตอนที่ 604: Switch และ TriStateCheckbox

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.state.ToggleableState
import androidx.compose.ui.unit.dp

@Composable
fun SwitchExamples() {
    var notificationsEnabled by remember { mutableStateOf(true) }
    var darkMode by remember { mutableStateOf(false) }
    var autoUpdate by remember { mutableStateOf(true) }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // Switch ธรรมดา
        Row(
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("การแจ้งเตือน")
            Switch(
                checked = notificationsEnabled,
                onCheckedChange = { notificationsEnabled = it }
            )
        }

        // Switch พร้อม Icon
        Row(
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("โหมดกลางคืน")
            Switch(
                checked = darkMode,
                onCheckedChange = { darkMode = it },
                thumbContent = if (darkMode) {
                    { Icon(Icons.Default.DarkMode, null, Modifier.size(SwitchDefaults.IconSize)) }
                } else null
            )
        }

        Divider()

        // TriStateCheckbox (Indeterminate state)
        var childState1 by remember { mutableStateOf(false) }
        var childState2 by remember { mutableStateOf(false) }

        val parentState = when {
            childState1 && childState2 -> ToggleableState.On
            !childState1 && !childState2 -> ToggleableState.Off
            else -> ToggleableState.Indeterminate
        }

        Row(verticalAlignment = Alignment.CenterVertically) {
            TriStateCheckbox(
                state = parentState,
                onClick = {
                    val newState = parentState != ToggleableState.On
                    childState1 = newState
                    childState2 = newState
                }
            )
            Text("เลือกทั้งหมด")
        }

        Row(verticalAlignment = Alignment.CenterVertically) {
            Spacer(Modifier.width(32.dp))
            Checkbox(checked = childState1, onCheckedChange = { childState1 = it })
            Text("รายการ 1")
        }

        Row(verticalAlignment = Alignment.CenterVertically) {
            Spacer(Modifier.width(32.dp))
            Checkbox(checked = childState2, onCheckedChange = { childState2 = it })
            Text("รายการ 2")
        }
    }
}
```

---

## ขั้นตอนที่ 605: Slider และ RangeSlider

```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import kotlin.math.roundToInt

@Composable
fun SliderExamples() {
    var volume by remember { mutableStateOf(0.5f) }
    var brightness by remember { mutableStateOf(0.7f) }
    var priceRange by remember { mutableStateOf(200f..800f) }

    Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // Slider ธรรมดา
        Text("ระดับเสียง: ${(volume * 100).roundToInt()}%")
        Slider(
            value = volume,
            onValueChange = { volume = it },
            modifier = Modifier.fillMaxWidth()
        )

        // Slider แบบ Steps (เลื่อนทีละ 10%)
        Text("ความสว่าง: ${(brightness * 100).roundToInt()}%")
        Slider(
            value = brightness,
            onValueChange = { brightness = it },
            steps = 9, // แบ่ง 10 ช่อง = 11 ตำแหน่ง (0,10,20,...,100)
            modifier = Modifier.fillMaxWidth()
        )

        // RangeSlider สำหรับช่วงราคา
        Text("ช่วงราคา: ฿${priceRange.start.roundToInt()} - ฿${priceRange.endInclusive.roundToInt()}")
        RangeSlider(
            value = priceRange,
            onValueChange = { priceRange = it },
            valueRange = 0f..1000f,
            modifier = Modifier.fillMaxWidth()
        )

        // Slider พร้อม custom colors
        var customValue by remember { mutableStateOf(0.3f) }
        Text("Custom Slider: ${(customValue * 100).roundToInt()}")
        Slider(
            value = customValue,
            onValueChange = { customValue = it },
            colors = SliderDefaults.colors(
                thumbColor = MaterialTheme.colorScheme.secondary,
                activeTrackColor = MaterialTheme.colorScheme.secondary,
                inactiveTrackColor = MaterialTheme.colorScheme.secondaryContainer
            )
        )
    }
}
```

---

*Part 33 จบแล้ว | ก่อนหน้า: [Part 32](../part32/README.md) | ถัดไป: [Part 34](../part34/README.md)*
