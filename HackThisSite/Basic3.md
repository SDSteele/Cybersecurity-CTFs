# HackThisSite — Basic Mission 3 Write-Up

### Challenge: Basic Mission 3

**Platform:** HackThisSite
**Category:** Web / Information Disclosure
**Difficulty:** Basic

---

## Objective

The goal of this mission is to find the password that Network Security Sam has uploaded to the website.

The challenge description essentially hints that Sam remembered to upload the password file, but something is still wrong.

The important clue is that the password isn't necessarily on the visible webpage. We need to inspect what the application is exposing to the browser.

---

## Step 1 — Inspect the Page Source

I started by viewing the HTML source of the challenge page.

The important thing here is to remember that **View Source shows the HTML the server sent to the browser**, including things that may not be visible on the rendered webpage.

While examining the source, I found a hidden form field referencing:

```html
<input type="hidden" name="file" value="password.php" />
```

This is the important discovery.

The browser doesn't display this field to the user, but the filename is still being sent to the client.

So now I know there is a file called:

```text
password.php
```

The official challenge page is under:

```text
/missions/basic/3/index.php
```

---

## Step 2 — Think About the Application Structure

At this point, rather than trying to guess the password, I should ask:

> "If the application is telling me the name of the password file, can I request that file directly?"

The challenge is essentially giving away the filename.

So I changed the URL from:

```text
https://www.hackthissite.org/missions/basic/3/index.php
```

to:

```text
https://www.hackthissite.org/missions/basic/3/password.php
```

This directly requests the file referenced by the hidden form field.

---

## Step 3 — Retrieve the Password

The `password.php` file responds with the password.

I can then copy that value and submit it to the Basic Mission 3 password form.

Mission complete.

---

# What I Learned

The main lesson here isn't really PHP exploitation. It's **information disclosure**.

The application exposed the name of a sensitive server-side file through HTML:

```html
value="password.php"
```

Even though the field was hidden from the normal webpage, **hidden HTML is not secret**.

Anything sent to the browser can potentially be inspected by the user.

This is an important distinction:

### Hidden ≠ Protected

For example:

```html
<input type="hidden" value="secret.txt">
```

doesn't actually protect `secret.txt`.

The browser still receives:

```text
secret.txt
```

A user can inspect the source and potentially request that resource directly.

---

# Security Lesson

A real application should never rely on:

* Hidden HTML fields
* Obscure filenames
* Unlinked files
* Client-side code
* Comments in HTML
* JavaScript variables

to protect sensitive information.

If a file contains credentials or other secrets, the server should enforce authorization before allowing the file to be retrieved.

In this challenge, the fundamental problem is that the password file is both **discoverable** and **directly accessible**.

---

## The Thought Process

The useful methodology to remember is:

```text
Look at the webpage
       ↓
Inspect the source
       ↓
Look for hidden fields / comments / filenames
       ↓
Find "password.php"
       ↓
Ask: "Can I request that file directly?"
       ↓
/missions/basic/3/password.php
       ↓
Retrieve password
       ↓
Submit password
```

### Key takeaway

**When doing web enumeration, don't only look at what the webpage displays. Look at what the application gives the browser.**

Source code, hidden fields, comments, JavaScript, filenames, endpoints, and other client-side information can reveal how the application is structured and point toward resources that aren't linked from the visible page.

That habit becomes much more important in later CTFs and real penetration tests.
