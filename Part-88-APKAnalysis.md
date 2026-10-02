# Part 88: Java Decompilation & Android APK Analysis
## ขั้นตอนที่ 6061-6130: APK Decompile, Smali, JADX, Frida Basics

---

## 88.1 APK Structure

```
APK = Android Package (ZIP ไฟล์)

โครงสร้างภายใน APK:
  META-INF/          → signing certificates
  AndroidManifest.xml → binary XML (ต้อง decode)
  classes.dex        → Dalvik bytecode (compiled Java/Kotlin)
  classes2.dex       → multi-dex support
  lib/
    armeabi-v7a/     → native .so libraries (32-bit ARM)
    arm64-v8a/       → native .so libraries (64-bit ARM)
    x86/             → native .so libraries (emulator)
  res/               → resources (drawables, layouts binary)
  resources.arsc     → compiled resources (strings, dims)
  assets/            → raw files (fonts, data, etc.)

DEX = Dalvik Executable
  DEX bytecode → Dalvik VM / ART (Android Runtime)
  Java/Kotlin source → .class → D8/R8 → .dex

Decompile Process:
  .dex → baksmali → .smali (readable bytecode)
  .dex → dex2jar → .jar → decompiler → .java
  .dex → JADX → .java directly (best quality)
```

---

## 88.2 Tools Setup

```bash
# ====== Install Tools ======

# JADX - best decompiler (DEX → Java)
brew install jadx  # Mac
# OR download: https://github.com/skylot/jadx/releases

# Apktool - full APK disassembly (includes resources)
brew install apktool

# adb - Android Debug Bridge (included in Android SDK)
# PATH: ~/Library/Android/sdk/platform-tools/

# Frida - dynamic instrumentation
pip3 install frida-tools

# ====== Basic Operations ======

# Decompile APK with JADX (GUI)
jadx-gui app.apk

# Decompile APK with JADX (command line)
jadx -d output_dir app.apk
# Output: output_dir/sources/ (Java) + output_dir/resources/

# Decompile with Apktool (smali + resources)
apktool d app.apk -o output_dir
# Output: output_dir/smali/ + output_dir/res/ + output_dir/AndroidManifest.xml

# Repackage (after modifying smali)
apktool b output_dir -o modified.apk

# Sign APK (required to install)
keytool -genkey -v -keystore debug.keystore -alias androiddebugkey \
        -keyalg RSA -keysize 2048 -validity 10000
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 \
          -keystore debug.keystore modified.apk androiddebugkey
zipalign -v 4 modified.apk modified_aligned.apk

# Install on device/emulator
adb install modified_aligned.apk

# ====== APK Analysis Commands ======

# List APK contents
unzip -l app.apk

# Decode manifest to readable XML
apktool d app.apk --no-src -o temp_dir
cat temp_dir/AndroidManifest.xml

# Extract strings
strings app.apk | grep -E "(http|https|api|key|secret|password)"

# List DEX classes
dexdump app.apk | grep "^Class descriptor"

# ====== AAPT - Android Asset Packaging Tool ======
aapt dump badging app.apk  # package info, version, permissions
aapt dump permissions app.apk  # requested permissions
```

---

## 88.3 Reading Smali Code

```smali
# Smali = human-readable DEX bytecode
# Similar to assembly language for Dalvik VM

# ====== Basic Smali Syntax ======

# Class declaration
.class public Lcom/example/shop/MainActivity;
.super Landroidx/appcompat/app/AppCompatActivity;

# Field declaration
.field private productList:Ljava/util/List;
.field private static final TAG:Ljava/lang/String; = "MainActivity"

# Method declaration
.method public onCreate(Landroid/os/Bundle;)V
    .registers 3        # number of registers used (v0..vN)
    
    # Parameters: p0 = this, p1 = savedInstanceState
    
    # Call super method
    invoke-super {p0, p1}, Landroidx/appcompat/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V
    
    # Set content view: this.setContentView(R.layout.activity_main)
    const v0, 0x7f0b001c  # R.layout.activity_main
    invoke-virtual {p0, v0}, Lcom/example/shop/MainActivity;->setContentView(I)V
    
    # Load string: String msg = "Hello"
    const-string v0, "Hello World"
    
    # Static method call: Log.d(TAG, "onCreate")
    sget-object v1, Lcom/example/shop/MainActivity;->TAG:Ljava/lang/String;
    invoke-static {v1, v0}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
    
    return-void
.end method

# ====== Smali Types ======
# V = void
# Z = boolean
# B = byte
# S = short
# C = char
# I = int
# J = long (takes 2 registers)
# F = float
# D = double (takes 2 registers)
# Ljava/lang/String; = object type (L prefix, ; suffix)
# [I = int array
# [[Ljava/lang/Object; = 2D Object array

# ====== Common Instructions ======

# Move: v0 = v1
move v0, v1
move-object v0, v1      # for objects

# Constants
const/4 v0, 0x1         # small int constant
const/16 v0, 0x100      # 16-bit constant
const v0, 0x7f0b001c    # 32-bit constant
const-string v0, "text"

# Arithmetic
add-int v0, v1, v2      # v0 = v1 + v2
mul-int v0, v1, v2
div-int v0, v1, v2
rem-int v0, v1, v2      # modulo

# Comparisons
if-eq v0, v1, :label    # if v0 == v1 goto label
if-ne v0, v1, :label
if-gt v0, v1, :label
if-eqz v0, :label       # if v0 == 0 (null check)
if-nez v0, :label       # if v0 != 0

# Array operations
new-array v0, v1, [I    # v0 = new int[v1]
aput v2, v0, v3         # v0[v3] = v2
aget v2, v0, v3         # v2 = v0[v3]
array-length v0, v1     # v0 = v1.length

# Object operations
new-instance v0, Lcom/example/Foo;
iput-object v1, p0, Lcom/example/Foo;->field:Ljava/lang/String;
iget-object v0, p0, Lcom/example/Foo;->field:Ljava/lang/String;
```

