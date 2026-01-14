# 12 — Security & Cryptography

Разбор вопросов о безопасности в Go: хранение паролей, JWT, TLS, rate limiting, OWASP Top 10 и работа с секретами.

**Навигация:** [← 11-message-queues-events](./11-message-queues-events.md) | [Трек Go](./README.md) | [13-reflection-codegen →](./13-reflection-codegen.md)

---

## Вопросы и разборы

### 1. Как безопасно хранить пароли?

**Зачем спрашивают.**
Хранение паролей — базовая задача безопасности. Интервьюер проверяет понимание hashing vs encryption, salt, и современных алгоритмов.

**Короткий ответ.**
Пароли хранятся как hash, не plaintext. Используй bcrypt (стандарт) или Argon2 (современный). Никогда MD5/SHA без salt. Hash включает salt, не нужен отдельный.

**Детальный разбор.**

**Почему hash, а не encryption:**
```
┌─────────────────────────────────────────────────────────────┐
│ Encryption:                                                  │
│   plaintext ──► ciphertext ──► plaintext                    │
│   Двусторонняя: можно расшифровать с ключом                 │
│   Проблема: утечка ключа = все пароли скомпрометированы     │
│                                                              │
│ Hashing:                                                     │
│   plaintext ──► hash                                        │
│   Односторонняя: нельзя восстановить пароль                 │
│   Проверка: hash(input) == stored_hash                      │
└─────────────────────────────────────────────────────────────┘
```

**Сравнение алгоритмов:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Алгоритм        │ Особенности                              │
├─────────────────┼──────────────────────────────────────────┤
│ MD5, SHA1       │ НЕБЕЗОПАСНЫ: быстрые, rainbow tables     │
│ SHA256+salt     │ Лучше, но всё ещё слишком быстрый        │
├─────────────────┼──────────────────────────────────────────┤
│ bcrypt          │ Стандарт: медленный, adaptive cost       │
│                 │ Salt встроен, широко поддерживается      │
├─────────────────┼──────────────────────────────────────────┤
│ Argon2          │ Современный: memory-hard, GPU-resistant  │
│                 │ Победитель Password Hashing Competition  │
│                 │ Argon2id — рекомендуемый вариант         │
├─────────────────┼──────────────────────────────────────────┤
│ scrypt          │ Memory-hard, используется в crypto       │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**

**bcrypt (рекомендуется для большинства случаев):**
```go
import "golang.org/x/crypto/bcrypt"

const bcryptCost = 12  // увеличивай со временем (12-14 для 2024)

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)
    if err != nil {
        return "", err
    }
    return string(bytes), nil
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

// Использование
func RegisterUser(username, password string) error {
    hash, err := HashPassword(password)
    if err != nil {
        return err
    }

    // Сохраняем hash, не password
    return db.CreateUser(username, hash)
}

func LoginUser(username, password string) (*User, error) {
    user, err := db.GetUserByUsername(username)
    if err != nil {
        // Timing-safe: всегда проверяем hash даже если user не найден
        bcrypt.CompareHashAndPassword([]byte("$2a$12$dummy"), []byte(password))
        return nil, ErrInvalidCredentials
    }

    if !CheckPassword(password, user.PasswordHash) {
        return nil, ErrInvalidCredentials
    }

    return user, nil
}
```

**Argon2 (более современный):**
```go
import "golang.org/x/crypto/argon2"

type Argon2Params struct {
    Memory      uint32
    Iterations  uint32
    Parallelism uint8
    SaltLength  uint32
    KeyLength   uint32
}

var DefaultParams = Argon2Params{
    Memory:      64 * 1024,  // 64 MB
    Iterations:  3,
    Parallelism: 4,
    SaltLength:  16,
    KeyLength:   32,
}

func HashPasswordArgon2(password string) (string, error) {
    salt := make([]byte, DefaultParams.SaltLength)
    if _, err := rand.Read(salt); err != nil {
        return "", err
    }

    hash := argon2.IDKey(
        []byte(password),
        salt,
        DefaultParams.Iterations,
        DefaultParams.Memory,
        DefaultParams.Parallelism,
        DefaultParams.KeyLength,
    )

    // Формат: $argon2id$v=19$m=65536,t=3,p=4$salt$hash
    b64Salt := base64.RawStdEncoding.EncodeToString(salt)
    b64Hash := base64.RawStdEncoding.EncodeToString(hash)

    return fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version, DefaultParams.Memory, DefaultParams.Iterations,
        DefaultParams.Parallelism, b64Salt, b64Hash), nil
}

func VerifyPasswordArgon2(password, encodedHash string) (bool, error) {
    // Парсим параметры из hash
    params, salt, hash, err := decodeArgon2Hash(encodedHash)
    if err != nil {
        return false, err
    }

    // Вычисляем hash с теми же параметрами
    otherHash := argon2.IDKey([]byte(password), salt,
        params.Iterations, params.Memory, params.Parallelism, params.KeyLength)

    // Constant-time comparison
    return subtle.ConstantTimeCompare(hash, otherHash) == 1, nil
}
```

