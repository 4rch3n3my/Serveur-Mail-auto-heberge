# Serveur Mail Auto-hébergé

Déploiement d'un serveur mail complet sur infrastructure Proxmox VE, avec configuration SPF, DKIM et DMARC sur un domaine interne `homelab.local`.

---

## Objectifs

- Déployer un serveur mail fonctionnel (Postfix + Dovecot) sur Debian 12
- Configurer un DNS interne (Bind9) avec la zone `homelab.local`
- Mettre en place les enregistrements SPF, DKIM et DMARC
- Tester l'envoi et la réception entre deux comptes locaux
- Comprendre et analyser les headers mail

---
| # | Projet | Description | Status |
|---|---|---|---|
| 00 | [Proxmox Setup](./00-Proxmox-Setup/README.md) | Installation et configuration d'un serveur mail Postfix / dovecot local | ✅ Terminé |
---

## Stack technique

| Composant | Rôle |
|-----------|------|
| Postfix | Serveur SMTP — envoi et réception des mails |
| Dovecot | Serveur IMAP — consultation des boîtes mail |
| OpenDKIM | Signature cryptographique des mails sortants |
| Bind9 | DNS interne — résolution de `homelab.local` |

---

## Infrastructure

```
Proxmox VE 8.4 (192.168.1.50)
├── mail-server (VM Debian 12)
│   ├── IP      : 192.168.1.26
│   ├── RAM     : 2 Go
│   ├── Disque  : 20 Go
│   └── Services : Postfix + Dovecot + OpenDKIM
│
└── dns-server (VM Debian 12 minimal)
    ├── IP      : 192.168.1.27
    ├── RAM     : 512 Mo
    ├── Disque  : 10 Go
    └── Services : Bind9
```

---

## Enregistrements DNS cibles

Zone : `homelab.local`

| Type | Nom | Valeur | Rôle |
|------|-----|--------|------|
| A | `mail.homelab.local` | `192.168.1.26` | Résolution du serveur mail |
| MX | `homelab.local` | `mail.homelab.local` | Routage des mails entrants |
| TXT | `homelab.local` | `v=spf1 ip4:192.168.1.26 -all` | SPF — autorise l'IP à envoyer |
| TXT | `default._domainkey` | `v=DKIM1; k=rsa; p=<clé publique>` | DKIM — clé de signature |
| TXT | `_dmarc.homelab.local` | `v=DMARC1; p=reject; rua=mailto:admin@homelab.local` | DMARC — politique de rejet |

---

## Statut des étapes

- [ ] Création de la branche `feat/mail-server`
- [ ] Déploiement VM dns-server
- [ ] Déploiement VM mail-server
- [ ] Installation et configuration Bind9
- [ ] Configuration zone `homelab.local`
- [ ] Installation et configuration Postfix
- [ ] Installation et configuration Dovecot
- [ ] Installation et configuration OpenDKIM
- [ ] Tests d'envoi entre comptes locaux
- [ ] Analyse des headers mail
- [ ] Documentation finale

---

## Comptes mail de test

| Compte | Usage |
|--------|-------|
| `thomas@homelab.local` | Compte principal |
| `admin@homelab.local` | Compte de test + réception DMARC |

---

## Références

- [Documentation Postfix](https://www.postfix.org/documentation.html)
- [Documentation Dovecot](https://doc.dovecot.org/)
- [OpenDKIM](http://www.opendkim.org/)
- [Bind9 — Debian Wiki](https://wiki.debian.org/Bind9)
- [Comprendre SPF, DKIM, DMARC — Cloudflare](https://www.cloudflare.com/learning/email-security/dmarc-dkim-spf/)
