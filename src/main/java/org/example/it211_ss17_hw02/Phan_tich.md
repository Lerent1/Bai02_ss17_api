# Phần 1 - Phân tích logic

Vấn đề nằm ở cách `JwtAuthenticationFilter` gọi phương thức `validateToken()` của `JwtTokenProvider`.

Trong đoạn mã hiện tại:

```java
//if (StringUtils.hasText(jwt) && tokenProvider.validateToken(jwt, null)) {
```

tham số thứ hai được truyền vào là `null`.

Tuy nhiên, theo cách triển khai phổ biến của `JwtTokenProvider`, phương thức `validateToken()` thường có dạng:

```java
public boolean validateToken(String token, UserDetails userDetails) {
    final String username = getUsernameFromJWT(token);
    return username.equals(userDetails.getUsername())
            && !isTokenExpired(token);
}
```

Phương thức này cần đối tượng `UserDetails` để:

- Lấy username của người dùng từ hệ thống.
- So sánh username trong JWT với username thực tế.
- Kiểm tra token có hết hạn hay không.

Khi truyền `null`:

```java
//tokenProvider.validateToken(jwt, null)
```

thì câu lệnh:

```java
//userDetails.getUsername()
```

sẽ không thể thực hiện được, dẫn đến:

- Phát sinh `NullPointerException`, hoặc
- Phương thức trả về `false`.

Kết quả là điều kiện `if` không được thỏa mãn và đoạn mã tải thông tin người dùng sẽ không được thực thi:

```java
UserDetails userDetails = userDetailsService.loadUserByUsername(username);
```

Do đó:

- Authentication không được tạo.
- `SecurityContextHolder` không được cập nhật.
- Spring Security coi request là chưa xác thực.
- Các API được bảo vệ trả về lỗi `403 Forbidden` hoặc `401 Unauthorized`.

## Kết luận

Nguyên nhân lỗi là do `JwtAuthenticationFilter` gọi:

```java
//tokenProvider.validateToken(jwt, null)
```

trong khi `validateToken()` cần một đối tượng `UserDetails` hợp lệ để kiểm tra username và trạng thái của token.
Cần tải `UserDetails` trước, sau đó mới gọi `validateToken(jwt, userDetails)` và thiết lập `Authentication` vào `SecurityContextHolder`.