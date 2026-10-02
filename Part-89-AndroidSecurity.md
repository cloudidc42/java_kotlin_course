# Part 89: Android App Security Analysis
## ขั้นตอนที่ 6131-6200: Frida Basics, SSL Pinning, Certificate Analysis, OWASP Mobile

---

## 89.1 OWASP Mobile Top 10

```
OWASP Mobile Top 10 (2024):

M1: Improper Credential Usage
  Hard-coded credentials, insecure credential storage
  Fix: KeyStore, EncryptedSharedPreferences

M2: Inadequate Supply Chain Security  
  Third-party library vulnerabilities
  Fix: dependency audit, SBOM

M3: Insecure Authentication/Authorization
  Weak biometric, broken session management
  Fix: strong auth, proper session lifecycle

M4: Insufficient Input/Output Validation
  SQL injection in SQLite, XSS in WebView
  Fix: parameterized queries, WebView security settings

M5: Insecure Communication
  Clear text HTTP, improper TLS validation
  Fix: HTTPS only, certificate pinning

M6: Inadequate Privacy Controls
  Excessive permissions, data leakage in logs
  Fix: minimal permissions, log scrubbing

M7: Insufficient Binary Protections
  No obfuscation, debug enabled in production
  Fix: R8/ProGuard, debuggable=false

M8: Security Misconfiguration
  Exported components, backup allowed
  Fix: android:exported=false, android:allowBackup=false

M9: Insecure Data Storage
  Sensitive data in plaintext SharedPreferences
  Fix: EncryptedSharedPreferences, Room with SQLCipher

M10: Insufficient Cryptography
  MD5 for passwords, ECB mode AES
  Fix: bcrypt/Argon2 for passwords, AES-256-GCM
```

---

## 89.2 Frida Basics

```
Frida = dynamic instrumentation toolkit
สามารถ inject JavaScript code เข้าไป runtime ของ app

Prerequisites:
  - Root device / rooted emulator
  - OR: frida-gadget embedded in APK (non-rooted)

Install:
  pip3 install frida frida-tools

frida-server:
  # Download frida-server for device architecture
  # adb push frida-server-<ver>-android-arm64 /data/local/tmp/frida-server
  # adb shell chmod +x /data/local/tmp/frida-server
  # adb shell /data/local/tmp/frida-server &
```

```javascript
// ====== Frida Script Examples ======

// 1. Hook a method and log calls
Java.perform(function() {
    // Get class
    var MainActivity = Java.use('com.example.shop.MainActivity');
    
    // Hook method
    MainActivity.login.implementation = function(email, password) {
        // Log before
        console.log('[*] login() called');
        console.log('[*] email: ' + email);
        console.log('[*] password: ' + password);  // Can see plaintext!
        
        // Call original method
        var result = this.login(email, password);
        
        // Log after
        console.log('[*] login() returned: ' + result);
        
        return result;
    };
});

// 2. Hook multiple overloads
Java.perform(function() {
    var TextUtils = Java.use('android.text.TextUtils');
    
    // When method has multiple overloads, specify signature
    TextUtils.isEmpty.overload('java.lang.CharSequence').implementation = function(s) {
        var result = this.isEmpty(s);
        console.log('isEmpty("' + s + '") = ' + result);
        return result;
    };
});

// 3. Create Java objects from Frida
Java.perform(function() {
    // Create string
    var str = Java.use('java.lang.String').$new('Hello');
    
    // Call static method
    var Base64 = Java.use('android.util.Base64');
    var encoded = Base64.encodeToString(str.getBytes(), 0);
    console.log('Base64: ' + encoded);
    
    // Access static field
    var Build = Java.use('android.os.Build');
    console.log('MODEL: ' + Build.MODEL.value);
});

// 4. Find instances and call methods
Java.perform(function() {
    Java.choose('com.example.shop.UserSession', {
        onMatch: function(instance) {
            console.log('Found UserSession: ' + instance);
            console.log('Token: ' + instance.getToken());
        },
        onComplete: function() {
            console.log('Search complete');
        }
    });
});

// Run script:
// frida -U -n com.example.shop -l script.js
// frida -U -f com.example.shop -l script.js  (launch + hook)
```

