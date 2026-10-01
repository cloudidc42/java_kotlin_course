# Part 34: Advanced Smali & APK Patching
## ขั้นตอนที่ 2281-2350: Reverse Engineering ขั้นสูง

---

## 34.1 Advanced Smali Analysis

```smali
# ====== Reading Complex Methods ======

# Java source:
# public List<String> processUsers(List<User> users) {
#     List<String> results = new ArrayList<>();
#     for (User user : users) {
#         if (user.isActive() && user.getAge() >= 18) {
#             results.add(user.getName().toUpperCase());
#         }
#     }
#     return results;
# }

# Smali equivalent:
.method public processUsers(Ljava/util/List;)Ljava/util/List;
    .registers 8
    .annotation system Ldalvik/annotation/Signature;
        value = {
            "(", "Ljava/util/List<", "Lcom/example/User;", ">;)",
            "Ljava/util/List<", "Ljava/lang/String;", ">;"
        }
    .end annotation
    
    # List<String> results = new ArrayList<>();
    new-instance v0, Ljava/util/ArrayList;
    invoke-direct {v0}, Ljava/util/ArrayList;-><init>()V
    
    # Iterator for the for-each loop
    invoke-interface {p1}, Ljava/util/List;->iterator()Ljava/util/Iterator;
    move-result-object v1
    
    :loop_start
    # hasNext()
    invoke-interface {v1}, Ljava/util/Iterator;->hasNext()Z
    move-result v2
    if-eqz v2, :loop_end     # if !hasNext goto end
    
    # User user = iterator.next()
    invoke-interface {v1}, Ljava/util/Iterator;->next()Ljava/lang/Object;
    move-result-object v3
    check-cast v3, Lcom/example/User;
    
    # if (user.isActive())
    invoke-virtual {v3}, Lcom/example/User;->isActive()Z
    move-result v4
    if-eqz v4, :loop_start   # if !active skip
    
    # && user.getAge() >= 18
    invoke-virtual {v3}, Lcom/example/User;->getAge()I
    move-result v4
    const/16 v5, 0x12          # 0x12 = 18
    if-lt v4, v5, :loop_start  # if age < 18 skip
    
    # results.add(user.getName().toUpperCase())
    invoke-virtual {v3}, Lcom/example/User;->getName()Ljava/lang/String;
    move-result-object v4
    invoke-virtual {v4}, Ljava/lang/String;->toUpperCase()Ljava/lang/String;
    move-result-object v4
    invoke-interface {v0, v4}, Ljava/util/List;->add(Ljava/lang/Object;)Z
    
    goto :loop_start
    
    :loop_end
    return-object v0
.end method
```

---

## 34.2 Native Method (JNI) Identification

```smali
# JNI Methods in Smali
# Java: public native String computeHash(String input);
# Smali equivalent is just a declaration:

.method public native computeHash(Ljava/lang/String;)Ljava/lang/String;
.end method

# To find JNI implementation:
# 1. Extract .so from APK: unzip app.apk lib/arm64-v8a/*.so
# 2. Use nm or objdump to find exports
# nm -D libapp.so | grep Java_
# Java_com_example_MainActivity_computeHash

# 3. Disassemble with Ghidra or radare2

# ====== Hooking JNI with Frida ======
# (Frida is a dynamic instrumentation toolkit)
/*
Java.perform(function() {
    var MainActivity = Java.use("com.example.MainActivity");
    
    // Hook Java method
    MainActivity.checkLicense.implementation = function() {
        console.log("checkLicense() called, returning true");
        return true;
    };
    
    // Hook constructor
    MainActivity.$init.implementation = function() {
        console.log("MainActivity created");
        this.$init();  // call original
    };
    
    // Read/modify fields
    MainActivity.isDebugMode.value = true;
});
*/
```

---

## 34.3 SSL Pinning Bypass

```smali
# SSL Pinning: app verifies server certificate against hardcoded hash
# Bypass: patch the pinning check to always return true

# ====== Method 1: Modify checkServerTrusted ======
# Find in jadx:
# void checkServerTrusted(X509Certificate[] chain, String authType)
#   throws CertificateException {
#     if (!this.isValid(chain[0])) {
#         throw new CertificateException("Certificate not trusted");
#     }
# }

# In Smali, find this method and make it empty:
.method public checkServerTrusted([Ljava/security/cert/X509Certificate;Ljava/lang/String;)V
    .registers 3
    .annotation system Ldalvik/annotation/Throws;
        value = {
            Ljava/security/cert/CertificateException;
        }
    .end annotation
    
    # ORIGINAL code here would throw exception
    # PATCH: just return without checking
    return-void
.end method

# ====== Method 2: OkHttp CertificatePinner bypass ======
# Find the CertificatePinner.check() call:
# invoke-virtual {v0, v1, v2}, Lokhttp3/CertificatePinner;->check(Ljava/lang/String;Ljava/util/List;)V

# Patch by replacing with nop instructions:
# nop
# nop

# ====== Method 3: TrustManager bypass ======
# Find class implementing X509TrustManager
# Replace checkServerTrusted with empty method (as above)

# ====== Frida approach (no APK modification needed) ======
/*
// frida -U -l bypass-ssl.js com.example.app

Java.perform(function() {
    // OkHttp3 bypass
    try {
        var OkHttpClient = Java.use('okhttp3.OkHttpClient');
        var CertificatePinner = Java.use('okhttp3.CertificatePinner');
        CertificatePinner.check.overload('java.lang.String', 'java.util.List')
            .implementation = function() {
                console.log("SSL pinning bypassed");
            };
    } catch(e) {}
    
    // TrustManager bypass
    var TrustManager = Java.registerClass({
        name: 'com.example.bypass.CustomTrustManager',
        implements: [Java.use('javax.net.ssl.X509TrustManager')],
        methods: {
            checkClientTrusted: function() {},
            checkServerTrusted: function() {},
            getAcceptedIssuers: function() { return []; }
        }
    });
});
*/
```

