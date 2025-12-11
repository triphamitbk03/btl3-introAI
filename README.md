# 🎬 BÀI TRÌNH BÀY AUTHENTICATION - 2 PHÚT

## 📋 Tổng quan

Trình bày phân tích hệ thống Authentication (Login/Register/Logout) theo 4 tính chất:

- **Availability** (Tính sẵn sàng)
- **Safety** (Tính an toàn)
- **Security** (Tính bảo mật)
- **Reliability** (Tính tin cậy)

**Thời gian:** 2 phút  
**Cấu trúc:** 6 phần (Giới thiệu + 4 tính chất + Demo)

---

## 🎤 KỊCH BẢN CHI TIẾT

### PHẦN 1: GIỚI THIỆU (15 giây)

> **"Xin chào, hôm nay tôi sẽ trình bày về hệ thống Authentication của dự án ASE-251.**
>
> **Hệ thống bao gồm 3 chức năng chính:**
>
> - **Register** - Đăng ký tài khoản mới
> - **Login** - Đăng nhập với email/password
> - **Logout** - Đăng xuất
>
> **Tôi sẽ phân tích qua 4 tính chất quan trọng: Availability, Safety, Security và Reliability."**

**🎬 Action:** Show slide hoặc code structure overview

---

### PHẦN 2: AVAILABILITY - TÍNH SẴN SÀNG (20 giây)

> **"Đầu tiên là Availability - khả năng hệ thống sẵn sàng phục vụ.**
>
> **Hệ thống đạt 6/10 điểm.**
>
> **✅ Điểm mạnh:**
>
> - **Deployed trên Render** - hệ thống đã được deploy lên production environment, accessible 24/7
> - Sử dụng **Async MongoDB operations** với Motor driver - xử lý nhiều requests đồng thời hiệu quả
> - **Auto-reconnect mechanism** - tự động kết nối lại database khi mất kết nối
> - **CORS enabled** - frontend có thể gọi API dễ dàng
>
> **⚠️ Điểm yếu:**
>
> - Chưa có **connection pooling** được config đúng cho production
> - Chưa có **retry mechanism** khi DB operation thất bại
> - Chưa có **health check endpoint** để monitor service status
>
> **Đây là code trong file db_client.py"**

**🎬 Action:** Show code `BE/app/database/db_client.py` lines 27-36

```python
async def get_database():
    """Get database instance. Auto-connect if not connected."""
    global client
    if client is None:
        await connect_to_mongo()  # ← Auto-reconnect!
    return client[DATABASE_NAME]

async def get_users_collection():
    """Get users collection with async client."""
    db = await get_database()
    return db["users"]  # ← Async operations
```

**🎬 Optional:** Show deployment proof

- Render dashboard hoặc
- Truy cập live API: `https://your-app.onrender.com/docs`
- Show API response time

---

### PHẦN 3: SAFETY - TÍNH AN TOÀN (25 giây)

> **"Tiếp theo là Safety - tính an toàn trong xử lý dữ liệu.**
>
> **Hệ thống đạt 6.5/10 điểm.**
>
> **✅ Các tính năng đã có:**
>
> - **Input validation** - kiểm tra email format với regex
> - **Role validation** - chỉ chấp nhận 2 role: lecturer và student
> - **Required fields check** - bắt buộc fullname, email, password
> - **Unique email constraint** - ngăn chặn duplicate accounts
> - **Blacklist mechanism** - chặn user bị cấm với flag is_blacklisted
>
> **❌ Vấn đề cần cải thiện:**
>
> - Chưa có **password complexity rules** - password "123" vẫn được chấp nhận
> - Chưa có **input sanitization** - có thể bị XSS attack qua trường fullname
>
> **Đây là các validation trong auth.py"**

**🎬 Action:** Show code `BE/app/routers/auth.py` lines 66-112

```python
# Email format validation
if not _EMAIL_REGEX.match(request.email):
    return JSONResponse(status_code=400, ...)

# Role validation - chỉ 2 role
if role not in ("lecturer", "student"):
    return JSONResponse(status_code=400, ...)

# Unique email check
existing_email = await users_collection.find_one({"email": request.email})
if existing_email:
    return JSONResponse(status_code=409, ...)  # Conflict

# Blacklist check
if user.get("is_blacklisted"):
    return JSONResponse(status_code=403, ...)  # Forbidden
```

---

### PHẦN 4: SECURITY - TÍNH BẢO MẬT (30 giây)

