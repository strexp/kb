---
tags:
  - v6-only
  - internal-srv
---

# Mail Service

!!! warning "Network Access"
    This service is strictly **IPv6 Only** and allow mails only from/to **within the DN42 network**.

We provide internal email services for our crew and automated systems within DN42.

## Server

| Service | Hostname | Port | Encryption | Auth Method |
| :--- | :--- | :--- | :--- | :--- |
| **IMAP** | `mx.nia.dn42` | **993** | SSL/TLS | Normal Password |
| **SMTP** | `mx.nia.dn42` | **587** | STARTTLS | Normal Password |
| **SMTP** | `mx.nia.dn42` | **465** | SSL/TLS | Normal Password |

## Hosted Domains

| Domain              | DKIM   | SPF  | DMARC |
| ------------------- | ------ | ---- | ----- |
| nia.dn42            | Y      | Y    | Y     |
| gensokyo.dn42       | Y      | Y    | Y     |

## Mail Test

Send a test email to the echo bot to verify your configuration.

- Recipient: `mail-test@nia.dn42`
- Expected Result: You will receive an auto-reply containing your original message headers.
