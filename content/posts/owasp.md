+++
date = '2025-06-04T00:10:38+07:00'
draft = false
title = 'OWASP'
author = 'Huy Dang Quang'
categories = ["Security"]
tags = ["owasp", "security"]
description = 'Những lỗ hổng bảo mật phổ biến'
+++

# Tổng quan

Bảo mật ứng dụng web đã trở thành mối quan tâm hàng đầu trong bối cảnh số hóa ngày càng gia tăng . **OWASP** (Open Web Application Security Project) là tổ chức phi lợi nhuận toàn cầu chuyên về bảo mật ứng dụng web, cung cấp danh sách **OWASP Top 10** - những lỗ hổng bảo mật nghiêm trọng nhất được cập nhật định kỳ.

[**OWASP Top 10 2021**](https://owasp.org/www-project-top-ten/) được phát hành nhằm đánh dấu 20 năm của tổ chức OWASP và vẫn là tiêu chuẩn phổ biến nhất. Danh sách này được xây dựng dựa trên dữ liệu từ hơn 200,000 tổ chức và khảo sát các chuyên gia trong ngành, bao gồm 2 triệu báo cáo bảo mật từ 144 nguồn công khai.

![](/static/images/owasp.png)

Theo thông [tin chính thức từ OWASP](https://owasp.org/www-project-top-ten/), họ đang thu thập dữ liệu cho phiên bản **OWASP Top 10 2025** từ tháng 1 năm 2024 và dự kiến công bố kết quả trong nửa đầu năm 2025. Quá trình thu thập dữ liệu đã kết thúc vào tháng 9 năm 2024, sau đó nhóm sẽ xem xét và phân tích dữ liệu để đưa ra danh sách cập nhật.

Dưới đây là bảng so sánh Rankings 2021 với dự đoán 2025

![](/static/images/owasp-2021-2025.png)


# A01:2021 - Broken Access Control (Kiểm soát Truy cập Bị phá vỡ)

Lỗ hổng này xảy ra khi người dùng có thể thực hiện các hành động ngoài phạm vi quyền hạn của họ.

## 🔴 Code Vulnerable

```go
// Lỗ hổng: Không kiểm tra quyền truy cập
func getUserData(w http.ResponseWriter, r *http.Request) {
    userID := r.URL.Query().Get("user_id")
    
    // VULNERABLE: Không kiểm tra người dùng hiện tại có quyền truy cập user_id này không
    user, err := db.Query("SELECT * FROM users WHERE id = ?", userID)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}
```

## ✅ Code An toàn

```go
// An toàn: Kiểm tra quyền truy cập
func getUserData(w http.ResponseWriter, r *http.Request) {
    userID := r.URL.Query().Get("user_id")
    currentUserID := getCurrentUserFromToken(r)
    
    // SECURE: Kiểm tra quyền truy cập
    if !hasAccessToUser(currentUserID, userID) {
        http.Error(w, "Forbidden", 403)
        return
    }
    
    user, err := db.Query("SELECT * FROM users WHERE id = ?", userID)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}

func hasAccessToUser(currentUserID, targetUserID string) bool {
    // Kiểm tra quyền: user chỉ có thể truy cập dữ liệu của chính mình
    // hoặc admin có thể truy cập tất cả
    if currentUserID == targetUserID {
        return true
    }
    
    return isAdmin(currentUserID)
}
```

- Lỗ hổng CSRF Protection (Cross-Site Request Forgery Protection)

```go
// An toàn: sử dụng thư viện github.com/gorilla/csrf
import "github.com/gorilla/csrf"

func main() {
    r := mux.NewRouter()
    
    // Thiết lập CSRF protection
    CSRF := csrf.Protect([]byte("32-byte-long-auth-key"))
    
    r.HandleFunc("/form", showForm)
    r.HandleFunc("/process", processForm).Methods("POST")
    
    http.ListenAndServe(":8000", CSRF(r))
}

func showForm(w http.ResponseWriter, r *http.Request) {
    // Lấy CSRF token
    token := csrf.Token(r)
    
    tmpl := `
    <form method="POST" action="/process">
        <input type="hidden" name="csrf_token" value="{{.}}">
        <input type="text" name="data">
        <button type="submit">Submit</button>
    </form>`
    
    t := template.Must(template.New("form").Parse(tmpl))
    t.Execute(w, token)
}
```

## Cách phòng tránh
- Sử dụng **middleware** xác thực và phân quyền với các framework như Gin, Echo...
- Implement Role-Based Access Control (**RBAC**), [tham khảo thêm](https://www.permit.io/blog/role-based-access-control-rbac-authorization-in-golang)
- Validate tất cả user input, có thể sử dụng thư viện [github.com/go-playground/validator](github.com/go-playground/validator)
- Áp dụng nguyên tắc "principle of least privilege" trong thiết kế hệ thống


# A02:2021 - Cryptographic Failures (Lỗi Mật mã học)

Lỗ hổng này trước đây được gọi là **Sensitive Data Exposure**, liên quan đến việc sử dụng mật mã hóa yếu, lỗi thời hoặc không được triển khai đúng cách.

## 🔴 Code Vulnerable

```go
// Lỗ hổng: Sử dụng mật mã yếu
import (
    "crypto/md5"
    "fmt"
)

func hashPassword(password string) string {
    // VULNERABLE: Sử dụng MD5 - thuật toán yếu
    hasher := md5.New()
    hasher.Write([]byte(password))
    return fmt.Sprintf("%x", hasher.Sum(nil))
}

func storeUser(username, password string) {
    // VULNERABLE: Lưu trữ mật khẩu với hash yếu
    hashedPassword := hashPassword(password)
    db.Exec("INSERT INTO users (username, password) VALUES (?, ?)", 
            username, hashedPassword)
}
```

## ✅ Code An toàn

```go
// An toàn: Sử dụng bcrypt
import (
    "golang.org/x/crypto/bcrypt"
    "log"
)

func hashPassword(password string) (string, error) {
    // SECURE: Sử dụng bcrypt với cost 14
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 14)
    return string(bytes), err
}

func verifyPassword(password, hash string) bool {
    // SECURE: So sánh an toàn với bcrypt
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}

func storeUser(username, password string) error {
    // SECURE: Hash mật khẩu an toàn trước khi lưu
    hashedPassword, err := hashPassword(password)
    if err != nil {
        return err
    }
    
    _, err = db.Exec("INSERT INTO users (username, password) VALUES (?, ?)", 
                    username, hashedPassword)
    return err
}
```

## Cách phòng tránh
- Sử dụng `crypto/rand` thay vì `math/rand`cho việc tạo số ngẫu nhiên an toàn
- Implement bcrypt cho password hashing với [golang.org/x/crypto/bcrypt](golang.org/x/crypto/bcrypt)
- Tránh sử dụng các thuật toán yếu như MD5, SHA1, DES, RC4
- Sử dụng TLS 1.3 cho giao tiếp và AES-GCM cho symmetric encryption, [tham khảo thêm](https://dzone.com/articles/implementing-testing-cryptographic-primitives-go)


# A03:2021 - Injection (Tiêm mã độc)

SQL injection là lỗ hổng phổ biến nhất.

## 🔴 Code Vulnerable

```go
// Lỗ hổng: SQL Injection
func getUser(w http.ResponseWriter, r *http.Request) {
    username := r.URL.Query().Get("username")
    
    // VULNERABLE: String concatenation cho SQL query
    query := "SELECT * FROM users WHERE username = '" + username + "'"
    rows, err := db.Query(query)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    
    // Process rows...
}

// Lỗ hổng: Command Injection
func executeCommand(w http.ResponseWriter, r *http.Request) {
    filename := r.URL.Query().Get("file")
    
    // VULNERABLE: Trực tiếp sử dụng user input trong command
    cmd := exec.Command("cat", "/var/log/"+filename)
    output, err := cmd.Output()
    if err != nil {
        http.Error(w, "Command failed", 500)
        return
    }
    
    w.Write(output)
}

// Lỗ hổng: Cross-Site Scripting (XSS) Prevention
func renderPage(w http.ResponseWriter, userInput string) {
    fmt.Fprintf(w, "<h1>Xin chào %s</h1>", userInput)
}
```

## ✅ Code An toàn

```go
// An toàn: Sử dụng Prepared Statements
func getUser(w http.ResponseWriter, r *http.Request) {
    username := r.URL.Query().Get("username")
    
    // SECURE: Sử dụng parameterized query
    query := "SELECT * FROM users WHERE username = ?"
    rows, err := db.Query(query, username)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    
    // Process rows...
}

// An toàn: Validate input và whitelist
func executeCommand(w http.ResponseWriter, r *http.Request) {
    filename := r.URL.Query().Get("file")
    
    // SECURE: Validate input với whitelist
    allowedFiles := map[string]bool{
        "access.log":  true,
        "error.log":   true,
        "system.log":  true,
    }
    
    if !allowedFiles[filename] {
        http.Error(w, "Invalid file", 400)
        return
    }
    
    // SECURE: Sử dụng filepath.Join để tránh path traversal
    safePath := filepath.Join("/var/log", filepath.Clean(filename))
    
    // Kiểm tra path không escape khỏi thư mục cho phép
    if !strings.HasPrefix(safePath, "/var/log/") {
        http.Error(w, "Invalid path", 400)
        return
    }
    
    content, err := ioutil.ReadFile(safePath)
    if err != nil {
        http.Error(w, "File not found", 404)
        return
    }
    
    w.Write(content)
}

// AN TOÀN: Escape HTML input
func renderPage(w http.ResponseWriter, userInput string) {
    escapedInput := template.HTMLEscapeString(userInput)
    fmt.Fprintf(w, "<h1>Xin chào %s</h1>", escapedInput)
}
```

## Cách phòng tránh
- Luôn sử dụng **parameterized queries** và **prepared statements**
- Validate và sanitize input với các thư viện chuyên dụng 
- Sử dụng ORM có built-in protection như **GORM**
- Tránh string concatenation khi xây dựng SQL queries

# A04:2021 - Insecure Design (Thiết kế Không An toàn)

Đây là lỗ hổng về thiết kế kiến trúc hệ thống thiếu controls bảo mật.

## Cách phòng tránh
- Thực hiện threat modeling từ giai đoạn thiết kế, [tham khảo thêm](https://www.cybersecurity-insiders.com/7-steps-to-implement-secure-design-patterns-a-robust-foundation-for-software-security/)
- Implement secure design patterns và sử dụng các framework bảo mật
- Code review bảo mật thường xuyên và sử dụng static analysis tools 
- Thiết lập security requirements rõ ràng từ đầu dự án


# A05:2021 - Security Misconfiguration (Cấu hình Bảo mật Sai)

Lỗ hổng xảy ra khi cấu hình bảo mật không đúng hoặc không đầy đủ. Điều này có thể bao gồm default passwords, expose sensitive endpoints, hoặc disable security features.

## Cách phòng tránh
- Sử dụng environment variables cho configuration thay vì hardcode
- Implement secure defaults và automated configuration validation, [tham khảo thêm](https://www.reco.ai/learn/security-misconfiguration)
- Thực hiện regular security audits và sử dụng configuration scanners
- Tránh expose debug information và sensitive endpoints trong production


Bonus: **Secure File Upload**

```go
import (
    "path/filepath"
    "strings"
)

func isAllowedFileType(filename string) bool {
    allowedExtensions := []string{".jpg", ".jpeg", ".png", ".gif", ".pdf"}
    ext := strings.ToLower(filepath.Ext(filename))
    
    for _, allowed := range allowedExtensions {
        if ext == allowed {
            return true
        }
    }
    return false
}

func uploadFile(w http.ResponseWriter, r *http.Request) {
    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }
    defer file.Close()

    // Kiểm tra loại file
    if !isAllowedFileType(header.Filename) {
        http.Error(w, "File type not allowed", http.StatusBadRequest)
        return
    }

    // Giới hạn kích thước file (10MB)
    if header.Size > 10*1024*1024 {
        http.Error(w, "File too large", http.StatusBadRequest)
        return
    }

    // Xử lý file upload...
}
```

# A06:2021 - Vulnerable and Outdated Components (Thành phần Dễ tổn thương)

Lỗ hổng này liên quan đến việc sử dụng các thư viện, framework với lỗ hổng đã biết hoặc không được cập nhật. Đây là vấn đề mà nhiều tổ chức gặp phải khi quản lý dependencies.

## Cách phòng tránh
- Sử dụng `go mod` để quản lý dependencies một cách hiệu quả
- Chạy `govulncheck` thường xuyên để kiểm tra vulnerabilities
- Sử dụng Nancy dependency scanner để phát hiện lỗ hổng trong dependencies
- Thiết lập automated dependency updates và monitoring


# A07:2021 - Identification and Authentication Failures (Lỗi Xác thực)

Lỗ hổng trong việc xác thực danh tính người dùng, bao gồm weak password policies, session fixation, và poor session management.

## Cách phòng tránh
- Implement JWT với proper expiration và secure secret management
- Sử dụng strong password hashing với bcrypt 
- Thiết lập secure session management và implement multi-factor authentication 
- Sử dụng **rate limiting** để chống **brute force attacks**

Rate Limiting

```go
import (
    "golang.org/x/time/rate"
    "net/http"
)

func rateLimitMiddleware(limiter *rate.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

func main() {
    // Cho phép 10 requests mỗi giây
    limiter := rate.NewLimiter(10, 1)
    
    mux := http.NewServeMux()
    mux.HandleFunc("/api", apiHandler)
    
    http.ListenAndServe(":8080", rateLimitMiddleware(limiter)(mux))
}
```


# A08:2021 - Software and Data Integrity Failures (Lỗi Toàn vẹn)

Lỗ hổng tập trung vào việc không xác minh tính toàn vẹn của software và data. Điều này bao gồm insecure CI/CD pipelines và auto-update without verification.

## Cách phòng tránh
- Implement checksum verification cho packages và dependencies 
- Sử dụng digital signatures cho software packages 
- Thiết lập secure CI/CD pipeline với proper validation 
- Implement dependency verification và supply chain security 


# A09:2021 - Security Logging and Monitoring Failures (Lỗi Giám sát)

Thiếu logging, monitoring và response kịp thời có thể dẫn đến việc không phát hiện được các cuộc tấn công. Ngoài ra việc ghi log không an toàn (làm lộ thông tin nhạy cảm) là một vấn đề.

## Cách phòng tránh
- Sử dụng structured logging với logrus, zap,...
- Implement monitoring và alerting system hiệu quả
- Thiết lập security audit trails và log retention policies
- Regular log analysis và automated threat detection, [tham khảo thêm](https://cloud.google.com/architecture/security-log-analytics)

Secure Logging

```go
import "github.com/sirupsen/logrus"

func setupSecureLogging() *logrus.Logger {
    log := logrus.New()
    log.SetFormatter(&logrus.JSONFormatter{})
    
    // Không log sensitive data
    log.AddHook(&sanitizerHook{})
    
    return log
}

type sanitizerHook struct{}

func (hook *sanitizerHook) Fire(entry *logrus.Entry) error {
    // Loại bỏ sensitive fields
    delete(entry.Data, "password")
    delete(entry.Data, "token")
    delete(entry.Data, "secret")
    return nil
}

func (hook *sanitizerHook) Levels() []logrus.Level {
    return logrus.AllLevels
}
```

# A10:2021 - Server-Side Request Forgery (SSRF)

SSRF xảy ra khi ứng dụng web lấy remote resource mà không validate destination URL

## 🔴 Code Vulnerable

```go
// Lỗ hổng: SSRF - không validate URL
func fetchData(w http.ResponseWriter, r *http.Request) {
    url := r.URL.Query().Get("url")
    
    // VULNERABLE: Trực tiếp request đến URL do user cung cấp
    resp, err := http.Get(url)
    if err != nil {
        http.Error(w, "Request failed", 500)
        return
    }
    defer resp.Body.Close()
    
    body, _ := ioutil.ReadAll(resp.Body)
    w.Write(body)
}
```

## ✅ Code An toàn

```go
// An toàn: Validate URL và restrict requests
import (
    "net"
    "net/url"
    "strings"
)

func fetchData(w http.ResponseWriter, r *http.Request) {
    urlStr := r.URL.Query().Get("url")
    
    // SECURE: Validate và sanitize URL
    if !isAllowedURL(urlStr) {
        http.Error(w, "URL not allowed", 400)
        return
    }
    
    // SECURE: Sử dụng custom client với restrictions
    client := &http.Client{
        Transport: &http.Transport{
            DialContext: restrictedDialContext,
        },
    }
    
    resp, err := client.Get(urlStr)
    if err != nil {
        http.Error(w, "Request failed", 500)
        return
    }
    defer resp.Body.Close()
    
    body, _ := ioutil.ReadAll(resp.Body)
    w.Write(body)
}

func isAllowedURL(urlStr string) bool {
    u, err := url.Parse(urlStr)
    if err != nil {
        return false
    }
    
    // Chỉ cho phép HTTPS
    if u.Scheme != "https" {
        return false
    }
    
    // Whitelist domains
    allowedDomains := []string{
        "api.example.com",
        "secure.partner.com",
    }
    
    for _, domain := range allowedDomains {
        if u.Host == domain {
            return true
        }
    }
    
    return false
}

func restrictedDialContext(ctx context.Context, network, addr string) (net.Conn, error) {
    host, _, err := net.SplitHostPort(addr)
    if err != nil {
        return nil, err
    }
    
    // Resolve hostname to IP
    ips, err := net.LookupIP(host)
    if err != nil {
        return nil, err
    }
    
    // Block private IP ranges
    for _, ip := range ips {
        if isPrivateIP(ip) {
            return nil, fmt.Errorf("access to private IP not allowed")
        }
    }
    
    // Use default dialer
    dialer := &net.Dialer{}
    return dialer.DialContext(ctx, network, addr)
}

func isPrivateIP(ip net.IP) bool {
    privateRanges := []string{
        "10.0.0.0/8",
        "172.16.0.0/12",
        "192.168.0.0/16",
        "127.0.0.0/8",
        "169.254.0.0/16",
    }
    
    for _, cidr := range privateRanges {
        _, subnet, _ := net.ParseCIDR(cidr)
        if subnet.Contains(ip) {
            return true
        }
    }
    
    return false
}
```

## Cách phòng tránh
- Implement URL validation và sanitization nghiêm ngặt
- Sử dụng allowlist cho trusted domains và IP ranges
- Thiết lập network segmentation và restrict outbound connections
- Validate và filter user-provided URLs trước khi sử dụng, [tham khảo thêm](https://blog.stackademic.com/security-in-golang-how-to-protect-your-applications-from-common-vulnerabilities-a6e388872e82)
