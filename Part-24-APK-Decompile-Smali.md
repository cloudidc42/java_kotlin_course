# Part 24: APK Decompile & Smali
## ขั้นตอนที่ 1571-1650: Reverse Engineering Android APK

---

## 24.1 เครื่องมือ Decompile APK

```
เครื่องมือหลัก:

1. apktool
   - Disassemble APK → Smali code + resources
   - Reassemble modified APK
   - ติดตั้ง: brew install apktool / apt install apktool

2. jadx
   - Decompile DEX/APK → Java/Kotlin source
   - มี GUI (jadx-gui) และ CLI
   - Download: https://github.com/skylot/jadx/releases

3. dex2jar
   - แปลง .dex → .jar
   - ดู .jar ด้วย JD-GUI

4. apksigner / zipalign
   - Sign APK หลังแก้ไข

5. aapt / aapt2
   - Android Asset Packaging Tool
   - ดู resource ใน APK
```

---

## 24.2 การใช้ apktool

```bash
# ====== Installation ======
# macOS
brew install apktool

# Ubuntu/Debian
sudo apt install apktool

# Windows (ใช้ scoop)
scoop install apktool

# ====== Decompile APK ======
# Decode APK (ได้ smali + resources)
apktool d app.apk

# ระบุ output directory
apktool d app.apk -o decoded_app

# Force overwrite existing output
apktool d app.apk -f -o decoded_app

# ====== โครงสร้างที่ได้ ======
decoded_app/
├── AndroidManifest.xml    (decoded XML)
├── apktool.yml            (metadata)
├── res/                   (resources)
│   ├── layout/
│   │   └── activity_main.xml
│   ├── values/
│   │   ├── strings.xml
│   │   └── colors.xml
│   └── drawable/
│       └── icon.png
└── smali/                 (disassembled Dalvik bytecode)
    └── com/example/app/
        ├── MainActivity.smali
        └── BuildConfig.smali

# ====== Recompile ======
apktool b decoded_app -o rebuilt.apk

# ====== Sign APK ======
# Generate keystore (ทำครั้งแรก)
keytool -genkeypair -v -keystore debug.keystore -alias androiddebugkey \
        -keyalg RSA -keysize 2048 -validity 10000 \
        -storepass android -keypass android

# Sign
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
          -keystore debug.keystore rebuilt.apk androiddebugkey \
          -storepass android

# Align
zipalign -v 4 rebuilt.apk final.apk

# Verify
apksigner verify --print-certs final.apk

# ====== ดู APK info ======
aapt dump badging app.apk | head -20
aapt list app.apk | head -30
```

---

## 24.3 การใช้ jadx

```bash
# ====== Installation ======
# Download jadx-gui จาก https://github.com/skylot/jadx/releases
# หรือ
brew install jadx

# ====== Decompile to Java ======
jadx app.apk

# ระบุ output
jadx -d output_dir app.apk

# Export Gradle project
jadx -d project --export-gradle app.apk

# Decompile with debug info
jadx --show-bad-code app.apk

# ====== CLI search ======
# Search for string
jadx app.apk -d out && grep -r "api.example.com" out/sources/

# Decompile specific class
jadx app.apk --input-file classes.dex

# ====== jadx-gui tips ======
# เปิด jadx-gui
jadx-gui

# Hotkeys:
# Ctrl+N = Navigate to class
# Ctrl+F = Find in file
# Ctrl+Shift+F = Find in all files
# F5 = Refresh decompile
# Ctrl+Alt+F7 = Find usages
```

---

## 24.4 Smali Language

```smali
# ====== Smali = Dalvik Assembly Language ======
#
# Java source:
#   public class MainActivity extends AppCompatActivity {
#       private String message = "Hello";
#
#       @Override
#       protected void onCreate(Bundle savedInstanceState) {
#           super.onCreate(savedInstanceState);
#           setContentView(R.layout.activity_main);
#           String text = getMessage();
#           Log.d("TAG", text);
#       }
#
#       public String getMessage() {
#           return message;
#       }
#   }

# Smali equivalent:
.class public Lcom/example/app/MainActivity;
.super Landroidx/appcompat/app/AppCompatActivity;
.source "MainActivity.kt"

# Field declaration
.field private message:Ljava/lang/String;

# Constructor
.method public constructor <init>()V
    .registers 2

    invoke-direct {p0}, Landroidx/appcompat/app/AppCompatActivity;-><init>()V

    const-string v0, "Hello"
    iput-object v0, p0, Lcom/example/app/MainActivity;->message:Ljava/lang/String;

    return-void
.end method

# onCreate method
.method protected onCreate(Landroid/os/Bundle;)V
    .registers 4
    .param p1, "savedInstanceState"

    # super.onCreate(savedInstanceState)
    invoke-super {p0, p1}, Landroidx/appcompat/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V

    # setContentView(R.layout.activity_main)
    const v0, 0x7f0b001c          # resource ID
    invoke-virtual {p0, v0}, Lcom/example/app/MainActivity;->setContentView(I)V

    # String text = getMessage()
    invoke-virtual {p0}, Lcom/example/app/MainActivity;->getMessage()Ljava/lang/String;
    move-result-object v0

    # Log.d("TAG", text)
    const-string v1, "TAG"
    invoke-static {v1, v0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    return-void
.end method

# getMessage method
.method public getMessage()Ljava/lang/String;
    .registers 2

    iget-object v0, p0, Lcom/example/app/MainActivity;->message:Ljava/lang/String;
    return-object v0
.end method
```

