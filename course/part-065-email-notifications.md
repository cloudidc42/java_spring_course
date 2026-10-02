# Part 065: Email and Notification Services

## Overview

Modern applications need reliable communication channels — email confirmations, push alerts, SMS
OTPs, and live in-app notifications. This part builds a complete notification system from a
single-mailbox prototype to a production-grade multi-channel service with async retry, template
rendering, and user-preference gating.

---

## 1. Project Setup

### Maven dependencies

```xml
<dependencies>
    <!-- Spring Boot Starter Mail -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>

    <!-- Thymeleaf for HTML templates -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>org.thymeleaf.extras</groupId>
        <artifactId>thymeleaf-extras-springsecurity6</artifactId>
    </dependency>

    <!-- SendGrid -->
    <dependency>
        <groupId>com.sendgrid</groupId>
        <artifactId>sendgrid-java</artifactId>
        <version>4.10.1</version>
    </dependency>

    <!-- AWS SES SDK -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>ses</artifactId>
        <version>2.21.29</version>
    </dependency>

    <!-- Firebase Admin SDK (FCM push) -->
    <dependency>
        <groupId>com.google.firebase</groupId>
        <artifactId>firebase-admin</artifactId>
        <version>9.2.0</version>
    </dependency>

    <!-- Twilio (SMS) -->
    <dependency>
        <groupId>com.twilio.sdk</groupId>
        <artifactId>twilio</artifactId>
        <version>9.14.1</version>
    </dependency>

    <!-- Spring WebSocket -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>

    <!-- Spring Retry -->
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>

    <!-- Spring Batch (for bulk sends) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 5000
          writetimeout: 5000

notification:
  email:
    from: "noreply@myshop.com"
    from-name: "MyShop"
  sendgrid:
    api-key: ${SENDGRID_API_KEY}
  ses:
    region: us-east-1
    from: "noreply@myshop.com"
  firebase:
    credentials-file: classpath:firebase-service-account.json
  twilio:
    account-sid: ${TWILIO_ACCOUNT_SID}
    auth-token: ${TWILIO_AUTH_TOKEN}
    from-number: "+15005550006"
```

---

## 2. Domain Models

```java
package com.example.notification.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;
import java.util.Set;

@Entity
@Table(name = "notification_preferences")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NotificationPreference {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long userId;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
        name = "notification_channels",
        joinColumns = @JoinColumn(name = "preference_id")
    )
    @Enumerated(EnumType.STRING)
    @Column(name = "channel")
    private Set<NotificationChannel> enabledChannels;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(
        name = "notification_types_enabled",
        joinColumns = @JoinColumn(name = "preference_id")
    )
    @Enumerated(EnumType.STRING)
    @Column(name = "notification_type")
    private Set<NotificationType> enabledTypes;

    private String emailAddress;
    private String phoneNumber;
    private String fcmToken;
}
```

```java
package com.example.notification.domain;

public enum NotificationChannel {
    EMAIL, PUSH, SMS, IN_APP
}
```

```java
package com.example.notification.domain;

public enum NotificationType {
    ORDER_PLACED,
    ORDER_CONFIRMED,
    ORDER_SHIPPED,
    ORDER_DELIVERED,
    ORDER_CANCELLED,
    PAYMENT_SUCCESS,
    PAYMENT_FAILED,
    ACCOUNT_CREATED,
    PASSWORD_RESET,
    PROMOTION
}
```

```java
package com.example.notification.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "notification_log")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NotificationLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;

    @Enumerated(EnumType.STRING)
    private NotificationChannel channel;

    @Enumerated(EnumType.STRING)
    private NotificationType type;

    @Column(length = 2000)
    private String recipient;

    @Column(length = 500)
    private String subject;

    @Enumerated(EnumType.STRING)
    private NotificationStatus status;

    private int retryCount;

    @Column(length = 1000)
    private String errorMessage;

    private Instant createdAt;
    private Instant updatedAt;

    @PrePersist
    void onCreate() { this.createdAt = Instant.now(); this.updatedAt = Instant.now(); }

    @PreUpdate
    void onUpdate() { this.updatedAt = Instant.now(); }
}
```

```java
package com.example.notification.domain;

public enum NotificationStatus {
    PENDING, SENT, FAILED, SKIPPED
}
```

---

## 3. Email Service with JavaMailSender

