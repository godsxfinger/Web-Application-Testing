# What is Header Injection?

Misconfigurations and flawed business logic can expose websites to a variety of attacks via the HTTP Host header. We'll outline the high-level methodology for identifying websites that are vulnerable to HTTP Host header attacks and demonstrate how you can exploit this for the following kinds of attacks:

 - Password reset poisoning.
 - Web cache poisoning.
 - Exploiting classic server-side vulnerabilities
 - Bypassing authentication
 - Virtual host brute-forcing
 - Routing-based SSRF
 - Connection state attacks
<br>
# What is the HTTP Host header?

The HTTP Host header is a mandatory request header as of HTTP/1.1. It specifies the domain name that the client wants to access. For example, when a user visits `https://example.com/directory`, their browser will compose a request containing a Host header as follows:

```
GET /directory HTTP/1.1 
Host: example.com
```

In some cases, such as when the request has been forwarded by an intermediary system, the Host value may be altered before it reaches the intended back-end component.
<br><br>
# What is the purpose of the HTTP Host header?

The purpose of the HTTP Host header is to help identify which back-end component the client wants to communicate with. If requests didn't contain Host headers, or if the Host header was malformed in some way, this could lead to issues when routing incoming requests to the intended application.

Historically, this ambiguity didn't exist because each IP address would only host content for a single domain. Nowadays, largely due to the ever-growing trend for cloud-based solutions and outsourcing much of the related architecture, it is common for multiple websites and applications to be accessible at the same IP address. This approach has also increased in popularity partly as a result of IPv4 address exhaustion.

When multiple applications are accessible via the same IP address, this is most commonly a result of one of the following scenarios.

### Virtual hosting

One possible scenario is when a single web server hosts multiple websites or applications. This could be multiple websites with a single owner, but it is also possible for websites with different owners to be hosted on a single, shared platform. This is less common than it used to be, but still occurs with some cloud-based SaaS solutions.

In either case, although each of these distinct websites will have a different domain name, they all share a common IP address with the server. Websites hosted in this way on a single server are known as "virtual hosts".

To a normal user accessing the website, a virtual host is often indistinguishable from a website being hosted on its own dedicated server.
<br><br>
### Routing traffic via an intermediary

Another common scenario is when websites are hosted on distinct back-end servers, but all traffic between the client and servers is routed through an intermediary system. This could be a simple load balancer or a reverse proxy server of some kind. This setup is especially prevalent in cases where clients access the website via a content delivery network (CDN).

In this case, even though the websites are hosted on separate back-end servers, all of their domain names resolve to a single IP address of the intermediary component. This presents some of the same challenges as virtual hosting because the reverse proxy or load balancer needs to know the appropriate back-end to which it should route each request.
<br><br>
### How does the HTTP Host header solve this problem?

In both of these scenarios, the Host header is relied on to specify the intended recipient. A common analogy is the process of sending a letter to somebody who lives in an apartment building. The entire building has the same street address, but behind this street address there are many different apartments that each need to receive the correct mail somehow. One solution to this problem is simply to include the apartment number or the recipient's name in the address. In the case of HTTP messages, the Host header serves a similar purpose.

When a browser sends the request, the target URL will resolve to the IP address of a particular server. When this server receives the request, it refers to the Host header to determine the intended back-end and forwards the request accordingly.
<br><br>
# How to test for vulnerabilities using the HTTP Host header

To test whether a website is vulnerable to attack via the HTTP Host header, you will need an intercepting proxy, such as Burp Proxy, and manual testing tools like Burp Repeater and Burp Intruder.

In short, you need to identify whether you are able to modify the Host header and still reach the target application with your request. If so, you can use this header to probe the application and observe what effect this has on the response.

### Supply an arbitrary Host header

When probing for Host header injection vulnerabilities, the first step is to test what happens when you supply an arbitrary, unrecognized domain name via the Host header.