---

## 24.5 Smali Registers & Opcodes

```smali
# ====== Register types ======
# p0 = this (first parameter in instance methods)
# p1, p2, ... = method parameters
# v0, v1, ... = local variables

# ====== Type descriptors ======
# V = void
# Z = boolean
# B = byte
# S = short
# C = char
# I = int
# J = long
# F = float
# D = double
# Ljava/lang/String; = Object type (full class path with L prefix and ; suffix)
# [I = int array
# [[Ljava/lang/String; = String 2D array

# ====== Common opcodes ======

# Move
const/4 v0, 0x1          # v0 = 1 (small int)
const/16 v0, 0x64        # v0 = 100
const v0, 0x12345678     # v0 = large int
const-string v0, "Hello" # v0 = "Hello"
const-class v0, Lcom/example/Foo; # v0 = Foo.class
move v0, v1              # v0 = v1
move-object v0, v1       # v0 = v1 (object)
move-result v0           # v0 = result of previous invoke
move-result-object v0    # v0 = object result of previous invoke

# Math
add-int v0, v1, v2       # v0 = v1 + v2
add-int/2addr v0, v1     # v0 += v1
sub-int v0, v1, v2       # v0 = v1 - v2
mul-int v0, v1, v2       # v0 = v1 * v2
div-int v0, v1, v2       # v0 = v1 / v2
rem-int v0, v1, v2       # v0 = v1 % v2
neg-int v0, v1           # v0 = -v1

# Comparison & branches
if-eq v0, v1, :label     # if v0 == v1 goto label
if-ne v0, v1, :label     # if v0 != v1 goto label
if-lt v0, v1, :label     # if v0 < v1 goto label
if-gt v0, v1, :label     # if v0 > v1 goto label
if-ge v0, v1, :label     # if v0 >= v1 goto label
if-le v0, v1, :label     # if v0 <= v1 goto label
if-eqz v0, :label        # if v0 == 0/null goto label
if-nez v0, :label        # if v0 != 0/null goto label
goto :label              # unconditional jump

# Invoke
invoke-virtual {v0, v1}, Lclass;->method(Param;)ReturnType; 
# instance method on object v0 with args v1

invoke-static {v0, v1}, Lclass;->method(Param;)ReturnType;
# static method

invoke-direct {v0}, Lclass;-><init>()V
# constructor or private method

invoke-super {p0, p1}, Lclass;->method(Param;)ReturnType;
# super method call

invoke-interface {v0, v1}, Linterface;->method(Param;)ReturnType;
# interface method

# Fields
iget-object v0, p0, Lclass;->field:LFieldType;  # v0 = this.field
iput-object v1, p0, Lclass;->field:LFieldType;  # this.field = v1
sget-object v0, Lclass;->field:LFieldType;       # v0 = Class.field (static)
sput-object v0, Lclass;->field:LFieldType;       # Class.field = v0

# Array
new-array v0, v1, [I       # v0 = new int[v1]
aget v0, v1, v2            # v0 = v1[v2]
aput v0, v1, v2            # v1[v2] = v0
array-length v0, v1        # v0 = v1.length

# Object
new-instance v0, Lclass;   # v0 = new class (before <init>!)
instance-of v0, v1, Lclass; # v0 = (v1 instanceof class)
check-cast v0, Lclass;     # (Class)v0

# Return
return v0                  # return int/float
return-object v0           # return object
return-void                # return void
```

---

## 24.6 Smali Modification Examples

```smali
# ====== Example 1: Bypass a boolean check ======

# Original smali (isPremium check):
.method public checkPremium()Z
    .registers 2
    
    # return false (not premium)
    const/4 v0, 0x0
    return v0
.end method

# Modified (always premium):
.method public checkPremium()Z
    .registers 2
    
    # return true (premium!)
    const/4 v0, 0x1
    return v0
.end method

# ====== Example 2: Extract hardcoded string ======

# Smali with hardcoded API key:
const-string v0, "AIzaSyB_secret_api_key_here"
invoke-static {v0}, Lcom/google/firebase/FirebaseApp;->initializeApp(Ljava/lang/String;)V

# Extract and study → find endpoint patterns

# ====== Example 3: Insert logging ======

# Original:
.method private processPayment(D)V
    .registers 4
    # ... payment logic ...
    return-void
.end method

# Modified with logging:
.method private processPayment(D)V
    .registers 6

    # Log the amount
    const-string v0, "PAYMENT_LOG"
    invoke-static {p1, p2}, Ljava/lang/String;->valueOf(D)Ljava/lang/String;
    move-result-object v1
    invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    # ... original payment logic ...
    return-void
.end method
```

