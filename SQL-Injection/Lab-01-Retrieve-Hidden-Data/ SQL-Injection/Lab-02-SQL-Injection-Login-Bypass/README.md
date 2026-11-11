
# Lab 02 — SQL Injection: Subverting Application Logic (Login Bypass)

![Platform](https://img.shields.io/badge/Platform-PortSwigger%20Web%20Security%20Academy-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice-green)
![Vulnerability](https://img.shields.io/badge/Vulnerability-SQL%20Injection-blue)
![Status](https://img.shields.io/badge/Status-Solved-brightgreen)

**Lab:** [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)  
**Tools:** Firefox, Firefox Developer Tools, Burp Suite (Proxy, HTTP History, Repeater)  
**Vulnerability Classification:** CWE-89 — SQL Injection  
**Testing Environment:** PortSwigger Web Security Academy

> **Disclaimer:** This write-up documents an authorized exercise conducted in an intentionally vulnerable training environment. Only test systems you own or have explicit permission to assess.

---

## 1. Objective

The application contains a SQL injection vulnerability in its login functionality.

The objective was to exploit the `username` parameter to authenticate as the **administrator** without knowing the account's password.

I explored four workflows:

1. Direct login form manipulation using Firefox.
2. HTTP request interception using Burp Suite Proxy.
3. Request modification and replay using Burp Suite Repeater.
4. Authenticated session-cookie transfer using Burp Suite and Firefox Developer Tools.

**Important:** The first three workflows deliver the same SQL injection payload. The fourth demonstrates reuse of an authenticated session created after the login bypass.

### Lab Scope

| Attribute | Details |
|---|---|
| Target endpoint | `/login` |
| HTTP method | POST |
| Vulnerable parameter | `username` |
| Injection technique | SQL string termination and comment |
| Target account | `administrator` |
| Expected outcome | Authentication without a valid password |
| Lab status | Solved |

---

## 2. Vulnerability Analysis

### 2.1 Identifying the Injection Point

The application provides a login form that accepts a username and password.

Submitting the form generates an HTTP request similar to:

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

csrf=<valid-token>&username=administrator&password=abc
```

The request contains user-controlled login parameters and a CSRF token.

### 2.2 Understanding the SQL Query

According to the lab scenario, the application uses a query equivalent to:

```sql
SELECT * FROM users
WHERE username = 'administrator'
AND password = 'abc';
```

Under normal conditions, both the username and password must match an existing user record.

The vulnerability occurs because user-controlled input is incorporated into SQL without proper parameterization.

### 2.3 Root Cause

**Improper separation of SQL instructions and user-supplied data.**

The application allows SQL syntax entered through the `username` parameter to modify the authentication query.

This enables an attacker to bypass the password-verification condition.

---

## 3. SQL Injection Payload

### Payload Used

```text
administrator'--
```

### Login Values

| Field | Input |
|---|---|
| Username | `administrator'--` |
| Password | `a` |

The password `a` is an arbitrary value. It is not the administrator's actual password.

### Payload Breakdown

| Component | Purpose |
|---|---|
| `administrator` | Specifies the target username |
| `'` | Closes the SQL string literal |
| `--` | Comments out the remaining SQL query |

### Modified SQL Query

```sql
SELECT * FROM users
WHERE username = 'administrator'--' AND password = 'a';
```

The database effectively evaluates:

```sql
SELECT * FROM users
WHERE username = 'administrator';
```

### How the Injection Works

1. The single quote terminates the original username string.
2. The SQL comment sequence `--` causes the remaining password condition to be ignored.
3. The query retrieves the administrator's user record without verifying the supplied password.
4. The vulnerable application treats the returned record as successful authentication.

**Result:** The attacker-controlled request bypasses the password check and authenticates as the administrator.

---

## 4. Method 1 — Direct Login Form Manipulation

### Procedure

**Step 1: Open the login page**

Opened the PortSwigger lab in Firefox and navigated to the login page.

**Step 2: Enter the SQL injection payload**

Entered the following values:

```text
Username: administrator'--
Password: a
```

**Step 3: Submit the login form**

Clicked **Log in** to submit the credentials.

**Step 4: Verify the result**

The application redirected to the administrator account page.

The page displayed:

```text
Your username is: administrator
```

The PortSwigger lab also displayed the **Solved** status.

### Result

Successfully bypassed the application's password-verification condition directly through the browser login form.

This demonstrated that the vulnerability could be exploited without using a dedicated HTTP interception tool.

### Screenshot Evidence

**Figure 1: Login form containing the SQL injection payload.**

![Login form with SQL injection payload](screenshots/01-browser-login.png)

**Figure 2: Administrator account page confirming successful login bypass.**

![Administrator login successful](screenshots/02-browser-solved.png)

---

## 5. Method 2 — Burp Suite Proxy Interception

Burp Suite Proxy was used to inspect and modify the login request before forwarding it to the application.

### Procedure

**Step 1: Enable interception**

Configured the browser to route traffic through Burp Suite.

Navigated to:

**Proxy → Intercept → Intercept On**

**Step 2: Capture the login request**

Submitted the login form to capture the HTTP request.

The intercepted request used the `POST` method and targeted `/login`.

**Step 3: Inspect the request parameters**

Opened the **Params** tab.

The request contained:

- `session` — Session cookie
- `csrf` — CSRF protection token
- `username` — Login username
- `password` — Login password

**Step 4: Modify the username parameter**

Set the username parameter to:

```text
administrator'--
```

Used a dummy password value while preserving the valid CSRF token and session cookie.

**Step 5: Forward the request**

Clicked **Forward** to send the modified request to the application.

**Step 6: Verify authentication**

The browser reached the administrator account page and displayed the lab's **Solved** status.

### Result

Successfully reproduced the SQL injection vulnerability through live HTTP request interception.

### Technical Observation

The CSRF token and session cookie remained valid during the test.

CSRF protection prevents a different class of attack and does not prevent SQL injection within submitted form parameters.

### Screenshot Evidence

**Figure 3: Intercepted login request showing the username and password parameters.**

![Burp Proxy intercepted request](screenshots/03-proxy-intercept.png)

**Figure 4: SQL injection payload in the intercepted username parameter.**

![Modified username in Burp Proxy](screenshots/04-proxy-payload.png)

**Figure 5: Successful administrator authentication after forwarding the request.**

![Administrator login through Proxy](screenshots/05-proxy-solved.png)

---

## 6. Method 3 — Burp Suite Repeater

Burp Suite Repeater was used to modify and replay the login request while examining the server's response.

### Procedure

**Step 1: Capture the login request**

Opened **Proxy → HTTP History** and located the `POST /login` request.

**Step 2: Send the request to Repeater**

Right-clicked the request and selected **Send to Repeater**.

**Step 3: Modify the request parameters**

Changed the username parameter to:

```text
administrator'--
```

Kept a dummy password value.

An example URL-encoded request body is:

```text
csrf=<valid-token>&username=administrator%27--&password=a
```

Here, `%27` represents the single quote character.

**Step 4: Send the modified request**

Clicked **Send/Go** in Burp Repeater.

**Step 5: Analyze the response**

The server returned a response similar to:

```http
HTTP/1.1 302 Found
Location: /my-account?id=administrator
Set-Cookie: session=<redacted>
```

The redirect to the administrator account and the newly issued session cookie were strong indicators of successful authentication.

**Step 6: Verify the result**

Accessed the administrator account page using the authenticated session.

The application displayed the administrator username, and the lab status was **Solved**.

### Result

Successfully reproduced the authentication bypass and analyzed the response using Burp Suite Repeater.

### Technical Observation

A `302 Found` redirect and a new session cookie are useful authentication indicators, but they do not universally prove successful login.

The administrator account page and the lab's **Solved** status provided confirmation.

### Screenshot Evidence

**Figure 6: Original login request sent to Burp Suite Repeater.**

![Original login request in Repeater](screenshots/06-repeater-original.png)

**Figure 7: Modified login request containing the SQL injection payload.**

![Injected login request in Repeater](screenshots/07-repeater-payload.png)

**Figure 8: HTTP response showing the administrator redirect and session-cookie issuance.**

![Repeater authentication response](screenshots/08-repeater-response.png)

---

## 7. Method 4 — Authenticated Session-Cookie Transfer

This workflow explored how an authenticated session obtained through Burp Suite could be reused in Firefox.

It was not a separate SQL injection technique.

### Procedure

**Step 1: Obtain an authenticated session**

Submitted the SQL injection payload through Burp Suite Repeater.

The successful authentication response included a session cookie:

```http
Set-Cookie: session=<authenticated-session-value>
```

**Step 2: Open Firefox Developer Tools**

Opened the lab website in Firefox and pressed **F12**.

Navigated to:

**Storage → Cookies → Lab Domain**

**Step 3: Locate the session cookie**

Identified the cookie named:

```text
session
```

The browser displayed its value and attributes, including `HttpOnly`, `Secure`, and `SameSite`.

**Step 4: Replace the session value**

Copied the authenticated session value from the Burp Suite response.

Replaced the browser's existing session cookie value with the authenticated value.

**Step 5: Verify the authenticated session**

Navigated to:

```text
/my-account?id=administrator
```

The application recognized the authenticated session and displayed:

```text
Your username is: administrator
```

### Result

Successfully accessed the administrator account in Firefox using the authenticated session obtained through Burp Suite.

### Technical Observation

Session cookies are commonly used as bearer credentials.

A server can associate a valid session cookie with an authenticated account without requiring the user to submit credentials on every request.

The following attributes provide important protections:

- **HttpOnly:** Prevents JavaScript access to the cookie through APIs such as `document.cookie`.
- **Secure:** Restricts transmission of the cookie to secure HTTPS connections.
- **SameSite:** Controls when the browser sends the cookie in cross-site contexts.

These attributes do not prevent someone who already possesses a valid session value from manually reusing it.

**Important:** Reusing a valid authenticated session does not independently establish a session-management vulnerability. The demonstrated vulnerability remains SQL injection in the login functionality.

### Screenshot Evidence

**Figure 9: Firefox Developer Tools displaying the session cookie and its attributes.**

![Firefox session cookie storage](screenshots/09-session-cookie.png)

**Figure 10: Administrator account accessed after transferring the authenticated session.**

![Administrator session transferred successfully](screenshots/10-session-transfer-solved.png)

---

## 8. Comparison of the Four Workflows

| Feature | Browser Form | Burp Proxy | Burp Repeater | Session Transfer |
|---|---|---|---|---|
| Main technique | Direct form submission | Live request interception | Request modification and replay | Authenticated session reuse |
| Primary tool | Firefox | Burp Proxy | Burp Repeater | Burp + Firefox DevTools |
| SQL injection submitted | Yes | Yes | Yes | No new injection |
| HTTP request visibility | Limited | Full intercepted request | Full request and response | Cookie and session information |
| Main evidence | Administrator account page | Modified request and browser result | Redirect and authenticated response | Authenticated account page |
| Best use | Quick proof of concept | Live traffic manipulation | Detailed response analysis | Understanding session handling |

The first three workflows exploited the same SQL injection vulnerability.

The fourth demonstrated how an authenticated session established after successful exploitation could be reused in another browser context.

---

## 9. Technical Observations

### 9.1 SQL Comment Bypasses Password Verification

The payload:

```text
administrator'--
```

causes the vulnerable SQL query to ignore the password-verification condition.

Unlike some SQL injection payloads, this technique does not require `OR 1=1`.

### 9.2 Authentication Can Be Confirmed Through HTTP Behavior

The successful login request produced a redirect to the administrator account and issued a session cookie.

The authenticated account page and lab completion status confirmed the result.

### 9.3 CSRF Protection Does Not Prevent SQL Injection

The login request included a CSRF token.

However, a valid CSRF token does not prevent SQL syntax from being injected through a vulnerable username parameter.

### 9.4 Session Cookies Represent Authenticated State

After successful authentication, the application issued a session cookie associated with the administrator account.

Reusing the valid cookie allowed the browser to access the authenticated session.

### 9.5 Different Tools Reveal Different Layers

- The browser demonstrated the visible login bypass.
- Proxy revealed the intercepted request parameters.
- Repeater exposed the HTTP response and redirect behavior.
- Developer Tools demonstrated session-cookie storage and reuse.

---

## 10. Security Impact

### Observed Impact

The vulnerability allowed unauthorized authentication as the administrator without knowledge of the account's password.

### Potential Real-World Impact

- **Account Takeover:** Unauthorized access to another user's account.
- **Administrative Access:** Access to privileged functionality when the affected account has administrative permissions.
- **Sensitive Data Exposure:** Unauthorized access to information available to the compromised account.
- **Unauthorized Actions:** Modification or deletion of application data within the account's privileges.
- **Session Misuse:** A valid authenticated session may be reused by someone who obtains its session cookie.

### Vulnerability Classification

- **CWE-89:** Improper Neutralization of Special Elements Used in an SQL Command.
- **CWE-287:** Improper Authentication — relevant to the resulting authentication bypass.

---

## 11. Remediation and Prevention

### 11.1 Use Parameterized Queries

Parameterized queries separate SQL instructions from user-controlled input.

**Illustrative Python example:**

```python
cursor.execute(
    "SELECT id, password_hash FROM users WHERE username = %s",
    (username,)
)

user = cursor.fetchone()

if user and password_hasher.verify(user["password_hash"], password):
    authenticate_user(user["id"])
else:
    reject_login()
```

*Note: The example assumes an appropriate password-hashing library and application-defined authentication functions. Placeholder syntax depends on the database driver.*

### 11.2 Secure Password Storage

Store passwords using an appropriate password-hashing algorithm such as Argon2id.

Never store or compare plaintext passwords directly in SQL queries.

### 11.3 Principle of Least Privilege

Restrict database accounts to the minimum permissions required for their intended operations.

### 11.4 Secure Session Management

- Regenerate session identifiers after successful authentication.
- Use appropriate `HttpOnly`, `Secure`, and `SameSite` cookie attributes.
- Apply suitable session expiration and idle timeouts.
- Invalidate sessions after logout.

### 11.5 Logging and Monitoring

Monitor unusual authentication activity and suspicious input patterns.

Logging and monitoring help detect attacks but do not replace secure query construction.

### 11.6 Security Testing

Regularly test authentication workflows for SQL injection, insecure session handling, and authorization weaknesses.

---

## 12. Key Learnings

This lab strengthened my understanding of:

1. SQL injection in authentication queries.
2. How SQL comments can bypass password-verification conditions.
3. Direct testing through a browser login form.
4. HTTP request interception using Burp Suite Proxy.
5. Request replay and response analysis using Burp Suite Repeater.
6. Interpreting HTTP redirects and session-cookie issuance.
7. Understanding authenticated session-cookie transfer.
8. Differentiating SQL injection from session-management behavior.
9. Applying parameterized queries and secure password verification.

---

## 13. References

- [PortSwigger — SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger — Login Bypass Lab](https://portswigger.net/web-security/sql-injection/lab-login-bypass)
- [PortSwigger — Authentication Vulnerabilities](https://portswigger.net/web-security/authentication)
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [MITRE — CWE-89](https://cwe.mitre.org/data/definitions/89.html)
- [MITRE — CWE-287](https://cwe.mitre.org/data/definitions/287.html)

### Related Medium Article

A detailed article explaining the four workflows and their practical differences:

**[Bypassing a Login Form with SQL Injection: Four Ways to Get There] https://medium.com/@PranjaliMandavi/bypassing-a-login-form-with-sql-injection-four-ways-to-get-there-and-what-each-one-teaches-you-d7b32ad54dab**

---

**Disclaimer:** All testing was conducted in PortSwigger Web Security Academy, an intentionally vulnerable training environment, for authorized educational purposes.