Some intercepting proxies derive the target IP address from the Host header directly, which makes this kind of testing all but impossible; any changes you made to the header would just cause the request to be sent to a completely different IP address. However, Burp Suite accurately maintains the separation between the Host header and the target IP address. This separation allows you to supply any arbitrary or malformed Host header that you want, while still making sure that the request is sent to the intended target.

Sometimes, you will still be able to access the target website even when you supply an unexpected Host header. This could be for a number of reasons. For example, servers are sometimes configured with a default or fallback option in case they receive requests for domain names that they don't recognize. If your target website happens to be the default, you're in luck. In this case, you can begin studying what the application does with the Host header and whether this behavior is exploitable.

On the other hand, as the Host header is such a fundamental part of how the websites work, tampering with it often means you will be unable to reach the target application at all. The front-end server or load balancer that received your request may simply not know where to forward it, resulting in an "`Invalid Host header`" error of some kind. This is especially likely if your target is accessed via a CDN. In this case, you should move on to trying some of the techniques outlined below.
<br><br>
### Check for flawed validation

Instead of receiving an "`Invalid Host header`" response, you might find that your request is blocked as a result of some kind of security measure. For example, some websites will validate whether the Host header matches the SNI from the TLS handshake. This doesn't necessarily mean that they're immune to Host header attacks.

You should try to understand how the website parses the Host header. This can sometimes reveal loopholes that can be used to bypass the validation. For example, some parsing algorithms will omit the port from the Host header, meaning that only the domain name is validated. If you are also able to supply a non-numeric port, you can leave the domain name untouched to ensure that you reach the target application, while potentially injecting a payload via the port.

`GET /example HTTP/1.1 Host: vulnerable-website.com:bad-stuff-here`

Other sites will try to apply matching logic to allow for arbitrary subdomains. In this case, you may be able to bypass the validation entirely by registering an arbitrary domain name that ends with the same sequence of characters as a whitelisted one:

`GET /example HTTP/1.1 Host: notvulnerable-website.com`

Alternatively, you could take advantage of a less-secure subdomain that you have already compromised:

`GET /example HTTP/1.1 Host: hacked-subdomain.vulnerable-website.com`
<br><br>
### Send ambiguous requests

The code that validates the host and the code that does something vulnerable with it often reside in different application components or even on separate servers. By identifying and exploiting discrepancies in how they retrieve the Host header, you may be able to issue an ambiguous request that appears to have a different host depending on which system is looking at it.

The following are just a few examples of how you may be able to create ambiguous requests.
<br><br>
#### Inject duplicate Host headers

One possible approach is to try adding duplicate Host headers. Admittedly, this will often just result in your request being blocked. However, as a browser is unlikely to ever send such a request, you may occasionally find that developers have not anticipated this scenario. In this case, you might expose some interesting behavioral quirks.

Different systems and technologies will handle this case differently, but it is common for one of the two headers to be given precedence over the other one, effectively overriding its value. When systems disagree about which header is the correct one, this can lead to discrepancies that you may be able to exploit. Consider the following request:

`GET /example HTTP/1.1 Host: vulnerable-website.com Host: bad-stuff-here`

Let's say the front-end gives precedence to the first instance of the header, but the back-end prefers the final instance. Given this scenario, you could use the first header to ensure that your request is routed to the intended target and use the second header to pass your payload into the server-side code.
<br><br>
#### Supply an absolute URL

Although the request line typically specifies a relative path on the requested domain, many servers are also configured to understand requests for absolute URLs.

The ambiguity caused by supplying both an absolute URL and a Host header can also lead to discrepancies between different systems. Officially, the request line should be given precedence when routing the request but, in practice, this isn't always the case. You can potentially exploit these discrepancies in much the same way as duplicate Host headers.

`GET https://vulnerable-website.com/ HTTP/1.1 Host: bad-stuff-here`

Note that you may also need to experiment with different protocols. Servers will sometimes behave differently depending on whether the request line contains an HTTP or an HTTPS URL.
<br><br>
#### Add line wrapping

You can also uncover quirky behavior by indenting HTTP headers with a space character. Some servers will interpret the indented header as a wrapped line and, therefore, treat it as part of the preceding header's value. Other servers will ignore the indented header altogether.

