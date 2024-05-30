# HTTP Methods?
The HTTP specification includes a collection of methods that are used to interact with server-side resources. There are commonly referred to as HTTP request methods or HTTP verbs and are intended to cover all possible types of interaction with resources.
While HTTP request methods typically perform different operations, there is an overlap in functionality, and depending on the task, several HTTP requests will have to be made before it is complete. There are also HTTP method properties to consider, including whether a HTTP request is safe, idempotent, or cacheable.
<br><br>
# What are different HTTP Methods?
HTTP offers a number of methods that can be used to perform actions on the web server (the HTTP 1.1 standard refers to them as methods but they are also commonly described as verbs). While GET and POST are by far the most common methods that are used to access information provided by a web server, HTTP allows several other (and somewhat less known) methods. Some of these can be used for nefarious purposes if the web server is misconfigured.
RFC 7231 – Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content defines the following valid HTTP request methods, or verbs:
<br>
### - GET
The HTTP GET method is called by the client to retrieve the current representation of a specified resource. It is the primary operation for most servers.
<br>
### - HEAD
The HTTP HEAD method is the same as HTTP GET but only transfers the status line and the HTTP response Headers. No message body is included.
<br>
### - POST
The HTTP POST method is called by the client to interact directly with the resource.
<br>
### - PUT
The HTTP PUT method is called by the client to replace the specified resource and all of its current representations, if any, on the server. It is also used to create resources.
<br>
### - DELETE
The HTTP DELETE method is called by the client to remove the specified resource and all of its current representations from the server.
<br>
### - CONNECT
The HTTP CONNECT method is called by the client to establish a tunnel to the origin server, identified by the request-target.
<br><br>
However, most web applications only need to respond to GET and POST requests, receiving user data in the URL query string or appended to the request respectively. The standard `<a href=""></a>` style links as well as forms defined without a method trigger a GET request; form data submitted via `<form method='POST'></form>` trigger POST requests. JavaScript and AJAX calls may send methods other than GET and POST but should usually not need to do that.
Since the other methods are so rarely used, many developers do not know, or fail to take into consideration, how the web server or application framework’s implementation of these methods impact the security features of the application.
<br><br>

# Testing Objectives
- Enumerate supported HTTP methods.
- Test for access control bypass.
- Test XST vulnerabilities.
- Test HTTP method overriding techniques.
<br>

# How to Test
## Discover the Supported Methods:
To perform this test, the tester needs some way to figure out which `HTTP methods` are supported by the web server that is being examined.
While the OPTIONS `HTTP method` provides a direct way to do that, verify the server’s response by issuing requests using different methods. This can be achieved by manual testing or something like the http-methods Nmap script.

To use the http-methods **Nmap** script to test the endpoint `/index.php` on the server localhost using HTTPS, issue the command:<br>
`nmap -p 443 --script http-methods --script-args http-methods.url-path='/index.php' localhost`

When testing an application that has to accept other methods, e.g. a RESTful Web Service, test it thoroughly to make sure that all endpoints accept only the methods that they require.
<br><br>
## Testing the PUT Method
1. Capture the base request of the target with a web proxy.
2. Change the request method to PUT and add test.html file and send the request to the application server.
```
PUT /test.html HTTP/1.1
Host: testing-website

<html>
HTTP PUT Method is Enabled
</html>
```
3. If the server response with 2XX success codes or 3XX redirections and then confirm by GET request for test.html file. The application is vulnerable. If the HTTP PUT method is not allowed on base URL or request, try other paths in the system.

``NOTE: If you are successful in uploading a web shell you should overwrite it or ensure that the security team of the target are aware and remove the component promptly after your proof-of-concept.``

