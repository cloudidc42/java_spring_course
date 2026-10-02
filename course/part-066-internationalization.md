# Part 066: Internationalization (i18n) and Localization

## Overview

Internationalization (i18n) is the process of designing your application so it can be adapted to
different languages and regions without code changes. Localization (l10n) is the adaptation itself.
This part covers everything from basic `MessageSource` wiring to ICU4J plural rules, timezone
handling, database content translation, and building a five-language e-commerce storefront.

---

## 1. Project Setup

### Maven dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- ICU4J for advanced formatting (plurals, date, number) -->
    <dependency>
        <groupId>com.ibm.icu</groupId>
        <artifactId>icu4j</artifactId>
        <version>74.2</version>
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
  messages:
    basename: i18n/messages
    encoding: UTF-8
    cache-duration: 3600          # seconds; -1 to disable cache in dev
    fallback-to-system-locale: false
  web:
    locale: en
    locale-resolver: accept-header   # or: fixed | session | cookie | parameter
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

app:
  supported-locales: en,th,ja,de,ar
  default-locale: en
  default-timezone: UTC
```

---

## 2. MessageSource Configuration

```java
package com.example.i18n.config;

import org.springframework.context.MessageSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.ReloadableResourceBundleMessageSource;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor;

import java.util.List;
import java.util.Locale;

@Configuration
public class I18nConfig implements WebMvcConfigurer {

    /** Available languages. */
    private static final List<Locale> SUPPORTED = List.of(
        Locale.ENGLISH,
        Locale.forLanguageTag("th"),
        Locale.JAPAN,
        Locale.GERMANY,
        Locale.forLanguageTag("ar")
    );

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource ms = new ReloadableResourceBundleMessageSource();
        ms.setBasenames(
            "classpath:i18n/messages",
            "classpath:i18n/validation",
            "classpath:i18n/errors"
        );
        ms.setDefaultEncoding("UTF-8");
        ms.setCacheSeconds(3600);
        ms.setFallbackToSystemLocale(false);
        return ms;
    }

    /**
     * Use Accept-Language header, fall back to English if unsupported.
     */
    @Bean
    public LocaleResolver localeResolver() {
        AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
        resolver.setSupportedLocales(SUPPORTED);
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }

    /**
     * ?lang=th  query parameter overrides Accept-Language.
     */
    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }

    /**
     * Make Bean Validation use the same MessageSource so @NotNull etc.
     * read from messages.properties.
     */
    @Bean
    public LocalValidatorFactoryBean validator(MessageSource messageSource) {
        LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
        factory.setValidationMessageInterpolator(
            new org.springframework.validation.beanvalidation
                .MessageInterpolatorFactory(messageSource).createInterpolator()
        );
        return factory;
    }
}
```

---

## 3. Message Property Files

### src/main/resources/i18n/messages.properties  (English — default)

```properties
# Navigation
nav.home=Home
nav.products=Products
nav.cart=Cart
nav.account=Account
nav.logout=Logout

# Product page
product.add_to_cart=Add to Cart
product.out_of_stock=Out of Stock
product.price=Price
product.available=Available
product.description=Description