**Миграция cost factor:**
```go
func NeedsRehash(hash string) bool {
    cost, err := bcrypt.Cost([]byte(hash))
    if err != nil {
        return true
    }
    return cost < bcryptCost
}

func LoginWithRehash(username, password string) (*User, error) {
    user, err := db.GetUserByUsername(username)
    if err != nil {
        return nil, ErrInvalidCredentials
    }

    if !CheckPassword(password, user.PasswordHash) {
        return nil, ErrInvalidCredentials
    }

    // Rehash если cost устарел
    if NeedsRehash(user.PasswordHash) {
        newHash, _ := HashPassword(password)
        db.UpdatePasswordHash(user.ID, newHash)
    }

    return user, nil
}
```

**Типичные ошибки.**
1. **Plaintext storage** — никогда
2. **MD5/SHA без salt** — rainbow tables
3. **Низкий cost** — brute force быстрый
4. **Timing attack** — разное время ответа

**На интервью.**
Объясни разницу между hash и encryption. Покажи bcrypt с правильным cost. Расскажи про constant-time comparison.

*Частые follow-up вопросы:*
- Как выбрать bcrypt cost?
- Когда Argon2 лучше bcrypt?
- Как мигрировать на новый алгоритм?

---

### 2. Как работать с JWT в Go?

**Зачем спрашивают.**
JWT — стандарт для stateless authentication. Интервьюер проверяет понимание структуры, алгоритмов подписи и best practices.

**Короткий ответ.**
JWT состоит из header.payload.signature. Подписывается секретом (HS256) или ключом (RS256). Библиотека: golang-jwt/jwt. Важны: expiration, refresh tokens, secure storage.

**Детальный разбор.**

**Структура JWT:**
```
┌─────────────────────────────────────────────────────────────┐
│ eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.                       │
│ eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTY│
│ yMzkwMjJ9.                                                   │
│ SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c                 │
│                                                              │
│ Header:   {"alg": "HS256", "typ": "JWT"}                    │
│ Payload:  {"sub": "1234567890", "name": "John", "iat": ...} │
│ Signature: HMACSHA256(base64(header) + "." + base64(payload),│
│            secret)                                           │
└─────────────────────────────────────────────────────────────┘
```

**Алгоритмы подписи:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Алгоритм        │ Особенности                              │
├─────────────────┼──────────────────────────────────────────┤
│ HS256 (HMAC)    │ Симметричный: один секрет                │
│                 │ Быстрый, простой                         │
│                 │ Для: один сервис                         │
├─────────────────┼──────────────────────────────────────────┤
│ RS256 (RSA)     │ Асимметричный: private/public key        │
│                 │ Медленнее, но flexible                   │
│                 │ Для: микросервисы (public key verify)    │
├─────────────────┼──────────────────────────────────────────┤
│ ES256 (ECDSA)   │ Асимметричный, короткие ключи            │
│                 │ Для: mobile, IoT                         │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**

**Создание и валидация JWT:**
```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID   string   `json:"user_id"`
    Username string   `json:"username"`
    Roles    []string `json:"roles"`
    jwt.RegisteredClaims
}

type JWTService struct {
    secretKey     []byte
    accessExpiry  time.Duration
    refreshExpiry time.Duration
}

func NewJWTService(secret string) *JWTService {
    return &JWTService{
        secretKey:     []byte(secret),
        accessExpiry:  15 * time.Minute,
        refreshExpiry: 7 * 24 * time.Hour,
    }
}

func (s *JWTService) GenerateAccessToken(user *User) (string, error) {
    claims := Claims{
        UserID:   user.ID,
        Username: user.Username,
        Roles:    user.Roles,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(s.accessExpiry)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
            Subject:   user.ID,
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(s.secretKey)
}

func (s *JWTService) ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Проверяем алгоритм
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return s.secretKey, nil
    })

    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }

    return claims, nil
}
```

**Refresh Token flow:**
```go
type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func (s *JWTService) GenerateTokenPair(user *User) (*TokenPair, error) {
    accessToken, err := s.GenerateAccessToken(user)
    if err != nil {
        return nil, err
    }

    // Refresh token — случайная строка, хранится в БД
    refreshToken := generateSecureToken(32)

    // Сохраняем refresh token
    err = s.db.SaveRefreshToken(user.ID, refreshToken, time.Now().Add(s.refreshExpiry))
    if err != nil {
        return nil, err
    }

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

func (s *JWTService) RefreshTokens(refreshToken string) (*TokenPair, error) {
    // Проверяем refresh token в БД
    userID, err := s.db.ValidateRefreshToken(refreshToken)
    if err != nil {
        return nil, ErrInvalidRefreshToken
    }

    // Инвалидируем старый refresh token (rotation)
    s.db.DeleteRefreshToken(refreshToken)

    user, _ := s.db.GetUserByID(userID)
    return s.GenerateTokenPair(user)
}

func generateSecureToken(length int) string {
    bytes := make([]byte, length)
    rand.Read(bytes)
    return base64.URLEncoding.EncodeToString(bytes)
}
```

