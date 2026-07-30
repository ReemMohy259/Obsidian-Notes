==It is a web security attack where a malicious website tricks a user’s browser into sending unwanted requests to another website where the user is already logged in.==
#### Simple Example
Suppose you are logged into `https://bank.com` 
* Your browser stores a session cookie like `SESSIONID=abc123`
* Now you visit a malicious website `https://evil.com`
That site **secretly** sends this request:
```html
<form action="https://bank.com/transfer" method="POST">
    <input name="to" value="attacker">
    <input name="amount" value="10000">
</form>

<script>
    document.forms[0].submit();
</script>
```
##### What Happens
Your browser automatically includes the bank session cookie `Cookie: SESSIONID=abc123`
So the bank thinks *"This request came from the authenticated user."* and performs the transfer.

Browsers automatically send:
- cookies
- session IDs
with requests to the target domain. Attackers exploit this behavior.

**That is CSRF.**

---
### How Spring Security Prevents CSRF

Spring Security generates a **random token.**

Example:
```html
<input type="hidden" name="_csrf" value="a1b2c3">
```

When sending **POST/PUT/DELETE** requests:
- client must include correct CSRF token
- otherwise request is rejected


> [!NOTE] Note
> GET requests should only read data. no fear
> CSRF mainly targets data modification (POST, PUT, PATCH, DELETE)

---
### Why REST APIs Often Disable CSRF

**REST APIs usually:**
- are stateless
- use JWT tokens or Basic Auth
- don't rely on browser sessions/cookies

So CSRF protection is **unnecessary**.

That's why you often see `http.csrf(AbstractHttpConfigurer::disable);`

---
### Finally

| **Uses Cookies/Sessions?**   | **CSRF Needed?** |
| ---------------------------- | ---------------- |
| Traditional MVC web app      | Yes              |
| REST API with JWT            | Usually no       |
| REST API with HTTP Basic     | Usually no       |
| Session-based authentication | Yes              |

