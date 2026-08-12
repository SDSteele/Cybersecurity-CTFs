# HackThisSite — Basic Mission 4 Write-Up

### Challenge: Basic Mission 4

**Platform:** HackThisSite
**Category:** Web / Parameter Manipulation / Input Validation
**Difficulty:** Basic

---

## Objective

The objective of Basic Mission 4 is to obtain Network Security Sam's password.

The challenge explains that Sam hardcoded his password into a script but created a password-reminder function that would email the password to him whenever he forgot it.

The important question becomes:

> **Does the server actually enforce who the password can be sent to, or does it simply trust the recipient supplied by the browser?**

That is what this challenge is designed to teach.

---

# Step 1 — Read the Challenge

The page provides a button similar to:

```text
Send password to Sam
```

At first glance, it appears that clicking this button should simply send Sam his password.

However, whenever I encounter a form that performs an interesting action, I want to know:

* Where is the request being sent?
* What HTTP method is being used?
* What parameters are being submitted?
* Are any parameters hidden from the normal interface?
* Can those parameters be modified?

This leads directly to inspecting the page source.

---

# Step 2 — Inspect the HTML Source

Looking at the source, the password-reminder form contains:

```html
<form action="/missions/basic/4/level4.php" method="post">
    <input type="hidden" name="to" value="sam@hackthissite.org">
    <input type="submit" value="Send password to Sam">
</form>
```

The important part is:

```html
<input type="hidden" name="to" value="sam@hackthissite.org">
```

The `to` parameter determines the recipient of the password.

The field is marked:

```text
type="hidden"
```

which means it isn't displayed as an editable field in the normal webpage.

But **hidden does not mean server-controlled or secure**.

The browser still has the value.

---

# Step 3 — Identify the Vulnerable Parameter

The form submits a POST request to:

```text
/missions/basic/4/level4.php
```

with a parameter named:

```text
to
```

and its default value is:

```text
sam@hackthissite.org
```

So the request conceptually looks like:

```text
POST /missions/basic/4/level4.php

to=sam@hackthissite.org
```

The application appears to be trusting the value supplied by the client.

That immediately gives us something to test:

> What happens if I change `to` to an email address I control?

---

# Step 4 — Modify the Hidden Field

There are several ways to do this.

One simple approach is to use the browser's developer tools and modify the HTML.

The original field:

```html
<input type="hidden" name="to" value="sam@hackthissite.org">
```

can temporarily be changed so that the recipient is my own email address.

For example:

```html
<input type="text" name="to" value="MY_EMAIL_ADDRESS">
```

The important part isn't actually changing `hidden` to `text`.

The important part is changing the value of:

```text
to
```

to an email address I control.

Another approach would be to intercept the POST request with a proxy such as Burp Suite and modify the parameter before allowing the request to reach the server.

---

# Step 5 — Submit the Request

After changing the recipient, I submit the form.

The browser sends the modified request to:

```text
/missions/basic/4/level4.php
```

The server processes the supplied `to` parameter.

Instead of sending the password to:

```text
sam@hackthissite.org
```

the password is sent to the address I supplied.

I then check the mailbox associated with the address I used.

The resulting email contains Sam's password.

I can use that password to complete Basic Mission 4.

---

# The Vulnerability

The fundamental vulnerability is **trusting client-controlled input**.

The application assumes that the recipient specified by the browser is trustworthy.

But the browser is controlled by the user.

Anything in the HTML can be modified before it is submitted.

For example:

```html
<input type="hidden" name="to" value="sam@hackthissite.org">
```

does **not** mean:

```text
Only Sam can receive this.
```

It actually means:

```text
The browser is currently going to submit this value.
```

There is a huge security difference between those two concepts.

---

# Why `type="hidden"` Doesn't Provide Security

This is one of the most important lessons from the challenge.

HTML hidden fields are useful for passing information between pages, but they provide **no meaningful access control**.

A user can:

1. View the source.
2. Open developer tools.
3. Modify the DOM.
4. Intercept the HTTP request.
5. Construct their own HTTP request entirely.