**Middleware:**
```go
func JWTMiddleware(jwtService *JWTService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }

            // Bearer token
            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 || parts[0] != "Bearer" {
                http.Error(w, "Invalid auth header", http.StatusUnauthorized)
                return
            }

            claims, err := jwtService.ValidateToken(parts[1])
            if err != nil {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }

            // Добавляем claims в context
            ctx := context.WithValue(r.Context(), "claims", claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

**Типичные ошибки.**
1. **JWT в localStorage** — XSS vulnerable, лучше httpOnly cookie
2. **Нет expiration** — токен валиден вечно
3. **Секрет в коде** — из env/vault
4. **Не проверять alg** — алгоритм none attack

**На интервью.**
Покажи структуру JWT. Объясни refresh token flow. Расскажи про HS256 vs RS256.

*Частые follow-up вопросы:*
- Как отозвать JWT до expiration?
- Где хранить токен на клиенте?
- Когда использовать session вместо JWT?

---

### 3. Что такое crypto/rand и почему не math/rand?

**Зачем спрашивают.**
Выбор RNG критичен для безопасности. Интервьюер проверяет понимание разницы между PRNG и CSPRNG.

**Короткий ответ.**
`math/rand` — псевдослучайный, предсказуемый при известном seed. `crypto/rand` — криптографически стойкий, использует OS entropy. Для security только `crypto/rand`.

**Детальный разбор.**

**Сравнение:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ math/rand       │ crypto/rand                              │
├─────────────────┼──────────────────────────────────────────┤
│ Псевдослучайный │ Криптографически стойкий                 │
│ Предсказуемый   │ Непредсказуемый                          │
│ Быстрый         │ Медленнее                                │
│ Детерминирован  │ Использует OS entropy                    │
│                 │ (/dev/urandom, CryptGenRandom)           │
├─────────────────┼──────────────────────────────────────────┤
│ Для:            │ Для:                                     │
│ • Симуляции     │ • Ключи, токены, nonces                  │
│ • Тесты         │ • Пароли, API keys                       │
│ • Игры          │ • Salt, IV                               │
│ • Шаффл         │ • Всё security-related                   │
└─────────────────┴──────────────────────────────────────────┘
```

**Проблема math/rand:**
```go
import "math/rand"

// НЕБЕЗОПАСНО: seed из времени
rand.Seed(time.Now().UnixNano())
token := rand.Int63()

// Атакующий может:
// 1. Узнать примерное время генерации
// 2. Перебрать seed за секунды
// 3. Предсказать все "случайные" значения
```

**Пример.**

**Безопасная генерация токенов:**
```go
import "crypto/rand"

// Безопасный случайный токен
func GenerateToken(length int) (string, error) {
    bytes := make([]byte, length)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

// API key
func GenerateAPIKey() (string, error) {
    bytes := make([]byte, 32)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return hex.EncodeToString(bytes), nil
}

// Безопасное случайное число
func RandomInt(max int64) (int64, error) {
    n, err := rand.Int(rand.Reader, big.NewInt(max))
    if err != nil {
        return 0, err
    }
    return n.Int64(), nil
}

// UUID v4 (crypto-safe)
func GenerateUUID() (string, error) {
    uuid := make([]byte, 16)
    if _, err := rand.Read(uuid); err != nil {
        return "", err
    }

    // Set version (4) and variant (RFC 4122)
    uuid[6] = (uuid[6] & 0x0f) | 0x40
    uuid[8] = (uuid[8] & 0x3f) | 0x80

    return fmt.Sprintf("%x-%x-%x-%x-%x",
        uuid[0:4], uuid[4:6], uuid[6:8], uuid[8:10], uuid[10:16]), nil
}
```

**Безопасный шаффл:**
```go
import (
    "crypto/rand"
    "math/big"
)

// Fisher-Yates с crypto/rand
func SecureShuffle[T any](slice []T) error {
    for i := len(slice) - 1; i > 0; i-- {
        j, err := rand.Int(rand.Reader, big.NewInt(int64(i+1)))
        if err != nil {
            return err
        }
        slice[i], slice[j.Int64()] = slice[j.Int64()], slice[i]
    }
    return nil
}
```

