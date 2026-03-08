
# Serveur Mail Auto-hébergé

Déploiement d'un serveur mail complet sur infrastructure Proxmox VE, avec Postfix, Dovecot, OpenDKIM et Bind9 sur domaine interne `homelab.local`.

---

## Objectifs

- Déployer un serveur DNS interne (Bind9) avec zone `homelab.local`
- Déployer un serveur mail fonctionnel (Postfix + Dovecot)
- Configurer SPF, DKIM et DMARC
- Valider la signature DKIM dans les headers
- Tester l'envoi et la réception depuis un client mail (Thunderbird)

---

## Stack technique

| Composant | Rôle | Port |
|-----------|------|------|
| Bind9 | DNS interne — zone `homelab.local` | 53 |
| Postfix | Serveur SMTP — envoi et réception | 25 |
| Dovecot | Serveur IMAP — consultation boîtes mail | 143 |
| OpenDKIM | Signature cryptographique des mails | 12301 (milter) |

---

## Infrastructure

```
Proxmox VE 8.4 (192.168.1.50)
├── dns-server (VM Debian 12)
│   ├── IP      : 192.168.1.28
│   ├── RAM     : 512 Mo
│   ├── Disque  : 10 Go
│   └── Service : Bind9
│
└── mail-server (VM Debian 12)
    ├── IP      : 192.168.1.27
    ├── RAM     : 2 Go
    ├── Disque  : 20 Go
    └── Services : Postfix + Dovecot + OpenDKIM

Laptop Debian 13 (192.168.1.24)
└── Thunderbird — client IMAP
```

---

## Partie 1 — DNS interne avec Bind9

### Installation

```bash
apt update && apt install -y bind9 bind9utils dnsutils
```

### Configuration des forwarders

`/etc/bind/named.conf.options` — permet la résolution DNS externe (internet) en plus de la zone locale :

```
options {
    directory "/var/cache/bind";
    forwarders {
        192.168.1.1;
        8.8.8.8;
    };
    forward only;
    dnssec-validation auto;
    listen-on-v6 { any; };
};
```

### Déclaration de la zone

`/etc/bind/named.conf.local` :

```
zone "homelab.local" {
    type master;
    file "/etc/bind/zones/homelab.local.db";
};
```

### Fichier de zone

`/etc/bind/zones/homelab.local.db` :

```
$TTL    604800
@       IN      SOA     dns-server.homelab.local. admin.homelab.local. (
                          2026030803  ; Serial
                          604800      ; Refresh
                          86400       ; Retry
                          2419200     ; Expire
                          604800 )    ; Negative Cache TTL

; Nameserver
@               IN      NS      dns-server.homelab.local.

; Enregistrements A
dns-server      IN      A       192.168.1.28
mail-server     IN      A       192.168.1.27

; Enregistrement MX
@               IN      MX      10 mail-server.homelab.local.

; SPF
@               IN      TXT     "v=spf1 ip4:192.168.1.27 -all"

; DMARC
_dmarc          IN      TXT     "v=DMARC1; p=reject; rua=mailto:admin@homelab.local"

; DKIM (clé découpée en segments de 255 caractères max)
default._domainkey      IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAz2OIb8FP2Kt2ytlh62bq5cK7HRuGfrbtvLIQQH8m2xhdKsn/sfQxiuIUROyUxhRGgkTmZvoMnMCxw4/Vx7IL8zh47CNdBjSQetn1zsKId/sFAiTwcZbSaTsLnuxZwFMBXLdNyzeZQe6Lhwm7jZQ3ssTU2Kgf4p1D1VoSlmJhwNhKhGB4InDbqlfisxsIg+JyFnd5joBHSSfmRp"
          "GdL9CgimAh8Bp36Bn8bC8lm/9VA1gZjII4miHxJRslM2P6/ejCVhIyozm8HoNX7o2S6j1tOLTf00iw3Q0tgjHURCm4CEu0cTySIHcW22Z6J6871l7MAPWe6YfqTgdg48irYUUukQIDAQAB" )
```

### Vérification et rechargement

```bash
named-checkconf
named-checkzone homelab.local /etc/bind/zones/homelab.local.db
systemctl reload bind9
```

### Tests de validation DNS

```bash
dig @192.168.1.28 mail-server.homelab.local A
dig @192.168.1.28 homelab.local MX
dig @192.168.1.28 homelab.local TXT
dig @192.168.1.28 default._domainkey.homelab.local TXT
```

### Pointer les VMs vers le DNS interne

