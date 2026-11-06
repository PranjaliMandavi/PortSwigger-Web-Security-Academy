
# Lab 01 — SQL Injection in WHERE Clause: Retrieving Hidden Data

![Platform](https://img.shields.io/badge/Platform-PortSwigger%20Web%20Security%20Academy-orange)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice-green)
![Vulnerability](https://img.shields.io/badge/Vulnerability-SQL%20Injection-blue)
![Status](https://img.shields.io/badge/Status-Solved-brightgreen)

**Lab:** [SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)  
**Tools:** Firefox, Burp Suite (Proxy, HTTP History, Repeater)  
**Vulnerability Classification:** CWE-89 — SQL Injection  
**Environment:** PortSwigger Web Security Academy

> **Disclaimer:** This write-up documents an authorized exercise conducted in an intentionally vulnerable training environment for educational purposes.

---

## 1. Objective

The application displays products based on their selected category and normally hides unreleased products.

The objective of this lab was to exploit a SQL injection vulnerability in the `category` parameter and retrieve **all products, including unreleased items**.

### Lab Scope

| Attribute | Details |
|---|---|
| Endpoint | `/filter` |
| HTTP Method | GET |
| Vulnerable Parameter | `category` |
| Injection Technique | Boolean tautology and SQL comment |
| Expected Result | Display hidden/unreleased products |
| Lab Status | Solved |

---

## 2. Vulnerability Analysis

### 2.1 Identifying the Injection Point

When the **Gifts** category is selected, the browser sends a request similar to:

```http
GET /filter?category=Gifts HTTP/1.1
```

The application uses the category value to filter products stored in its database.

### 2.2 Understanding the SQL Query

The lab describes a backend query equivalent to:

```sql
SELECT * FROM products
WHERE category = 'Gifts' AND released = 1;
```

This query contains two filtering conditions:

- `category = 'Gifts'` — Returns products belonging to the Gifts category.
- `released = 1` — Ensures that only released products are displayed.

The vulnerability occurs because user-controlled input is incorporated into SQL in a way that allows it to modify the query's structure.

### 2.3 Root Cause

The root cause is **improper separation of SQL instructions and user-supplied data**.

Instead of treating the category value exclusively as data, the application allows SQL operators and comments within the input to affect database query execution.

---

## 3. SQL Injection Payload

### Payload Used

```sql
Gifts' OR 1=1--
```

### URL Representation

```text
/filter?category=Gifts'+OR+1=1--
```

The `+` characters represent spaces in the URL query parameter.

### Payload Breakdown

| Component | Purpose |
|---|---|
| `Gifts` | Original category value |
| `'` | Closes the existing SQL string literal |
| `OR` | Introduces an alternative condition |
| `1=1` | Always evaluates to TRUE |
| `--` | Comments out the remaining SQL statement |

### Modified SQL Query

```sql
SELECT * FROM products
WHERE category = 'Gifts' OR 1=1--' AND released = 1;
```

### How the Injection Works

1. The single quote closes the original category string.
2. `OR 1=1` makes the WHERE condition true for every row.
3. `--` comments out the remaining SQL, including the release restriction.
4. The application returns products regardless of their category or release status.

**Result:** The intended product filtering conditions are bypassed, allowing unreleased products to be displayed.

---

## 4. Method 1 — Direct Browser URL Manipulation

### Procedure

**Step 1: Identify the vulnerable parameter**

Opened the PortSwigger lab in Firefox and selected the **Gifts** category.

The browser generated the following URL:

```text
/filter?category=Gifts
```

**Step 2: Inject the SQL payload**

Modified the category parameter directly in the browser address bar:

```text
/filter?category=Gifts'+OR+1=1--
```

**Step 3: Submit the modified request**

Pressed Enter to send the modified URL to the application.

**Step 4: Verify the result**

The application displayed additional products, including items that were normally hidden.

The PortSwigger lab also displayed the **Solved** status, confirming successful exploitation.

### Result

Successfully exploited the SQL injection vulnerability through direct URL manipulation without using an interception tool.

### Screenshot Evidence

**Figure 1: SQL injection payload submitted through the browser URL.**

![Browser URL with SQL injection payload](screenshots/01-browser-payload.png)

**Figure 2: Successful retrieval of hidden products and lab completion.**

![PortSwigger lab solved](screenshots/02-browser-solved.png)

---

## 5. Method 2 — Burp Suite Proxy Interception

Burp Suite Proxy was used to intercept and modify the HTTP request before it reached the application.

### Procedure

**Step 1: Configure and enable interception**

Configured Firefox to route HTTP traffic through Burp Suite.

Navigated to:

**Proxy → Intercept → Intercept On**

**Step 2: Capture the original request**

Selected the Gifts category in the browser.

Burp Suite intercepted the following request:

```http
GET /filter?category=Gifts HTTP/1.1
```

The request contained the original category value and the application's session cookie.

**Step 3: Identify the vulnerable parameter**

Opened the **Params** tab and located the `category` URL parameter.

The original value was:

```text
Gifts
```

**Step 4: Modify the parameter**

Replaced the category value with:

```sql
Gifts' OR 1=1--
```

**Step 5: Forward the modified request**

Clicked **Forward** to send the injected request to the application.

**Step 6: Analyze the result**

The browser displayed additional product records, demonstrating that the injected SQL condition bypassed the original filtering logic.

The request was also recorded in **Proxy → HTTP History**.

### Result

Successfully reproduced the SQL injection vulnerability by intercepting and modifying a live HTTP request.

This method demonstrated how Burp Suite Proxy provides visibility and control over request parameters before they are processed by the server.

### Screenshot Evidence

**Figure 3: Original category-filter request intercepted using Burp Suite Proxy.**

![Original HTTP request intercepted](screenshots/03-proxy-intercept.png)

**Figure 4: SQL injection payload inserted into the category parameter using the Params tab.**

![Modified category parameter in Burp Proxy](screenshots/04-proxy-payload.png)

**Figure 5: Application displaying additional products after forwarding the injected request.**

![Browser response after Proxy modification](screenshots/05-proxy-result.png)

---

## 6. Method 3 — Burp Suite Repeater

Burp Suite Repeater was used to manually modify, resend, and analyze HTTP requests.

### Procedure

**Step 1: Locate the original request**

Navigated to:

**Proxy → HTTP History**

Identified the original category-filter request:

```http
GET /filter?category=Gifts HTTP/1.1
```

**Step 2: Send the request to Repeater**

Right-clicked the request and selected **Send to Repeater**.

**Step 3: Modify the request**

Opened the Repeater tab and changed the request line to:

```http
GET /filter?category=Gifts'+OR+1=1-- HTTP/1.1
```

**Step 4: Submit the modified request**

Clicked **Send/Go** to forward the injected request to the server.

**Step 5: Analyze the HTTP response**

The server returned an HTTP `200 OK` response.

The injected request produced a significantly larger response than the original request.

### Response Comparison

| Observation | Original Request | Injected Request |
|---|---|---|
| HTTP Method | GET | GET |
| Category Parameter | `Gifts` | `Gifts' OR 1=1--` |
| HTTP Status | 200 OK | 200 OK |
| Response Size | Approximately 7,948 bytes | Approximately 14,625 bytes |
| Application Output | Filtered products | Additional products |

The increased response size suggested that additional content had been returned.

However, response size alone is not definitive proof of SQL injection.

The rendered response displayed additional products and the PortSwigger lab completion message, confirming successful exploitation.

### Result

Successfully reproduced the vulnerability using Burp Suite Repeater and compared the baseline and injected HTTP responses.

This method demonstrated the usefulness of request replay and response analysis during web application security testing.

### Screenshot Evidence

**Figure 6: Sending the captured request from HTTP History to Burp Suite Repeater.**

![Send request to Repeater](screenshots/06-send-to-repeater.png)

**Figure 7: Modified HTTP request containing the SQL injection payload.**

![SQL injection request in Repeater](screenshots/07-repeater-request.png)

**Figure 8: Rendered response confirming successful SQL injection and lab completion.**

![Successful exploitation in Repeater](screenshots/08-repeater-solved.png)

---

## 7. Comparison of the Three Methods

All three methods demonstrated the same underlying SQL injection vulnerability using the same payload.

However, each method offered different levels of visibility, control, and testing efficiency.

| Feature | Browser URL | Burp Proxy | Burp Repeater |
|---|---|---|---|
| Testing Speed | Fast | Moderate | Fast after setup |
| Request Visibility | Limited | Full intercepted request | Full request and response |
| Request Modification | Primarily URL | Parameters, headers, cookies, body | Parameters, headers, cookies, body |
| Response Analysis | Rendered page | Browser and HTTP History | Raw and rendered response |
| Repeatability | Manual | Re-interception or history workflow | Easy repeated testing |
| Best Use | Quick proof of concept | Live request interception | Detailed testing and analysis |

**Practical takeaway:** The browser is useful for quick validation, Proxy for intercepting and modifying live traffic, and Repeater for controlled request replay and response comparison.

---

## 8. Technical Observations

### 8.1 Response Size Difference

The baseline response was approximately **7,948 bytes**, while the injected response was approximately **14,625 bytes**.

This difference supported the observation that the application returned additional content after the SQL injection payload was submitted.

### 8.2 Successful Response Without SQL Errors

The injected request returned `200 OK` rather than a visible SQL error.

This demonstrates that SQL injection can succeed without generating an obvious error message.

### 8.3 Reflected Input and HTML Encoding

The submitted category value was reflected in the application output using HTML entity encoding, such as:

```html
Gifts&apos; OR 1=1--
```

HTML encoding reduces the risk of HTML injection in this specific output context.

However, this observation does not establish that the application is free from cross-site scripting vulnerabilities.

### 8.4 No Elevated Privileges Required

The exploitation did not require elevated application privileges.

The vulnerability originated from unsafe SQL query construction rather than a failure of session authentication.

### 8.5 Role of SQL Comments

The `OR 1=1` condition caused the query's filtering expression to evaluate as true.

The `--` comment removed the remaining SQL restriction, including `AND released = 1`.

Together, these components enabled the successful filtering bypass.

---

## 9. Security Impact

### Observed Impact

The vulnerability allowed unauthorized retrieval of unreleased products that the application was intended to hide.

### Potential Real-World Impact

Depending on the affected query and database permissions, SQL injection vulnerabilities may lead to:

- **Sensitive Data Exposure:** Unauthorized retrieval of confidential database records.
- **Business Logic Bypass:** Access to restricted or unreleased information.
- **Authentication Bypass:** Manipulation of vulnerable login queries.
- **Data Modification:** Unauthorized changes to stored records.
- **Data Deletion:** Loss of database information where permissions allow it.

**Classification:** CWE-89 — Improper Neutralization of Special Elements Used in an SQL Command.

The severity of SQL injection depends on the sensitivity of the affected data, the vulnerable query, and the database account's privileges.

---

## 10. Remediation and Prevention

### 10.1 Parameterized Queries

The primary defense against SQL injection is the use of **parameterized queries (prepared statements)**.

Parameterized queries separate SQL instructions from user-controlled input, ensuring that input values are treated as data.

**Secure Python Example:**

```python
cursor.execute(
    "SELECT * FROM products WHERE category = %s AND released = 1",
    (category,)
)
```

*Note: Placeholder syntax depends on the database driver.*

### 10.2 Input Validation

Validate category values against an expected allow-list.

For example, only predefined product categories should be accepted.

Input validation provides additional protection but does not replace parameterized queries.

### 10.3 Principle of Least Privilege

Configure database accounts with only the permissions required for their intended operations.

### 10.4 Server-Side Access Control

Restricted records should be protected through appropriate server-side authorization controls rather than relying solely on user-influenced query filters.

### 10.5 Logging and Monitoring

Monitor suspicious request patterns, unexpected database errors, and unusual application behavior.

Detection rules should consider context to reduce false positives.

### 10.6 Defense in Depth

A Web Application Firewall (WAF) may help detect or block known SQL injection patterns.

However, a WAF is an additional defense and cannot replace secure query construction.

---

## 11. Key Learnings

This lab strengthened my understanding of:

1. Identifying SQL injection points in HTTP GET parameters.
2. Understanding how Boolean expressions affect SQL queries.
3. Using SQL comments to bypass application filtering conditions.
4. Testing vulnerabilities through direct browser URL manipulation.
5. Intercepting and modifying HTTP requests using Burp Suite Proxy.
6. Replaying requests and analyzing responses using Burp Suite Repeater.
7. Comparing baseline and modified HTTP responses.
8. Understanding how parameterized queries prevent SQL injection.

---

## 12. References

- [PortSwigger — SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger — Retrieving Hidden Data Lab](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
- [OWASP — SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [MITRE — CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)

### Related Medium Article

I also documented this lab in a detailed Medium article covering three exploitation approaches and their practical differences.

**Article:** [One Lab, Three Ways — SQL Injection Walkthrough](PASTE_YOUR_MEDIUM_ARTICLE_URL_HERE)

---

**Disclaimer:** This write-up documents an authorized educational exercise conducted in PortSwigger Web Security Academy. No unauthorized systems were tested.