```java
package com.example.notification.service.email;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.ClassPathResource;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;
import org.thymeleaf.context.Context;
import org.thymeleaf.spring6.SpringTemplateEngine;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;
    private final SpringTemplateEngine templateEngine;

    @Value("${notification.email.from}")
    private String fromAddress;

    @Value("${notification.email.from-name}")
    private String fromName;

    /**
     * Send a simple HTML email rendered from a Thymeleaf template.
     */
    public void sendHtmlEmail(String to, String subject,
                              String templateName, Map<String, Object> variables) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(
                    mimeMessage,
                    MimeMessageHelper.MULTIPART_MODE_MIXED_RELATED,
                    StandardCharsets.UTF_8.name()
            );

            Context context = new Context();
            context.setVariables(variables);
            String htmlContent = templateEngine.process(templateName, context);

            helper.setFrom(fromAddress, fromName);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);   // true = isHtml

            mailSender.send(mimeMessage);
            log.info("HTML email sent to={} subject={}", to, subject);

        } catch (Exception e) {
            log.error("Failed to send email to={}", to, e);
            throw new EmailSendException("Email send failed", e);
        }
    }

    /**
     * Send email with file attachment.
     */
    public void sendEmailWithAttachment(String to, String subject,
                                        String templateName, Map<String, Object> variables,
                                        File attachment, String attachmentName) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(
                    mimeMessage, true, StandardCharsets.UTF_8.name()
            );

            Context context = new Context();
            context.setVariables(variables);
            String htmlContent = templateEngine.process(templateName, context);

            helper.setFrom(fromAddress, fromName);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);
            helper.addAttachment(attachmentName, attachment);

            mailSender.send(mimeMessage);
            log.info("Email with attachment sent to={}", to);

        } catch (Exception e) {
            throw new EmailSendException("Email with attachment failed", e);
        }
    }

    /**
     * Send email with inline image (embedded in HTML body).
     */
    public void sendEmailWithInlineImage(String to, String subject,
                                         String templateName, Map<String, Object> variables,
                                         String imageCid, String imageClasspath) {
        try {
            MimeMessage mimeMessage = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(
                    mimeMessage, true, StandardCharsets.UTF_8.name()
            );

            Context context = new Context();
            context.setVariables(variables);
            // The template references <img src="cid:logoImage"> – match imageCid below
            String htmlContent = templateEngine.process(templateName, context);

            helper.setFrom(fromAddress, fromName);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);
            helper.addInline(imageCid, new ClassPathResource(imageClasspath));

            mailSender.send(mimeMessage);
            log.info("Email with inline image sent to={}", to);

        } catch (Exception e) {
            throw new EmailSendException("Email with inline image failed", e);
        }
    }
}
```

### Custom exception

```java
package com.example.notification.service.email;

public class EmailSendException extends RuntimeException {
    public EmailSendException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## 4. Thymeleaf Email Templates

### src/main/resources/templates/email/order-confirmation.html

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <title>Order Confirmation</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f5f5f5; margin: 0; padding: 0; }
        .container { max-width: 600px; margin: 40px auto; background: #fff;
                     border-radius: 8px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,.1); }
        .header { background: #1a73e8; color: #fff; padding: 24px 32px; }
        .header h1 { margin: 0; font-size: 24px; }
        .body { padding: 32px; }
        .order-table { width: 100%; border-collapse: collapse; margin: 16px 0; }
        .order-table th { background: #f0f4ff; text-align: left; padding: 10px; }
        .order-table td { padding: 10px; border-bottom: 1px solid #eee; }
        .total-row td { font-weight: bold; font-size: 16px; }
        .footer { background: #f0f4ff; padding: 16px 32px; font-size: 12px; color: #888; }
        .btn { display: inline-block; background: #1a73e8; color: #fff;
               padding: 12px 24px; border-radius: 4px; text-decoration: none; margin-top: 16px; }
    </style>
</head>
<body>
<div class="container">
    <div class="header">
        <!-- Inline logo: cid:logoImage must match addInline() call -->
        <img th:src="'cid:logoImage'" src="logo-placeholder.png"
             alt="MyShop" height="40" style="margin-bottom:8px;"/>
        <h1>Order Confirmed!</h1>
    </div>
    <div class="body">
        <p>Hi <strong th:text="${customerName}">Customer</strong>,</p>
        <p>Thank you for your order. Here's a summary:</p>

        <p><strong>Order #:</strong> <span th:text="${order.orderNumber}">ORD-001</span></p>
        <p><strong>Date:</strong>
            <span th:text="${#temporals.format(order.placedAt, 'dd MMM yyyy HH:mm')}">01 Jan 2025</span>
        </p>

        <table class="order-table">
            <thead>
            <tr>
                <th>Item</th>
                <th>Qty</th>
                <th>Unit Price</th>
                <th>Subtotal</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="item : ${order.items}">
                <td th:text="${item.productName}">Widget</td>
                <td th:text="${item.quantity}">1</td>
                <td th:text="${#numbers.formatCurrency(item.unitPrice)}">$10.00</td>
                <td th:text="${#numbers.formatCurrency(item.subtotal)}">$10.00</td>
            </tr>
            <tr class="total-row">
                <td colspan="3">Total</td>
                <td th:text="${#numbers.formatCurrency(order.total)}">$10.00</td>
            </tr>
            </tbody>
        </table>

        <p><strong>Shipping to:</strong> <span th:text="${order.shippingAddress}">123 Main St</span></p>

        <a class="btn" th:href="${trackingUrl}">Track Your Order</a>
    </div>
    <div class="footer">
        <p>You received this email because you placed an order on MyShop.
           &copy; 2025 MyShop Inc. All rights reserved.</p>
    </div>
</div>
</body>
</html>
```