Due to the highly inconsistent handling of this case, there will often be discrepancies between different systems that process your request. For example, consider the following request:

`GET /example HTTP/1.1 Host: bad-stuff-here Host: vulnerable-website.com`

The website may block requests with multiple Host headers, but you may be able to bypass this validation by indenting one of them like this. If the front-end ignores the indented header, the request will be processed as an ordinary request for `vulnerable-website.com`. Now let's say the back-end ignores the leading space and gives precedence to the first header in the case of duplicates. This discrepancy might allow you to pass arbitrary values via the "wrapped" Host header.
<br><br>
### Inject host override headers

Even if you can't override the Host header using an ambiguous request, there are other possibilities for overriding its value while leaving it intact. This includes injecting your payload via one of several other HTTP headers that are designed to serve just this purpose, albeit for more innocent use cases.

As we've already discussed, websites are often accessed via some kind of intermediary system, such as a load balancer or a reverse proxy. In this kind of architecture, the Host header that the back-end server receives may contain the domain name for one of these intermediary systems. This is usually not relevant for the requested functionality.

To solve this problem, the front-end may inject the `X-Forwarded-Host` header, containing the original value of the Host header from the client's initial request. For this reason, when an `X-Forwarded-Host` header is present, many frameworks will refer to this instead. You may observe this behavior even when there is no front-end that uses this header.

You can sometimes use `X-Forwarded-Host` to inject your malicious input while circumventing any validation on the Host header itself.

`GET /example HTTP/1.1 Host: vulnerable-website.com X-Forwarded-Host: bad-stuff-here`

Although `X-Forwarded-Host` is the de facto standard for this behavior, you may come across other headers that serve a similar purpose, including:

- `X-Host`
- `X-Forwarded-Server`
- `X-HTTP-Host-Override`
- `Forwarded`
<br><br>

# How to exploit the HTTP Host header.

Once you have identified that you can pass arbitrary hostnames to the target application, you can start to look for ways to exploit it.

In this section, we'll provide some examples of common HTTP Host header attacks that you may be able to construct. We've also created some deliberately vulnerable websites so that you can see how these exploits work and put what you've learned to the test.

We'll cover the following examples:

### Password reset poisoning

Attackers can sometimes use the Host header for password reset poisoning attacks.
<br><br>
### Web cache poisoning via the Host header

When probing for potential Host header attacks, you will often come across seemingly vulnerable behavior that isn't directly exploitable. For example, you may find that the Host header is reflected in the response markup without HTML-encoding, or even used directly in script imports. Reflected, client-side vulnerabilities, such as XSS, are typically not exploitable when they're caused by the Host header. There is no way for an attacker to force a victim's browser to issue an incorrect host in a useful manner.

However, if the target uses a web cache, it may be possible to turn this useless, reflected vulnerability into a dangerous, stored one by persuading the cache to serve a poisoned response to other users.

To construct a web cache poisoning attack, you need to elicit a response from the server that reflects an injected payload. The challenge is to do this while preserving a cache key that will still be mapped to other users' requests. If successful, the next step is to get this malicious response cached. It will then be served to any users who attempt to visit the affected page.

Standalone caches typically include the Host header in the cache key, so this approach usually works best on integrated, application-level caches. That said, the techniques discussed earlier can sometimes enable you to poison even standalone web caches.
<br><br>
### Exploiting classic server-side vulnerabilities

Every HTTP header is a potential vector for exploiting classic server-side vulnerabilities, and the Host header is no exception. For example, you should try the usual SQL injection probing techniques via the Host header. If the value of the header is passed into a SQL statement, this could be exploitable.
<br><br>
### Accessing restricted functionality

For fairly obvious reasons, it is common for websites to restrict access to certain functionality to internal users only. However, some websites `access control` features make flawed assumptions that allow you to bypass these restrictions by making simple modifications to the Host header. This can expose an increased attack surface for other exploits.
<br><br>
### Accessing internal websites with virtual host brute-forcing

