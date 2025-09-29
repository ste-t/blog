+++
date = '2025-09-25T15:37:51+02:00'
draft = false
title = 'Building my own email server'

translationKey = 'email-server'
+++

I have owned the domain `tste.dev` for years now, and I've always considered hosting my own email server.
It is no trivial task, but it means having better control of how my own data is stored and handled,
especially considering how much sensitive information is unfortunately transmitted via email.

## Understanding email

When you compose and send an email, your client hands the message off to a Mail Transfer Agent (MTA)
using the SMTP protocol (Simple Mail Transfer Protocol). The MTA then looks up the recipient's domain via DNS MX records to determine
which mail server should receive the message.

Your MTA then connects to such server over SMTP and transmits the email. The recipient's server stores
the message in a mailbox until the recipient's client retrieves it. Such operation usually happens through IMAP
(Internet Message Access Protocol) or the older POP3 (Post Office Protocol).

> **Did you know?** RAS syndrome (Redundant Acronym Syndrome) is the redundant use of one or more of the words contained in an acronym.

### Modern security standards

#### Sender Policy Framework ([RFC7208](https://datatracker.ietf.org/doc/html/rfc7208))

SPF is essentially a list of servers allowed to send email on behalf of a domain.
It is stored as a TXT DNS record. When a receiving mail server gets a message, it checks the sender's domain
against the SPF record. However, SPF alone only validates the path the email took; it doesn't ensure the message
body or headers weren't tampered with.

#### DomainKeys Identified Mail ([RFC6376](https://datatracker.ietf.org/doc/html/rfc6376))