**Генерация паролей:**
```go
const (
    lowercase = "abcdefghijklmnopqrstuvwxyz"
    uppercase = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    digits    = "0123456789"
    special   = "!@#$%^&*"
)

func GeneratePassword(length int, useSpecial bool) (string, error) {
    charset := lowercase + uppercase + digits
    if useSpecial {
        charset += special
    }

    result := make([]byte, length)
    for i := range result {
        idx, err := rand.Int(rand.Reader, big.NewInt(int64(len(charset))))
        if err != nil {
            return "", err
        }
        result[i] = charset[idx.Int64()]
    }

    return string(result), nil
}
```

**math/rand v2 (Go 1.22+):**
```go
import "math/rand/v2"

// v2 автоматически использует хороший seed
// Но всё ещё НЕ для криптографии!
n := rand.IntN(100)  // v2 API

// Для детерминированных тестов
source := rand.NewPCG(1, 2)
r := rand.New(source)
n = r.IntN(100)
```

**Типичные ошибки.**
1. **math/rand для токенов** — предсказуемы
2. **time.Now() как seed** — легко угадать
3. **Фиксированный seed** — все токены одинаковые
4. **rand.Read без проверки ошибки** — может вернуть меньше bytes

**На интервью.**
Объясни разницу PRNG vs CSPRNG. Покажи безопасную генерацию токена. Расскажи про entropy source.

*Частые follow-up вопросы:*
- Откуда crypto/rand берёт entropy?
- Можно ли использовать math/rand v2 для security?
- Как генерировать cryptographically secure ID?

---

### 4. Как реализовать Rate Limiting?

**Зачем спрашивают.**
Rate limiting защищает от DDoS и abuse. Интервьюер проверяет знание алгоритмов и практическую реализацию.

**Короткий ответ.**
Rate limiting ограничивает количество запросов. Алгоритмы: Token Bucket, Sliding Window. Реализация: in-memory (x/time/rate), Redis для distributed.

**Детальный разбор.**

**Алгоритмы:**
```
┌─────────────────────────────────────────────────────────────┐
│ Token Bucket:                                                │
│                                                              │
│   Bucket: [●●●●●○○○○○]  capacity=10, rate=1/sec             │
│                                                              │
│   Request → берёт токен → если есть, пропускает             │
│   Время → добавляет токены (до capacity)                    │
│   Плюс: burst allowed, smooth                               │
├─────────────────────────────────────────────────────────────┤
│ Sliding Window Log:                                          │
│                                                              │
│   [req1, req2, req3, ..., reqN]  window=1min                │
│                                                              │
│   Request → подсчёт в окне → если < limit, пропускает       │
│   Плюс: точный, no burst                                    │
│   Минус: память на каждый request                           │
├─────────────────────────────────────────────────────────────┤
│ Sliding Window Counter:                                      │
│                                                              │
│   Prev bucket: 5 requests | Current bucket: 3 requests      │
│   weighted = prev * (1-elapsed%) + current                  │
│                                                              │
│   Плюс: низкое потребление памяти                           │
│   Минус: приблизительный                                    │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**golang.org/x/time/rate (Token Bucket):**
```go
import "golang.org/x/time/rate"

type RateLimiter struct {
    limiters sync.Map  // key → *rate.Limiter
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{
        rate:  r,
        burst: burst,
    }
}

func (l *RateLimiter) GetLimiter(key string) *rate.Limiter {
    limiter, exists := l.limiters.Load(key)
    if exists {
        return limiter.(*rate.Limiter)
    }

    newLimiter := rate.NewLimiter(l.rate, l.burst)
    l.limiters.Store(key, newLimiter)
    return newLimiter
}

func (l *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Key: IP или user ID
        key := getClientIP(r)

        limiter := l.GetLimiter(key)
        if !limiter.Allow() {
            http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func getClientIP(r *http.Request) string {
    // Проверяем X-Forwarded-For для proxy
    xff := r.Header.Get("X-Forwarded-For")
    if xff != "" {
        ips := strings.Split(xff, ",")
        return strings.TrimSpace(ips[0])
    }

    ip, _, _ := net.SplitHostPort(r.RemoteAddr)
    return ip
}
```

**Redis-based (distributed):**
```go
import "github.com/redis/go-redis/v9"

type RedisRateLimiter struct {
    client *redis.Client
    limit  int
    window time.Duration
}

func (l *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().UnixNano()
    windowStart := now - l.window.Nanoseconds()

    pipe := l.client.Pipeline()

    // Удаляем старые записи
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", windowStart))

    // Добавляем текущий запрос
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})

    // Подсчитываем в окне
    countCmd := pipe.ZCard(ctx, key)

    // TTL для cleanup
    pipe.Expire(ctx, key, l.window)

    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }

    count := countCmd.Val()
    return count <= int64(l.limit), nil
}