---

## 34.4 Obfuscation Analysis

```java
// ====== ProGuard/R8 obfuscation ======
// Before obfuscation:
public class UserAuthenticationManager {
    private String secretApiKey;
    
    public boolean authenticateUser(String username, String password) {
        return validateCredentials(username, password);
    }
}

// After obfuscation (jadx output):
public class a {
    private String a;
    
    public boolean a(String b, String c) {
        return this.b(b, c);
    }
}

// ====== Deobfuscation techniques ======

// 1. String constants (not obfuscated):
//    const-string v0, "https://api.example.com/auth"
//    → strings often reveal purpose

// 2. Method signatures can hint at purpose:
//    a(Ljava/lang/String;Ljava/lang/String;)Z → (String, String) → boolean
//    likely: boolean authenticate(String, String)

// 3. Class relationships:
//    extends Landroid/os/AsyncTask → background operation
//    implements Ljava/lang/Runnable → thread
//    extends Landroidx/lifecycle/ViewModel → Android ViewModel

// 4. XML resources (not obfuscated):
//    strings.xml: "Welcome back, %s!"
//    layout XML: field IDs still meaningful

// 5. R.class mapping:
//    Resource IDs still map to meaningful names
//    const v0, 0x7f0c0045  → R.string.welcome_message

// Reconstruct from jadx:
// 1. Right click → "Rename" in jadx-gui
// 2. Build rename map in .cfg file
// 3. Use -printmapping flag in ProGuard to get mapping
```

---

## 34.5 Root Detection Bypass

```smali
# Apps detect root and refuse to run
# Common detection methods:

# 1. Check for su binary
# if (new File("/system/bin/su").exists()) { ... }

# In Smali:
# const-string v0, "/system/bin/su"
# new-instance v1, Ljava/io/File;
# invoke-direct {v1, v0}, Ljava/io/File;-><init>(Ljava/lang/String;)V
# invoke-virtual {v1}, Ljava/io/File;->exists()Z
# move-result v2
# if-nez v2, :rooted   ← patch this to: goto :not_rooted

# 2. Check for Superuser.apk
# PackageManager.getPackageInfo("com.noshufou.android.su", ...)

# 3. Build.TAGS check
# if ("test-keys".equals(Build.TAGS)) { ... }

# Patch approach: Find the isRooted() or checkRoot() method
# and make it always return false (0):

.method public isRooted()Z
    .registers 2
    
    # PATCH: always return false
    const/4 v0, 0x0
    return v0
    
    # Original code below is now unreachable...
.end method

# ====== Frida approach ======
/*
Java.perform(function() {
    // RootBeer library bypass
    var RootBeer = Java.use('com.scottyab.rootbeer.RootBeer');
    RootBeer.isRooted.implementation = function() { return false; };
    RootBeer.isRootedWithoutBusyBoxCheck.implementation = function() { return false; };
    
    // File.exists bypass for su
    var File = Java.use('java.io.File');
    File.exists.implementation = function() {
        var path = this.getAbsolutePath();
        if (path.includes('su') || path.includes('superuser')) {
            console.log('Hiding: ' + path);
            return false;
        }
        return this.exists();
    };
});
*/
```

---

## 34.6 Complete APK Modification Workflow