# Order
order.summary=Order Summary
order.total=Total
order.place_order=Place Order
order.thank_you=Thank you for your order!
order.number=Order #{0}
order.item_count={0,choice,0#No items|1#1 item|1<{0} items}

# Greeting (uses MessageFormat)
greeting.welcome=Welcome, {0}!
greeting.items_in_cart=You have {0,choice,0#no items|1#one item|1<{0} items} in your cart.

# Errors
error.not_found=The requested resource was not found.
error.server=An unexpected error occurred. Please try again later.
error.access_denied=You do not have permission to access this resource.
```

### src/main/resources/i18n/messages_th.properties  (Thai)

```properties
nav.home=หน้าแรก
nav.products=สินค้า
nav.cart=ตะกร้า
nav.account=บัญชี
nav.logout=ออกจากระบบ

product.add_to_cart=เพิ่มลงตะกร้า
product.out_of_stock=สินค้าหมด
product.price=ราคา
product.available=มีสินค้า
product.description=รายละเอียด

order.summary=สรุปคำสั่งซื้อ
order.total=ยอดรวม
order.place_order=สั่งซื้อ
order.thank_you=ขอบคุณสำหรับคำสั่งซื้อของคุณ!
order.number=คำสั่งซื้อ #{0}
order.item_count={0,choice,0#ไม่มีสินค้า|1#1 รายการ|1<{0} รายการ}

greeting.welcome=ยินดีต้อนรับ, {0}!
greeting.items_in_cart=คุณมี {0,choice,0#ไม่มีสินค้า|1#1 ชิ้น|1<{0} ชิ้น} ในตะกร้า

error.not_found=ไม่พบทรัพยากรที่ร้องขอ
error.server=เกิดข้อผิดพลาดที่ไม่คาดคิด กรุณาลองใหม่ภายหลัง
error.access_denied=คุณไม่มีสิทธิ์เข้าถึงทรัพยากรนี้
```

### src/main/resources/i18n/messages_ja.properties  (Japanese)

```properties
nav.home=ホーム
nav.products=商品
nav.cart=カート
nav.account=アカウント
nav.logout=ログアウト

product.add_to_cart=カートに追加
product.out_of_stock=在庫切れ
product.price=価格
product.available=在庫あり
product.description=説明

order.summary=注文概要
order.total=合計
order.place_order=注文する
order.thank_you=ご注文ありがとうございます！
order.number=注文番号 #{0}
order.item_count={0,choice,0#商品なし|1#1点|1<{0}点}

greeting.welcome={0}様、ようこそ！
greeting.items_in_cart=カートに{0,choice,0#商品がありません|1#1点|1<{0}点}あります。

error.not_found=お探しのリソースが見つかりませんでした。
error.server=予期しないエラーが発生しました。後でもう一度お試しください。
error.access_denied=このリソースへのアクセス権限がありません。
```

### src/main/resources/i18n/messages_de.properties  (German)

```properties
nav.home=Startseite
nav.products=Produkte
nav.cart=Warenkorb
nav.account=Konto
nav.logout=Abmelden

product.add_to_cart=In den Warenkorb
product.out_of_stock=Nicht vorrätig
product.price=Preis
product.available=Verfügbar
product.description=Beschreibung

order.summary=Bestellübersicht
order.total=Gesamt
order.place_order=Bestellen
order.thank_you=Vielen Dank für Ihre Bestellung!
order.number=Bestellung #{0}
order.item_count={0,choice,0#Keine Artikel|1#1 Artikel|1<{0} Artikel}

greeting.welcome=Willkommen, {0}!
greeting.items_in_cart=Sie haben {0,choice,0#keine Artikel|1#einen Artikel|1<{0} Artikel} im Warenkorb.

error.not_found=Die angeforderte Ressource wurde nicht gefunden.
error.server=Ein unerwarteter Fehler ist aufgetreten. Bitte versuchen Sie es später erneut.
error.access_denied=Sie haben keine Berechtigung, auf diese Ressource zuzugreifen.
```

### src/main/resources/i18n/messages_ar.properties  (Arabic)

```properties
nav.home=الرئيسية
nav.products=المنتجات
nav.cart=سلة التسوق
nav.account=الحساب
nav.logout=تسجيل الخروج

product.add_to_cart=أضف إلى السلة
product.out_of_stock=نفذت الكمية
product.price=السعر
product.available=متوفر
product.description=الوصف

order.summary=ملخص الطلب
order.total=الإجمالي
order.place_order=تأكيد الطلب
order.thank_you=شكراً على طلبك!
order.number=الطلب رقم #{0}
order.item_count={0,choice,0#لا توجد عناصر|1#عنصر واحد|1<{0} عناصر}

greeting.welcome=مرحباً، {0}!
greeting.items_in_cart=لديك {0,choice,0#لا توجد عناصر|1#عنصر واحد|1<{0} عناصر} في سلة التسوق.

error.not_found=المورد المطلوب غير موجود.
error.server=حدث خطأ غير متوقع. يرجى المحاولة مرة أخرى لاحقاً.
error.access_denied=ليس لديك صلاحية الوصول إلى هذا المورد.
```

### src/main/resources/i18n/validation.properties

```properties
# Bean Validation messages (referenced by {key} in @NotNull etc.)
jakarta.validation.constraints.NotBlank.message=Field must not be blank
jakarta.validation.constraints.NotNull.message=Field must not be null
jakarta.validation.constraints.Size.message=Field size must be between {min} and {max}
jakarta.validation.constraints.Email.message=Must be a valid email address
jakarta.validation.constraints.Min.message=Must be at least {value}
jakarta.validation.constraints.Max.message=Must not exceed {value}
jakarta.validation.constraints.Pattern.message=Invalid format
```

---

## 4. LocaleResolver Strategies

### Session-based resolver (user persists locale across requests)

```java
package com.example.i18n.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.SessionLocaleResolver;
import java.util.Locale;

@Configuration
public class SessionLocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }
}
```

### Cookie-based resolver (locale survives browser restart)

```java
package com.example.i18n.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.CookieLocaleResolver;
import java.time.Duration;
import java.util.Locale;

@Configuration
public class CookieLocaleConfig {

    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver resolver = new CookieLocaleResolver("MYSHOP_LANG");
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setCookieMaxAge(Duration.ofDays(365));
        resolver.setCookieSecure(true);
        resolver.setCookiePath("/");
        return resolver;
    }
}
```

### Custom locale resolver (reads from user profile + falls back to header)

```java
package com.example.i18n.resolver;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.servlet.LocaleResolver;

import java.util.List;
import java.util.Locale;

@RequiredArgsConstructor
public class UserProfileLocaleResolver implements LocaleResolver {

    private static final List<Locale> SUPPORTED = List.of(
        Locale.ENGLISH,
        Locale.forLanguageTag("th"),
        Locale.JAPAN,
        Locale.GERMANY,
        Locale.forLanguageTag("ar")
    );

    private final UserLocaleRepository userLocaleRepository;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // 1. ?lang parameter
        String langParam = request.getParameter("lang");
        if (langParam != null && !langParam.isBlank()) {
            Locale requested = Locale.forLanguageTag(langParam);
            if (SUPPORTED.contains(requested)) return requested;
        }

        // 2. Authenticated user's saved preference
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated() && !"anonymousUser".equals(auth.getPrincipal())) {
            String email = auth.getName();
            Locale saved = userLocaleRepository.findLocaleByEmail(email);
            if (saved != null && SUPPORTED.contains(saved)) return saved;
        }

        // 3. Accept-Language header
        List<Locale.LanguageRange> ranges = Locale.LanguageRange.parse(
            request.getHeader("Accept-Language") == null
                ? "en" : request.getHeader("Accept-Language")
        );
        Locale best = Locale.lookup(ranges, SUPPORTED);
        return best != null ? best : Locale.ENGLISH;
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {
        // Persist locale to user profile if authenticated
        var auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated() && !"anonymousUser".equals(auth.getPrincipal())) {
            userLocaleRepository.updateLocaleByEmail(auth.getName(), locale);
        }
    }
}
```

---

## 5. Using MessageSource in Code

```java
package com.example.i18n.service;

import lombok.RequiredArgsConstructor;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Service;

import java.util.Locale;

@Service
@RequiredArgsConstructor
public class MessageService {

    private final MessageSource messageSource;

    /**
     * Get message using current request locale (set by LocaleContextHolder).
     */
    public String get(String key, Object... args) {
        return messageSource.getMessage(key, args, LocaleContextHolder.getLocale());
    }

    /**
     * Get message for an explicit locale.
     */
    public String get(String key, Locale locale, Object... args) {
        return messageSource.getMessage(key, args, locale);
    }

    /**
     * Get message with a fallback if key does not exist.
     */
    public String getOrDefault(String key, String fallback, Object... args) {
        return messageSource.getMessage(key, args, fallback, LocaleContextHolder.getLocale());
    }
}
```

### Usage in a REST controller

```java
package com.example.i18n.controller;

import com.example.i18n.service.MessageService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;
    private final MessageService messageService;

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        var product = productService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(
                messageService.get("error.not_found")));
        return ResponseEntity.ok(ProductResponse.from(product));
    }

    @PostMapping("/{id}/cart")
    public ResponseEntity<Map<String, String>> addToCart(
            @PathVariable Long id,
            @RequestParam(defaultValue = "1") int qty) {

        int cartSize = productService.addToCart(id, qty);
        String msg = messageService.get("greeting.items_in_cart", cartSize);
        return ResponseEntity.ok(Map.of("message", msg));
    }
}
```

---

## 6. Date, Number, and Currency Formatting

```java
package com.example.i18n.util;

import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.text.NumberFormat;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.time.format.FormatStyle;
import java.util.Currency;
import java.util.Locale;

@Component
public class LocaleAwareFormatter {

    /**
     * Format a price in the locale's currency format.
     * Uses the currency symbol of the provided currency code.
     */
    public String formatCurrency(BigDecimal amount, String currencyCode) {
        Locale locale = LocaleContextHolder.getLocale();
        NumberFormat fmt = NumberFormat.getCurrencyInstance(locale);
        fmt.setCurrency(Currency.getInstance(currencyCode));
        return fmt.format(amount);
    }

    /**
     * Format using locale-default currency.
     */
    public String formatCurrency(BigDecimal amount) {
        NumberFormat fmt = NumberFormat.getCurrencyInstance(LocaleContextHolder.getLocale());
        return fmt.format(amount);
    }

    /**
     * Format a number with locale-appropriate thousands separator and decimals.
     */
    public String formatNumber(Number number) {
        return NumberFormat.getNumberInstance(LocaleContextHolder.getLocale())
                           .format(number);
    }

    /**
     * Format a percentage (e.g., 0.15 → "15%").
     */
    public String formatPercent(double value) {
        return NumberFormat.getPercentInstance(LocaleContextHolder.getLocale())
                           .format(value);
    }

    /**
     * Format a LocalDate with locale-appropriate short format.
     */
    public String formatDate(LocalDate date) {
        return date.format(
            DateTimeFormatter.ofLocalizedDate(FormatStyle.SHORT)
                             .withLocale(LocaleContextHolder.getLocale())
        );
    }

    /**
     * Format a LocalDateTime with locale-appropriate medium format.
     */
    public String formatDateTime(LocalDateTime dateTime) {
        return dateTime.format(
            DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
                             .withLocale(LocaleContextHolder.getLocale())
        );
    }

    /**
     * Convert a UTC instant to a ZonedDateTime in user's timezone and format it.
     */
    public String formatInstant(Instant instant, String zoneIdStr) {
        ZoneId zone = ZoneId.of(zoneIdStr);
        ZonedDateTime zdt = instant.atZone(zone);
        return DateTimeFormatter
            .ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(LocaleContextHolder.getLocale())
            .format(zdt);
    }
}
```

### Formatting demo

```java
// English (en) + USD
formatter.formatCurrency(new BigDecimal("1234.50"), "USD")  // → "$1,234.50"
formatter.formatNumber(1234567.89)                          // → "1,234,567.89"
formatter.formatPercent(0.075)                              // → "8%"

// German (de) + EUR
formatter.formatCurrency(new BigDecimal("1234.50"), "EUR")  // → "1.234,50 €"
formatter.formatNumber(1234567.89)                          // → "1.234.567,89"

// Japanese (ja) + JPY
formatter.formatCurrency(new BigDecimal("1234"), "JPY")     // → "¥1,234"

// Arabic (ar) + SAR
formatter.formatCurrency(new BigDecimal("1234.50"), "SAR")  // → "١٬٢٣٤٫٥٠ ر.س."
```

---

## 7. Timezone Handling

```java
package com.example.i18n.timezone;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.ZoneId;
import java.time.ZoneOffset;

/**
 * Reads X-Timezone header (e.g. "Asia/Bangkok") and stores it in a thread-local
 * so services can use it for user-facing date formatting.
 */
@Slf4j
@Component
@Order(1)
public class TimezoneFilter implements Filter {

    private static final ThreadLocal<ZoneId> ZONE_HOLDER = new ThreadLocal<>();

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        try {
            String tz = ((HttpServletRequest) req).getHeader("X-Timezone");
            ZONE_HOLDER.set(parseZone(tz));
            chain.doFilter(req, res);
        } finally {
            ZONE_HOLDER.remove();
        }
    }

    public static ZoneId current() {
        ZoneId z = ZONE_HOLDER.get();
        return z != null ? z : ZoneOffset.UTC;
    }

    private ZoneId parseZone(String tz) {
        if (tz == null || tz.isBlank()) return ZoneOffset.UTC;
        try {
            return ZoneId.of(tz);
        } catch (Exception e) {
            log.warn("Invalid X-Timezone header: {}", tz);
            return ZoneOffset.UTC;
        }
    }
}
```

```java
package com.example.i18n.timezone;

import java.time.*;

/**
 * Utility for converting between UTC stored values and user-local display values.
 */
public class TimezoneConverter {

    private TimezoneConverter() {}

    public static ZonedDateTime toUserZone(Instant utcInstant) {
        return utcInstant.atZone(TimezoneFilter.current());
    }

    public static Instant toUtc(LocalDateTime userLocal) {
        return userLocal.atZone(TimezoneFilter.current()).toInstant();
    }

    /** Determine if the current user's timezone is observing DST. */
    public static boolean isInDst() {
        ZoneId zone = TimezoneFilter.current();
        return zone.getRules().isDaylightSavings(Instant.now());
    }
}
```

---

## 8. ICU4J for Advanced Formatting

ICU4J handles plural rules, ordinals, and complex message patterns that `java.text.MessageFormat`
cannot.

```java
package com.example.i18n.icu;

import com.ibm.icu.text.MessageFormat;
import com.ibm.icu.text.PluralRules;
import lombok.RequiredArgsConstructor;
import org.springframework.context.MessageSource;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Component;

import java.util.Locale;
import java.util.Map;

@Component
@RequiredArgsConstructor
public class IcuMessageService {

    private final MessageSource messageSource;

    /**
     * Format a message that uses ICU4J MessageFormat syntax.
     * Pattern example:
     *   "{count, plural, =0{No items} one{# item} other{# items}}"
     */
    public String formatIcu(String pattern, Map<String, Object> args) {
        Locale locale = LocaleContextHolder.getLocale();
        MessageFormat fmt = new MessageFormat(pattern, locale);
        return fmt.format(args);
    }

    /**
     * Look up a message key from MessageSource, then format it with ICU.
     */
    public String getMessage(String key, Map<String, Object> args) {
        Locale locale = LocaleContextHolder.getLocale();
        String pattern = messageSource.getMessage(key, null, locale);
        MessageFormat fmt = new MessageFormat(pattern, locale);
        return fmt.format(args);
    }
}
```

### ICU message pattern examples (messages_icu.properties)

```properties
# English plural
cart.items.count={count, plural, =0{Your cart is empty.} one{You have 1 item in your cart.} other{You have # items in your cart.}}

# Russian plural (3 forms)
cart.items.count_ru={count, plural, one{# товар} few{# товара} many{# товаров} other{# товара}}

# Ordinal (1st, 2nd, 3rd, 4th…)
order.rank={rank, selectordinal, one{#st order} two{#nd order} few{#rd order} other{#th order}}

# Gender-sensitive message
order.greeting={gender, select, male{Dear Mr. {name}} female{Dear Ms. {name}} other{Dear {name}}}

# Date skeleton
date.format={0, date, ::yMMMd}
```

### ICU usage example

```java
@Test
void testIcuPlural() {
    // English
    LocaleContextHolder.setLocale(Locale.ENGLISH);

    String zero = icuService.formatIcu(
        "{count, plural, =0{Your cart is empty.} one{You have 1 item.} other{You have # items.}}",
        Map.of("count", 0));
    assertThat(zero).isEqualTo("Your cart is empty.");

    String many = icuService.formatIcu(
        "{count, plural, =0{Your cart is empty.} one{You have 1 item.} other{You have # items.}}",
        Map.of("count", 5));
    assertThat(many).isEqualTo("You have 5 items.");
}
```

---

## 9. i18n Error Messages in REST APIs

### Global exception handler with locale-aware messages

```java
package com.example.i18n.exception;

import com.example.i18n.service.MessageService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@RestControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    private final MessageService messageService;

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ApiError(
                HttpStatus.NOT_FOUND.value(),
                messageService.get("error.not_found"),
                Instant.now()
            ));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiError> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ApiError(
                HttpStatus.FORBIDDEN.value(),
                messageService.get("error.access_denied"),
                Instant.now()
            ));
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {

        Map<String, List<String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.groupingBy(
                FieldError::getField,
                Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
            ));

        ApiValidationError error = new ApiValidationError(
            HttpStatus.UNPROCESSABLE_ENTITY.value(),
            messageService.get("error.validation"),
            Instant.now(),
            fieldErrors
        );
        return ResponseEntity.unprocessableEntity().body(error);
    }
}
```

```java
package com.example.i18n.exception;

import lombok.*;
import java.time.Instant;
import java.util.*;

@Data
@AllArgsConstructor
public class ApiError {
    private int status;
    private String message;
    private Instant timestamp;
}

@Data
@AllArgsConstructor
public class ApiValidationError {
    private int status;
    private String message;
    private Instant timestamp;
    private Map<String, List<String>> fieldErrors;
}
```

### Validation messages tied to locale

```java
package com.example.i18n.dto;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class RegisterRequest {

    @NotBlank(message = "{register.name.required}")
    @Size(min = 2, max = 100, message = "{register.name.size}")
    private String name;

    @NotBlank(message = "{register.email.required}")
    @Email(message = "{register.email.invalid}")
    private String email;

    @NotBlank(message = "{register.password.required}")
    @Size(min = 8, message = "{register.password.min}")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*\\d).+$",
             message = "{register.password.pattern}")
    private String password;
}
```

### src/main/resources/i18n/validation.properties

```properties
register.name.required=Name is required
register.name.size=Name must be between {min} and {max} characters
register.email.required=Email address is required
register.email.invalid=Please enter a valid email address
register.password.required=Password is required
register.password.min=Password must be at least {min} characters
register.password.pattern=Password must contain at least one uppercase letter and one digit
error.validation=Validation failed — please check your input
```

### src/main/resources/i18n/validation_th.properties

```properties
register.name.required=กรุณากรอกชื่อ
register.name.size=ชื่อต้องมีความยาวระหว่าง {min} ถึง {max} ตัวอักษร
register.email.required=กรุณากรอกที่อยู่อีเมล
register.email.invalid=กรุณากรอกที่อยู่อีเมลที่ถูกต้อง
register.password.required=กรุณากรอกรหัสผ่าน
register.password.min=รหัสผ่านต้องมีอย่างน้อย {min} ตัวอักษร
register.password.pattern=รหัสผ่านต้องมีตัวอักษรพิมพ์ใหญ่อย่างน้อย 1 ตัวและตัวเลขอย่างน้อย 1 ตัว
error.validation=การตรวจสอบล้มเหลว — กรุณาตรวจสอบข้อมูล
```

---

## 10. Database Content Translation Patterns

### Pattern A — Translation table (recommended for many languages)

```java
package com.example.i18n.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /** Default/fallback name (stored in English). */
    private String name;

    /** Default/fallback description. */
    @Column(length = 2000)
    private String description;

    private String sku;
    @Column(precision = 10, scale = 2)
    private java.math.BigDecimal price;
}
```

```java
package com.example.i18n.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "product_translations",
       uniqueConstraints = @UniqueConstraint(columnNames = {"product_id", "locale"}))
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ProductTranslation {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false, length = 10)
    private String locale;        // e.g. "th", "ja", "de", "ar"

    @Column(nullable = false)
    private String name;

    @Column(length = 2000)
    private String description;
}
```

```java
package com.example.i18n.repository;