> **"Phần quan trọng nhất - Security, tính bảo mật.**
>
> **Hệ thống chỉ đạt 4/10 điểm - đây là vấn đề NGHIÊM TRỌNG cần khắc phục.**
>
> **✅ Điểm mạnh:**
>
> - **Password hashing với bcrypt** - không lưu plaintext password
> - **Secure password verification** - dùng bcrypt.verify để tránh timing attack
> - **HTTP status codes chuẩn** - 401 Unauthorized, 403 Forbidden, 409 Conflict
>
> **🔴 VẤN ĐỀ CRITICAL:**
>
> - **KHÔNG CÓ JWT/TOKEN** - logout chỉ là dummy endpoint return success, không có session management thực sự
> - **MongoDB URI hardcoded** - username password nằm ngay trong source code
> - **CORS allow all origins** - không secure cho production environment
> - **Không có rate limiting** - dễ bị brute force attack
>
> **Đây là code password hashing và các security issues"**

**🎬 Action:** Show code `BE/app/routers/auth.py` lines 22-42 và 213-216

```python
# ✅ GOOD: Password hashing với bcrypt
_pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__truncate_error=False,
)

def _hash_password(password: str) -> str:
    if len(password.encode("utf-8")) > _MAX_PASSWORD_BYTES:
        raise HTTPException(status_code=400, ...)
    return _pwd_context.hash(password)  # ← Secure hashing

def _verify_password(plain_password: str, hashed_password: str) -> bool:
    return _pwd_context.verify(plain_password, hashed_password)  # ← Safe verify
```

```python
# ❌ BAD: Logout dummy - không làm gì cả!
@router.post("/logout", response_model=AuthResponse)
async def logout():
    """Logout user"""
    return AuthResponse(status="success", message="Logout successful")
    # ← Không invalidate token, không clear session!
```

**Show:** `BE/app/config/settings.py`

```python
# ❌ BAD: Hardcoded credentials
MONGODB_URI = "mongodb+srv://hungnguyen2205_db_user:ASE-251-2025@..."
# ← Password exposed trong code!
```

---

### PHẦN 5: RELIABILITY - TÍNH TIN CẬY (25 giây)

> **"Cuối cùng là Reliability - độ tin cậy của hệ thống.**
>
> **Hệ thống đạt 5.5/10 điểm.**
>
> **✅ Điểm mạnh:**
>
> - **Type hints với Pydantic** - tự động validation request/response
> - **Consistent error format** - tất cả lỗi đều có code và message
> - **Auto-increment user_id** - format UYYYYMM0001 theo năm và tháng
> - **Datetime tracking** - lưu created_at cho mỗi user
>
> **⚠️ VẤN ĐỀ NGHIÊM TRỌNG:**
>
> - **Race condition trong user ID generation** - 2 requests đồng thời có thể tạo duplicate user_id
> - **Không có transaction support** - có thể bị data inconsistency
> - **Không có logging** - không track được operations, khó debug
>
> **Đây là race condition trong code"**

**🎬 Action:** Show code `BE/app/routers/auth.py` lines 48-54

```python
async def _generate_user_id(users_collection) -> str:
    """Generate user_id like UYYYYMM0001, incrementing per month."""
    now = datetime.utcnow()
    prefix = f"U{now.year}{now.month:02d}"

    # ⚠️ RACE CONDITION HERE!
    count = await users_collection.count_documents({"user_id": {"$regex": f"^{prefix}"}})
    sequence = count + 1  # ← NOT ATOMIC! Có thể duplicate!

    return f"{prefix}{sequence:04d}"
```

**Giải thích:**

- Request 1: count = 5, sequence = 6
- Request 2 (đồng thời): count = 5, sequence = 6 ← DUPLICATE!
- Cả 2 tạo user_id = "U20251206"

---

### PHẦN 6: DEMO & KẾT LUẬN (15 giây)

> **"Bây giờ tôi sẽ demo test suite để verify các tính năng.**
>
> **Chúng ta có 42 test cases covering 4 quality attributes."**

**🎬 Action:** Chạy terminal commands

```bash
# Terminal 1: Chạy tất cả tests
cd /Users/phamnguyenviettri/Ses251/ASE-251/BE
pytest tests/test_auth*.py -v --tb=short

# Kết quả mong đợi:
# ✓ 42 passed in 2.5s
```

> **"Tổng kết:**
>
> - ✅ **Availability: 6/10** - Async operations tốt, cần thêm connection pooling
> - ✅ **Safety: 6.5/10** - Có validation cơ bản, thiếu password policy
> - ⚠️ **Security: 4/10** - CRITICAL: Cần JWT, rate limiting, fix hardcoded credentials
> - ⚠️ **Reliability: 5.5/10** - Race condition cần fix bằng UUID hoặc atomic counter
>
> **Overall: 5.5/10 - Hệ thống hoạt động nhưng CẦN cải thiện security và reliability để production-ready.**
>
> **Cảm ơn các bạn đã theo dõi!"**