// Sliding Window Counter (более эффективный)
func (l *RedisRateLimiter) AllowSlidingWindow(ctx context.Context, key string) (bool, error) {
    script := redis.NewScript(`
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local current_window = math.floor(now / window)
        local prev_window = current_window - 1

        local current_key = key .. ":" .. current_window
        local prev_key = key .. ":" .. prev_window

        local current_count = tonumber(redis.call("GET", current_key) or "0")
        local prev_count = tonumber(redis.call("GET", prev_key) or "0")

        local elapsed = (now % window) / window
        local weighted = prev_count * (1 - elapsed) + current_count

        if weighted >= limit then
            return 0
        end

        redis.call("INCR", current_key)
        redis.call("EXPIRE", current_key, window * 2 / 1000000000)

        return 1
    `)

    result, err := script.Run(ctx, l.client, []string{key},
        l.limit, l.window.Nanoseconds(), time.Now().UnixNano()).Int()

    return result == 1, err
}
```

**Rate limit headers:**
```go
func (l *RateLimiter) MiddlewareWithHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := getClientIP(r)
        limiter := l.GetLimiter(key)

        reservation := limiter.Reserve()
        if !reservation.OK() {
            http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
            return
        }

        delay := reservation.Delay()
        if delay > 0 {
            reservation.Cancel()

            w.Header().Set("Retry-After", fmt.Sprintf("%.0f", delay.Seconds()))
            w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", l.burst))
            w.Header().Set("X-RateLimit-Remaining", "0")

            http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
            return
        }

        // Успех
        tokens := limiter.Tokens()
        w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", l.burst))
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%.0f", tokens))

        next.ServeHTTP(w, r)
    })
}
```

**Типичные ошибки.**
1. **IP без X-Forwarded-For** — за proxy все с одного IP
2. **Нет burst** — легитимные пики блокируются
3. **In-memory для distributed** — каждый pod свой лимит
4. **Нет Retry-After** — клиент не знает когда повторить

**На интервью.**
Объясни Token Bucket. Покажи distributed rate limiting. Расскажи про rate limit headers.

*Частые follow-up вопросы:*
- Как rate limit по user vs по IP?
- Как реализовать tiered limits (free vs premium)?
- Как защититься от distributed DDoS?

---

### 5. Как работать с TLS/mTLS в Go?

**Зачем спрашивают.**
TLS — основа безопасной коммуникации. Интервьюер проверяет понимание конфигурации и mutual TLS.

**Короткий ответ.**
TLS шифрует трафик. mTLS — взаимная аутентификация (сервер и клиент). Go: `tls.Config` с сертификатами. Production: Let's Encrypt или internal CA.

**Детальный разбор.**

**TLS vs mTLS:**
```
┌─────────────────────────────────────────────────────────────┐
│ TLS (server-only):                                           │
│                                                              │
│   Client ──────────► Server                                 │
│           verify server cert                                │
│           (клиент анонимен)                                 │
│                                                              │
│   Для: веб-сайты, публичные API                             │
├─────────────────────────────────────────────────────────────┤
│ mTLS (mutual):                                               │
│                                                              │
│   Client ◄────────► Server                                  │
│           verify each other                                 │
│           (оба аутентифицированы)                           │
│                                                              │
│   Для: микросервисы, service mesh, zero trust               │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**HTTPS сервер:**
```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    server := &http.Server{
        Addr:    ":443",
        Handler: mux,
        TLSConfig: &tls.Config{
            MinVersion: tls.VersionTLS12,
            CurvePreferences: []tls.CurveID{
                tls.X25519,
                tls.CurveP256,
            },
            CipherSuites: []uint16{
                tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
                tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
                tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
                tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
            },
        },
    }

    log.Fatal(server.ListenAndServeTLS("server.crt", "server.key"))
}
```

**mTLS сервер:**
```go
func NewMTLSServer(certFile, keyFile, caFile string) (*http.Server, error) {
    // Загружаем CA для проверки клиентов
    caCert, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        ClientCAs:  caCertPool,
        ClientAuth: tls.RequireAndVerifyClientCert,  // mTLS
        MinVersion: tls.VersionTLS12,
    }

    return &http.Server{
        Addr:      ":443",
        TLSConfig: tlsConfig,
    }, nil
}

// Получение информации о клиенте
func handler(w http.ResponseWriter, r *http.Request) {
    if r.TLS != nil && len(r.TLS.PeerCertificates) > 0 {
        clientCert := r.TLS.PeerCertificates[0]
        fmt.Fprintf(w, "Hello, %s\n", clientCert.Subject.CommonName)
    }
}
```

