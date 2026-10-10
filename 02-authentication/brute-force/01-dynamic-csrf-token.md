# Dynamic CSRF Token During Brute Force

> **Lab environment:** Pikachu / DVWA-style authorized lab environment  
> **Purpose:** Educational testing in an authorized local environment only.

## 1. Observation

The login request contained both a password and a CSRF token.

For example:

```http
POST /login

username=admin
password=test123
token=8d12f...
```

During brute-force testing, the password needs to change for each attempt.

However, I noticed that the CSRF token also changed after each request.

This means that simply loading a password dictionary into Burp Intruder is not enough.

If the next request still uses an old token, the request may fail at the CSRF-token validation stage before the password is even checked.

---

## 2. Initial Hypothesis

The application appeared to follow a stateful process:

```text
Request 1
   ↓
Server validates Token A
   ↓
Server generates Token B
   ↓
Response 1 contains Token B
   ↓
Request 2 must use Token B
   ↓
Server generates Token C
   ↓
Response 2 contains Token C
```

This means that the next request depends on information returned by the previous response.

The requests are therefore not completely independent.

---

## 3. Why Normal Brute Force Fails

A normal brute-force process may look like this:

```text
Request 1 → password1
Request 2 → password2
Request 3 → password3
```

But with a dynamic CSRF token, the real process becomes:

```text
Request 1
password = password1
token = Token A
        ↓
Response contains Token B
        ↓
Request 2
password = password2
token = Token B
        ↓
Response contains Token C
        ↓
Request 3
password = password3
token = Token C
```

Therefore, two values are changing:

- the password;
- the CSRF token.

The password comes from a dictionary.

The token must be extracted from the previous response.

---

## 4. Testing with Burp Suite

In Burp Suite, the idea is to make the attack process preserve this state transition.

Conceptually:

```text
Response N
      │
      │ extract new token
      ▼
Token N+1
      │
      ▼
Request N+1
```

The password position uses a normal payload list.

The token position uses the newly extracted token from the previous server response.

This allows each new request to use the latest valid token.

---

## 5. Why Sequential Requests Matter

If the server invalidates or replaces the token after every request, sending multiple requests at the same time can cause problems.

For example:

```text
Request A ─┐
Request B ─┼──→ all depend on the same current token
Request C ─┘
```

If Request A is processed first, the server may generate a new token.

Requests B and C may then still contain the old token and fail.

Therefore, in this kind of stateful scenario, sequential requests are usually more appropriate than concurrent requests.

Conceptually:

```text
Request 1
   ↓
Response 1
   ↓
extract new token
   ↓
Request 2
   ↓
Response 2
   ↓
extract new token
   ↓
Request 3
```

---

## 6. What I Learned

Before this experiment, I tended to think of HTTP requests as separate and independent operations.

This experiment helped me understand that many Web applications maintain state across multiple requests.

The important idea is:

```text
Current request
      ↓
Changes server-side state
      ↓
Response contains new state
      ↓
Next request depends on it
```

A CSRF token is therefore not just another static parameter.

It may represent part of the application's current state.

---

## 7. Security Testing Insight

When automated testing fails, I should not immediately assume that the password dictionary or Burp configuration is wrong.

Instead, I should inspect whether the request contains values that change over time.

Examples include:

- CSRF tokens;
- session identifiers;
- nonces;
- one-time parameters;
- dynamically generated hidden fields.

The key question is:

> Does the next request depend on something returned by the previous response?

If the answer is yes, the testing process must preserve that state.

---

## 8. Key Takeaway

The main lesson from this case was not simply how to configure Burp Suite.

It was understanding that authentication requests can form a stateful sequence.

```text
Request
   ↓
Response
   ↓
Extract state
   ↓
Next request
```

This changed the way I think about automated Web security testing.