import com.example.i18n.domain.ProductTranslation;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface ProductTranslationRepository
        extends JpaRepository<ProductTranslation, Long> {

    Optional<ProductTranslation> findByProductIdAndLocale(Long productId, String locale);
}
```

### Translated product service

```java
package com.example.i18n.service;

import com.example.i18n.domain.*;
import com.example.i18n.dto.ProductDto;
import com.example.i18n.repository.*;
import lombok.RequiredArgsConstructor;
import org.springframework.context.i18n.LocaleContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Locale;

@Service
@RequiredArgsConstructor
public class TranslatedProductService {

    private final ProductRepository productRepository;
    private final ProductTranslationRepository translationRepository;

    @Transactional(readOnly = true)
    public ProductDto getProduct(Long id) {
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));

        String locale = LocaleContextHolder.getLocale().getLanguage(); // "th", "en", "ja"…
        ProductTranslation translation =
            translationRepository.findByProductIdAndLocale(id, locale)
                .orElse(null); // fall back to default if no translation

        return ProductDto.builder()
            .id(product.getId())
            .sku(product.getSku())
            .price(product.getPrice())
            .name(translation != null ? translation.getName() : product.getName())
            .description(translation != null ? translation.getDescription() : product.getDescription())
            .build();
    }

    @Transactional
    public void saveTranslation(Long productId, String locale,
                                String name, String description) {
        Product product = productRepository.getReferenceById(productId);
        ProductTranslation t = translationRepository
            .findByProductIdAndLocale(productId, locale)
            .orElseGet(() -> ProductTranslation.builder()
                .product(product).locale(locale).build());
        t.setName(name);
        t.setDescription(description);
        translationRepository.save(t);
    }
}
```

### Pattern B — JSON column (PostgreSQL JSONB, simpler schema)

```java
package com.example.i18n.domain;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.util.Map;

