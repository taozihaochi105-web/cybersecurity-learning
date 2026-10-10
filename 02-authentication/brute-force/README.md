Brute Force Protection Bypass — Learning Notes
Lab environment: Pikachu Web Vulnerability Testing Platform
Purpose: Educational testing in an authorized local environment only.

1. What I wanted to understand
When learning brute-force attacks, I initially thought the process was simply:
Username + Password Dictionary
          ↓
      Send Requests
          ↓
     Find Correct Password

However, real login pages often introduce additional mechanisms such as:
CSRF Token
CAPTCHA
Session
Request validation
Rate limiting

These mechanisms change the way automated requests work.
During my experiments, I encountered three interesting cases:
1. A CSRF token changes after every request
2. CAPTCHA validation exists only on the client side
3. CAPTCHA is validated by the server but can be reused

The important part of these experiments was not simply bypassing the protection.
My goal was to understand:
Where is the validation performed, what state does the server maintain, and how can network behavior reveal the application's internal logic?