Companies sometimes make the mistake of hosting publicly accessible websites and private, internal sites on the same server. Servers typically have both a public and a private IP address. As the internal hostname may resolve to the private IP address, this scenario can't always be detected simply by looking at DNS records:

`www.example.com: 12.34.56.78 intranet.example.com: 10.0.0.132`

In some cases, the internal site might not even have a public DNS record associated with it. Nonetheless, an attacker can typically access any virtual host on any server that they have access to, provided they can guess the hostnames. If they have discovered a hidden domain name through other means, such as `information disclosure`, they could simply request this directly. Otherwise, they can use tools like Burp Intruder to brute-force virtual hosts using a simple wordlist of candidate subdomains.
<br><br>
### Routing-based SSRF

It is sometimes also possible to use the Host header to launch high-impact, routing-based SSRF attacks. These are sometimes known as "Host header SSRF attacks", and were explored in depth by PortSwigger Research in [Cracking the lens: targeting HTTP's hidden attack-surface](https://portswigger.net/research/cracking-the-lens-targeting-https-hidden-attack-surface).

Classic SSRF vulnerabilities are usually based on XXE or exploitable business logic that sends HTTP requests to a URL derived from user-controllable input. Routing-based SSRF, on the other hand, relies on exploiting the intermediary components that are prevalent in many cloud-based architectures. This includes in-house load balancers and reverse proxies.

Although these components are deployed for different purposes, fundamentally, they receive requests and forward them to the appropriate back-end. If they are insecurely configured to forward requests based on an unvalidated Host header, they can be manipulated into misrouting requests to an arbitrary system of the attacker's choice.

These systems make fantastic targets. They sit in a privileged network position that allows them to receive requests directly from the public web, while also having access to much, if not all, of the internal network. This makes the Host header a powerful vector for SSRF attacks, potentially transforming a simple load balancer into a gateway to the entire internal network.

You can use Burp Collaborator to help identify these vulnerabilities. If you supply the domain of your Collaborator server in the Host header, and subsequently receive a DNS lookup from the target server or another in-path system, this indicates that you may be able to route requests to arbitrary domains.

Having confirmed that you can successfully manipulate an intermediary system to route your requests to an arbitrary public server, the next step is to see if you can exploit this behavior to access internal-only systems. To do this, you'll need to identify private IP addresses that are in use on the target's internal network. In addition to any IP addresses that are leaked by the application, you can also scan hostnames belonging to the company to see if any resolve to a private IP address. If all else fails, you can still identify valid IP addresses by simply brute-forcing standard private IP ranges, such as `192.168.0.0/16`.
<br><br>
### Connection state attacks

For performance reasons, many websites reuse connections for multiple request/response cycles with the same client. Poorly implemented HTTP servers sometimes work on the dangerous assumption that certain properties, such as the Host header, are identical for all HTTP/1.1 requests sent over the same connection. This may be true of requests sent by a browser, but isn't necessarily the case for a sequence of requests sent from Burp Repeater. This can lead to a number of potential issues.

For example, you may occasionally encounter servers that only perform thorough validation on the first request they receive over a new connection. In this case, you can potentially bypass this validation by sending an innocent-looking initial request then following up with your malicious one down the same connection.

Many reverse proxies use the Host header to route requests to the correct back-end. If they assume that all requests on the connection are intended for the same host as the initial request, this can provide a useful vector for a number of Host header attacks, including routing-based SSRF, password reset poisoning, and cache poisoning.
<br><br>
### SSRF via a malformed request line

Custom proxies sometimes fail to validate the request line properly, which can allow you to supply unusual, malformed input with unfortunate results.

For example, a reverse proxy might take the path from the request line, prefix it with `http://backend-server`, and route the request to that upstream URL. This works fine if the path starts with a `/` character, but what if starts with an `@` character instead?

`GET @private-intranet/example HTTP/1.1`

The resulting upstream URL will be `http://backend-server@private-intranet/example`, which most HTTP libraries interpret as a request to access `private-intranet` with the username `backend-server`.
<br><br>
