# Part 104: Java Security - Cryptography and Secure Coding

## เนื้อหาในส่วนนี้
- Java Cryptography Architecture (JCA/JCE)
- Symmetric and Asymmetric Encryption
- Hashing and Message Authentication Codes (MAC)
- Digital Signatures
- TLS/HTTPS Configuration
- Keystore and Truststore Management
- Secure Random Number Generation
- Spring Boot HTTPS setup
- Common Security Vulnerabilities

---

## 1. Symmetric Encryption (AES)

```java
import javax.crypto.*;
import javax.crypto.spec.*;
import java.security.*;
import java.util.Base64;

public class AesEncryption {
    
    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;   // 96 bits
    private static final int GCM_TAG_LENGTH = 128;  // 128 bits
    
    // Generate a random AES-256 key
    public static SecretKey generateKey() throws NoSuchAlgorithmException {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);  // 256-bit key
        return keyGen.generateKey();
    }
    
    // Encrypt using AES-GCM (authenticated encryption)
    public static byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
        // Random IV (never reuse IV with same key!)
        byte[] iv = new byte[GCM_IV_LENGTH];
        new SecureRandom().nextBytes(iv);
        
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec params = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, params);
        
        byte[] ciphertext = cipher.doFinal(plaintext);
        
        // Prepend IV to ciphertext (IV is not secret, just unique)
        byte[] result = new byte[GCM_IV_LENGTH + ciphertext.length];
        System.arraycopy(iv, 0, result, 0, GCM_IV_LENGTH);
        System.arraycopy(ciphertext, 0, result, GCM_IV_LENGTH, ciphertext.length);
        
        return result;
    }
    
    // Decrypt AES-GCM
    public static byte[] decrypt(byte[] combined, SecretKey key) throws Exception {
        // Extract IV and ciphertext
        byte[] iv = new byte[GCM_IV_LENGTH];
        byte[] ciphertext = new byte[combined.length - GCM_IV_LENGTH];
        System.arraycopy(combined, 0, iv, 0, GCM_IV_LENGTH);
        System.arraycopy(combined, GCM_IV_LENGTH, ciphertext, 0, ciphertext.length);
        
        Cipher cipher = Cipher.getInstance(ALGORITHM);
        GCMParameterSpec params = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.DECRYPT_MODE, key, params);
        
        return cipher.doFinal(ciphertext);  // Throws if tampered!
    }
    
    // Key from password using PBKDF2
    public static SecretKey deriveKeyFromPassword(char[] password, byte[] salt) 
            throws Exception {
        javax.crypto.SecretKeyFactory factory = 
            javax.crypto.SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256");
        KeySpec spec = new PBEKeySpec(password, salt, 310_000, 256);  // 310K iterations
        SecretKey tmp = factory.generateSecret(spec);
        return new SecretKeySpec(tmp.getEncoded(), "AES");
    }
    
    public static void main(String[] args) throws Exception {
        SecretKey key = generateKey();
        
        String message = "Sensitive data: credit card 4111-1111-1111-1111";
        byte[] encrypted = encrypt(message.getBytes(), key);
        byte[] decrypted = decrypt(encrypted, key);
        
        System.out.println("Original:  " + message);
        System.out.println("Encrypted: " + Base64.getEncoder().encodeToString(encrypted));
        System.out.println("Decrypted: " + new String(decrypted));
        
        // From password
        byte[] salt = new byte[16];
        new SecureRandom().nextBytes(salt);
        SecretKey derivedKey = deriveKeyFromPassword("myPassword123!".toCharArray(), salt);
        byte[] enc = encrypt(message.getBytes(), derivedKey);
        System.out.println("Password-derived key encryption works: " + 
            new String(decrypt(enc, derivedKey)).equals(message));
    }
}

import java.security.spec.KeySpec;
import java.security.spec.PBEKeySpec;
```

---

## 2. Asymmetric Encryption (RSA)

