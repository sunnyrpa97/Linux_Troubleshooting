# TLS Handshake Failure Analysis

A guide to diagnosing and resolving TLS handshake failures that occur in a pattern.

## Overview

A failing TLS handshake is a common but complex issue. When it happens in a pattern (e.g., failing at certain times, for certain users, or with certain servers), it provides a crucial clue for diagnosis.

This document outlines the most common causes, sorted by likelihood, and provides a methodology for diagnosing the problem.

## Common Causes of Patterned Failures

### 1. Version or Protocol Mismatch (Very Common)
The client and server must agree on a TLS version (e.g., TLS 1.2, TLS 1.3). A pattern of failure suggests this agreement isn't always happening.

- **Cause:** A server or client might be configured to only accept a specific range of TLS versions.
- **Pattern Example:**
  - **Failure with older systems:** The handshake fails only for users on Windows 7 (which only supports up to TLS 1.2 by default) trying to connect to a server that has **disabled all versions below TLS 1.3**.
  - **Failure with new systems:** A modern client that only tries TLS 1.3 might fail to connect to an very old server that only supports TLS 1.0 or 1.1 (which are now widely disabled for security reasons).

### 2. Cipher Suite Mismatch (Very Common)
Even if the versions match, the client and server must agree on a cipher suite (the set of algorithms used for encryption, authentication, and key exchange).

- **Cause:** Administrators often harden servers by disabling older, insecure cipher suites (e.g., those using RSA key exchange, CBC mode, or SHA-1).
- **Pattern Example:**
  - The handshake works from a modern browser (which supports modern ciphers like ECDHE) but fails from a legacy application or IoT device that only offers a weak, now-disabled cipher suite like `TLS_RSA_WITH_3DES_EDE_CBC_SHA`.

### 3. Certificate Issues
Problems with the server's certificate are a prime culprit.

- **Cause:**
  - **Expired Certificate:** The most common certificate error. If it's failing in a pattern, it might have just expired.
  - **Missing Intermediate Certificate:** The server is not sending the full certificate chain. This causes issues with **certain clients** because some clients (like browsers) can "fill in the missing chain" from a cache, while others (like CLI tools `curl`, `openssl`) cannot.
  - **Certificate Name Mismatch:** The certificate is issued for `www.example.com` but someone is accessing `api.example.com` or just `example.com`.
  - **Untrusted Certificate Authority (CA):** The certificate is signed by a CA that the client does not trust. This is very common with:
    - Self-signed certificates (often used in development).
    - Corporate/internal PKI certificates that are not installed on the client device.
- **Pattern Example:**
  - A script run from a server works, but the same call from a developer's laptop fails. This is a classic sign of a missing intermediate certificate or an internal CA trust issue.
  - Accessing a site via its IP address fails, but accessing via its domain name works (name mismatch).

### 4. Server Name Indication (SNI) Issues
SNI is an extension that allows a client to specify which hostname it's connecting to *during the handshake*. This is crucial for servers that host multiple websites on a single IP address.

- **Cause:** The client doesn't send the SNI extension, or sends an incorrect one.
- **Pattern Example:**
  - The handshake fails when connecting to a specific service on a shared hosting platform, but works for others on the same IP. The server doesn't know which certificate to present because the client didn't tell it the hostname.

### 5. Network Issues and Middleboxes
Hardware between the client and server can interfere with the pure TLS protocol.

- **Cause:**
  - **Firewalls/Proxies:** Corporate firewalls or transparent proxies may actively block certain TLS versions or cipher suites they deem insecure. They might even try to intercept the traffic (SSL Inspection), acting as a "Man-in-the-Middle." If the client does not trust the proxy's CA certificate, the handshake will fail.
  - **Deep Packet Inspection (DPI):** Similar to firewalls, these can disrupt the initial handshake packets.
- **Pattern Example:**
  - The connection works from a home network but fails from a corporate office network. This is a massive red flag for a corporate firewall/proxy interception issue.
  - The connection works from most regions but fails from a specific country that employs heavy network filtering.

### 6. Resource Exhaustion on the Server
The failure might not be cryptographic at all.

- **Cause:** The server may be out of memory, CPU, or, more specifically, have no available space in its SSL session cache or no more ephemeral ports available for new connections.
- **Pattern Example:**
  - Handshakes start failing under high load or during a traffic spike, but work perfectly under normal conditions.

### 7. Client or Server Clock Skew
Certificates are valid only between a "Not Before" and "Not After" date. This validation uses the system time.

- **Cause:** If the client's clock is significantly wrong (e.g., off by days or years), it will think a valid certificate is either not yet valid or has expired.
- **Pattern Example:**
  - A handshake fails on a single virtual machine or device but works everywhere else. Check the system time on the failing device.

## Diagnosis Guide

### 1. Isolate the Pattern
Document *exactly* when it works and when it doesn't.
- Does it fail for all clients or just some?
- From all networks or just one?
- For all server endpoints or just one?

### 2. Use `openssl s_client` (Your Best Tool)
This command-line tool is invaluable for testing. Run it from both a working and a failing environment and compare the outputs.

**Basic test:**
```bash
openssl s_client -connect example.com:443 -servername example.com

*Look at the certificate chain, the negotiated protocol version, and the cipher suite.*

**Test against a specific version (to rule out version mismatch):**
```bash
openssl s_client -connect example.com:443 -tls1_2  # Force TLS 1.2
openssl s_client -connect example.com:443 -tls1_3  # Force TLS 1.3
```

**Test with a specific cipher (to rule out cipher mismatch):**
```bash
openssl s_client -connect example.com:443 -cipher 'ECDHE'
```

### 3. Use Online Tools
Websites like **SSL Labs' SSL Server Test** ([https://ssllabs.com/ssltest/](https://ssllabs.com/ssltest/)) are excellent. You provide your domain, and it gives a deep analysis of your server's TLS configuration, highlighting misconfigurations, supported versions, and cipher suites.

### 4. Check Client-Side Logs
Enable debug logging in your client application. It will often provide a specific error message like:
- `"WRONG_VERSION_NUMBER"`
- `"HANDSHAKE_FAILURE"`
- `"CERTIFICATE_UNTRUSTED"`

### 5. Check Server-Side Logs
The server's error logs (e.g., in nginx, Apache) will often contain the exact reason for the handshake failure from its perspective.

## Summary

Start by testing with `openssl s_client` from both a working and a failing client. The difference in the output will almost certainly point you directly to the root cause, whether it's a certificate chain issue, a protocol version, or a cipher suite.
```