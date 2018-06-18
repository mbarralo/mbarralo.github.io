---
layout: post
title:  Security - CSRF
tags: security
---

This will be the first post of a **Web Security Series**. To start this series let's address one of the known web security vulnerabilities - Cross Site Request Forgery, know simply as CSRF (or XSRF).

## CSRF

CSRF attacks exploit the fact that the browser sends cookies and authentication headers automatically to the target site.

Say for instance you have logged on to your banking site  - `my.bank`.

If a malicious site contains a form, like:

```html
<form action="https://my.bank/transfer" method="post">
    <input type="hidden" name="account" value="666"/>
    <input type="hidden" name="amount" value="1000"/>

    <button type="submit" value="win money">win money</button>
</form>
```

The browser will automatically send the cookies and header information it has to the target site.

Even with no session cookies or other (i.e. stateless rest services), **http basic authentication** is still an issue.


## Solution

Use a CSRF Synchronizer Token. This is something that it is impossible for the attacker site to know about. On **POST/PUT/DELETE** this will be added to the payload as an HTTP  parameter.

With spring-security module the protection is enabled by default.
We can customize.

#### Spring Config
```java
   // disable if needed
   .csrf().disable()

   // specify a custom/your own CSRF token provider
   // enable cookies for postman
   .csrf().csrfTokenRepository(new CookieCsrfTokenRepository())
```


#### Example Form
```html
    <input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}" />
```


### Caveats

* Beware *CORS* enablement. If the *CORS* is too relaxed can undermine *CSRf* protection.

* Even with stateless applications. If the server returns  stateless cookie (authentication) the browser will still send those as part of the request.