```java
import java.security.*;
import java.security.spec.*;
import javax.crypto.*;
import java.util.Base64;

public class RsaEncryption {
    
    // Generate RSA key pair (2048 or 4096 bits)
    public static KeyPair generateKeyPair() throws NoSuchAlgorithmException {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);  // 2048 minimum, prefer 4096 for high security
        return keyGen.generateKeyPair();
    }
    
    // Encrypt with RSA-OAEP (use public key)
    public static byte[] encrypt(byte[] plaintext, PublicKey publicKey) throws Exception {
        // Only encrypt small amounts of data (< key size - overhead)
        // For large data: encrypt symmetric key with RSA, data with AES
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(plaintext);
    }
    
    // Decrypt with private key
    public static byte[] decrypt(byte[] ciphertext, PrivateKey privateKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return cipher.doFinal(ciphertext);
    }
    
    // Hybrid encryption: RSA + AES (for large data)
    public static EncryptedData hybridEncrypt(byte[] data, PublicKey rsaPublicKey) throws Exception {
        // 1. Generate random AES key
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        SecretKey aesKey = keyGen.generateKey();
        
        // 2. Encrypt data with AES
        AesEncryption aes = new AesEncryption();
        byte[] encryptedData = AesEncryption.encrypt(data, aesKey);
        
        // 3. Encrypt AES key with RSA public key
        byte[] encryptedKey = encrypt(aesKey.getEncoded(), rsaPublicKey);
        
        return new EncryptedData(encryptedKey, encryptedData);
    }
    
    public static byte[] hybridDecrypt(EncryptedData ed, PrivateKey rsaPrivateKey) throws Exception {
        // 1. Decrypt AES key with RSA private key
        byte[] aesKeyBytes = decrypt(ed.encryptedKey(), rsaPrivateKey);
        SecretKey aesKey = new javax.crypto.spec.SecretKeySpec(aesKeyBytes, "AES");
        
        // 2. Decrypt data with AES
        return AesEncryption.decrypt(ed.encryptedData(), aesKey);
    }
    
    // Save/Load keys
    public static String publicKeyToBase64(PublicKey key) {
        return Base64.getEncoder().encodeToString(key.getEncoded());
    }
    
    public static PublicKey base64ToPublicKey(String base64) throws Exception {
        byte[] keyBytes = Base64.getDecoder().decode(base64);
        X509EncodedKeySpec spec = new X509EncodedKeySpec(keyBytes);
        return KeyFactory.getInstance("RSA").generatePublic(spec);
    }
    
    public static void main(String[] args) throws Exception {
        KeyPair keyPair = generateKeyPair();
        
        String message = "Secret message for RSA";
        byte[] encrypted = encrypt(message.getBytes(), keyPair.getPublic());
        byte[] decrypted = decrypt(encrypted, keyPair.getPrivate());
        
        System.out.println("Original: " + message);
        System.out.println("Decrypted: " + new String(decrypted));
        
        // Hybrid encryption for large data
        byte[] bigData = "Large data that won't fit in RSA block".repeat(100).getBytes();
        EncryptedData hybrid = hybridEncrypt(bigData, keyPair.getPublic());
        byte[] recoveredData = hybridDecrypt(hybrid, keyPair.getPrivate());
        System.out.println("Hybrid works: " + new String(recoveredData).equals(new String(bigData)));
    }
}

record EncryptedData(byte[] encryptedKey, byte[] encryptedData) {}

import java.security.spec.X509EncodedKeySpec;
```

---

## 3. Hashing and HMAC

```java
import java.security.*;
import javax.crypto.*;
import javax.crypto.spec.*;
import java.util.*;

public class HashingAndMac {
    
    // One-way hash (for integrity, not passwords!)
    public static String sha256(byte[] data) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(data);
        return HexFormat.of().formatHex(hash);
    }
    
    // SHA-256 with salt (still not for passwords - use BCrypt/Argon2/PBKDF2!)
    public static String sha256WithSalt(byte[] data, byte[] salt) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        digest.update(salt);
        byte[] hash = digest.digest(data);
        return HexFormat.of().formatHex(hash);
    }
    
    // HMAC-SHA256: message authentication (both integrity AND authenticity)
    public static byte[] hmacSha256(byte[] data, byte[] key) throws Exception {
        Mac mac = Mac.getInstance("HmacSHA256");
        mac.init(new SecretKeySpec(key, "HmacSHA256"));
        return mac.doFinal(data);
    }
    
    // Verify HMAC (timing-safe comparison)
    public static boolean verifyHmac(byte[] data, byte[] key, byte[] expectedMac) throws Exception {
        byte[] actualMac = hmacSha256(data, key);
        return MessageDigest.isEqual(actualMac, expectedMac);  // Constant-time comparison!
    }
    
    // File checksum
    public static String fileChecksum(java.io.InputStream input) throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] buffer = new byte[8192];
        int read;
        while ((read = input.read(buffer)) != -1) {
            digest.update(buffer, 0, read);
        }
        return HexFormat.of().formatHex(digest.digest());
    }
    
    // Password hashing - use BCrypt in Spring, Argon2, or PBKDF2
    // NEVER use MD5 or plain SHA for passwords!
    public static void passwordHashingExample() {
        // In Spring Security:
        // PasswordEncoder encoder = new BCryptPasswordEncoder(12);
        // String hash = encoder.encode("myPassword");
        // boolean matches = encoder.matches("myPassword", hash);
        
        // BCrypt handles salt internally, includes:
        // $2a$12$<22-char-salt><31-char-hash>
        System.out.println("Use BCrypt, Argon2, or PBKDF2 for passwords!");
    }
    
    public static void main(String[] args) throws Exception {
        String data = "Important message";
        
        // Hash
        String hash = sha256(data.getBytes());
        System.out.println("SHA-256: " + hash);
        
        // HMAC
        byte[] key = new byte[32];
        new SecureRandom().nextBytes(key);
        byte[] mac = hmacSha256(data.getBytes(), key);
        System.out.println("HMAC-SHA256: " + HexFormat.of().formatHex(mac));
        System.out.println("HMAC valid: " + verifyHmac(data.getBytes(), key, mac));
        
        // Tampered data should fail
        System.out.println("HMAC tampered: " + 
            verifyHmac("Tampered message".getBytes(), key, mac));
    }
}
```