**mTLS клиент:**
```go
func NewMTLSClient(certFile, keyFile, caFile string) (*http.Client, error) {
    // Загружаем клиентский сертификат
    cert, err := tls.LoadX509KeyPair(certFile, keyFile)
    if err != nil {
        return nil, err
    }

    // Загружаем CA для проверки сервера
    caCert, err := os.ReadFile(caFile)
    if err != nil {
        return nil, err
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
        MinVersion:   tls.VersionTLS12,
    }

    return &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: tlsConfig,
        },
    }, nil
}
```

**Auto TLS с Let's Encrypt:**
```go
import "golang.org/x/crypto/acme/autocert"

func main() {
    manager := &autocert.Manager{
        Cache:      autocert.DirCache("certs"),  // кеш сертификатов
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("example.com", "www.example.com"),
    }

    server := &http.Server{
        Addr:      ":443",
        Handler:   myHandler,
        TLSConfig: manager.TLSConfig(),
    }

    // HTTP redirect to HTTPS
    go http.ListenAndServe(":80", manager.HTTPHandler(nil))

    log.Fatal(server.ListenAndServeTLS("", ""))  // сертификаты из manager
}
```

**Certificate rotation:**
```go
type CertReloader struct {
    certFile string
    keyFile  string
    cert     atomic.Pointer[tls.Certificate]
}

func NewCertReloader(certFile, keyFile string) (*CertReloader, error) {
    r := &CertReloader{certFile: certFile, keyFile: keyFile}
    if err := r.Reload(); err != nil {
        return nil, err
    }
    return r, nil
}

func (r *CertReloader) Reload() error {
    cert, err := tls.LoadX509KeyPair(r.certFile, r.keyFile)
    if err != nil {
        return err
    }
    r.cert.Store(&cert)
    return nil
}

func (r *CertReloader) GetCertificate(clientHello *tls.ClientHelloInfo) (*tls.Certificate, error) {
    return r.cert.Load(), nil
}

// Использование
func main() {
    reloader, _ := NewCertReloader("server.crt", "server.key")

    // Watch файлы и reload
    go func() {
        watcher, _ := fsnotify.NewWatcher()
        watcher.Add("server.crt")
        for range watcher.Events {
            reloader.Reload()
        }
    }()

    server := &http.Server{
        TLSConfig: &tls.Config{
            GetCertificate: reloader.GetCertificate,
        },
    }
}
```

**Типичные ошибки.**
1. **InsecureSkipVerify: true** — отключает проверку сертификата
2. **TLS 1.0/1.1** — устаревшие, уязвимые
3. **Слабые cipher suites** — RC4, DES
4. **Self-signed в production** — warning в браузере

**На интервью.**
Объясни mTLS flow. Покажи конфигурацию сервера с ClientAuth. Расскажи про certificate rotation.

*Частые follow-up вопросы:*
- Как работает certificate chain?
- Когда использовать mTLS vs API keys?
- Как отлаживать TLS проблемы?

---

### 6. Что такое OWASP Top 10 и как защититься в Go?

**Зачем спрашивают.**
OWASP Top 10 — стандартный чеклист уязвимостей. Интервьюер проверяет awareness и знание защитных мер.

**Короткий ответ.**
OWASP Top 10 — список критичных web-уязвимостей. В Go: prepared statements против SQL injection, html/template против XSS, CSRF tokens, input validation.

**Детальный разбор.**

**OWASP Top 10 (2021) и защита в Go:**
```
┌──────────────────────────────────────────────────────────────┐
│ A01: Broken Access Control                                   │
│     Защита: RBAC middleware, проверка ownership             │
├──────────────────────────────────────────────────────────────┤
│ A02: Cryptographic Failures                                  │
│     Защита: crypto/rand, TLS, bcrypt для паролей            │
├──────────────────────────────────────────────────────────────┤
│ A03: Injection (SQL, Command, LDAP)                          │
│     Защита: prepared statements, input validation           │
├──────────────────────────────────────────────────────────────┤
│ A04: Insecure Design                                         │
│     Защита: threat modeling, secure by default              │
├──────────────────────────────────────────────────────────────┤
│ A05: Security Misconfiguration                               │
│     Защита: secure defaults, no debug in prod               │
├──────────────────────────────────────────────────────────────┤
│ A06: Vulnerable Components                                   │
│     Защита: govulncheck, dependabot                         │
├──────────────────────────────────────────────────────────────┤
│ A07: Authentication Failures                                 │
│     Защита: rate limiting, secure session, MFA              │
├──────────────────────────────────────────────────────────────┤
│ A08: Software and Data Integrity                             │
│     Защита: signed artifacts, verify checksums              │
├──────────────────────────────────────────────────────────────┤
│ A09: Logging and Monitoring Failures                         │
│     Защита: structured logging, audit trail                 │
├──────────────────────────────────────────────────────────────┤
│ A10: SSRF                                                    │
│     Защита: whitelist URLs, disable redirects               │
└──────────────────────────────────────────────────────────────┘
```

**Пример.**

