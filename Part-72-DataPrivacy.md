# Part 72: Data Privacy & GDPR Compliance
## ขั้นตอนที่ 4941-5010: PII Handling, Encryption, Right to Erasure

---

## 72.1 GDPR & PDPA Fundamentals

```
GDPR (General Data Protection Regulation) = กฎหมาย EU
PDPA (Personal Data Protection Act) = กฎหมายไทย (คล้ายกัน)

PII (Personally Identifiable Information) = ข้อมูลส่วนบุคคล:
  - ชื่อ, อีเมล, เบอร์โทรศัพท์
  - เลขบัตรประชาชน, เลขบัตรเครดิต
  - IP address, Cookie, Device ID
  - ตำแหน่งที่อยู่ (location data)
  - Biometric data (ลายนิ้วมือ, ใบหน้า)

GDPR Rights:
  ✓ Right to Access: ขอดูข้อมูลของตัวเอง
  ✓ Right to Rectification: แก้ไขข้อมูล
  ✓ Right to Erasure: ขอลบข้อมูล ("Right to be Forgotten")
  ✓ Right to Portability: ขอ export ข้อมูล
  ✓ Right to Object: คัดค้านการประมวลผล

Developer Responsibilities:
  1. Privacy by Design: คิดถึง privacy ตั้งแต่ออกแบบ
  2. Data Minimization: เก็บเฉพาะที่จำเป็น
  3. Purpose Limitation: ใช้ข้อมูลตามที่แจ้ง
  4. Encryption: เข้ารหัสข้อมูล sensitive
  5. Audit Trail: บันทึกการเข้าถึงข้อมูล
```

---

## 72.2 PII Encryption at Rest

```java
import javax.crypto.*;
import javax.crypto.spec.*;
import java.util.Base64;

// Encrypt PII fields before storing in DB
@Service
class EncryptionService {
    
    private final SecretKey encryptionKey;
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 16;
    
    EncryptionService(@Value("${encryption.key}") String base64Key) {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        this.encryptionKey = new SecretKeySpec(keyBytes, "AES");
    }
    
    public String encrypt(String plaintext) {
        try {
            byte[] iv = new byte[GCM_IV_LENGTH];
            new java.security.SecureRandom().nextBytes(iv);
            
            var cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, encryptionKey, 
                new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv));
            
            byte[] encrypted = cipher.doFinal(
                plaintext.getBytes(java.nio.charset.StandardCharsets.UTF_8));
            
            // Prepend IV to ciphertext (IV is not secret)
            byte[] combined = new byte[iv.length + encrypted.length];
            System.arraycopy(iv, 0, combined, 0, iv.length);
            System.arraycopy(encrypted, 0, combined, iv.length, encrypted.length);
            
            return Base64.getEncoder().encodeToString(combined);
        } catch (Exception e) {
            throw new RuntimeException("Encryption failed", e);
        }
    }
    
    public String decrypt(String ciphertext) {
        try {
            byte[] combined = Base64.getDecoder().decode(ciphertext);
            
            // Extract IV and encrypted data
            byte[] iv = new byte[GCM_IV_LENGTH];
            byte[] encrypted = new byte[combined.length - GCM_IV_LENGTH];
            System.arraycopy(combined, 0, iv, 0, GCM_IV_LENGTH);
            System.arraycopy(combined, GCM_IV_LENGTH, encrypted, 0, encrypted.length);
            
            var cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, encryptionKey,
                new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv));
            
            return new String(cipher.doFinal(encrypted),
                java.nio.charset.StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }
}

// JPA converter for automatic encryption
@javax.persistence.Converter
class EncryptedStringConverter implements javax.persistence.AttributeConverter<String, String> {
    
    // Spring can't inject into @Converter easily, use static holder
    private static EncryptionService ENCRYPTION_SERVICE;
    
    @jakarta.annotation.PostConstruct
    static void setEncryptionService(EncryptionService service) {
        ENCRYPTION_SERVICE = service;
    }
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        return attribute == null ? null : ENCRYPTION_SERVICE.encrypt(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        return dbData == null ? null : ENCRYPTION_SERVICE.decrypt(dbData);
    }
}

// Entity with encrypted PII fields
@Entity
@Table(name = "users")
class User {
    
    @Id
    @GeneratedValue
    private java.util.UUID id;
    
    // Email stored encrypted in DB
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "email_encrypted")
    private String email;
    
    // Phone stored encrypted
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "phone_encrypted")
    private String phone;
    
    // Hash of email for lookup (can't search encrypted data directly)
    @Column(name = "email_hash", unique = true)
    private String emailHash;
    
    public void setEmail(String email) {
        this.email = email;
        this.emailHash = hashEmail(email);
    }
    
    private String hashEmail(String email) {
        try {
            var digest = java.security.MessageDigest.getInstance("SHA-256");
            return Base64.getEncoder().encodeToString(
                digest.digest(email.toLowerCase().getBytes()));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## 72.3 Right to Erasure Implementation

```java
@Service
@org.springframework.transaction.annotation.Transactional
class GdprService {
    
