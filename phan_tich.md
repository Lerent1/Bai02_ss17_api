1. Mục đích của JwtAuthenticationFilter

JwtAuthenticationFilter có nhiệm vụ:

Lấy JWT từ Header Authorization.
Kiểm tra tính hợp lệ của JWT.
Trích xuất username từ JWT.
Tải thông tin người dùng từ UserDetailsService.
Tạo Authentication.
Đưa Authentication vào SecurityContextHolder.

Sau khi hoàn thành các bước trên, Spring Security mới xem người dùng là đã đăng nhập.

2. Luồng xử lý hiện tại

Đoạn mã hiện tại:

String jwt = getJwtFromRequest(request);

if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt, null)) {
String username = tokenProvider.getUsernameFromJWT(jwt);

    UserDetails userDetails =
            userDetailsService.loadUserByUsername(username);
}
3. Điểm không nhất quán

Theo mô hình JWT trong Lesson 3, phương thức:

boolean validateToken(String token, UserDetails userDetails)

thường được triển khai như sau:

public boolean validateToken(String token, UserDetails userDetails) {

    final String username = extractUsername(token);

    return username.equals(userDetails.getUsername())
            && !isTokenExpired(token);
}

Phương thức này cần:

JWT
UserDetails

để kiểm tra:

username.equals(userDetails.getUsername())
4. Lỗi trong JwtAuthenticationFilter

Hiện tại filter gọi:

tokenProvider.validateToken(jwt, null)

Tức là:

userDetails = null

Khi chạy:

userDetails.getUsername()

sẽ gây:

NullPointerException

hoặc

validateToken() trả về false
5. Hậu quả

Điều kiện:

if (StringUtils.hasText(jwt)
&& tokenProvider.validateToken(jwt, null))

luôn thất bại.

Do đó:

String username = tokenProvider.getUsernameFromJWT(jwt);

không được thực thi.

Tiếp theo:

UserDetails userDetails =
userDetailsService.loadUserByUsername(username);

không được gọi.

Và:

SecurityContextHolder.getContext()

không bao giờ nhận Authentication.

Spring Security xem request là:

Unauthenticated Request

nên trả về:

403 Forbidden

hoặc

401 Unauthorized
6. Nguyên nhân gốc rễ

Sai thứ tự xử lý:

Đang làm
JWT
↓
validateToken(jwt, null)
↓
Load UserDetails
Đúng phải là
JWT
↓
Lấy username
↓
Load UserDetails
↓
validateToken(jwt, userDetails)
↓
Tạo Authentication
↓
SecurityContextHolder
7. Kết luận

Lỗi xảy ra vì:

tokenProvider.validateToken(jwt, null)

được gọi trước khi tải UserDetails.

Trong khi validateToken() cần UserDetails để kiểm tra username và trạng thái token.

Kết quả là JWT không thể được xác thực hợp lệ, Authentication không được tạo và Spring Security từ chối truy cập các API bảo vệ.