**SQL Injection (A03):**
```go
// УЯЗВИМО
query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", userInput)
db.Query(query)  // userInput = "1; DROP TABLE users;--"

// ЗАЩИТА: prepared statements
db.Query("SELECT * FROM users WHERE id = $1", userInput)

// Или с sqlx
db.Get(&user, "SELECT * FROM users WHERE id = $1", userInput)
```

**XSS (Cross-Site Scripting):**
```go
// УЯЗВИМО
func handler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    fmt.Fprintf(w, "<h1>Hello, %s</h1>", name)  // XSS!
}

// ЗАЩИТА: html/template автоматически escapes
var tmpl = template.Must(template.New("").Parse(`<h1>Hello, {{.Name}}</h1>`))

func handler(w http.ResponseWriter, r *http.Request) {
    name := r.URL.Query().Get("name")
    tmpl.Execute(w, map[string]string{"Name": name})
}

// Или явный escape
html.EscapeString(userInput)
```

**CSRF Protection:**
```go
import "github.com/gorilla/csrf"

func main() {
    // CSRF middleware
    csrfMiddleware := csrf.Protect(
        []byte("32-byte-long-auth-key-here-----"),
        csrf.Secure(true),  // только HTTPS
        csrf.HttpOnly(true),
    )

    r := mux.NewRouter()
    r.HandleFunc("/form", formHandler)
    r.HandleFunc("/submit", submitHandler).Methods("POST")

    http.ListenAndServe(":8080", csrfMiddleware(r))
}

func formHandler(w http.ResponseWriter, r *http.Request) {
    // Вставляем токен в форму
    tmpl.Execute(w, map[string]interface{}{
        csrf.TemplateTag: csrf.TemplateField(r),
    })
}

// Шаблон:
// <form method="POST" action="/submit">
//     {{ .csrfField }}
//     <input type="text" name="data">
//     <button>Submit</button>
// </form>
```

**Command Injection:**
```go
// УЯЗВИМО
cmd := exec.Command("sh", "-c", "echo " + userInput)

// ЗАЩИТА: передавать аргументы отдельно
cmd := exec.Command("echo", userInput)

// Или валидация/whitelist
func runCommand(action string) error {
    allowed := map[string]bool{"start": true, "stop": true, "restart": true}
    if !allowed[action] {
        return errors.New("invalid action")
    }
    return exec.Command("systemctl", action, "myservice").Run()
}
```

**Path Traversal:**
```go
// УЯЗВИМО
filePath := filepath.Join("/uploads", userInput)  // userInput = "../etc/passwd"
os.ReadFile(filePath)

// ЗАЩИТА
func SecureFilePath(base, userInput string) (string, error) {
    // Очищаем путь
    cleaned := filepath.Clean(userInput)

    // Проверяем, что не выходит за base
    fullPath := filepath.Join(base, cleaned)
    if !strings.HasPrefix(fullPath, filepath.Clean(base)+string(os.PathSeparator)) {
        return "", errors.New("invalid path")
    }

    return fullPath, nil
}
```

**Security Headers:**
```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Prevent XSS
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")

        // CSP
        w.Header().Set("Content-Security-Policy", "default-src 'self'")

        // HSTS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

        next.ServeHTTP(w, r)
    })
}
```

**Типичные ошибки.**
1. **String concatenation для SQL** — injection
2. **text/template вместо html/template** — XSS
3. **Нет CSRF для форм** — state-changing actions
4. **Sensitive data в logs** — passwords, tokens

**На интервью.**
Покажи примеры injection и защиты. Объясни, как html/template защищает от XSS. Расскажи про security headers.

*Частые follow-up вопросы:*
- Как проводить security review кода?
- Какие static analyzers для security?
- Как настроить CSP?

---

### 7. Как безопасно работать с секретами?

**Зачем спрашивают.**
Утечка секретов — частая причина breaches. Интервьюер проверяет понимание best practices хранения и передачи секретов.

**Короткий ответ.**
Секреты не в коде и не в git. Используй: environment variables, secret managers (Vault, AWS Secrets Manager), Kubernetes secrets. Ротация, audit logging.

**Детальный разбор.**

**Иерархия безопасности:**
```
┌─────────────────────────────────────────────────────────────┐
│ ПЛОХО → ЛУЧШЕ → ХОРОШО                                       │
├─────────────────────────────────────────────────────────────┤
│ В коде         → .env файл      → Environment variables     │
│                                 → Secret manager (Vault)    │
│                                 → Cloud secrets (AWS SM)    │
│                                                              │
│ Plain text     → Encrypted      → Hardware HSM              │
│                                                              │
│ Shared secrets → Per-env secrets→ Short-lived tokens        │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Environment variables:**
```go
import "os"