---

## 89.3 SSL Pinning Analysis

```javascript
// SSL Pinning: App pins expected certificate/public key
// Prevents MITM even with trusted CA

// Common SSL Pinning libraries:
// - OkHttp CertificatePinner
// - TrustKit
// - Android network_security_config.xml

// ====== Finding SSL Pinning in JADX ======
// Search for: CertificatePinner, TrustManager, checkServerTrusted
// network_security_config.xml → <pin-set> element

// ====== Frida SSL Pinning Bypass ======
// Universal bypass (handles most common implementations)

Java.perform(function() {
    
    // Bypass OkHttp CertificatePinner
    var CertificatePinner = Java.use('okhttp3.CertificatePinner');
    CertificatePinner.check.overload('java.lang.String', 'java.util.List')
        .implementation = function(hostname, peerCertificates) {
            console.log('[*] Bypassing SSL pinning for: ' + hostname);
            // Don't call original = bypass
        };
    
    // Alternative overload
    CertificatePinner.check.overload('java.lang.String', '[Ljava.security.cert.Certificate;')
        .implementation = function(hostname, certs) {
            console.log('[*] Bypassing SSL pinning (v2) for: ' + hostname);
        };
    
    // Bypass TrustManagerImpl
    var TrustManagerImpl = Java.use('com.android.org.conscrypt.TrustManagerImpl');
    TrustManagerImpl.verifyChain.implementation = function(untrustedChain, trustAnchorChain, host, clientAuth, ocspData, tlsSctData) {
        return untrustedChain;
    };
    
    // Bypass WebView SSL errors
    var WebViewClient = Java.use('android.webkit.WebViewClient');
    WebViewClient.onReceivedSslError.implementation = function(view, handler, error) {
        handler.proceed();  // Accept all SSL errors
    };
});

// ====== Network Security Config Analysis ======
// res/xml/network_security_config.xml

/*
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2025-01-01">
            <!-- SHA-256 of public key -->
            <pin digest="SHA-256">base64encodedpublickeyhash</pin>
            <pin digest="SHA-256">backup_key_hash</pin>
        </pin-set>
    </domain-config>
</network-security-config>
*/

// Extract pinned certificates:
// 1. Find pins in network_security_config.xml
// 2. Decode base64 → SHA-256 of public key
// 3. Compare with server certificate
```

---

## 89.4 Secure Android Development

```kotlin
// ====== EncryptedSharedPreferences ======
import androidx.security.crypto.*

class SecureStorage(context: Context) {
    
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    private val prefs = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    
    fun saveToken(token: String) = prefs.edit().putString("auth_token", token).apply()
    fun getToken(): String? = prefs.getString("auth_token", null)
    fun clearToken() = prefs.edit().remove("auth_token").apply()
}

// ====== Android Keystore ======
// Hardware-backed key storage (cannot be extracted even with root on some devices)

class KeystoreEncryption {
    private val keyAlias = "MyAppKey"
    
    fun generateKey() {
        val keyGenSpec = KeyGenParameterSpec.Builder(
            keyAlias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .setUserAuthenticationRequired(false)
            .build()
        
        KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
            .apply { init(keyGenSpec) }
            .generateKey()
    }
    
    fun encrypt(plaintext: ByteArray): Pair<ByteArray, ByteArray> {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        val key = keyStore.getKey(keyAlias, null) as SecretKey
        
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, key)
        
        val iv = cipher.iv
        val ciphertext = cipher.doFinal(plaintext)
        return Pair(iv, ciphertext)
    }
    
    fun decrypt(iv: ByteArray, ciphertext: ByteArray): ByteArray {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        val key = keyStore.getKey(keyAlias, null) as SecretKey
        
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(128, iv))
        
        return cipher.doFinal(ciphertext)
    }
}

// ====== Biometric Authentication ======
class BiometricAuth(private val activity: AppCompatActivity) {
    
    fun authenticate(onSuccess: () -> Unit, onFailure: (String) -> Unit) {
        val executor = ContextCompat.getMainExecutor(activity)
        
        val biometricPrompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onSuccess()
                }
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onFailure(errString.toString())
                }
                override fun onAuthenticationFailed() {
                    onFailure("Authentication failed")
                }
            }
        )
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Authenticate")
            .setSubtitle("Use biometric to access your account")
            .setNegativeButtonText("Use PIN")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG
            )
            .build()
        
        biometricPrompt.authenticate(promptInfo)
    }
}

// ====== Secure WebView ======
class SecureWebView(context: Context) : WebView(context) {
    init {
        settings.apply {
            javaScriptEnabled = false              // disable if not needed
            allowFileAccess = false
            allowFileAccessFromFileURLs = false
            allowUniversalAccessFromFileURLs = false
            domStorageEnabled = false
            databaseEnabled = false
            setSupportZoom(false)
        }
    }
}
```