For example, an attacker doesn't necessarily need the webpage at all.

They could conceptually submit:

```http
POST /missions/basic/4/level4.php
Host: example.com
Content-Type: application/x-www-form-urlencoded

to=attacker@example.com
```

If the server accepts that value without additional authorization checks, the application has a vulnerability.

---

# Security Impact

In the context of this challenge, the impact is simply obtaining the password.

In a real application, the same design mistake could have much more serious consequences.

Imagine a password-reset system containing:

```html
<input type="hidden" name="email" value="victim@example.com">
```

If the backend trusts that field without validating the authenticated user's authority, an attacker may be able to manipulate the recipient and potentially redirect sensitive information.

This general class of problem is related to **parameter tampering** and **client-side trust**.

---

# How a Secure Application Should Handle This

Sensitive decisions should be made server-side.

The application should not trust a recipient merely because the browser supplied it.

Instead, the server should determine:

```text
Who is authenticated?
        ↓
What account are they authorized to access?
        ↓
What email address is associated with that account?
        ↓
Send the sensitive information only there
```

If a recipient must be supplied by the user, the server should validate that the user is authorized to send the information to that recipient.

---

# Methodology

The useful methodology from this challenge is:

```text
Observe the application
        ↓
Identify interesting functionality
        ↓
Inspect the HTML
        ↓
Find hidden/client-controlled parameters
        ↓
Determine what the parameter controls
        ↓
Ask whether the server validates it
        ↓
Modify the parameter
        ↓
Submit the request
        ↓
Observe the result
```

This is a much more transferable skill than simply memorizing the solution.

---

# Key Takeaways

### 1. Never assume hidden HTML is secure

```html
type="hidden"
```

only controls how the browser displays the field.

It does not prevent the user from changing it.

---

### 2. The client is untrusted

Anything sent by the browser should be considered potentially attacker-controlled.

That includes:

* Hidden form fields
* Cookies
* URL parameters
* POST parameters
* JavaScript variables
* Form values
* HTTP headers

---

### 3. Inspect forms

When testing a web application, pay attention to:

```html
<form action="..." method="...">
```

and especially:

```html
<input name="..." value="...">
```

These can reveal how the application expects requests to be constructed.

---

### 4. Ask what the parameter actually controls

Finding:

```text
to=sam@hackthissite.org
```

isn't enough.

The important realization is:

> **The `to` parameter controls where a sensitive credential is sent.**

That makes it an interesting security boundary to test.

---

# Basic 3 → Basic 4 Progression

These two challenges are worth remembering together.

### Basic 3

The source revealed:

```text
password.php
```

The mistake was essentially:

> **Sensitive resource exposed and directly accessible.**

The approach was:

```text
Inspect source
      ↓
Find filename
      ↓
Request file
      ↓
Retrieve password
```

### Basic 4

The source revealed:

```text
to=sam@hackthissite.org
```

The mistake was:

> **The server trusted a client-controlled parameter.**

The approach was:

```text
Inspect source
      ↓
Find hidden parameter
      ↓
Understand what it controls
      ↓
Modify parameter
      ↓
Submit request
      ↓
Receive password
```

This is an important progression because I'm beginning to move from **finding information** to **manipulating application behavior**.

---

## Skills Practiced

* HTML source-code inspection
* Understanding HTML forms
* Identifying hidden form fields
* HTTP POST request concepts
* Client-side vs. server-side trust
* Parameter manipulation
* Basic web application enumeration
* Understanding insecure direct trust of user input
* Thinking about authorization boundaries

---

## Final Lesson

The biggest lesson from Basic 4 is:

> **Never confuse a client-side restriction with a security control.**

The browser is not a trusted environment.

If a value matters to security, authorization, or the handling of sensitive information, the server must validate it.

Basic 4 demonstrates this with an intentionally simple example: the webpage hides the email recipient, but the server accepts whatever recipient the client provides.

That is the mindset I want to carry into more advanced web challenges:

**Don't just ask what the webpage allows me to do. Ask what the server will actually accept.**