Sur chaque VM (`dns-server` et `mail-server`) :

```bash
nano /etc/resolv.conf
```

```
nameserver 192.168.1.28
search homelab.local
```

---

## Partie 2 — Postfix (SMTP)

### Installation

```bash
apt update && apt install -y postfix
# Pendant l'install : Internet Site / homelab.local
```

### Configuration `/etc/postfix/main.cf`

```
# Identité
myhostname = mail-server.homelab.local
mydomain = homelab.local
myorigin = $mydomain

# Interfaces et protocoles
inet_interfaces = all
inet_protocols = all

# Domaines locaux
mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost

# Réseau autorisé
mynetworks = 127.0.0.0/8 192.168.1.0/24

# Pas de relay externe
relayhost =

# Boîtes mail format Maildir
home_mailbox = Maildir/

# Bannière SMTP (sans version pour la sécu)
smtpd_banner = $myhostname ESMTP

# Restrictions relay
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

# TLS
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may
smtp_tls_security_level=may

# Aliases
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# Divers
biff = no
append_dot_mydomain = no
compatibility_level = 3.6
mailbox_size_limit = 0
recipient_delimiter = +

# OpenDKIM milter
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:12301
non_smtpd_milters = inet:localhost:12301
```

### Hostname système

```bash
hostnamectl set-hostname mail-server.homelab.local
echo "homelab.local" > /etc/mailname
postfix reload
```

---

## Partie 3 — Dovecot (IMAP)

### Installation

```bash
apt install -y dovecot-core dovecot-imapd
```

### `/etc/dovecot/conf.d/10-mail.conf`

```
mail_location = maildir:~/Maildir
```

### `/etc/dovecot/conf.d/10-auth.conf`

```
disable_plaintext_auth = no
auth_mechanisms = plain login
auth_username_format = %Ln
```

> `%Ln` — supprime la partie `@domaine` du username. Nécessaire quand le client mail envoie `user@homelab.local` au lieu de juste `user`.

### Création des utilisateurs et boîtes mail

```bash
useradd -m -s /bin/bash thomas
useradd -m -s /bin/bash admin
passwd thomas
passwd admin

mkdir -p /home/thomas/Maildir/{new,cur,tmp}
mkdir -p /home/admin/Maildir/{new,cur,tmp}
chown -R thomas:thomas /home/thomas/Maildir
chown -R admin:admin /home/admin/Maildir
```

### Redémarrage

```bash
systemctl restart dovecot
ss -tlnp | grep ':143'
```

---

## Partie 4 — OpenDKIM

### Installation

```bash
apt install -y opendkim opendkim-tools
```

### `/etc/opendkim.conf`

```
Syslog                  yes
SyslogSuccess           yes
LogWhy                  yes
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no
OversignHeaders         From
KeyTable                /etc/opendkim/KeyTable
SigningTable            refile:/etc/opendkim/SigningTable
ExternalIgnoreList      /etc/opendkim/TrustedHosts
InternalHosts           /etc/opendkim/TrustedHosts
Socket                  inet:12301@localhost
PidFile                 /run/opendkim/opendkim.pid
UMask                   002
UserID                  opendkim
```

### Fichiers de configuration

`/etc/opendkim/TrustedHosts` :
```
127.0.0.1
localhost
192.168.1.0/24
homelab.local
mail-server.homelab.local
```

`/etc/opendkim/KeyTable` :
```
default._domainkey.homelab.local homelab.local:default:/etc/opendkim/keys/homelab.local/default.private
```

`/etc/opendkim/SigningTable` :
```
*@homelab.local default._domainkey.homelab.local
```

### Génération de la clé DKIM

```bash
mkdir -p /etc/opendkim/keys/homelab.local
opendkim-genkey -b 2048 -d homelab.local -D /etc/opendkim/keys/homelab.local -s default -v
chown -R opendkim:opendkim /etc/opendkim/keys
```

La clé publique à publier en DNS :

```bash
cat /etc/opendkim/keys/homelab.local/default.txt
```

---

## Partie 5 — Validation

### Test envoi en ligne de commande

```bash
apt install -y mailutils
echo "Test DKIM" | mail -s "Test DKIM signature" thomas@homelab.local
cat /home/thomas/Maildir/new/<fichier>
```

### Lecture d'un header DKIM valide

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=homelab.local;
        s=default; t=1772995248;
        bh=+qlNhYuQiot52MMJ53YSV0rt4q35P6JeHyKLUgbd7Ek=;
        h=Subject:To:Date:From:From;
        b=IXzdr6Ey...