    private final UserRepository userRepository;
    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;
    
    GdprService(UserRepository userRepository,
                OrderRepository orderRepository,
                AuditLogRepository auditLogRepository) {
        this.userRepository = userRepository;
        this.orderRepository = orderRepository;
        this.auditLogRepository = auditLogRepository;
    }
    
    // Data Access Request: export all user data
    public UserDataExport exportUserData(java.util.UUID userId) {
        var user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        var orders = orderRepository.findByUserId(userId.toString());
        
        auditLogRepository.save(new AuditLog(
            userId, "DATA_EXPORT", "User requested data export",
            java.time.Instant.now()
        ));
        
        return new UserDataExport(
            userId,
            user.getEmail(),
            user.getName(),
            user.getPhone(),
            user.getCreatedAt(),
            orders.stream().map(o -> new OrderSummary(
                o.getId(), o.getTotal(), o.getStatus(), o.getCreatedAt()
            )).toList()
        );
    }
    
    // Right to Erasure: anonymize/delete user data
    public void eraseUserData(java.util.UUID userId, String requestReason) {
        var user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        // Anonymize PII (not delete, because orders may be needed for legal/financial reasons)
        user.setEmail("anonymized-" + userId + "@deleted.local");
        user.setPhone(null);
        user.setName("Deleted User");
        user.setDeletedAt(java.time.Instant.now());
        user.setStatus("ANONYMIZED");
        userRepository.save(user);
        
        // Delete marketing data (not legally required)
        deleteMarketingData(userId);
        
        // Audit trail (required by GDPR Article 5 - accountability)
        auditLogRepository.save(new AuditLog(
            userId, "DATA_ERASURE",
            "User data anonymized. Reason: " + requestReason,
            java.time.Instant.now()
        ));
        
        // Publish event for other services to anonymize their data
        // eventPublisher.publish(new UserDataErasureEvent(userId));
    }
    
    // Rectification: update PII
    public void rectifyUserData(java.util.UUID userId, UpdatePiiRequest request) {
        var user = userRepository.findById(userId)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        var changes = new java.util.ArrayList<String>();
        
        if (request.newEmail() != null && !request.newEmail().equals(user.getEmail())) {
            user.setEmail(request.newEmail());
            changes.add("email");
        }
        if (request.newPhone() != null) {
            user.setPhone(request.newPhone());
            changes.add("phone");
        }
        
        userRepository.save(user);
        
        auditLogRepository.save(new AuditLog(
            userId, "DATA_RECTIFICATION",
            "Fields updated: " + String.join(", ", changes),
            java.time.Instant.now()
        ));
    }
    
    private void deleteMarketingData(java.util.UUID userId) {
        // Delete email preferences, marketing consent, etc.
    }
    
    record UserDataExport(
        java.util.UUID userId,
        String email,
        String name,
        String phone,
        java.time.Instant memberSince,
        java.util.List<OrderSummary> orders
    ) {}
    