### src/main/resources/templates/email/password-reset.html

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="UTF-8"/>
    <title>Password Reset</title>
    <style>
        body { font-family: Arial, sans-serif; background:#f5f5f5; }
        .container { max-width:600px; margin:40px auto; background:#fff;
                     border-radius:8px; padding:32px; }
        .btn { display:inline-block; background:#e53935; color:#fff;
               padding:12px 24px; border-radius:4px; text-decoration:none; }
    </style>
</head>
<body>
<div class="container">
    <h2>Password Reset Request</h2>
    <p>Hi <span th:text="${userName}">User</span>,</p>
    <p>We received a request to reset your password. Click the button below.
       The link is valid for <strong th:text="${expiryMinutes}">30</strong> minutes.</p>
    <p>
        <a class="btn" th:href="${resetUrl}">Reset Password</a>
    </p>
    <p>If you didn't request this, you can safely ignore this email.</p>
    <hr/>
    <p style="font-size:12px;color:#888;">
        Or copy this URL: <span th:text="${resetUrl}">https://...</span>
    </p>
</div>
</body>
</html>
```

---

## 5. SendGrid Integration

```java
package com.example.notification.service.email;

import com.sendgrid.*;
import com.sendgrid.helpers.mail.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.thymeleaf.context.Context;
import org.thymeleaf.spring6.SpringTemplateEngine;

import java.io.IOException;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class SendGridEmailService {

    private final SendGrid sendGrid;
    private final SpringTemplateEngine templateEngine;

    @Value("${notification.email.from}")
    private String fromEmail;

    @Value("${notification.email.from-name}")
    private String fromName;

    public void send(String toEmail, String toName, String subject,
                     String templateName, Map<String, Object> variables) {

        Email from = new Email(fromEmail, fromName);
        Email to   = new Email(toEmail, toName);

        Context ctx = new Context();
        ctx.setVariables(variables);
        String htmlContent = templateEngine.process(templateName, ctx);

        Content content = new Content("text/html", htmlContent);
        Mail mail = new Mail(from, subject, to, content);

        Request request = new Request();
        try {
            request.setMethod(Method.POST);
            request.setEndpoint("mail/send");
            request.setBody(mail.build());

            Response response = sendGrid.api(request);
            if (response.getStatusCode() >= 400) {
                throw new EmailSendException(
                    "SendGrid error " + response.getStatusCode() + ": " + response.getBody(), null);
            }
            log.info("SendGrid email sent to={} status={}", toEmail, response.getStatusCode());

        } catch (IOException e) {
            throw new EmailSendException("SendGrid IO error", e);
        }
    }
}
```

### SendGrid bean configuration

```java
package com.example.notification.config;

import com.sendgrid.SendGrid;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SendGridConfig {

    @Value("${notification.sendgrid.api-key}")
    private String apiKey;

    @Bean
    public SendGrid sendGrid() {
        return new SendGrid(apiKey);
    }
}
```

---

## 6. Amazon SES Integration

```java
package com.example.notification.service.email;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.thymeleaf.context.Context;
import org.thymeleaf.spring6.SpringTemplateEngine;
import software.amazon.awssdk.services.ses.SesClient;
import software.amazon.awssdk.services.ses.model.*;

import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class SesEmailService {

    private final SesClient sesClient;
    private final SpringTemplateEngine templateEngine;

    @Value("${notification.ses.from}")
    private String fromEmail;

    public void send(String to, String subject,
                     String templateName, Map<String, Object> variables) {

        Context ctx = new Context();
        ctx.setVariables(variables);
        String htmlBody = templateEngine.process(templateName, ctx);

        SendEmailRequest request = SendEmailRequest.builder()
            .source(fromEmail)
            .destination(Destination.builder().toAddresses(to).build())
            .message(
                Message.builder()
                    .subject(Content.builder().data(subject).charset("UTF-8").build())
                    .body(Body.builder()
                        .html(Content.builder().data(htmlBody).charset("UTF-8").build())
                        .build())
                    .build()
            )
            .build();

        SendEmailResponse response = sesClient.sendEmail(request);
        log.info("SES email sent to={} messageId={}", to, response.messageId());
    }
}
```

```java
package com.example.notification.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.auth.credentials.DefaultCredentialsProvider;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.ses.SesClient;

@Configuration
public class SesConfig {

    @Value("${notification.ses.region}")
    private String region;

    @Bean
    public SesClient sesClient() {
        return SesClient.builder()
            .region(Region.of(region))
            .credentialsProvider(DefaultCredentialsProvider.create())
            .build();
    }
}
```

---

## 7. Firebase Cloud Messaging (Push Notifications)

```java
package com.example.notification.config;

import com.google.auth.oauth2.GoogleCredentials;
import com.google.firebase.FirebaseApp;
import com.google.firebase.FirebaseOptions;
import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;

import java.io.IOException;

@Slf4j
@Configuration
public class FirebaseConfig {

    @Value("${notification.firebase.credentials-file}")
    private Resource credentialsResource;

    @PostConstruct
    public void initialize() throws IOException {
        if (FirebaseApp.getApps().isEmpty()) {
            FirebaseOptions options = FirebaseOptions.builder()
                .setCredentials(
                    GoogleCredentials.fromStream(credentialsResource.getInputStream())
                )
                .build();
            FirebaseApp.initializeApp(options);
            log.info("Firebase initialized");
        }
    }
}
```

```java
package com.example.notification.service.push;

import com.google.firebase.messaging.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Slf4j
@Service
public class FcmPushService {

    /**
     * Send push notification to a single device token.
     */
    public String sendToDevice(String fcmToken, String title, String body,
                               Map<String, String> data) {
        try {
            Message message = Message.builder()
                .setToken(fcmToken)
                .setNotification(Notification.builder()
                    .setTitle(title)
                    .setBody(body)
                    .build())
                .putAllData(data)
                .setAndroidConfig(AndroidConfig.builder()
                    .setPriority(AndroidConfig.Priority.HIGH)
                    .build())
                .setApnsConfig(ApnsConfig.builder()
                    .setAps(Aps.builder().setSound("default").build())
                    .build())
                .build();

            String messageId = FirebaseMessaging.getInstance().send(message);
            log.info("FCM sent messageId={} to token={}", messageId, fcmToken);
            return messageId;

        } catch (FirebaseMessagingException e) {
            log.error("FCM send failed for token={}", fcmToken, e);
            throw new PushSendException("FCM send failed", e);
        }
    }

    /**
     * Send to multiple tokens at once (up to 500 per call).
     */
    public BatchResponse sendToMultiple(List<String> tokens, String title,
                                        String body, Map<String, String> data) {
        try {
            MulticastMessage message = MulticastMessage.builder()
                .addAllTokens(tokens)
                .setNotification(Notification.builder()
                    .setTitle(title)
                    .setBody(body)
                    .build())
                .putAllData(data)
                .build();

            BatchResponse response = FirebaseMessaging.getInstance().sendEachForMulticast(message);
            log.info("FCM multicast: successCount={} failureCount={}",
                     response.getSuccessCount(), response.getFailureCount());
            return response;

        } catch (FirebaseMessagingException e) {
            throw new PushSendException("FCM multicast failed", e);
        }
    }

    /**
     * Send to a topic (e.g., "promotions" or "order-{orderId}").
     */
    public String sendToTopic(String topic, String title, String body,
                              Map<String, String> data) {
        try {
            Message message = Message.builder()
                .setTopic(topic)
                .setNotification(Notification.builder()
                    .setTitle(title)
                    .setBody(body)
                    .build())
                .putAllData(data)
                .build();

            String messageId = FirebaseMessaging.getInstance().send(message);
            log.info("FCM topic={} messageId={}", topic, messageId);
            return messageId;

        } catch (FirebaseMessagingException e) {
            throw new PushSendException("FCM topic send failed", e);
        }
    }

    /**
     * Subscribe tokens to a topic.
     */
    public void subscribeToTopic(List<String> tokens, String topic) {
        try {
            TopicManagementResponse response =
                FirebaseMessaging.getInstance().subscribeToTopic(tokens, topic);
            log.info("FCM topic subscribe successCount={}", response.getSuccessCount());
        } catch (FirebaseMessagingException e) {
            throw new PushSendException("FCM topic subscribe failed", e);
        }
    }
}
```

```java
package com.example.notification.service.push;

public class PushSendException extends RuntimeException {
    public PushSendException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## 8. SMS with Twilio

```java
package com.example.notification.config;

import com.twilio.Twilio;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TwilioConfig {

    @Value("${notification.twilio.account-sid}")
    private String accountSid;

    @Value("${notification.twilio.auth-token}")
    private String authToken;

    @PostConstruct
    public void init() {
        Twilio.init(accountSid, authToken);
    }
}
```

```java
package com.example.notification.service.sms;

import com.twilio.rest.api.v2010.account.Message;
import com.twilio.type.PhoneNumber;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class TwilioSmsService {

    @Value("${notification.twilio.from-number}")
    private String fromNumber;

    public String sendSms(String toNumber, String text) {
        Message message = Message.creator(
                new PhoneNumber(toNumber),
                new PhoneNumber(fromNumber),
                text
        ).create();

        log.info("SMS sent sid={} to={} status={}",
                 message.getSid(), toNumber, message.getStatus());
        return message.getSid();
    }

    public String sendOtp(String toNumber, String otp) {
        String text = "Your MyShop OTP is: " + otp + ". Valid for 10 minutes. Do not share.";
        return sendSms(toNumber, text);
    }
}
```

---

## 9. In-App Notifications with WebSocket (STOMP)

```java
package com.example.notification.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/notifications")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

```java
package com.example.notification.service.inapp;

import lombok.Data;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

import java.time.Instant;

@Slf4j
@Service
@RequiredArgsConstructor
public class InAppNotificationService {

    private final SimpMessagingTemplate messagingTemplate;

    /**
     * Push a notification to a specific user's personal queue.
     * Client subscribes to /user/queue/notifications
     */
    public void sendToUser(Long userId, InAppNotification notification) {
        messagingTemplate.convertAndSendToUser(
            userId.toString(),
            "/queue/notifications",
            notification
        );
        log.info("In-app notification sent to userId={} type={}", userId, notification.getType());
    }

    /**
     * Broadcast to all subscribers of a topic (e.g., global announcements).
     */
    public void broadcast(String topic, InAppNotification notification) {
        messagingTemplate.convertAndSend("/topic/" + topic, notification);
    }
}
```

```java
package com.example.notification.service.inapp;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.Instant;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class InAppNotification {
    private String type;
    private String title;
    private String message;
    private String actionUrl;
    private Instant timestamp;
    private boolean read;
}
```

### WebSocket controller

```java
package com.example.notification.controller;

import com.example.notification.service.inapp.InAppNotification;
import com.example.notification.service.inapp.InAppNotificationService;
import lombok.RequiredArgsConstructor;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;
import java.security.Principal;
import java.time.Instant;

@Controller
@RequiredArgsConstructor
public class NotificationWebSocketController {

    private final InAppNotificationService inAppService;

    /**
     * Client sends ACK when it reads a notification.
     */
    @MessageMapping("/notifications/ack")
    public void acknowledgeNotification(Long notificationId, Principal principal) {
        // Mark notification as read in DB
        // (not shown — uses a NotificationRepository)
    }
}
```

---

## 10. Notification Preferences Management

```java
package com.example.notification.repository;

import com.example.notification.domain.NotificationPreference;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface NotificationPreferenceRepository
        extends JpaRepository<NotificationPreference, Long> {
    Optional<NotificationPreference> findByUserId(Long userId);
}
```

```java
package com.example.notification.service;

import com.example.notification.domain.*;
import com.example.notification.repository.NotificationPreferenceRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Set;

@Service
@RequiredArgsConstructor
public class NotificationPreferenceService {

    private final NotificationPreferenceRepository preferenceRepository;

    @Transactional(readOnly = true)
    public NotificationPreference getPreference(Long userId) {
        return preferenceRepository.findByUserId(userId)
            .orElseGet(() -> createDefaultPreference(userId));
    }

    @Transactional
    public NotificationPreference updatePreference(Long userId,
                                                   Set<NotificationChannel> channels,
                                                   Set<NotificationType> types) {
        NotificationPreference pref = preferenceRepository.findByUserId(userId)
            .orElseGet(() -> NotificationPreference.builder().userId(userId).build());

        pref.setEnabledChannels(channels);
        pref.setEnabledTypes(types);
        return preferenceRepository.save(pref);
    }

    @Transactional
    public void updateFcmToken(Long userId, String token) {
        NotificationPreference pref = getOrCreate(userId);
        pref.setFcmToken(token);
        preferenceRepository.save(pref);
    }

    public boolean canSend(Long userId, NotificationChannel channel, NotificationType type) {
        NotificationPreference pref = preferenceRepository.findByUserId(userId).orElse(null);
        if (pref == null) return true; // default: allow
        return pref.getEnabledChannels().contains(channel)
            && pref.getEnabledTypes().contains(type);
    }

    private NotificationPreference createDefaultPreference(Long userId) {
        return NotificationPreference.builder()
            .userId(userId)
            .enabledChannels(Set.of(NotificationChannel.values()))
            .enabledTypes(Set.of(NotificationType.values()))
            .build();
    }

    private NotificationPreference getOrCreate(Long userId) {
        return preferenceRepository.findByUserId(userId)
            .orElseGet(() -> NotificationPreference.builder().userId(userId).build());
    }
}
```

### REST API for preferences

```java
package com.example.notification.controller;

import com.example.notification.domain.*;
import com.example.notification.service.NotificationPreferenceService;
import lombok.*;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.Set;

@RestController
@RequestMapping("/api/v1/notification-preferences")
@RequiredArgsConstructor
public class NotificationPreferenceController {

    private final NotificationPreferenceService preferenceService;

    @GetMapping
    public ResponseEntity<NotificationPreference> get(@AuthenticationPrincipal Long userId) {
        return ResponseEntity.ok(preferenceService.getPreference(userId));
    }

    @PutMapping
    public ResponseEntity<NotificationPreference> update(
            @AuthenticationPrincipal Long userId,
            @RequestBody PreferenceRequest body) {
        return ResponseEntity.ok(
            preferenceService.updatePreference(userId, body.getChannels(), body.getTypes())
        );
    }

    @PutMapping("/fcm-token")
    public ResponseEntity<Void> updateFcmToken(
            @AuthenticationPrincipal Long userId,
            @RequestBody FcmTokenRequest body) {
        preferenceService.updateFcmToken(userId, body.getToken());
        return ResponseEntity.noContent().build();
    }

    @Data
    static class PreferenceRequest {
        private Set<NotificationChannel> channels;
        private Set<NotificationType>   types;
    }

    @Data
    static class FcmTokenRequest {
        private String token;
    }
}
```

---

## 11. Async Notification Sending with Retry

```java
package com.example.notification.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.retry.annotation.EnableRetry;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
@EnableRetry
public class AsyncConfig {

    @Bean(name = "notificationExecutor")
    public Executor notificationExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("notification-");
        executor.setRejectedExecutionHandler((r, e) ->
            // Log and discard if queue full; in production: push to dead-letter queue
            System.err.println("Notification task rejected: queue full")
        );
        executor.initialize();
        return executor;
    }
}
```

```java
package com.example.notification.service;

import com.example.notification.domain.*;
import com.example.notification.service.email.EmailService;
import com.example.notification.service.push.FcmPushService;
import com.example.notification.service.sms.TwilioSmsService;
import com.example.notification.service.inapp.InAppNotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class AsyncNotificationService {

    private final EmailService emailService;
    private final FcmPushService fcmPushService;
    private final TwilioSmsService smsService;
    private final InAppNotificationService inAppService;
    private final NotificationPreferenceService preferenceService;
    private final NotificationLogService logService;

    @Async("notificationExecutor")
    @Retryable(
        retryFor = { Exception.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 2000, multiplier = 2.0, maxDelay = 30_000)
    )
    public void sendEmail(Long userId, String toEmail, String subject,
                          String template, Map<String, Object> vars) {

        if (!preferenceService.canSend(userId, NotificationChannel.EMAIL,
                NotificationType.ORDER_CONFIRMED)) {
            logService.log(userId, NotificationChannel.EMAIL,
                           NotificationStatus.SKIPPED, toEmail, subject, null);
            return;
        }

        emailService.sendHtmlEmail(toEmail, subject, template, vars);
        logService.log(userId, NotificationChannel.EMAIL,
                       NotificationStatus.SENT, toEmail, subject, null);
    }

    @Recover
    public void recoverEmail(Exception e, Long userId, String toEmail,
                             String subject, String template, Map<String, Object> vars) {
        log.error("Email delivery failed permanently for userId={} email={}", userId, toEmail, e);
        logService.log(userId, NotificationChannel.EMAIL,
                       NotificationStatus.FAILED, toEmail, subject, e.getMessage());
    }

    @Async("notificationExecutor")
    @Retryable(retryFor = Exception.class, maxAttempts = 3,
               backoff = @Backoff(delay = 1000, multiplier = 2.0))
    public void sendPush(Long userId, String fcmToken, String title,
                         String body, Map<String, String> data) {
        if (!preferenceService.canSend(userId, NotificationChannel.PUSH,
                NotificationType.ORDER_CONFIRMED)) return;

        fcmPushService.sendToDevice(fcmToken, title, body, data);
        logService.log(userId, NotificationChannel.PUSH, NotificationStatus.SENT,
                       fcmToken, title, null);
    }

    @Recover
    public void recoverPush(Exception e, Long userId, String fcmToken,
                            String title, String body, Map<String, String> data) {
        log.error("Push delivery failed permanently for userId={}", userId, e);
        logService.log(userId, NotificationChannel.PUSH, NotificationStatus.FAILED,
                       fcmToken, title, e.getMessage());
    }

    @Async("notificationExecutor")
    @Retryable(retryFor = Exception.class, maxAttempts = 2,
               backoff = @Backoff(delay = 3000))
    public void sendSms(Long userId, String phone, String text) {
        if (!preferenceService.canSend(userId, NotificationChannel.SMS,
                NotificationType.ORDER_CONFIRMED)) return;

        smsService.sendSms(phone, text);
        logService.log(userId, NotificationChannel.SMS, NotificationStatus.SENT,
                       phone, text.substring(0, Math.min(50, text.length())), null);
    }
}
```

---

## 12. Notification Log Service

```java
package com.example.notification.service;

import com.example.notification.domain.*;
import com.example.notification.repository.NotificationLogRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class NotificationLogService {

    private final NotificationLogRepository logRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(Long userId, NotificationChannel channel, NotificationStatus status,
                    String recipient, String subject, String errorMessage) {
        NotificationLog entry = NotificationLog.builder()
            .userId(userId)
            .channel(channel)
            .status(status)
            .recipient(recipient)
            .subject(subject)
            .errorMessage(errorMessage)
            .build();
        logRepository.save(entry);
    }
}
```

---

## 13. Real Example — Order Lifecycle Notification System

### Order event record

```java
package com.example.notification.event;

import com.example.notification.domain.NotificationType;
import lombok.Builder;
import lombok.Data;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Data
@Builder
public class OrderEvent {
    private NotificationType type;
    private Long orderId;
    private String orderNumber;
    private Long customerId;
    private String customerEmail;
    private String customerPhone;
    private String customerName;
    private String customerFcmToken;
    private List<OrderItemDto> items;
    private BigDecimal total;
    private String shippingAddress;
    private String trackingNumber;
    private Instant occurredAt;
}
```

```java
package com.example.notification.event;

import lombok.Data;
import java.math.BigDecimal;

@Data
public class OrderItemDto {
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
    private BigDecimal subtotal;
}
```

### Order notification orchestrator

```java
package com.example.notification.service;

import com.example.notification.domain.NotificationType;
import com.example.notification.event.OrderEvent;
import com.example.notification.service.inapp.InAppNotification;
import com.example.notification.service.inapp.InAppNotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderNotificationService {

    private final AsyncNotificationService asyncNotification;
    private final InAppNotificationService inAppService;

    @EventListener
    public void handleOrderEvent(OrderEvent event) {
        log.info("Handling order notification type={} orderId={}",
                 event.getType(), event.getOrderId());

        switch (event.getType()) {
            case ORDER_PLACED       -> onOrderPlaced(event);
            case ORDER_CONFIRMED    -> onOrderConfirmed(event);
            case ORDER_SHIPPED      -> onOrderShipped(event);
            case ORDER_DELIVERED    -> onOrderDelivered(event);
            case ORDER_CANCELLED    -> onOrderCancelled(event);
            default -> log.warn("Unhandled order event type={}", event.getType());
        }
    }

    private void onOrderPlaced(OrderEvent e) {
        Map<String, Object> vars = buildOrderVars(e);
        vars.put("trackingUrl", "https://myshop.com/orders/" + e.getOrderId());

        asyncNotification.sendEmail(
            e.getCustomerId(), e.getCustomerEmail(),
            "Order Received - #" + e.getOrderNumber(),
            "email/order-confirmation", vars
        );

        sendInApp(e.getCustomerId(), "Order Placed",
            "Your order #" + e.getOrderNumber() + " has been placed!",
            "/orders/" + e.getOrderId());
    }

    private void onOrderConfirmed(OrderEvent e) {
        asyncNotification.sendPush(
            e.getCustomerId(), e.getCustomerFcmToken(),
            "Order Confirmed!",
            "Order #" + e.getOrderNumber() + " is confirmed and being prepared.",
            Map.of("orderId", String.valueOf(e.getOrderId()),
                   "type", "ORDER_CONFIRMED")
        );
        sendInApp(e.getCustomerId(), "Order Confirmed",
            "Your order #" + e.getOrderNumber() + " is confirmed.",
            "/orders/" + e.getOrderId());
    }

    private void onOrderShipped(OrderEvent e) {
        // Email with tracking info
        Map<String, Object> vars = buildOrderVars(e);
        vars.put("trackingNumber", e.getTrackingNumber());
        vars.put("trackingUrl", "https://myshop.com/orders/" + e.getOrderId() + "/track");

        asyncNotification.sendEmail(
            e.getCustomerId(), e.getCustomerEmail(),
            "Your order is on the way! - #" + e.getOrderNumber(),
            "email/order-shipped", vars
        );

        // SMS with tracking number
        asyncNotification.sendSms(
            e.getCustomerId(), e.getCustomerPhone(),
            "MyShop: Order #" + e.getOrderNumber() + " shipped! Tracking: "
            + e.getTrackingNumber() + " Track: https://myshop.com/orders/"
            + e.getOrderId() + "/track"
        );

        sendInApp(e.getCustomerId(), "Order Shipped!",
            "Order #" + e.getOrderNumber() + " is on its way.",
            "/orders/" + e.getOrderId() + "/track");
    }

    private void onOrderDelivered(OrderEvent e) {
        asyncNotification.sendPush(
            e.getCustomerId(), e.getCustomerFcmToken(),
            "Order Delivered!",
            "Order #" + e.getOrderNumber() + " has been delivered. Enjoy!",
            Map.of("orderId", String.valueOf(e.getOrderId()), "type", "ORDER_DELIVERED")
        );
        sendInApp(e.getCustomerId(), "Delivered!",
            "Order #" + e.getOrderNumber() + " was delivered.",
            "/orders/" + e.getOrderId() + "/review");
    }

    private void onOrderCancelled(OrderEvent e) {
        Map<String, Object> vars = buildOrderVars(e);

        asyncNotification.sendEmail(
            e.getCustomerId(), e.getCustomerEmail(),
            "Order Cancelled - #" + e.getOrderNumber(),
            "email/order-cancelled", vars
        );

        sendInApp(e.getCustomerId(), "Order Cancelled",
            "Order #" + e.getOrderNumber() + " has been cancelled.",
            "/orders/" + e.getOrderId());
    }

    private Map<String, Object> buildOrderVars(OrderEvent e) {
        return new java.util.HashMap<>(Map.of(
            "customerName",    e.getCustomerName(),
            "order",           e,
            "shippingAddress", e.getShippingAddress()
        ));
    }

    private void sendInApp(Long userId, String title, String message, String actionUrl) {
        inAppService.sendToUser(userId, InAppNotification.builder()
            .type("ORDER")
            .title(title)
            .message(message)
            .actionUrl(actionUrl)
            .timestamp(Instant.now())
            .read(false)
            .build());
    }
}
```

### Publishing order events from the order service

```java
package com.example.order.service;

import com.example.notification.domain.NotificationType;
import com.example.notification.event.OrderEvent;
import com.example.order.domain.Order;
import lombok.RequiredArgsConstructor;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    public void confirmOrder(Order order) {
        // ... business logic ...

        eventPublisher.publishEvent(OrderEvent.builder()
            .type(NotificationType.ORDER_CONFIRMED)
            .orderId(order.getId())
            .orderNumber(order.getOrderNumber())
            .customerId(order.getCustomerId())
            .customerEmail(order.getCustomerEmail())
            .customerPhone(order.getCustomerPhone())
            .customerName(order.getCustomerName())
            .customerFcmToken(order.getCustomerFcmToken())
            .items(order.getItems().stream()
                .map(i -> {
                    var dto = new com.example.notification.event.OrderItemDto();
                    dto.setProductName(i.getProductName());
                    dto.setQuantity(i.getQuantity());
                    dto.setUnitPrice(i.getUnitPrice());
                    dto.setSubtotal(i.getSubtotal());
                    return dto;
                }).toList())
            .total(order.getTotal())
            .shippingAddress(order.getShippingAddress())
            .occurredAt(java.time.Instant.now())
            .build());
    }
}
```

---

## 14. Testing Email Templates

```java
package com.example.notification.service.email;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.mail.javamail.JavaMailSender;