```

| Champ | Signification |
|-------|---------------|
| `a=rsa-sha256` | Algorithme de signature |
| `d=homelab.local` | Domaine signataire |
| `s=default` | Sélecteur DKIM |
| `bh=` | Hash du corps du mail |
| `b=` | Signature cryptographique |

### Configuration Thunderbird

```
Serveur entrant IMAP
  Hôte     : 192.168.1.27
  Port     : 143
  Sécurité : Aucune
  Auth     : Mot de passe normal
  Username : thomas (sans @homelab.local)

Serveur sortant SMTP
  Hôte     : 192.168.1.27
  Port     : 25
  Sécurité : Aucune
  Auth     : Mot de passe normal
```

---

## Problèmes rencontrés et résolutions

### 1. APT — erreur de résolution DNS après modification de `/etc/resolv.conf`

**Symptôme :**
```
Erreur temporaire de résolution de « deb.debian.org »
```

**Cause :** Bind9 configuré sans forwarders — il ne savait pas résoudre les domaines externes.

**Fix :** Ajout des forwarders dans `/etc/bind/named.conf.options` :
```
forwarders { 192.168.1.1; 8.8.8.8; };
forward only;
```

---

### 2. Clé DKIM refusée par Bind9 — syntax error

**Symptôme :**
```
dns_rdata_fromtext: syntax error
zone homelab.local/IN: not loaded due to errors.
```

**Cause :** La clé DKIM RSA 2048 bits dépasse la limite de 255 caractères par segment TXT DNS.

**Fix :** Découper la clé en plusieurs segments entre parenthèses :
```
default._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
    "p=MIIBIjAN...première moitié..."
    "seconde moitié...IDAQAB" )
```

---

### 3. Dovecot — `Auth process broken` au démarrage

**Symptôme :**
```
auth: Fatal: Unknown authentication mechanism 'PLAI'
```

**Cause :** Faute de frappe dans `/etc/dovecot/conf.d/10-auth.conf` — `plain` avait été tronqué en `plai`.

**Fix :**
```
auth_mechanisms = plain login
```

---

### 4. Thunderbird — échec d'authentification IMAP

**Symptôme :**
```
pam_unix(dovecot:auth): check pass; user unknown
ruser=thomas@homelab.local
```

**Cause :** Thunderbird envoie `thomas@homelab.local` comme username, mais PAM cherche `thomas` dans `/etc/passwd`.

**Fix :** Ajout de `auth_username_format = %Ln` dans `/etc/dovecot/conf.d/10-auth.conf`. Le paramètre `%Ln` tronque tout ce qui suit le `@`.

---

### 5. `From` des mails affichant `root@mail-server` au lieu de `root@homelab.local`

**Symptôme :** Header `From: root <root@mail-server>` sans le domaine complet.

**Cause :** Le hostname système était `mail-server` au lieu de `mail-server.homelab.local`.

**Fix :**
```bash
hostnamectl set-hostname mail-server.homelab.local
echo "homelab.local" > /etc/mailname
postfix reload
```

---

### 6. DNS — résolution `.local` interceptée par mDNS

**Symptôme :** `opendkim-testkey` retourne `record not found` malgré un enregistrement DNS valide.

**Cause :** Le suffixe `.local` est réservé au protocole mDNS (Multicast DNS / Avahi). Certains outils système envoient les requêtes `.local` en mDNS plutôt qu'en DNS unicast.

**Contournement :** Validation directe avec `host` en spécifiant le serveur DNS explicitement :
```bash
host -t TXT default._domainkey.homelab.local 192.168.1.28
```
La signature DKIM a été validée par l'envoi réel d'un mail et l'inspection du header `DKIM-Signature`.

---

## Résultat final

Mail envoyé depuis Thunderbird (`thomas@homelab.local`) vers `admin@homelab.local` :

```
Return-Path: <thomas@homelab.local>
DKIM-Signature: v=1; a=rsa-sha256; d=homelab.local; s=default;
Received: from [192.168.1.24] by mail-server.homelab.local (Postfix)
From: Thomas <thomas@homelab.local>
Subject: test circuit complet
```

✅ Postfix reçoit et délivre les mails  
✅ OpenDKIM signe chaque mail sortant  
✅ Dovecot expose les boîtes via IMAP  
✅ Thunderbird se connecte et envoie  
✅ SPF, DKIM, DMARC publiés en DNS interne  

