
# What happens when you type `https://www.google.com` in your browser? 



### ✅ Step 1: **Browser Input Handling**

* You type: `www.google.com`
* Browser checks:

  * Is it a **URL or a search query**? It sees `.com` → treated as URL.
  * Browser automatically prepends `https://` → so full URL becomes:

    ```
    https://www.google.com/
    ```

---

### ✅ Step 2: **Browser Internal Caches**

Browser checks multiple caches before asking OS:

* **HSTS Preload** list: Is `google.com` HTTPS-only? → ✅ Yes, it’s on the preload list.
* **DNS Cache**: Browser checks if it already has an IP for `www.google.com`
* **HTTP Cache**: For resource reuse.
* If no usable DNS cache, browser sends request to OS.

---

### ✅ Step 3: **OS Resolver Call**

Browser uses OS APIs to resolve the domain:

```c
getaddrinfo("www.google.com", ...)
```

* OS checks `/etc/hosts` (Linux) or `C:\Windows\System32\drivers\etc\hosts` → nothing for `www.google.com`
* OS now queries the **configured DNS resolver** (usually from DHCP or `/etc/resolv.conf`)

---

## 🔍 Step 4: **DNS Resolution – Recursive Resolver Flow**

Let's say your resolver is `8.8.8.8` (Google Public DNS). It does the following:

### ✅ 4.1: Check Cache

If the resolver has `www.google.com` cached → it returns immediately.

If not → begins **iterative DNS resolution**:

---

### 🌍 4.2: Query Root Server (`.`)

* Query: **“What is the nameserver for `com` TLD?”**
* Root server responds:

```text
Use these .com nameservers:
ns1.gtld-servers.net
ns2.gtld-servers.net
...
```

---

### 🔤 4.3: Query `.com` TLD Server

* Query: “What are the authoritative nameservers for `google.com`?”

> ✅ If `.com` has the delegation for `google.com`, it replies:

```text
google.com nameservers are:
ns1.google.com
ns2.google.com
...
```

> ❌ **If `.com` doesn’t have delegation (no NS for `google.com`)**:

* It responds with **NXDOMAIN** (non-existent domain).
* The resolution **fails here** → browser shows **“Server not found”**.
* This is rare for real domains but **expected for typos or expired domains**.

---

### 🧭 4.4: Query Google’s Authoritative Nameserver

* Query: “What is the A or AAAA record for `www.google.com`?”
* Response:

```text
A www.google.com → 142.250.72.36 (one of many)
```

---

## 🔄 Step 5: **OS → Browser with IP**

The IP is now passed back to the browser.

Example:

```json
www.google.com → 142.250.72.36
```

> This IP could belong to **Google Frontend / Edge Server** close to your location (via geo-aware DNS).

---

## 🌐 Step 6: **TCP Connection (3-Way Handshake)**

* Browser initiates a TCP connection to `142.250.72.36` on port `443`:

```
Client → SYN → Google
Google → SYN-ACK → Client
Client → ACK → Google
```

OS-level call like:

```c
socket(), connect()
```

---

## 🔐 Step 7: **TLS Handshake**

* Browser sends **Client Hello** with:

  * TLS version, cipher suites, SNI = `www.google.com`
* Server replies with:

  * **Server Hello**, certificate, key exchange, etc.

➡️ Certificate is signed by a trusted CA and matches `www.google.com`.

* Client and server **negotiate symmetric session keys**

---

## 🌍 Step 8: **HTTPS Request**

Browser sends encrypted HTTP request:

```http
GET / HTTP/1.1
Host: www.google.com
User-Agent: Chrome/Edge/Firefox
```

---

## 💻 Step 9: **Google Web Server Response**

* Google uses its global frontend server to respond:

```http
HTTP/2 200 OK
Content-Type: text/html
Content-Encoding: gzip
```

* HTML page is returned (compressed), with references to:

  * CSS, JS, logos, etc.

---

## 🎨 Step 10: **Browser Rendering Engine**

* Parses HTML → DOM
* Parses CSS → CSSOM
* Builds render tree
* Executes JavaScript
* Performs layout and paint operations
* Composites layers
* Displays the Google homepage

---

## 🎯 Edge Cases: What If `.com` TLD Doesn’t Have Entry?

### ❌ Scenario: `.com` doesn’t have NS record for `google.com`

* `.com` responds with **NXDOMAIN** (non-existent domain)
* Resolver immediately stops
* Error bubbles up to browser:

```
ERR_NAME_NOT_RESOLVED
```

➡️ Meaning: There is no such domain under `.com`.

### 🛑 Resolution Ends at `.com` Level

If no delegation exists at `.com` for `google.com`, the resolver will **not** try:

* Other TLDs
* Wildcard guessing
* Search engines (unless browser forces it)

---

## 🧠 Summary Table of Flow

| Step | Layer         | Description                              |
| ---- | ------------- | ---------------------------------------- |
| 1    | Browser       | Detects URL, forces HTTPS                |
| 2    | Browser       | Checks internal caches                   |
| 3    | OS            | Checks hosts file, calls DNS             |
| 4    | Recursive DNS | Contacts root → `.com` → `google.com` NS |
| 5    | OS            | Returns resolved IP                      |
| 6    | TCP/IP        | 3-way handshake to IP:443                |
| 7    | TLS           | Secure encrypted handshake               |
| 8    | HTTP/2        | GET / request sent                       |
| 9    | Web Server    | Sends HTML page                          |
| 10   | Browser       | Renders Google homepage                  |

---

## 📌 Bonus: Where Caching Happens

* **Browser DNS cache** (seconds/minutes)
* **OS-level cache** (`nscd`, `systemd-resolved`, `dnsmasq`)
* **Recursive resolver cache** (e.g., Google DNS, Cloudflare)
* **CDN Edge DNS** or load balancing (e.g., Akamai, Google Frontend)

---