type Config struct {
    DBPassword   string
    APIKey       string
    JWTSecret    string
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        DBPassword: os.Getenv("DB_PASSWORD"),
        APIKey:     os.Getenv("API_KEY"),
        JWTSecret:  os.Getenv("JWT_SECRET"),
    }

    // Validate required secrets
    if cfg.DBPassword == "" {
        return nil, errors.New("DB_PASSWORD is required")
    }

    return cfg, nil
}

// Не логировать секреты!
func (c *Config) String() string {
    return fmt.Sprintf("Config{DBPassword: [REDACTED], APIKey: [REDACTED]}")
}
```

**HashiCorp Vault:**
```go
import vault "github.com/hashicorp/vault/api"

type VaultClient struct {
    client *vault.Client
}

func NewVaultClient(addr, token string) (*VaultClient, error) {
    config := vault.DefaultConfig()
    config.Address = addr

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, err
    }

    client.SetToken(token)
    return &VaultClient{client: client}, nil
}

func (v *VaultClient) GetSecret(path string) (map[string]interface{}, error) {
    secret, err := v.client.Logical().Read(path)
    if err != nil {
        return nil, err
    }

    if secret == nil {
        return nil, errors.New("secret not found")
    }

    return secret.Data, nil
}

// Использование
func LoadDBConfig(vault *VaultClient) (*DBConfig, error) {
    secret, err := vault.GetSecret("secret/data/database")
    if err != nil {
        return nil, err
    }

    data := secret["data"].(map[string]interface{})
    return &DBConfig{
        Host:     data["host"].(string),
        Password: data["password"].(string),
    }, nil
}
```

**AWS Secrets Manager:**
```go
import (
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/secretsmanager"
)

func GetAWSSecret(ctx context.Context, secretName string) (string, error) {
    cfg, err := config.LoadDefaultConfig(ctx)
    if err != nil {
        return "", err
    }

    client := secretsmanager.NewFromConfig(cfg)

    input := &secretsmanager.GetSecretValueInput{
        SecretId: aws.String(secretName),
    }

    result, err := client.GetSecretValue(ctx, input)
    if err != nil {
        return "", err
    }

    return *result.SecretString, nil
}

// С кешированием и auto-refresh
type SecretCache struct {
    client   *secretsmanager.Client
    cache    sync.Map
    ttl      time.Duration
}

type cachedSecret struct {
    value     string
    expiresAt time.Time
}

func (c *SecretCache) Get(ctx context.Context, name string) (string, error) {
    if cached, ok := c.cache.Load(name); ok {
        cs := cached.(cachedSecret)
        if time.Now().Before(cs.expiresAt) {
            return cs.value, nil
        }
    }

    // Fetch from AWS
    secret, err := c.fetch(ctx, name)
    if err != nil {
        return "", err
    }

    c.cache.Store(name, cachedSecret{
        value:     secret,
        expiresAt: time.Now().Add(c.ttl),
    })

    return secret, nil
}
```

**Kubernetes Secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=  # base64 encoded

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: myapp
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
```

**Secret detection в CI:**
```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0

    - uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: ${{ github.event.repository.default_branch }}

    - uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**.gitignore для секретов:**
```gitignore
# Secrets
.env
.env.*
*.pem
*.key
secrets/
credentials.json
```

**Типичные ошибки.**
1. **Секреты в git** — даже после удаления в истории
2. **Логирование секретов** — в стектрейсах, debug
3. **Hardcoded fallbacks** — `os.Getenv("X") || "default"`
4. **Общие секреты между env** — prod secret в staging

**На интервью.**
Покажи интеграцию с Vault. Объясни иерархию хранения секретов. Расскажи про ротацию и audit.

*Частые follow-up вопросы:*
- Как ротировать секреты без downtime?
- Vault vs AWS Secrets Manager — когда что?
- Как обнаружить leaked secrets?

---

## Практика

1. **Passwords:** Реализуй регистрацию/логин с bcrypt. Добавь rehash при login.

2. **JWT:** Реализуй access/refresh token flow. Добавь token revocation.

3. **Rate Limiting:** Настрой distributed rate limiter на Redis.

4. **TLS:** Настрой mTLS между двумя сервисами. Добавь certificate rotation.

5. **Security Audit:** Проведи OWASP review своего проекта. Исправь найденные issues.

6. **Secrets:** Интегрируй Vault. Настрой auto-renewal.

---

## Дополнительные материалы

- [OWASP Top 10](https://owasp.org/Top10/)
- [Go Cryptography](https://pkg.go.dev/crypto)
- [golang-jwt/jwt](https://github.com/golang-jwt/jwt)
- [HashiCorp Vault](https://www.vaultproject.io/docs)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [Let's Encrypt with Go](https://letsencrypt.org/docs/client-options/)
- Books: "Secure by Design" — Dan Bergh Johnsson
- Talks: "Secure Go" — Filippo Valsorda