DKIM adds a cryptographic signature to emails. The sending server uses a private key to sign specific headers and the body of the email,
and the corresponding public key is published in DNS. When the receiving server gets the email, it verifies the signature by fetching the
public key. This makes the email content virtually impossible to tamper with as long as the private key is securely stored.
The key pair is commonly RSA-SHA256 1024-4096 bits; newer setups may use Ed25519 ([RFC8463](https://datatracker.ietf.org/doc/html/rfc8463)).

#### Domain-based Message Authentication, Reporting, and Conformance ([RFC7489](https://datatracker.ietf.org/doc/html/rfc7489))

DMARC is a policy layer that ties SPF and DKIM together. It tells receiving servers how to handle messages that fail SPF and/or DKIM checks.
A domain owner publishes a DMARC record in DNS specifying:
- Which checks should pass to consider and email valid.
- What to do with failing messages (none, quarantine, reject).
- Where to send reports about authentication activity.

## Deliverability

Unlike tightly controlled messaging systems, email is built on an open, decentralized infrastructure where anyone can send
mail to anyone else. Such openness, combined with the lack of strong built-in sender authentication in the original SMTP protocol, makes email
particularly vulnerable to spam, spoofing, and abuse.

As a result, modern email delivery heavily depends on filters, blacklists, and reputation systems operated by receiving providers. The challenge
is that even legitimate messages can get misclassified or rejected if they come from a new domain, misconfigured server, or lack proper authentication.

Finally, keep in mind that email does not necessarily have any tolerance for downtime. If your receiving SMTP server fails,
some sending servers might retry, even for days, whilst others will drop the message instantly.

### Ensuring deliverability

#### Authentication

Properly configure SPF, DKIM, DMARC. Once you have a strong DMARC policy, you might want to look into [BIMI](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/).
Use TLS for in-transit encryption.

#### Consistency

Set up a PTR (reverse DNS) record to match your sending IP to your domain. This usually involves contacting your ISP or cloud provider.
Additionally, set up a correct HELO/EHLO hostname ([RFC5321](https://datatracker.ietf.org/doc/html/rfc5321)) on your mail server.

#### IP reputation

Your IP reputation needs to be clean. If trying to host from home, it's likely that your ISP-provided IP is already dirty, and they might also refuse
to allow outbound port 25 or to create a PTR record. If you're in this situation, you can set up an external SMTP relay. That would be either your own VPS
running the relay (useful if you want most of your architecture to still exist inside your own house) or one provided by many dedicated companies.

#### Monitoring

You can check your domain and IP reputation using tools like Google Postmaster Tools and Microsoft SNDS.

## The setup

> Apparently, providers like Microsoft block Oracle IPs no matter what, so I went the safe route and used Mailjet as an SMTP relay.
I also plan on implementing custom Postfix rules to route email shipped to known safe domains through my own SMTP server, in order
to maximize sovereignty over my own data.

I set up my server in an Oracle Cloud VM instance using [mailcow](https://mailcow.email), a Docker-based mailserver suite which takes advantage
of many well-known and long-used components. The two most fundamental pieces of software you should familiarize yourself with are:
- Dovecot: IMAP/POP server
- Postfix: Mail Transfer Agent

These can of course also be used standalone if you desire a more minimal setup or have a more niche use case.

## Ports

You need to enable the following inbound ports in your cloud or physical firewall:
| Service             | Protocol | Port    |
|---------------------|----------|---------|
| Postfix SMTP(S)     | TCP      | 25/465  |
| Postfix Submission  | TCP      | 587     |
| Dovecot IMAP(S)     | TCP      | 143/993 |
| Dovecot POP3(S)     | TCP      | 110/995 |
| Dovecot ManageSieve | TCP      | 4190    |
| HTTP(S)             | TCP      | 80/443  |

All outbound ports are likely already allowed.
If that's not your case, refer to [this table](https://docs.mailcow.email/getstarted/prerequisite-system/#outgoing-portshosts).

## DNS records

### A/AAAA

This simply points to the IP of your mail server.
```
mail.tste.dev. IN A 130.110.6.94
```

### PTR

A PTR (reverse DNS) record allows looking up a domain given an IP, improving trust with receiving servers.
You usually need to get in touch with your provider in order to get this set up.
```
94.6.110.130.in-addr.arpa. IN PTR mail.tste.dev
```
Notice how the octects are in reverse order.

### MX

An MX record tells sending mail servers which mail server to use to deliver emails for a specific domain.
```
tste.dev. IN MX 10 mail.tste.dev.
```

This means emails sent to `@tste.dev` will go to `mail.tste.dev`.
The number `10` is this record's priority, useful when multiple servers are available.

### SPF

{{< tabs tabTotal="2" >}}

{{% tab tabName="No relay" %}}
```
tste.dev. IN TXT "v=spf1 mx ~all"
```
This means the only allowed server is the one specified in your MX record.
The `~all` parameter signals to the receiving server to soft fail if the check doesn't succeed, meaning the email is delivered but marked as potentially dangerous.
{{% /tab %}}

{{% tab tabName="SMTP relay" %}}
```
tste.dev. IN TXT "v=spf1 include:spf.mailjet.com mx ~all"
```
This allows for spf.mailjet.com to send mail on your behalf as well.
{{% /tab %}}

{{< /tabs >}}

### DKIM

```
dkim._domainkey. IN TXT "v=DKIM1;k=rsa;t=s;s=email;p=MIIBIjANBg..."
```

The part before `._domainkey` is called label: you can list multiple public keys using different labels.
For instance, I created a DKIM record with the `mailjet` label and the corresponding key in order to allow
the relay to send mail on my behalf.
The value assigned to `p=` is the base64-encoded public key. It will later be generated during the Mailcow setup.

### DMARC

```
_dmarc.tste.dev. IN TXT "v=DMARC1; p=reject; sp=reject;
adkim=s; aspf=s; rua=mailto:dmarc@tste.dev;"
```

`p=reject` → Reject mail that fails DMARC checks.

`sp=reject` → Also reject mail from subdomains that fail.

`adkim=s` → DKIM alignment mode is strict (the DKIM signing domain must exactly match the From: domain).

`aspf=s` → SPF alignment mode is strict (the SPF-validated domain must exactly match the From: domain).

`rua=mailto:dmarc@tste.dev` → Aggregate reports about DMARC results should be sent to this email address.

## Software setup

> Keep in mind these steps take into account the specific challenges I faced on my system with my setup.
For a complete and generic guide, consult [mailcow's docs](https://docs.mailcow.email).

### Preliminary

Make sure you have these basic packages.

```sh
sudo dnf install -y git openssl curl gawk grep jq
```

Install docker.

```sh
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# This threw an error on my system, and
# after some research I managed to solve with
# sudo dnf install -y kernel-modules-extra-$(uname -r)
# and a reboot
sudo systemctl enable --now docker
```

Check your network interface's MTU with `ip l`. If it is an uncommon value (not 1500), add it to your
`/etc/docker/daemon.json`.

```json
{
  "selinux-enabled": true,
  "mtu": 9000
}
```

Do add the selinux-enabled entry if applicable, and keep in mind your MTU as you will need it in an upcoming step.

### Mailcow

```sh
sudo su
umask 0022
cd /opt

git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized

./generate_config.sh
vim mailcow.conf  # Set HTTP_REDIRECT=y if you do not intend on using a reverse proxy
```

Edit `docker-compose.yml`.
```yml
...
networks:
  mailcow-network:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-mailcow
      com.docker.network.driver.mtu: 9000
...
```

Start the containers.
```sh
docker compose pull
docker compose up -d
```

## Configuration

### First login

Once I got mailcow up and running, I logged into the admin interface, generated a new, strong, random password, and enabled TOTP 2FA.
I obviously recommend using a password manager such as [KeePassXC](https://github.com/keepassxreboot/keepassxc) as well as a FOSS mobile
authenticator app.

### DKIM key

The UI provides a menu to generate DKIM keys under System > Configuration > Options > ARC/DCIM keys.
However, it was throwing "Access denied or incomplete / invalid data", and I simply decided to generate the key pair myself
and import it from within the same menu.

> You can do 3072 or 4096 bits, but this is often unnecessary and can lead to issues with DNS record length limits.

```sh
openssl genrsa -out dkim.key 2048
cat dkim.key | xclip -c  # Copy to clipboard
```

Once this is done, you can copy and paste the shown DKIM record to complete your DNS configuration.

### Domains, mailboxes, aliases

To complete my setup, I created the `@tste.dev` domain, and added a few mailboxes.
Each mailbox is essentially a user with their own password.

Mailcow allows you to create aliases for your mailboxes. This can be useful in many different ways,
like spam mitigation, categorization, and aggregating emails into a single central inbox.

Each mailbox can also autonomously generate temporary email addresses with pseudo-random local names like `xiheha.jazi@tste.dev`.

## Privacy considerations

What some people fail to realize is that privacy and cybersecurity require defining a threat model.
Otherwise, the only safe system is encased in concrete and at the bottom of the Marianna Trench.

Oracle would have no issues reading my cloud volume. And disk encryption doesn't make sense because the decryption key would exist in RAM
as long as the machine would be running, which is always, and they can dump memory just as easily as they can dump my storage volume.

However, I doubt they're out there scanning every single cloud instance just in case someone has emails at rest on the disk.
I can also delete emails as soon as I've read them, which would only be countered by either scanning every single file as soon as
it is created or modified or by forensic techniques such as free inode scanning or raw signature-based file carving.

On the other hand Google does in fact actively scan every single email you receive or send for targeted advertising, customer behavior predictive analysis,
selling to data brokers, and who knows what other dystopian purposes.

Due to the aforementioned points, it's reasonable to say my setup is pretty safe from corporate data harvesting.
However, as discussed, Oracle could monitor or even alter anything going through my mail server; meaning that if my threat model was
a government or someone with government-level resources, then I could be compromised with relative ease.
However, if I was being specifically being targeted by such antagonist, I would definitely not be hosting on OCI nor describing my setup on the internet.

Finally, using a third party SMTP relay means that the corresponding company has access to the content of sent mail.
This is not of great concern to me since I very rarely send emails, my main worry was protecting my received messages.
Additionally, if transmitting sensitive information, I'd be using PGP. If my recipient can't or won't use PGP, then they're clearly not privacy-conscious;
even if I was using my own SMTP server, their provider and/or email client could still be eavesdropping on the conversation.
