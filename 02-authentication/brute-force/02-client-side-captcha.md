# CAPTCHA Validation Only on the Client Side

> **Lab environment:** Pikachu Web Vulnerability Testing Platform  
> **Purpose:** Educational testing in an authorized local environment only.

## 1. Observation

When testing the login page, I noticed an unusual behavior:

If I entered an incorrect CAPTCHA and clicked the login button, the page displayed a CAPTCHA error message, but Burp Suite captured no HTTP request.

The behavior looked like this:

```text
Enter incorrect CAPTCHA
        ↓
Click Login
        ↓
CAPTCHA error message appears
        ↓
No HTTP request is sent
```

This was different from what I initially expected.

I originally thought the browser would submit the request to the server first:

```text
Browser
   ↓
HTTP request
   ↓
Server validates CAPTCHA
   ↓
Server returns "CAPTCHA incorrect"
```

However, Burp showed that no request was generated.

---

## 2. Initial Hypothesis

Because no HTTP request was sent, I suspected that the CAPTCHA was being checked in the browser before form submission.

The likely process was:

```text
User clicks Login
        ↓
JavaScript validation
        ↓
Is CAPTCHA correct?
       / \
      /   \
    No     Yes
    ↓       ↓
return    return
false     true
    ↓       ↓
No HTTP   Submit form
request
```

This suggested that the CAPTCHA validation logic might be implemented in JavaScript on the client side.

---

## 3. Source Code Analysis

I then inspected the page source and found JavaScript similar to the following:

```javascript
function validate() {
    var inputCode =
        document.querySelector('#bf_client .vcode').value;

    if (inputCode.length <= 0) {
        alert("Please enter the CAPTCHA");
        return false;
    }

    if (inputCode != code) {
        alert("Incorrect CAPTCHA");
        createCode();
        return false;
    }

    return true;
}
```

The important part is:

```javascript
return false;
```

If the `validate()` function is called during form submission, returning `false` prevents the browser from submitting the form.

Conceptually:

```text
Incorrect CAPTCHA
       ↓
JavaScript detects the error
       ↓
return false
       ↓
Form submission is cancelled
       ↓
No HTTP request is generated
```

This matched the behavior I observed in Burp Suite.

---

## 4. How the CAPTCHA Is Generated

The page also contained JavaScript that generated the CAPTCHA value in the browser.

Conceptually:

```javascript
var code;

function createCode() {
    code = "";

    var codeLength = 5;

    for (var i = 0; i < codeLength; i++) {
        var charIndex = Math.floor(Math.random() * 36);
        code += selectChar[charIndex];
    }
}
```

This means the CAPTCHA value is generated and stored in a JavaScript variable on the client side.

The browser therefore knows both:

```text
Correct CAPTCHA value
        +
User input
```

and performs the comparison locally.

---

## 5. Why Client-Side Validation Is Not a Security Boundary

Client-side validation can improve user experience, but it should not be trusted as the only security control.

The browser normally follows the JavaScript logic:

```text
Browser UI
    ↓
JavaScript validation
    ↓
HTTP request
    ↓
Server
```

However, the server only receives HTTP requests.

A request created manually by a testing tool does not need to follow the page's JavaScript logic.

Therefore, if the server does not independently validate the CAPTCHA, the client-side check alone cannot enforce the security rule.

The important trust boundary is:

```text
Untrusted client
      ↓
HTTP request
      ↓
Trusted server-side validation
```

Security decisions should be enforced on the server side.

---

## 6. A More Careful Conclusion

One important lesson from this experiment was that:

```text
No HTTP request
```

does **not automatically prove** that the server has no CAPTCHA validation.

What it proves is:

> This particular failed CAPTCHA attempt was blocked by client-side logic before the request was sent.

To determine whether the server also performs CAPTCHA validation, I would need to send a request that bypasses the client-side JavaScript and then observe the server response.

Therefore, the reasoning process is:

```text
Incorrect CAPTCHA produces no request
        ↓
Client-side validation exists
        ↓
Inspect JavaScript
        ↓
Confirm form submission is blocked
        ↓
Send controlled request directly
        ↓
Check whether server also validates CAPTCHA
```

This distinction is important because a secure application may perform both:

```text
Client-side validation
        +
Server-side validation
```

---

## 7. How I Located the Relevant JavaScript

At first, I often tried to read the entire page source line by line.

This was inefficient.

A better approach is to search for functionality-related keywords.

For CAPTCHA validation, useful keywords include:

```text
validate
captcha
vcode
checkCode
createCode
```

In browser DevTools, useful locations include:

```text
DevTools
├── Sources
├── Elements
├── Network
└── Event Listeners
```

Using global search in DevTools makes it much easier to locate the relevant JavaScript.

---

## 8. What I Learned

This experiment helped me connect browser behavior, JavaScript, and HTTP requests.

My reasoning process was:

```text
Observe browser behavior
        ↓
Check Burp traffic
        ↓
Notice no request was sent
        ↓
Form a client-side validation hypothesis
        ↓
Inspect JavaScript
        ↓
Confirm the hypothesis
```

The key lesson was:

> Network behavior can help reveal where validation is taking place.

If a user action produces no HTTP request, I should first investigate browser-side logic.

If a request reaches the server and is rejected there, I should investigate server-side validation.

---

## 9. Key Takeaway

Before this experiment, I mainly focused on whether a CAPTCHA existed.

Now I ask a more useful question:

> Where is the CAPTCHA actually validated?

That leads to a better testing workflow:

```text
Observe
   ↓
Capture traffic
   ↓
Locate validation logic
   ↓
Identify the trust boundary
   ↓
Verify server-side enforcement
```

This was an important step in moving from simply using security tools to understanding how browser-side and server-side logic interact.