@Entity
@Table(name = "products_json")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ProductJson {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /** Map<locale, translated name>. Stored as JSONB in PostgreSQL. */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private Map<String, String> names;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private Map<String, String> descriptions;

    private String sku;
    private java.math.BigDecimal price;

    public String getName(String locale) {
        return names.getOrDefault(locale, names.getOrDefault("en", ""));
    }

    public String getDescription(String locale) {
        return descriptions.getOrDefault(locale, descriptions.getOrDefault("en", ""));
    }
}
```

---

## 11. Character Encoding — UTF-8 Everywhere

```java
package com.example.i18n.config;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.filter.CharacterEncodingFilter;

@Configuration
public class EncodingConfig {

    @Bean
    public FilterRegistrationBean<CharacterEncodingFilter> encodingFilter() {
        CharacterEncodingFilter filter = new CharacterEncodingFilter();
        filter.setEncoding("UTF-8");
        filter.setForceRequestEncoding(true);
        filter.setForceResponseEncoding(true);

        FilterRegistrationBean<CharacterEncodingFilter> bean = new FilterRegistrationBean<>(filter);
        bean.setOrder(Integer.MIN_VALUE); // must run first
        return bean;
    }
}
```

### application.yml (server encoding)

```yaml
server:
  servlet:
    encoding:
      charset: UTF-8
      enabled: true
      force: true
