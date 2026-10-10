# Brute Force Protection Bypass — Learning Notes

> **Lab environment:** Pikachu Web Vulnerability Testing Platform  
> **Purpose:** Educational testing in an authorized local environment only.

## 1. What I Wanted to Understand

When learning brute-force attacks, I initially thought the process was simply:

```text
Username + Password Dictionary
          ↓
      Send Requests
          ↓
     Find Correct Password
```

However, real login pages often introduce additional mechanisms such as:

- CSRF tokens
- CAPTCHA
- Session state
- Request validation
- Rate limiting

During my experiments, I encountered three interesting cases:

1. A CSRF token changes after every request.
2. CAPTCHA validation exists only on the client side.
3. CAPTCHA is validated by the server but can be reused.