---

## 89.5 ProGuard/R8 Configuration

```proguard
# proguard-rules.pro

# ====== Basic ======
-optimizationpasses 5
-dontusemixedcaseclassnames
-dontskipnonpubliclibraryclasses
-verbose

# ====== Keep entry points ======
-keep class com.example.shop.MainActivity { *; }

# ====== Keep data classes (for serialization) ======
-keepclassmembers class com.example.shop.data.** {
    <fields>;
    <init>(...);
}

# ====== Retrofit ======
-keepattributes Signature
-keepattributes Exceptions
-keep class retrofit2.** { *; }
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

# ====== Kotlinx Serialization ======
-keep @kotlinx.serialization.Serializable class ** { *; }
-keepclassmembers class ** {
    kotlinx.serialization.KSerializer serializer(...);
}

# ====== Obfuscation ======
# By default R8 obfuscates everything not kept
# Class names: com.example.LoginActivity → a.b.c
# Method names: validateEmail() → a()
# This makes decompiled code much harder to read

# ====== String Encryption (R8 feature) ======
-encryptstrings  # encrypt string literals in bytecode

# ====== Anti-tamper (detect if APK was modified) ======
# Signature verification in code
```

```kotlin
// Signature verification (detect repackaging)
object AppIntegrityCheck {
    
    fun verifySignature(context: Context): Boolean {
        return try {
            val packageInfo = context.packageManager.getPackageInfo(
                context.packageName,
                PackageManager.GET_SIGNATURES
            )
            
            val signatures = packageInfo.signatures
            val messageDigest = MessageDigest.getInstance("SHA")
            messageDigest.update(signatures[0].toByteArray())
            
            val signatureHash = android.util.Base64.encodeToString(
                messageDigest.digest(), android.util.Base64.DEFAULT
            ).trim()
            
            // Compare with expected signature hash (store in native code for security)
            signatureHash == getExpectedSignatureHash()
        } catch (e: Exception) {
            false
        }
    }
    
    private fun getExpectedSignatureHash(): String {
        // Store this in native .so to make it harder to bypass
        return "your_release_signature_hash_here"
    }
}
```

---

## สรุป Part 89

```
Android Security Summary:

Static Analysis (before running):
  JADX: decompile DEX → Java
  Apktool: smali + resources
  Look for: hardcoded keys, API endpoints, auth logic

Dynamic Analysis (while running):
  Frida: hook methods at runtime
  Charles/mitmproxy: intercept HTTP traffic
  adb logcat: read app logs

SSL Pinning:
  App pins expected certificate/public key
  Intercept requires bypass
  Common bypass: Frida + okhttp3.CertificatePinner.check override

Secure Development:
  EncryptedSharedPreferences: AES-256-GCM encrypted storage
  Android Keystore: hardware-backed key storage
  Biometric: strong authentication
  ProGuard/R8: obfuscation + code optimization

OWASP Mobile Top 10 Defense:
  M1: KeyStore, EncryptedSharedPreferences
  M5: HTTPS + cert pinning + network_security_config
  M7: R8 minify + obfuscate
  M9: SQLCipher, EncryptedFile
  
App Protection Layers:
  1. Root detection
  2. Emulator detection  
  3. Debugger detection
  4. Tampering detection (signature check)
  5. SSL pinning
  
  None are unbreakable, but raise the bar
  Defense in depth: assume client can be compromised
  → Don't trust client, verify server-side
```

➡️ [Part 90: Production Hardening & Zero-Trust Architecture](./Part-90-ProductionHardening.md)