---

## 4. Digital Signatures

```java
import java.security.*;
import java.security.spec.*;
import java.util.Base64;

public class DigitalSignature {
    
    // Sign data with private key (proves identity + integrity)
    public static byte[] sign(byte[] data, PrivateKey privateKey) throws Exception {
        Signature sig = Signature.getInstance("SHA256withRSA");
        sig.initSign(privateKey);
        sig.update(data);
        return sig.sign();
    }
    
    // Verify signature with public key
    public static boolean verify(byte[] data, byte[] signature, PublicKey publicKey) 
            throws Exception {
        Signature sig = Signature.getInstance("SHA256withRSA");
        sig.initVerify(publicKey);
        sig.update(data);
        return sig.verify(signature);
    }
    
    // Sign with ECDSA (faster than RSA, smaller signatures)
    public static KeyPair generateEcKeyPair() throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("EC");
        generator.initialize(new ECGenParameterSpec("secp256r1"));  // P-256 curve
        return generator.generateKeyPair();
    }
    
    public static byte[] ecSign(byte[] data, PrivateKey privateKey) throws Exception {
        Signature sig = Signature.getInstance("SHA256withECDSA");
        sig.initSign(privateKey);
        sig.update(data);
        return sig.sign();
    }
    
    public static boolean ecVerify(byte[] data, byte[] signature, PublicKey publicKey) 
            throws Exception {
        Signature sig = Signature.getInstance("SHA256withECDSA");
        sig.initVerify(publicKey);
        sig.update(data);
        return sig.verify(signature);
    }
    
    public static void main(String[] args) throws Exception {
        // RSA signatures
        KeyPairGenerator rsaGen = KeyPairGenerator.getInstance("RSA");
        rsaGen.initialize(2048);
        KeyPair rsaKeys = rsaGen.generateKeyPair();
        
        String document = "I hereby authorize payment of $10,000";
        byte[] signature = sign(document.getBytes(), rsaKeys.getPrivate());
        
        System.out.println("RSA Signature valid: " + 
            verify(document.getBytes(), signature, rsaKeys.getPublic()));
        System.out.println("RSA Signature tampered: " + 
            verify("I authorize $100".getBytes(), signature, rsaKeys.getPublic()));
        
        // ECDSA (smaller signature, faster)
        KeyPair ecKeys = generateEcKeyPair();
        byte[] ecSig = ecSign(document.getBytes(), ecKeys.getPrivate());
        System.out.println("ECDSA Signature valid: " + 
            ecVerify(document.getBytes(), ecSig, ecKeys.getPublic()));
        System.out.println("RSA sig size: " + signature.length + " bytes");
        System.out.println("ECDSA sig size: " + ecSig.length + " bytes");
    }
}

import java.security.spec.ECGenParameterSpec;
```

---

## 5. Spring Boot HTTPS Configuration

```yaml
# application.yml
server:
  port: 8443
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: myapp
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

```bash
# Generate self-signed certificate (development only)
keytool -genkeypair \
  -alias myapp \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore keystore.p12 \
  -storetype PKCS12 \
  -storepass changeit \
  -dname "CN=localhost, OU=Dev, O=MyOrg, L=Bangkok, S=Bangkok, C=TH"

# Import CA certificate to truststore
keytool -import -alias ca -keystore truststore.p12 -storetype PKCS12 \
  -storepass changeit -file ca.crt -noprompt