Leveraging the PUT method an attacker may be able to place arbitrary and potentially malicious content, into the system which may lead to remote code execution, defacing the site or denial of service.
<br><br>
## Testing for Access Control Bypass
Find a page to visit that has a security constraint such that a GET request would normally force a 302 redirect to a log in page or force a log in directly.
Issue requests using various methods such as HEAD, POST, PUT etc. as well as arbitrarily made up methods such as BILBAO, FOOBAR, CATS, etc. 
If the web application responds with a `HTTP/1.1 200 OK` that is not a log in page, it may be possible to bypass authentication or authorization.
The following example uses Nmap’s ncat.
```
$ ncat www.example.com 80
HEAD /admin HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
Date: Mon, 18 Aug 2008 22:44:11 GMT
Server: Apache
Set-Cookie: PHPSESSID=pKi...; path=/; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Set-Cookie: adminOnlyCookie1=...; expires=Tue, 18-Aug-2009 22:44:31 GMT; domain=www.example.com
Set-Cookie: adminOnlyCookie2=...; expires=Mon, 18-Aug-2008 22:54:31 GMT; domain=www.example.com
Set-Cookie: adminOnlyCookie3=...; expires=Sun, 19-Aug-2007 22:44:30 GMT; domain=www.example.com
Content-Language: EN
Connection: close
Content-Type: text/html; charset=ISO-8859-1
```

If the system appears vulnerable, issue CSRF-like attacks such as the following to exploit the issue more fully:

- HEAD /admin/createUser.php?member=myAdmin
- PUT /admin/changePw.php?member=myAdmin&passwd=foo123&confirm=foo123
- CATS /admin/groupEdit.php?group=Admins&member=myAdmin&action=add

Using the above three commands, modified to suit the application under test and testing requirements, a new user would be created, a password assigned, and the user made an administrator, all using blind request submission.
<br><br>
## Testing for Cross-Site Tracing Potential
*Note: in order to understand the logic and the goals of a cross-site tracing (XST) attack, one must be familiar with cross-site scripting attacks.*

The TRACE method, intended for testing and debugging, instructs the web server to reflect the received message back to the client. This method, while apparently harmless, can be successfully leveraged in some scenarios to steal legitimate users’ credentials. This attack technique was discovered by Jeremiah Grossman in 2003, in an attempt to bypass the HttpOnly attribute that aims to protect cookies from being accessed by JavaScript. However, the TRACE method can be used to bypass this protection and access the cookie even when this attribute is set.

Test for cross-site tracing potential by issuing a request such as the following:
```
$ ncat www.victim.com 80
TRACE / HTTP/1.1
Host: www.victim.com
Random: Header

HTTP/1.1 200 OK
Random: Header
...
```

The web server returned a 200 and reflected the random header that was set in place. To further exploit this issue:

```
$ ncat www.victim.com 80
TRACE / HTTP/1.1
Host: www.victim.com
Attack: <script>prompt()</script>
```

The above example works if the response is being reflected in the HTML context.

In older browsers, attacks were pulled using XHR technology, which leaked the headers when the server reflects them (e.g. Cookies, Authorization tokens, etc.) and bypassed security measures such as the HttpOnly attribute. This attack can be pulled in recent browsers only if the application integrates with technologies similar to Flash.
<br><br>
## Testing for HTTP Method Overriding
Some web frameworks provide a way to override the actual HTTP method in the request by emulating the missing HTTP verbs passing some custom header in the requests. The main purpose of this is to circumvent some middleware (e.g. proxy, firewall) limitation where methods allowed usually do not encompass verbs such as PUT or DELETE. The following alternative headers could be used to do such verb tunneling:

- X-HTTP-Method
- X-HTTP-Method-Override
- X-Method-Override

In order to test this, in the scenarios where restricted verbs such as PUT or DELETE return a “405 Method not allowed”, replay the same request with the addition of the alternative headers for HTTP method overriding, and observe how the system responds. The application should respond with a different status code (e.g. 200) in cases where method overriding is supported.

The web server in the following example does not allow the `DELETE` method and blocks it:
```
$ ncat www.example.com 80
DELETE /resource.html HTTP/1.1
Host: www.example.com

HTTP/1.1 405 Method Not Allowed
Date: Sat, 04 Apr 2020 18:26:53 GMT
Server: Apache
Allow: GET,HEAD,POST,OPTIONS
Content-Length: 320
Content-Type: text/html; charset=iso-8859-1
Vary: Accept-Encoding
```

After adding the `X-HTTP-Header`, the server responds to the request with a 200:

```
$ ncat www.example.com 80
DELETE /resource.html HTTP/1.1
Host: www.example.com
X-HTTP-Method: DELETE

HTTP/1.1 200 OK
Date: Sat, 04 Apr 2020 19:26:01 GMT
Server: Apache
```