    record OrderSummary(java.util.UUID id, double total, String status, java.time.Instant date) {}
    record UpdatePiiRequest(String newEmail, String newPhone) {}
}

// GDPR REST API
@RestController
@RequestMapping("/api/privacy")
class PrivacyController {
    
    private final GdprService gdprService;
    
    PrivacyController(GdprService gdprService) {
        this.gdprService = gdprService;
    }
    
    @GetMapping("/my-data")
    GdprService.UserDataExport exportMyData(
            @AuthenticationPrincipal org.springframework.security.oauth2.jwt.Jwt jwt) {
        return gdprService.exportUserData(
            java.util.UUID.fromString(jwt.getSubject()));
    }
    
    @DeleteMapping("/my-data")
    @org.springframework.http.ResponseStatus(org.springframework.http.HttpStatus.NO_CONTENT)
    void deleteMyData(
            @AuthenticationPrincipal org.springframework.security.oauth2.jwt.Jwt jwt,
            @RequestBody DeleteRequest request) {
        gdprService.eraseUserData(
            java.util.UUID.fromString(jwt.getSubject()),
            request.reason()
        );
    }
    
    record DeleteRequest(String reason) {}
}
```

---

## 72.4 Data Masking & Logging

```java
// Mask PII in logs
@Aspect
@Component
class PiiMaskingAspect {
    
    private static final org.slf4j.Logger log = 
        org.slf4j.LoggerFactory.getLogger(PiiMaskingAspect.class);
    
    // Mask email in all service method parameters
    @Around("execution(* com.example.service..*(..))")
    public Object maskPiiInLogs(org.aspectj.lang.ProceedingJoinPoint pjp) throws Throwable {
        // Log method call without PII
        log.debug("Calling: {}", pjp.getSignature().getName());
        return pjp.proceed();
    }
    
    static String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) return "***@***";
        return email.charAt(0) + "***" + email.substring(atIndex);
    }
    
    static String maskPhone(String phone) {
        if (phone == null) return null;
        if (phone.length() < 4) return "****";
        return "*".repeat(phone.length() - 4) + phone.substring(phone.length() - 4);
    }
    
    static String maskCreditCard(String cardNumber) {
        if (cardNumber == null) return null;
        return "**** **** **** " + cardNumber.replaceAll("\\s", "")
            .substring(Math.max(0, cardNumber.length() - 4));
    }
}

// Test masking
class PiiMaskingTest {
    @Test void testEmailMask() {
        assertThat(PiiMaskingAspect.maskEmail("user@example.com")).isEqualTo("u***@example.com");
    }
    @Test void testPhoneMask() {
        assertThat(PiiMaskingAspect.maskPhone("0812345678")).isEqualTo("******5678");
    }
}
```

---

## สรุป Part 72

```
Data Privacy Checklist:

Data Collection:
  ✓ Only collect what you need (data minimization)
  ✓ Get explicit consent before collecting
  ✓ Tell users WHY you collect (privacy policy)

Storage:
  ✓ Encrypt PII at rest (AES-256-GCM)
  ✓ Hash for lookup (SHA-256 + salt)
  ✓ Store minimal data (don't store what you don't need)
  ✓ Retention policy: delete after N months/years
  ✓ DB encryption + TLS in transit

Access:
  ✓ RBAC: only authorized roles can access PII
  ✓ Audit log: who accessed PII and when
  ✓ Mask PII in logs (email, phone, card number)
  ✓ Row-level security in DB

GDPR Implementation:
  /privacy/my-data GET  = data export (Article 20)
  /privacy/my-data DELETE = data erasure (Article 17)
  /privacy/my-data PUT = rectification (Article 16)
  Response time: 30 days maximum

Code Practices:
  @Convert(converter = EncryptedStringConverter.class) on PII fields
  Never log: passwords, tokens, full card numbers, SSN
  Use emailHash for searching (not email_encrypted directly)
  
Incident Response:
  GDPR requires breach notification within 72 hours to authority
  And "without undue delay" to affected users
```

➡️ [Part 73: Cost Optimization & FinOps](./Part-73-CostOptimization.md)