import jakarta.mail.internet.MimeMessage;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@SpringBootTest
class EmailServiceTest {

    @Autowired
    EmailService emailService;

    @MockBean
    JavaMailSender mailSender;

    @Test
    void shouldSendOrderConfirmationEmail() {
        MimeMessage mimeMessage = mock(MimeMessage.class);
        when(mailSender.createMimeMessage()).thenReturn(mimeMessage);
        doNothing().when(mailSender).send(any(MimeMessage.class));

        var order = buildSampleOrderVars();

        emailService.sendHtmlEmail(
            "customer@example.com",
            "Order Confirmed - #ORD-001",
            "email/order-confirmation",
            order
        );

        verify(mailSender).send(any(MimeMessage.class));
    }

    private Map<String, Object> buildSampleOrderVars() {
        var orderItem = new com.example.notification.event.OrderItemDto();
        orderItem.setProductName("Premium Widget");
        orderItem.setQuantity(2);
        orderItem.setUnitPrice(BigDecimal.valueOf(49.99));
        orderItem.setSubtotal(BigDecimal.valueOf(99.98));

        var order = com.example.notification.event.OrderEvent.builder()
            .orderNumber("ORD-001")
            .placedAt(Instant.now())
            .items(List.of(orderItem))
            .total(BigDecimal.valueOf(99.98))
            .shippingAddress("123 Main St, Springfield, IL 62701")
            .build();

        return Map.of(
            "customerName", "John Doe",
            "order", order,
            "trackingUrl", "https://myshop.com/orders/1/track"
        );
    }
}
```

### Testing with GreenMail (SMTP mock server)

```java
package com.example.notification.service.email;