---

## 📂 FILES CẦN MỞ TRƯỚC KHI QUAY

### 1. Authentication Router

**File:** `BE/app/routers/auth.py`

**Key sections:**

- Lines 21-28: Password hashing setup
- Lines 30-42: `_hash_password()` và `_verify_password()`
- Lines 48-54: `_generate_user_id()` - RACE CONDITION
- Lines 66-79: Email và required fields validation
- Lines 89-99: Role validation
- Lines 101-112: Unique email check
- Lines 173-180: Blacklist check
- Lines 213-216: Logout dummy

### 2. Database Client

**File:** `BE/app/database/db_client.py`

**Key sections:**

- Lines 13-22: `connect_to_mongo()` - connection setup
- Lines 27-36: `get_database()` - auto-reconnect
- Lines 38-42: `get_users_collection()` - async collection

### 3. Schemas

**File:** `BE/app/schemas/auth.py`

**Key sections:**

- Lines 5-11: `RegisterRequest` - Pydantic validation
- Lines 14-16: `LoginRequest`
- Lines 19-26: `LoginResponse`

### 4. Settings

**File:** `BE/app/config/settings.py`

**Key sections:**

- Lines 1-3: MongoDB URI hardcoded (SECURITY ISSUE)

### 5. Test Files

**Files:**

- `BE/tests/test_auth.py` (7 tests)
- `BE/tests/test_auth_security.py` (15 tests)
- `BE/tests/test_auth_safety.py` (20 tests)

---

## 🎬 TIMELINE CHI TIẾT (120 giây)

| Thời gian | Phần         | Nội dung                             | Action                  |
| --------- | ------------ | ------------------------------------ | ----------------------- |
| 0:00-0:15 | Intro        | Giới thiệu 3 chức năng + 4 tính chất | Show overview           |
| 0:15-0:35 | Availability | Async ops, auto-reconnect            | Show `db_client.py`     |
| 0:35-1:00 | Safety       | Validation rules, blacklist          | Show validation code    |
| 1:00-1:30 | Security     | Password hashing, JWT missing        | Show hashing + issues   |
| 1:30-1:55 | Reliability  | Pydantic, race condition             | Show user_id generation |
| 1:55-2:00 | Demo         | Run tests, kết luận                  | Run pytest command      |

---

## 🧪 COMMANDS ĐỂ CHẠY

### Trước khi quay:

```bash
# 1. Đảm bảo đang ở đúng branch
git checkout ASE-RateLimit

# 2. Vào thư mục BE
cd /Users/phamnguyenviettri/Ses251/ASE-251/BE

# 3. Kiểm tra server chạy được
python3 -m app.main
# Ctrl+C để stop

# 4. Test xem pytest hoạt động
pytest tests/test_auth.py -v
```

### Trong video:

**Command 1: Chạy tất cả auth tests**

```bash
pytest tests/test_auth*.py -v
```

**Command 2: Chạy với coverage (optional)**

```bash
pytest tests/test_auth*.py --cov=app.routers.auth --cov-report=term-missing
```

**Command 3: Chạy specific test file**

```bash
# Security tests
pytest tests/test_auth_security.py -v

# Safety tests
pytest tests/test_auth_safety.py -v
```

---

## 📊 KẾT QUẢ MONG ĐỢI