---

## 24.7 Complete Workflow Script

```bash
#!/bin/bash
# APK Analysis Workflow Script

APK="$1"
OUTPUT="analysis_$(date +%Y%m%d_%H%M%S)"

if [ -z "$APK" ]; then
    echo "Usage: $0 <app.apk>"
    exit 1
fi

echo "=== APK Analysis Workflow ==="
echo "File: $APK"
echo "Output: $OUTPUT"
mkdir -p "$OUTPUT"

# Step 1: Basic info
echo ""
echo "--- Step 1: Basic APK Info ---"
aapt dump badging "$APK" 2>/dev/null | grep -E "^package|^application" | head -5
aapt list "$APK" | wc -l | xargs echo "Files in APK:"

# Step 2: Decompile with apktool
echo ""
echo "--- Step 2: apktool decode ---"
apktool d "$APK" -f -o "$OUTPUT/apktool" 2>/dev/null

echo "Smali files:"
find "$OUTPUT/apktool/smali" -name "*.smali" | wc -l

echo "Interesting strings in smali:"
grep -r "http\|https\|api\|key\|password\|secret\|token" \
     "$OUTPUT/apktool/smali" --include="*.smali" -l 2>/dev/null | head -10

# Step 3: Decompile with jadx
echo ""
echo "--- Step 3: jadx decompile ---"
jadx "$APK" -d "$OUTPUT/jadx" 2>/dev/null

echo "Java files:"
find "$OUTPUT/jadx/sources" -name "*.java" | wc -l

# Step 4: Search interesting patterns
echo ""
echo "--- Step 4: Pattern Search ---"
SEARCH_DIR="$OUTPUT/jadx/sources"

echo "API endpoints:"
grep -r "http\|https" "$SEARCH_DIR" --include="*.java" -h 2>/dev/null \
     | grep -oE '"https?://[^"]*"' | sort -u | head -10

echo ""
echo "Hardcoded credentials (potential):"
grep -r "password\|apikey\|api_key\|secret\|token\|auth" "$SEARCH_DIR" \
     --include="*.java" -i -h 2>/dev/null \
     | grep '= "' | grep -v "//" | head -10

echo ""
echo "SQLite tables:"
grep -r "CREATE TABLE\|createTable" "$SEARCH_DIR" --include="*.java" -h 2>/dev/null \
     | grep -oE '"[^"]*"' | head -10

# Step 5: Permissions
echo ""
echo "--- Step 5: Permissions ---"
if [ -f "$OUTPUT/apktool/AndroidManifest.xml" ]; then
    grep "uses-permission" "$OUTPUT/apktool/AndroidManifest.xml" \
         | grep -oE 'android\.permission\.[A-Z_]+' | sort
fi

echo ""
echo "=== Analysis complete: $OUTPUT ==="
```

---

## 24.8 Smali Cheat Sheet

```
ประเภท Register:
  p0, p1, p2... = parameters (p0 = this สำหรับ instance methods)
  v0, v1, v2... = local variables

ประเภท Type:
  I = int      J = long (wide)   F = float    D = double (wide)
  Z = boolean  B = byte          S = short    C = char   V = void
  L<classname>; = Object         [T = array of T

Invoke types:
  invoke-virtual    = instance method
  invoke-static     = static method
  invoke-direct     = private/constructor
  invoke-super      = superclass method
  invoke-interface  = interface method

เทคนิคการอ่าน Smali:
  1. หา .method และ .end method เพื่อดู scope
  2. p0 = this, p1 = first arg, ...
  3. move-result / move-result-object หลัง invoke = รับค่า return
  4. iget = อ่าน field, iput = เขียน field
  5. if-* = conditional jump, goto = unconditional jump
```

---

## สรุป Part 24

| เครื่องมือ | หน้าที่ |
|-----------|---------|
| apktool | Decode APK → Smali + Resources |
| jadx | Decompile DEX → Java/Kotlin source |
| dex2jar | แปลง DEX → JAR |
| jarsigner | เซ็นชื่อ APK |
| zipalign | Optimize APK |
| aapt | ดู APK metadata |

**คำเตือน**: การ decompile APK ควรทำเพื่อ:
- วิเคราะห์ malware / security research
- Debug แอปตัวเอง
- ทำความเข้าใจการทำงานของ Android
- CTF challenges

ไม่ควรใช้เพื่อ reverse engineer แอปของผู้อื่นเพื่อประโยชน์ส่วนตน

➡️ [Part 25: Advanced Java Topics](./Part-25-Advanced-Java.md)