import com.icegreen.greenmail.configuration.GreenMailConfiguration;
import com.icegreen.greenmail.junit5.GreenMailExtension;
import com.icegreen.greenmail.util.ServerSetupTest;
import jakarta.mail.internet.MimeMessage;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@TestPropertySource(properties = {
    "spring.mail.host=localhost",
    "spring.mail.port=3025",
    "spring.mail.username=test",
    "spring.mail.password=test"
})
class EmailIntegrationTest {

    @RegisterExtension
    static GreenMailExtension greenMail = new GreenMailExtension(ServerSetupTest.SMTP)
        .withConfiguration(GreenMailConfiguration.aConfig().withUser("test", "test"))
        .withPerMethodLifecycle(false);

    @Autowired
    EmailService emailService;

    @Test
    void emailArrives() throws Exception {
        emailService.sendHtmlEmail(
            "user@example.com", "Test Subject",
            "email/password-reset",
            Map.of("userName", "Alice", "expiryMinutes", 30,
                   "resetUrl", "https://myshop.com/reset?token=abc")
        );

        MimeMessage[] messages = greenMail.getReceivedMessages();
        assertThat(messages).hasSize(1);
        assertThat(messages[0].getSubject()).isEqualTo("Test Subject");
        assertThat(messages[0].getAllRecipients()[0].toString())
            .isEqualTo("user@example.com");
    }
}
```

---

## Summary Table

| Feature | Technology | Class |
|---|---|---|
| HTML email | `JavaMailSender` + Thymeleaf | `EmailService` |
| Email attachment | `MimeMessageHelper.addAttachment` | `EmailService` |
| Inline image | `MimeMessageHelper.addInline` | `EmailService` |
| Transactional email (cloud) | SendGrid SDK | `SendGridEmailService` |
| Transactional email (AWS) | Amazon SES SDK v2 | `SesEmailService` |
| Push notification | Firebase Admin SDK | `FcmPushService` |
| SMS | Twilio SDK | `TwilioSmsService` |
| In-app real-time | STOMP / SimpMessagingTemplate | `InAppNotificationService` |
| User preferences | JPA entity + REST API | `NotificationPreferenceService` |
| Async + retry | `@Async` + `@Retryable` | `AsyncNotificationService` |
| Audit log | `NotificationLog` entity | `NotificationLogService` |
| Template testing | GreenMail extension | `EmailIntegrationTest` |

---

## Next Part Preview

**Part 066: Internationalization (i18n) and Localization** — learn how to serve your Spring Boot
application in multiple languages using `MessageSource`, resolve locales from headers, parameters,
or cookies, format dates/numbers/currencies per locale with ICU4J, translate database content,
and build a fully i18n-aware e-commerce site that speaks English, Thai, Japanese, German, and
Arabic.