```bash
#!/bin/bash
# Full APK modification workflow

APK_IN="original.apk"
APK_OUT="patched.apk"
WORK_DIR="patching_work"

echo "=== APK Patching Workflow ==="

# Step 1: Decode
echo "[1/7] Decoding APK..."
apktool d "$APK_IN" -o "$WORK_DIR" -f
if [ $? -ne 0 ]; then echo "FAILED: apktool decode"; exit 1; fi

# Step 2: Find target smali files
echo "[2/7] Searching for target code..."
TARGET_FILES=$(grep -rl "checkLicense\|isRooted\|isPremium" \
                    "$WORK_DIR/smali" --include="*.smali")
echo "Found in:"
echo "$TARGET_FILES"

# Step 3: Make backup
echo "[3/7] Creating backup..."
cp -r "$WORK_DIR/smali" "$WORK_DIR/smali_backup"

# Step 4: Apply patches
echo "[4/7] Applying patches..."

# Patch 1: Bypass license check
# Find the method and replace return value with true (0x1)
for file in $TARGET_FILES; do
    # Look for checkLicense method returning boolean
    if grep -q "checkLicense\(\)Z" "$file"; then
        echo "Patching: $file"
        # Use python for complex patching
        python3 << 'PYTHON'
import re

with open("$file", 'r') as f:
    content = f.read()

# Replace: const/4 v0, 0x0; return v0  →  const/4 v0, 0x1; return v0
# in checkLicense method
content = re.sub(
    r'(\.method.*checkLicense.*?\n.*?)const/4 v(\d+), 0x0\n\s+return v\2',
    r'\g<1>const/4 v\2, 0x1\n    return v\2',
    content,
    flags=re.DOTALL
)

with open("$file", 'w') as f:
    f.write(content)
PYTHON
    fi
done

# Step 5: Rebuild
echo "[5/7] Rebuilding APK..."
apktool b "$WORK_DIR" -o "unsigned.apk"

# Step 6: Generate/use keystore
echo "[6/7] Signing..."
if [ ! -f debug.keystore ]; then
    keytool -genkeypair -v \
            -keystore debug.keystore \
            -alias androiddebugkey \
            -keyalg RSA -keysize 2048 \
            -validity 10000 \
            -storepass android -keypass android \
            -dname "CN=Android Debug,O=Android,C=US" 2>/dev/null
fi

jarsigner -verbose \
          -sigalg SHA256withRSA \
          -digestalg SHA-256 \
          -keystore debug.keystore \
          -storepass android \
          unsigned.apk androiddebugkey

# Step 7: Zipalign
echo "[7/7] Optimizing..."
zipalign -v 4 unsigned.apk "$APK_OUT"

echo ""
echo "=== Done ==="
echo "Original: $APK_IN"
echo "Patched:  $APK_OUT"
echo ""
echo "Install with: adb install -r $APK_OUT"

# Verify
echo "Verification:"
apksigner verify --verbose "$APK_OUT" 2>/dev/null | head -5
```

---

## 34.7 Frida Complete Setup

```bash
# ====== Frida Setup ======

# Install frida-tools on PC
pip install frida-tools

# Push frida-server to device
# Download from: https://github.com/frida/frida/releases
# Pick frida-server-<version>-android-arm64.xz

adb push frida-server-16.1.4-android-arm64 /data/local/tmp/frida-server
adb shell chmod +x /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &

# Verify
frida-ps -U | head -20          # list running processes
frida-ps -Ua                    # list installed apps

# ====== Frida Script Examples ======
cat > hook.js << 'EOF'
Java.perform(function() {
    
    // Log all method calls on a class
    var ClassName = Java.use("com.example.PaymentService");
    var methods = ClassName.class.getDeclaredMethods();
    methods.forEach(function(method) {
        var methodName = method.getName();
        try {
            ClassName[methodName].overloads.forEach(function(overload) {
                overload.implementation = function() {
                    var args = Array.prototype.join.call(arguments, ", ");
                    console.log("[*] " + methodName + "(" + args + ")");
                    var ret = this[methodName].apply(this, arguments);
                    console.log("[*] " + methodName + " returned: " + ret);
                    return ret;
                };
            });
        } catch(e) {}
    });
    
    // Hook String.equals to find hardcoded passwords/keys
    var String = Java.use("java.lang.String");
    String.equals.implementation = function(other) {
        var result = this.equals(other);
        if (this.toString().length > 8) {
            console.log("String.equals: " + this + " == " + other + " = " + result);
        }
        return result;
    };
});
EOF

# Run against running app
frida -U -l hook.js com.example.app

# Or spawn (restart app with frida)
frida -U -l hook.js --no-pause -f com.example.app
```

---

## สรุป Part 34

```
Advanced Reverse Engineering Tools:

Static Analysis:
  jadx       = Java source decompile
  apktool    = APK → Smali
  Ghidra     = Native (.so) analysis (free, NSA)
  IDA Pro    = Native analysis (paid, industry standard)
  
Dynamic Analysis:
  Frida      = Runtime instrumentation (hook any method)
  objection  = Frida-based automation (SSL bypass, root bypass)
  Xposed     = System-level hooking (requires root)
  
Network:
  Burp Suite = HTTPS interception/manipulation
  mitmproxy  = Open-source proxy
  
Common Bypasses:
  Root Detection  = patch isRooted() to return false
  SSL Pinning     = patch checkServerTrusted() to empty
  License Check   = patch return value to true
  Emulator Check  = patch Build.* checks

⚠️  Ethics Note:
  ใช้เพื่อ security research / CTF / แอปตัวเอง
  ไม่ใช้ละเมิดทรัพย์สินทางปัญญาของผู้อื่น
```

➡️ [Part 35: GraphQL API](./Part-35-GraphQL.md)
