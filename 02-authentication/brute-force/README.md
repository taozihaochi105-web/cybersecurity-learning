# Brute Force Protection Bypass — Learning Notes

> **Lab environment:** Pikachu Web Vulnerability Testing Platform  
> **Purpose:** Educational testing in an authorized local environment only.

## Overview

During my learning, I encountered three different cases involving brute-force protection mechanisms.

## Cases

1. [Dynamic CSRF Token During Brute Force](./01-dynamic-csrf-token.md)
2. [CAPTCHA Validation Only on the Client Side](./02-client-side-captcha.md)
3. [Server-Side CAPTCHA Validation but CAPTCHA Reuse](./03-reusable-captcha.md)

## Key Questions I Learned to Ask

- Where is the validation performed?
- Is the control enforced on the client or server?
- Is the value bound to the session?
- Does the value change after each request?
- Can the same value be reused?
- What server-side state changes after each request?

## Key Takeaway

The important part of brute-force testing is not simply sending many password attempts.

I need to understand how the application manages state, tokens, CAPTCHA values, sessions, and validation logic across requests.