```

---

## 12. Thymeleaf i18n in Templates

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en"
      th:lang="${#locale.language}">
<head>
    <meta charset="UTF-8"/>
    <title th:text="#{nav.products}">Products</title>
    <!-- RTL support for Arabic -->
    <th:block th:if="${#locale.language == 'ar'}">
        <style>body { direction: rtl; text-align: right; }</style>
    </th:block>
</head>
<body>
<nav>
    <a th:href="@{/}" th:text="#{nav.home}">Home</a>
    <a th:href="@{/products}" th:text="#{nav.products}">Products</a>
    <a th:href="@{/cart}" th:text="#{nav.cart}">Cart</a>
    <a th:href="@{/account}" th:text="#{nav.account}">Account</a>
</nav>

<h1 th:text="#{nav.products}">Products</h1>

<!-- Language switcher -->
<div class="lang-switcher">
    <a th:href="@{/products(lang='en')}">English</a>
    <a th:href="@{/products(lang='th')}">ภาษาไทย</a>
    <a th:href="@{/products(lang='ja')}">日本語</a>
    <a th:href="@{/products(lang='de')}">Deutsch</a>
    <a th:href="@{/products(lang='ar')}">العربية</a>
</div>

<div th:each="product : ${products}" class="product-card">
    <h2 th:text="${product.name}">Product Name</h2>
    <p th:text="${product.description}">Description</p>
    <p>
        <span th:text="#{product.price}">Price</span>:
        <strong th:text="${formatter.formatCurrency(product.price)}">$0.00</strong>
    </p>
    <button th:text="#{product.add_to_cart}"
            th:disabled="${product.stock == 0}">Add to Cart</button>
    <span th:if="${product.stock == 0}" th:text="#{product.out_of_stock}">Out of Stock</span>
</div>
</body>
</html>
```