```bash
$ pytest tests/test_auth*.py -v

tests/test_auth.py::test_register_success_returns_created_user PASSED
tests/test_auth.py::test_register_rejects_duplicate_email PASSED
tests/test_auth.py::test_register_rejects_invalid_role PASSED
tests/test_auth.py::test_login_success_with_hashed_password PASSED
tests/test_auth.py::test_login_rejects_invalid_password PASSED
tests/test_auth.py::test_login_blocks_blacklisted_user PASSED
tests/test_auth.py::test_logout_returns_success PASSED

tests/test_auth_security.py::test_password_is_hashed_not_plaintext PASSED
tests/test_auth_security.py::test_same_password_produces_different_hashes PASSED
tests/test_auth_security.py::test_password_length_limit_prevents_dos PASSED
tests/test_auth_security.py::test_email_validation_rejects_invalid_formats PASSED
tests/test_auth_security.py::test_sql_injection_in_email_is_safe PASSED
tests/test_auth_security.py::test_nosql_injection_in_email_is_safe PASSED
tests/test_auth_security.py::test_required_fields_validation PASSED
tests/test_auth_security.py::test_only_valid_roles_accepted PASSED
tests/test_auth_security.py::test_valid_roles_accepted PASSED
tests/test_auth_security.py::test_blacklisted_user_cannot_login PASSED
tests/test_auth_security.py::test_login_error_does_not_leak_user_existence PASSED
tests/test_auth_security.py::test_duplicate_email_prevented PASSED
tests/test_auth_security.py::test_legacy_plaintext_password_still_works PASSED
tests/test_auth_security.py::test_empty_password_rejected PASSED
tests/test_auth_security.py::test_logout_always_succeeds PASSED

tests/test_auth_safety.py::test_valid_email_formats_accepted PASSED
tests/test_auth_safety.py::test_invalid_email_formats_rejected PASSED
tests/test_auth_safety.py::test_email_case_sensitivity PASSED
tests/test_auth_safety.py::test_all_required_fields_must_be_present PASSED
tests/test_auth_safety.py::test_empty_string_fields_rejected PASSED
tests/test_auth_safety.py::test_valid_roles_accepted PASSED
tests/test_auth_safety.py::test_invalid_roles_rejected PASSED
tests/test_auth_safety.py::test_password_max_length_enforced PASSED
tests/test_auth_safety.py::test_password_with_unicode_characters PASSED
tests/test_auth_safety.py::test_short_passwords_accepted_but_shouldnt PASSED
tests/test_auth_safety.py::test_duplicate_email_prevented PASSED
tests/test_auth_safety.py::test_login_requires_email_and_password PASSED
tests/test_auth_safety.py::test_login_with_empty_credentials PASSED
tests/test_auth_safety.py::test_blacklisted_user_blocked_from_login PASSED
tests/test_auth_safety.py::test_non_blacklisted_user_can_login PASSED
tests/test_auth_safety.py::test_none_password_rejected PASSED
tests/test_auth_safety.py::test_password_verification_handles_edge_cases PASSED
tests/test_auth_safety.py::test_user_data_saved_correctly PASSED
tests/test_auth_safety.py::test_login_returns_correct_user_data PASSED
tests/test_auth_safety.py::test_fullname_with_special_characters PASSED
tests/test_auth_safety.py::test_error_format_is_consistent PASSED

============================== 42 passed in 2.45s ==============================

Coverage: 70% of app.routers.auth
```

---

## 💡 TIPS KHI TRÌNH BÀY

### ✅ NÊN:

- Nói rõ ràng, tự tin, tốc độ vừa phải
- Point chuột vào code quan trọng khi giải thích
- Highlight màu sắc: 🟢 GREEN cho điểm mạnh, 🔴 RED cho issues
- Nhấn mạnh từ khóa: CRITICAL, RACE CONDITION, ASYNC, JWT
- Pause ngắn giữa các phần để người xem theo dõi
- Show terminal output to demonstrate working tests

### ❌ KHÔNG NÊN:

- Đọc từng dòng code chi tiết (chỉ explain ý chính)
- Nói quá nhanh hoặc quá chậm
- Skip phần demo test (quan trọng để prove claims)
- Để lỗi trong terminal khi chạy tests
- Quá 2 phút (strict timeline!)

---

## 🎯 KEY MESSAGES

1. **Availability: 6/10** - ✅ Deployed on Render, Async tốt, thiếu pooling & health check
2. **Safety: 6.5/10** - Validation cơ bản, thiếu password policy
3. **Security: 4/10** - 🔴 CRITICAL: No JWT, hardcoded creds
4. **Reliability: 5.5/10** - Race condition cần fix

**Overall: 5.5/10** - Hoạt động nhưng CẦN cải thiện để production-ready

---

## 📋 CHECKLIST TRƯỚC KHI QUAY

- [ ] Code editor đã mở các files cần thiết
- [ ] Terminal sẵn sàng ở thư mục `/BE`
- [ ] Test đã chạy thử và pass hết
- [ ] Server có thể start (python3 -m app.main)
- [ ] **Render deployment URL sẵn sàng** (để show live system)
- [ ] **Postman/curl command để test live API** (optional demo)
- [ ] Đã review script và timeline
- [ ] Camera/mic hoạt động tốt
- [ ] Screen resolution phù hợp (1920x1080 recommended)
- [ ] Font size trong editor đủ lớn để đọc (14-16pt)
- [ ] Dark/Light theme phù hợp với recording

---

## 📞 CONTACT & RESOURCES

**Project:** ASE-251  
**Branch:** ASE-RateLimit  
**Repository:** hunghehe2205/ASE-251

**Files liên quan:**

- `FEATURE_REQUIREMENTS.md` - Chi tiết requirements
- `BE/tests/test_auth_security.py` - Security tests
- `BE/tests/test_auth_safety.py` - Safety tests
- `BE/app/routers/auth.py` - Main authentication code

---

**Good luck with your presentation! 🎬🚀**