# Generate CSR for production (to sign by real CA)
keytool -certreq -alias myapp -keystore keystore.p12 -storetype PKCS12 \
  -storepass changeit -file myapp.csr
```

```java
// Redirect HTTP to HTTPS
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.apache.catalina.connector.Connector;

@Configuration
public class HttpsRedirectConfig {
    
    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> httpRedirectCustomizer() {
        return factory -> {
            Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
            connector.setScheme("http");
            connector.setPort(8080);
            connector.setSecure(false);
            connector.setRedirectPort(8443);  // Redirect to HTTPS
            factory.addAdditionalTomcatConnectors(connector);
        };
    }
    
    // Security headers
    @Bean
    public org.springframework.security.web.SecurityFilterChain securityHeaders(
            org.springframework.security.config.annotation.web.builders.HttpSecurity http) 
            throws Exception {
        http.headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)  // 1 year
                .preload(true))
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; " +
                    "script-src 'self' https://cdnjs.cloudflare.com; " +
                    "img-src 'self' data:; " +
                    "font-src 'self' https://fonts.gstatic.com"))
            .referrerPolicy(rp -> rp
                .policy(org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicy(pp -> pp
                .policy("camera=(), microphone=(), geolocation=(self)"))
        );
        return http.build();
    }
}
```

---

## 6. Secure Coding Practices

```java
import java.util.Arrays;

public class SecureCoding {
    
    // 1. Never log sensitive data
    public void badLogin(String username, String password) {
        System.out.println("Login: " + username + " password: " + password);  // NEVER!
    }
    
    public void goodLogin(String username, String password) {
        System.out.println("Login attempt: " + username);  // OK, no password
        // Mask in logs: "password=***"
    }
    
    // 2. Clear sensitive data from memory
    public boolean authenticateSecurely(char[] password) {
        try {
            boolean valid = checkPassword(password);
            return valid;
        } finally {
            Arrays.fill(password, '\0');  // Zero out password from memory!
        }
    }
    
    // 3. Prevent timing attacks - use constant-time comparison
    public boolean badTokenCompare(String expected, String actual) {
        return expected.equals(actual);  // Returns early on first mismatch!
    }
    
    public boolean goodTokenCompare(String expected, String actual) {
        return MessageDigest.isEqual(
            expected.getBytes(), actual.getBytes());  // Constant time!
    }
    
    // 4. Safe random numbers
    public String generateSecureToken(int bytes) {
        byte[] tokenBytes = new byte[bytes];
        new SecureRandom().nextBytes(tokenBytes);  // Cryptographically secure
        return Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);
    }
    
    // 5. Input validation for security
    public String sanitizeForLog(String input) {
        if (input == null) return "[null]";
        // Remove newlines to prevent log injection
        return input.replaceAll("[\r\n\t]", "_")
                   .substring(0, Math.min(input.length(), 200));  // Limit length
    }
    
    // 6. Avoid path traversal
    public java.io.File safeFileAccess(String userInput, String baseDir) {
        java.io.File base = new java.io.File(baseDir);
        java.io.File file = new java.io.File(base, userInput);
        
        try {
            if (!file.getCanonicalPath().startsWith(base.getCanonicalPath())) {
                throw new SecurityException("Path traversal detected: " + userInput);
            }
        } catch (java.io.IOException e) {
            throw new SecurityException("Invalid path", e);
        }
        return file;
    }
    
    // 7. XML External Entity (XXE) prevention
    public org.w3c.dom.Document safeXmlParse(java.io.InputStream xml) throws Exception {
        javax.xml.parsers.DocumentBuilderFactory factory = 
            javax.xml.parsers.DocumentBuilderFactory.newInstance();
        
        // Disable XXE
        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setXIncludeAware(false);
        factory.setExpandEntityReferences(false);
        
        return factory.newDocumentBuilder().parse(xml);
    }
    
    private boolean checkPassword(char[] password) { return true; }
}

import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;
```

---

## สรุป Part 104

| Topic | Best Practice |
|-------|--------------|
| Symmetric Encryption | AES-256-GCM (authenticated encryption) |
| Asymmetric Encryption | RSA-OAEP or ECDSA (P-256) |
| Password Hashing | BCrypt (cost≥12), Argon2, PBKDF2 |
| Key Derivation | PBKDF2WithHmacSHA256 (310K+ iterations) |
| Random Numbers | `SecureRandom` ONLY |
| Token Comparison | `MessageDigest.isEqual()` (constant-time) |
| TLS | TLS 1.3 preferred, TLS 1.2 minimum |
| Memory | Zero out char[] passwords after use |

---

**Part 105:** Final Project - Complete Production Application