---

## 13. i18n Testing

```java
package com.example.i18n;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class I18nApiTest {

    @Autowired MockMvc mockMvc;

    @ParameterizedTest
    @CsvSource({
        "en, Not Found",
        "th, ไม่พบทรัพยากรที่ร้องขอ",
        "ja, お探しのリソースが見つかりませんでした。",
        "de, Die angeforderte Ressource wurde nicht gefunden.",
        "ar, المورد المطلوب غير موجود."
    })
    void errorMessageIsLocalized(String lang, String expectedMsg) throws Exception {
        mockMvc.perform(get("/api/v1/products/99999")
                .header("Accept-Language", lang))
               .andExpect(status().isNotFound())
               .andExpect(jsonPath("$.message").value(expectedMsg));
    }

    @ParameterizedTest
    @CsvSource({
        "en, Add to Cart",
        "th, เพิ่มลงตะกร้า",
        "ja, カートに追加",
        "de, In den Warenkorb"
    })
    void productPageIsLocalized(String lang, String expectedButton) throws Exception {
        mockMvc.perform(get("/products/1").param("lang", lang))
               .andExpect(status().isOk())
               .andExpect(content().string(org.hamcrest.Matchers.containsString(expectedButton)));
    }
}
```

```java
package com.example.i18n.util;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.i18n.LocaleContextHolder;

import java.math.BigDecimal;
import java.util.Locale;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class LocaleAwareFormatterTest {

    @Autowired LocaleAwareFormatter formatter;

    @Test
    void currencyFormatsCorrectlyPerLocale() {
        LocaleContextHolder.setLocale(Locale.US);
        assertThat(formatter.formatCurrency(new BigDecimal("1234.50"), "USD"))
            .startsWith("$");

        LocaleContextHolder.setLocale(Locale.GERMANY);
        String de = formatter.formatCurrency(new BigDecimal("1234.50"), "EUR");
        assertThat(de).contains("€");

        LocaleContextHolder.setLocale(Locale.JAPAN);
        String ja = formatter.formatCurrency(new BigDecimal("1234"), "JPY");
        assertThat(ja).contains("¥");
    }
}
```