---

## 88.4 JADX: Decompiling to Java

```java
// JADX output: app.apk → Java code

// Input (in APK, obfuscated):
// class a extends b { ... }

// JADX output (readable):
public class LoginActivity extends AppCompatActivity {
    
    private static final String TAG = "LoginActivity";
    private EditText emailInput;
    private EditText passwordInput;
    private Button loginButton;
    
    // JADX preserves structure but loses original names (if obfuscated)
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        
        this.emailInput = (EditText) findViewById(R.id.email_input);
        this.passwordInput = (EditText) findViewById(R.id.password_input);
        this.loginButton = (Button) findViewById(R.id.login_button);
        
        this.loginButton.setOnClickListener(view -> {
            String email = this.emailInput.getText().toString();
            String password = this.passwordInput.getText().toString();
            
            // Validation
            if (!validateEmail(email)) {
                Toast.makeText(this, "Invalid email", Toast.LENGTH_SHORT).show();
                return;
            }
            
            // This reveals API endpoint!
            login(email, password);
        });
    }
    
    private void login(String email, String password) {
        // API call visible in decompiled code
        String apiUrl = "https://api.example.com/auth/login";
        JSONObject body = new JSONObject();
        body.put("email", email);
        body.put("password", password);
        
        // Make HTTP request...
    }
    
    // JADX tip: Search for secrets/keys
    // Ctrl+F or Ctrl+Shift+F to search across all decompiled files
    // Common patterns: "API_KEY", "SECRET", "token", "password"
}
```

---

## 88.5 Modifying APK (Smali Patching)

```bash
# Example: Remove root check (educational purposes)

# Step 1: Decompile
apktool d app.apk -o app_decompiled

# Step 2: Find root check in smali
grep -r "isRooted\|isDeviceRooted\|RootBeer\|checkForRoot" app_decompiled/smali/

# Step 3: Examine the method
cat app_decompiled/smali/com/example/security/SecurityCheck.smali
```

```smali
# Original method (check returns true if rooted):
.method public static isRooted()Z
    .registers 1
    
    # ... complex root detection code ...
    
    # Returns true (1) if rooted
    const/4 v0, 0x1
    return v0
.end method

# Patched version (always returns false):
.method public static isRooted()Z
    .registers 1
    
    const/4 v0, 0x0      # 0 = false
    return v0
.end method
```

```bash
# Step 4: Rebuild
apktool b app_decompiled -o modified.apk

# Step 5: Sign
keytool -genkeypair -v -keystore test.keystore -alias test -keyalg RSA \
        -keysize 2048 -validity 10000 -storepass testpass -keypass testpass \
        -dname "CN=Test, OU=Test, O=Test, L=Test, ST=Test, C=US"

apksigner sign --ks test.keystore --ks-pass pass:testpass modified.apk

# Step 6: Install
adb install modified.apk
```

---

## 88.6 Static Analysis Checklist

```
APK Security Analysis Checklist:

1. Permissions
   adb dump permissions app.apk
   Look for: READ_CONTACTS, CAMERA, LOCATION, READ_CALL_LOG
   Are all permissions justified?

2. Hardcoded Secrets
   strings app.apk | grep -iE "api[_-]?key|secret|password|token"
   In JADX: search for BuildConfig fields
   Watch for: Firebase keys, Stripe keys, internal APIs

3. Network Configuration
   res/xml/network_security_config.xml
   Look for: cleartext traffic allowed, certificate pinning bypass

4. Exported Components
   AndroidManifest.xml: android:exported="true"
   Activities/Services with no permission → accessible to other apps

5. WebView Security
   setJavaScriptEnabled(true) + addJavascriptInterface = RCE risk
   setAllowFileAccessFromFileURLs = local file read

6. Data Storage
   SharedPreferences in MODE_WORLD_READABLE (deprecated but check)
   SQLite without encryption (sensitive data?)
   Logs: Log.d/v calls in production build

7. Obfuscation Check
   R8/ProGuard: class names like a, b, c → obfuscated
   No obfuscation → full class/method names visible
   Check build.gradle: minifyEnabled true

8. Third-party Libraries
   META-INF/ → look for third-party library manifests
   Known vulnerable versions?
```

---

## สรุป Part 88

```
APK Analysis Tools:

JADX (recommended):
  Best decompiler: DEX → readable Java/Kotlin
  GUI: jadx-gui app.apk
  Search: Ctrl+Shift+F across all files

Apktool:
  Full decompile: smali + resources
  Rebuild + modify capability
  apktool d app.apk -o dir
  apktool b dir -o new.apk

Smali Key Points:
  Register-based VM (unlike stack-based JVM)
  p0 = this (for non-static methods)
  p1, p2... = parameters
  v0, v1... = local variables
  Types: I=int, Z=bool, Ljava/lang/String;=String

Common Analysis Goals:
  Find API endpoints
  Find API keys / secrets
  Understand authentication logic
  Analyze encryption/obfuscation
  SSL pinning implementation
  
Ethical Use:
  ✓ Own apps: debugging, optimization
  ✓ Security research with permission
  ✓ CTF challenges
  ✓ Malware analysis
  ✗ Unauthorized access to others' code
  ✗ Bypassing licensing
  ✗ Extracting proprietary algorithms

Repackaging:
  Always sign before install
  Zipalign for optimization
  Test on emulator first
```

➡️ [Part 89: Advanced Android Security & Frida](./Part-89-AndroidSecurity.md)
