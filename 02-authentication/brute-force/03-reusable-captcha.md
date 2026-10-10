# Server-Side CAPTCHA Validation but CAPTCHA Reuse

> **Lab environment:** Pikachu Web Vulnerability Testing Platform  
> **Purpose:** Educational testing in an authorized local environment only.

## 1. Observation

In this scenario, the CAPTCHA appeared to be validated by the server.

However, I suspected that a valid CAPTCHA might remain usable after one successful verification.

In other words, the server might verify the CAPTCHA correctly but fail to invalidate it immediately.

Conceptually:

```text
CAPTCHA = A7K2P

Request 1
Correct CAPTCHA + Wrong Password
        ↓
Password error

Request 2
Same CAPTCHA + Different Wrong Password
        ↓
Password error again
```

If the same CAPTCHA continues to pass verification, this suggests that the CAPTCHA is reusable.

---

## 2. Initial Hypothesis

My hypothesis was:

> The server validates the CAPTCHA, but the CAPTCHA is not invalidated immediately after successful verification.

This is different from a client-side CAPTCHA issue.

In this case, the request does reach the server.

The question is not:

> "Is the CAPTCHA validated?"

Instead, the question is:

> "What happens to the CAPTCHA after it has been successfully validated?"

---

## 3. Black-Box Testing Without Source Code

Source code is not required to test this behavior.

I can infer the server-side logic by replaying controlled requests and comparing responses.

The important variables are:

```text
Session
CAPTCHA
Password
```

To test CAPTCHA reuse correctly, I need to keep the same session while changing only the password.

---

## 4. First Request

I first submit:

```text
Correct CAPTCHA
+
Incorrect Password A
```

For example:

```http
POST /login

username=admin
password=wrongA
captcha=A7K2P
```

Suppose the response is:

```text
Incorrect username or password
```

This response is important.

It suggests that the CAPTCHA was accepted and the request reached the credential-validation stage.

Conceptually:

```text
Request
   ↓
CAPTCHA check
   ↓
CAPTCHA correct
   ↓
Password check
   ↓
Password incorrect
```

---

## 5. Replay the Same CAPTCHA

Next, I keep:

```text
Same session
Same CAPTCHA
```

but change the password:

```http
POST /login

username=admin
password=wrongB
captcha=A7K2P
```

There are two important possible results.

### Result A — CAPTCHA Can Be Reused

If the server again responds with:

```text
Incorrect username or password
```

then the request appears to have passed the CAPTCHA-validation stage again.

This suggests that the same CAPTCHA value is still valid.

Conceptually:

```text
CAPTCHA A7K2P
   ↓
Request 1 → accepted
   ↓
Request 2 → accepted
   ↓
Request 3 → accepted
```

This is evidence that the CAPTCHA can be reused.

---

### Result B — CAPTCHA Was Invalidated

If the second request returns:

```text
Incorrect CAPTCHA
```

then the server likely invalidated or rotated the CAPTCHA after the first successful verification.

Conceptually:

```text
CAPTCHA A7K2P
   ↓
Request 1 → accepted
   ↓
CAPTCHA invalidated
   ↓
Request 2 → rejected
```

---

## 6. Control Experiment

To make the conclusion stronger, I can compare several requests.

| Request | Session | CAPTCHA | Password | Expected Observation |
|---|---|---|---|---|
| 1 | Same | Correct X | Wrong A | Password error |
| 2 | Same | Correct X | Wrong B | Password error if reusable |
| 3 | Same | Correct X | Wrong C | Password error if reusable |
| 4 | Same | Incorrect Y | Wrong D | CAPTCHA error |

If Requests 1–3 all reach password validation, while Request 4 fails at CAPTCHA validation, this provides stronger evidence that:

> The server validates the CAPTCHA, but the valid CAPTCHA is reusable.

---

## 7. Why the Session Must Stay the Same

CAPTCHA values are often associated with server-side session state.

For example:

```text
Session ID
   ↓
Server-side session
   ↓
Stored CAPTCHA value
```

Conceptually:

```text
PHPSESSID = abc123
        ↓
Server session
        ↓
captcha = A7K2P
```

If I change the session cookie between requests, I may no longer be testing the same CAPTCHA state.

Therefore, during this experiment, I should keep the same session identifier.

For example:

```http
Cookie: PHPSESSID=abc123
```

should remain unchanged while testing CAPTCHA reuse.

---

## 8. Why Error Messages Matter

Different error messages can help reveal which part of the server-side logic was reached.

The application may conceptually behave like this:

```text
Request
   ↓
Is CAPTCHA correct?
   │
 ┌─┴─────────────┐
No               Yes
│                 │
CAPTCHA error     ↓
              Check credentials
                 │
              ┌──┴──┐
             No     Yes
             │       │
         Password   Login
          error     success
```

Therefore:

```text
CAPTCHA error
```

and:

```text
Password error
```

may indicate different execution paths.

This makes response messages useful as black-box signals.

---

## 9. A More Precise Conclusion

If the same CAPTCHA repeatedly produces a password error rather than a CAPTCHA error, I should not simply conclude:

> "CAPTCHA is broken."

A more precise conclusion is:

> The server-side CAPTCHA validation exists, but the CAPTCHA lifecycle may not enforce one-time use.

The issue is therefore related to:

```text
CAPTCHA lifecycle
        ↓
Validation
        ↓
Invalidation / rotation
```

rather than the complete absence of CAPTCHA validation.

---

## 10. Possible Server-Side Logic

A weak implementation may conceptually behave like this:

```text
If CAPTCHA is correct:
    Check username and password

If login succeeds:
    Generate a new CAPTCHA
```

In this design, a correct CAPTCHA may remain valid after failed password attempts.

A stronger design may instead follow:

```text
Receive CAPTCHA answer
        ↓
Verify CAPTCHA
        ↓
Consume / invalidate challenge
        ↓
Continue authentication logic
```

The exact implementation depends on the application's design, but the important security concept is that the challenge lifecycle should be intentionally controlled.

---

## 11. Page Refresh Can Change the State

One detail that can interfere with testing is refreshing the CAPTCHA image or login page.

The CAPTCHA image may be loaded from a separate endpoint.

For example:

```text
/inc/showvcode.php
```

Loading this endpoint may generate a new CAPTCHA and update the value stored in the current session.

Conceptually:

```text
Refresh CAPTCHA image
        ↓
Browser sends new request
        ↓
Server generates new CAPTCHA
        ↓
Session CAPTCHA value changes
```

Therefore, during testing, I should distinguish between:

```text
Replaying an existing request
```

and:

```text
Refreshing the browser page
```

because refreshing may silently change the server-side state.

---

## 12. What I Learned

This experiment helped me understand that validating a security control is not only about checking whether the control exists.

I also need to understand its lifecycle.

My reasoning process became:

```text
Does the server validate it?
        ↓
Is it bound to the session?
        ↓
Can it be reused?
        ↓
When does it expire?
        ↓
When is it invalidated?
```

This applies not only to CAPTCHA values, but also to other stateful security mechanisms.

---

## 13. Key Takeaway

The most important lesson from this case was:

> A security value can be validated correctly and still be weak if its lifecycle is poorly controlled.

For CAPTCHA testing, I should therefore ask:

- Is the CAPTCHA validated by the server?
- Is it bound to the current session?
- Can the same CAPTCHA be reused?
- Is it invalidated after successful verification?
- Does refreshing the page generate a new value?
- Does it expire after a period of time?

This experiment helped me move from simply testing input values to thinking about server-side state and challenge lifecycle.