---

## 14. Real Example — Multi-language E-commerce (5 Languages)

### Product listing endpoint

```java
package com.example.i18n.controller;

import com.example.i18n.dto.ProductDto;
import com.example.i18n.service.TranslatedProductService;
import com.example.i18n.util.LocaleAwareFormatter;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class I18nProductController {

    private final TranslatedProductService productService;
    private final LocaleAwareFormatter formatter;

    @GetMapping
    public ResponseEntity<Page<ProductDto>> listProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "name") String sort) {

        Pageable pageable = PageRequest.of(page, size, Sort.by(sort));
        Page<ProductDto> products = productService.listProducts(pageable);
        return ResponseEntity.ok(products);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProduct(id));
    }
}
```

### Language change endpoint

```java
package com.example.i18n.controller;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.LocaleResolver;

import java.util.Locale;

@RestController
@RequestMapping("/api/v1/locale")
@RequiredArgsConstructor
public class LocaleController {

    private final LocaleResolver localeResolver;

    @PostMapping("/{lang}")
    public ResponseEntity<Void> setLocale(@PathVariable String lang,
                                          HttpServletRequest request,
                                          HttpServletResponse response) {
        Locale locale = Locale.forLanguageTag(lang);
        localeResolver.setLocale(request, response, locale);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Summary Table

| Feature | Mechanism | Key Class/Config |
|---|---|---|
| Message lookup | `MessageSource` | `ReloadableResourceBundleMessageSource` |
| Locale from header | `AcceptHeaderLocaleResolver` | `I18nConfig` |
| Locale from param | `LocaleChangeInterceptor` (`?lang=`) | `I18nConfig` |
| Locale from session | `SessionLocaleResolver` | optional config |
| Locale from cookie | `CookieLocaleResolver` | optional config |
| Date/number formatting | `java.text.NumberFormat` + `DateTimeFormatter` | `LocaleAwareFormatter` |
| Advanced plural/gender | ICU4J `MessageFormat` | `IcuMessageService` |
| Timezone conversion | `TimezoneFilter` thread-local | `TimezoneConverter` |
| Validation messages | `LocalValidatorFactoryBean` | `I18nConfig.validator()` |
| DB content translation | Translation table pattern | `ProductTranslation` |
| DB content (JSON) | JSONB column | `ProductJson` |
| UTF-8 encoding | `CharacterEncodingFilter` | `EncodingConfig` |
| Thymeleaf templates | `#{key}` expressions | `messages_*.properties` |
| Testing | `MockMvc` + `Accept-Language` | `I18nApiTest` |

---

## Next Part Preview

**Part 067: Advanced Search and Filtering** — build powerful product search using JPA Criteria API,
Spring Data Specifications, QueryDSL predicates, RSQL query language for REST APIs, and
Elasticsearch bool queries with faceted search, autocomplete, and result ranking.
