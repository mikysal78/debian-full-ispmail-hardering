# Hardening server Debian trixie (Ansible)

Playbook Ansible modulare per l'hardening di server Debian 13 (trixie),
con ruoli opzionali per web server (nginx), mail server (Postfix +
Dovecot) e database (MariaDB).

## Struttura

```
playbook.yml                       # orchestrazione: common -> firewall/fail2ban -> ruoli di servizio
ispmail.yml                        # playbook standalone: installa Postfix+Dovecot+Roundcube via ispmail.sh
group_vars/all.yml.example         # tutte le opzioni di hardening, con commenti (copia -> all.yml)
group_vars/webservers.yml.example  # role_webserver: true per il gruppo [webservers]
group_vars/mailservers.yml.example # role_mailserver: true per il gruppo [mailservers]
group_vars/dbservers.yml.example   # role_database: true per il gruppo [dbservers]
inventory/hosts.ini.example         # inventario di esempio
roles/
  common/      utenti, SSH, sysctl, moduli kernel, PAM, auditd, AppArmor,
               aggiornamenti automatici, NTP, journald, banner legale
  firewall/    nftables (default-deny in ingresso)
  fail2ban/    ban automatico su SSH e, se attivi, nginx/postfix/dovecot
  webserver/   nginx: header di sicurezza, TLS moderno, no listing
  mailserver/  Postfix + Dovecot: TLS obbligatorio, anti-relay, SASL
  database/    MariaDB: bind locale, niente utenti anonimi/test/root remoto
  ispmail/     scarica ed esegue ispmail.sh (usato da ispmail.yml)
  ispmail_admin/ pannello web opzionale per domini/mailbox/alias (usato da ispmail.yml)
```

## Setup iniziale (file privati)

I file con i dati reali della tua infrastruttura (chiave pubblica SSH,
IP/reti whitelistate, domini mail/web, host dell'inventario) sono in
`.gitignore` e NON vengono committati: nel repo trovi solo i template
`*.example`. Al primo utilizzo copiali senza suffisso e personalizzali:

```bash
cp group_vars/all.yml.example group_vars/all.yml
cp group_vars/webservers.yml.example group_vars/webservers.yml
cp group_vars/mailservers.yml.example group_vars/mailservers.yml
cp group_vars/dbservers.yml.example group_vars/dbservers.yml
cp inventory/hosts.ini.example inventory/hosts.ini
```

Poi modifica `group_vars/all.yml` (chiave pubblica, whitelist fail2ban,
domini) e `inventory/hosts.ini` (IP/hostname reali) con i tuoi dati:
questi file restano solo in locale e non finiscono mai su GitHub.

## Requisiti

```bash
ansible-galaxy collection install -r requirements.yml
```

Serve `ansible-core >= 2.15` circa. Sui target: Debian trixie con
accesso root o sudo iniziale.

## Bootstrap (primo giro su un server nuovo)

Al primo accesso il server ha solo `root` e nessun firewall. Procedi così:

1. Compila `inventory/hosts.ini` con l'IP/hostname reale, nel gruppo giusto
   (`[webservers]`, `[mailservers]`, `[dbservers]` a seconda dei ruoli).
2. In `group_vars/all.yml` imposta:
   - `harden_create_admin_user: true`
   - `harden_admin_user` e soprattutto `harden_admin_pubkey` (la tua chiave
     pubblica reale)
   - lascia `ssh_password_authentication: "no"` solo se sei sicuro che la
     chiave funzioni; altrimenti impostala temporaneamente a `"yes"` per il
     primo giro e rimettila a `"no"` una volta verificato l'accesso.
3. Esegui il primo run autenticandoti come root sulla porta 22 di fabbrica
   (anche se nell'inventario hai già messo `ansible_port=2400` per i run
   successivi, va sovrascritto per questo primo giro):
   ```bash
   ansible-playbook playbook.yml -u root -k -e ansible_ssh_pass=xxx -e ansible_port=22
   ```
   (o con chiave se il provider l'ha già installata: `-u root -e ansible_port=22`)
4. **Verifica in un'altra finestra di terminale, PRIMA di chiudere la sessione
   corrente**, che riesci a collegarti con il nuovo utente sulla nuova porta:
   ```bash
   ssh -p 2400 <harden_admin_user>@<ip-server>
   ```
   Il playbook include un `assert` che blocca l'esecuzione se stai per
   disabilitare sia il login root che l'autenticazione a password senza aver
   configurato un utente admin con chiave — ma la prudenza (secondo terminale
   aperto) resta comunque consigliata quando si cambia la configurazione SSH
   da remoto.
5. Dai run successivi aggiungi `ansible_user=<harden_admin_user>
   ansible_port=2400` alla riga dell'host in `inventory/hosts.ini`, così non
   devi più passare `-u`/`-e` a mano.

## Porta SSH

Il playbook è già configurato con `ssh_port: 2400` (vedi
`group_vars/all.yml`). Il ruolo `firewall` apre automaticamente quella
porta in ingresso (non la 22), e il ruolo `fail2ban` monitora la jail
`sshd` sulla stessa porta. Se cambi ancora la porta, modifica solo
`ssh_port` in `group_vars/all.yml`: si propaga da sola a sshd, nftables
e fail2ban.

## Attivare i ruoli opzionali

Il modo più semplice è mettere l'host nel gruppo giusto in
`inventory/hosts.ini`:

```ini
[webservers]
web01 ansible_host=203.0.113.10

[mailservers]
mail01 ansible_host=203.0.113.11
```

`group_vars/webservers.yml` e `group_vars/mailservers.yml` attivano già
`role_webserver`/`role_mailserver` per quei gruppi. In alternativa puoi
forzare i flag per singolo host in `host_vars/<nome-host>.yml`.

Il ruolo `firewall` legge questi stessi flag per aprire automaticamente
le porte giuste (80/443 per il web, 25/465/587/143/993/995 per la mail).

### Installare la posta con ispmail.sh (ispmail.yml)

In alternativa al ruolo `mailserver` di questo playbook, `ispmail.yml` è
un playbook standalone che scarica ed esegue
[ispmail.sh](https://workaround.org/ispmail.sh) di Christoph Haas: installa
Postfix + Dovecot + Roundcube + Rspamd + MariaDB con TLS via Let's Encrypt,
seguendo la guida [ISPmail](https://workaround.org).

Il certificato TLS **non** lo richiede ispmail.sh (che lo farebbe via
`certbot --apache`, validazione HTTP-01): lo provisiona invece il ruolo
`ansible-dns` (validazione DNS-01), che lo copia in `/etc/ssl/acme/`. Il
ruolo `ispmail` lo copia da lì nel path standard di certbot
(`/etc/letsencrypt/live/<fqdn>/`) **prima** di lanciare lo script, che a
quel punto lo trova già presente e salta da solo la propria chiamata a
certbot.

1. In `group_vars/all.yml` (o `group_vars/mailservers.yml`) imposta:
   ```yaml
   role_mailserver: true              # firewall/fail2ban aprono comunque le porte mail
   mailserver_manage_service: false   # il ruolo Ansible "mailserver" di questo repo NON gira
   firewall_allowed_tcp_ports: [80, 443]   # servono a Roundcube/rspamd-ui via Apache (non più a certbot)
   ```
   `role_webserver` deve restare `false` su questi host: ispmail.sh installa
   Apache, non nginx, e i due andrebbero in conflitto sulle porte 80/443.
2. Esegui prima il ruolo `ansible-dns` per ottenere il certificato di
   `ispmail_fqdn` in `/etc/ssl/acme/`, poi `playbook.yml` (hardening +
   firewall), poi:
   ```bash
   ansible-playbook ispmail.yml -e ispmail_fqdn=mail.tuodominio.it
   ```
   (oppure valorizza `ispmail_fqdn` in `group_vars/mailservers.yml`/`host_vars`).

Note:
- Il ruolo si aspetta i file del certificato "piatti" sotto
  `/etc/ssl/acme/<nome>.fullchain.pem` e `<nome>.key`. Se `<nome>` non
  coincide con `ispmail_fqdn` (es. certificato sul dominio apex/wildcard
  invece che sul sottodominio `mail.*`, come nel caso tipico di un
  dominio mail su un sottodominio), imposta `ispmail_cert_name` di
  conseguenza — **verifica anche che quel certificato copra davvero
  l'FQDN del mail server** (SAN o wildcard), altrimenti Postfix/Dovecot/
  Roundcube presenteranno un certificato con nome diverso dall'hostname.
- Se il certificato manca in `/etc/ssl/acme/` quando lanci `ispmail.yml`,
  il ruolo si ferma subito con un errore esplicito (non lascia che sia
  ispmail.sh a scoprirlo a metà installazione).
- ispmail.sh normalmente si aspetta che sia `certbot --apache` (chiamato
  prima di configurare Apache) ad abilitare `mod_ssl` e il vhost
  `000-default-le-ssl.conf` di Roundcube. Siccome qui quella chiamata
  viene saltata, il ruolo `ispmail` **patcha lo script scaricato** per
  abilitare comunque `mod_ssl` e quel vhost — altrimenti l'installazione
  si fermerebbe da sola al controllo finale su Roundcube, prima ancora di
  arrivare a configurare TLS su Postfix, rspamd, DKIM e le quote.
- ispmail.sh installa un intero mail server in un solo colpo e non è
  pensato per essere rieseguito da zero: il ruolo `ispmail` lo esegue
  quindi una volta sola per host (marker `/root/.ispmail_installed`).
- Le password generate a fine installazione (DB `mailadmin`/`mailserver`,
  interfaccia web di rspamd) vengono stampate dallo script **solo una
  volta** e non sono recuperabili altrimenti. Il ruolo:
  1. mostra un riepilogo finale nell'output di `ansible-playbook` con URL
     di Roundcube/rspamd e tutte le password generate;
  2. lo salva anche sul **controller Ansible** (non sul mail server, di
     norma più esposto) in `secrets/<fqdn>.log`, permessi 0600;
  3. rimuove la copia dal server.

  `secrets/` è in `.gitignore` — copia comunque il contenuto in un
  password manager e poi cancella anche il file locale, es.
  `shred -u secrets/<fqdn>.log`.
- Lo script gira come root e scarica codice da un dominio di terze parti:
  è affidabile (guida ISPmail nota e mantenuta) ma, trattandosi comunque
  di uno script esterno eseguito con privilegi root, è buona norma
  rivederlo (`cat /usr/local/sbin/ispmail.sh` sul server dopo il primo
  download) prima di lanciare `ispmail.yml` su un server di produzione.

### Gestire domini/mailbox/alias: ISPmail Admin (opzionale)

`ispmail_admin_enabled: true` installa anche [ISPmail
Admin](http://ima.jungclaussen.com/), un pannello PHP che gestisce
direttamente le tabelle `virtual_domains`/`virtual_users`/`virtual_aliases`
create da `ispmail.sh` (aggiungere domini, caselle, alias, redirect —
alternativa a farlo via SQL diretto).

**Compromessi da conoscere prima di attivarlo**, perché non è un progetto
al livello di garanzie di `ispmail.sh`/workaround.org:
- si scarica **solo via HTTP** (`http://ima.jungclaussen.com`), **senza
  checksum pubblicati** — nessun modo di verificare l'integrità del download;
- richiede un utente MariaDB con **scrittura** (SELECT/UPDATE/INSERT/DELETE)
  su tutto il database `mailserver` — più ampio del semplice utente
  `mailserver` in sola lettura creato da ispmail.sh.

Per questo il ruolo `ispmail_admin`, per difendersi:
- lo espone su un **vhost dedicato** (`ispmail_admin_fqdn`, es.
  `ispmail.tuodominio.it` — non un path su `ispmail_fqdn`) ristretto via
  Apache (`Require ip`) solo alle reti in `ispmail_admin_allow_networks`
  (default: le stesse reti già whitelistate su fail2ban) — le porte 80/443
  restano pubbliche per Roundcube/rspamd-ui su `ispmail_fqdn`, ma questo
  vhost separato no;
- genera e salva da solo utente/password admin e password del DB
  sul controller Ansible in `secrets/<fqdn>-ima-admin.txt` e
  `secrets/<fqdn>-ima-db.txt` (stessa cartella, stesso trattamento delle
  password di ispmail.sh — copiale in un password manager);
- crea l'utente DB con permessi limitati alla sola tabella `mailserver`,
  non `ALL PRIVILEGES`.

`ispmail_admin_fqdn` deve risolvere via DNS verso questo host ed essere
coperto dal certificato TLS di `ispmail_fqdn` (SAN o wildcard) — il ruolo
non lo verifica automaticamente, solo un `assert` che sia stato impostato.

Se preferisci non installarlo, i domini/mailbox/alias restano comunque
gestibili a mano via SQL sul database `mailserver` (stesso schema che usa
ISPmail Admin: `virtual_domains`, `virtual_users`, `virtual_aliases`).

### Un hostname diverso per il webmail

Di default Roundcube risponde solo su `https://<ispmail_fqdn>/` (lo stesso
hostname usato per SMTP/IMAP). Se preferisci un hostname dedicato per il
webmail (es. `webmail.tuodominio.it`), imposta:

```yaml
ispmail_webmail_fqdn: "webmail.tuodominio.it"
```

Il ruolo aggiunge un `ServerAlias` al vhost di Roundcube scritto da
ispmail.sh: `ispmail_fqdn` resta l'hostname "vero" (Postfix/Dovecot/TLS),
`ispmail_webmail_fqdn` è solo un alias in più per raggiungerlo via browser.
Anche questo deve risolvere via DNS verso l'host ed essere coperto dal
certificato TLS.

## Whitelist fail2ban (IP/reti mai bannati)

`fail2ban_ignoreip` in `group_vars/all.yml` è una lista di IP/reti CIDR
che fail2ban ignora sempre, a prescindere dai tentativi falliti (utile
per il tuo IP fisso, la rete d'ufficio o una VPN di gestione):

```yaml
fail2ban_ignoreip:
  - "127.0.0.1/8"
  - "::1"
  - "203.0.113.10"      # il tuo IP fisso
  - "10.20.0.0/16"       # es. rete VPN/ufficio
```

Nota: questa whitelist agisce solo su fail2ban. Se vuoi che anche il
firewall (nftables) limiti la porta SSH alle sole reti fidate, usa
`firewall_allow_ssh_from` con le stesse reti (vedi sezione precedente).

## Esecuzione

```bash
ansible-playbook playbook.yml
ansible-playbook playbook.yml --limit webservers   # solo i webserver
ansible-playbook playbook.yml --check --diff       # dry-run
```

## Note importanti da leggere prima di andare in produzione

- **TLS/certificati**: i ruoli `webserver` e `mailserver` partono con
  certificati "snakeoil" self-signed. Per il web, genera i certificati
  reali (es. `certbot --nginx -d tuodominio.it`) prima di attivare i
  vhost con `webserver_enable_tls: true` con domini reali in
  `webserver_domains`. Per la mail, sostituisci i path
  `smtpd_tls_cert_file`/`smtpd_tls_key_file` in
  `roles/mailserver/templates/main.cf.j2` con i tuoi certificati.
- **DKIM/SPF/DMARC**: il ruolo mailserver installa `opendkim` ma non lo
  configura (chiavi/domini sono specifici per ognuno): vanno impostati a
  mano o con un ruolo dedicato prima di considerare la mail "pronta".
- **PAM e `pam-auth-update`**: `roles/common/tasks/pam.yml` modifica
  direttamente `/etc/pam.d/common-auth` e `common-password` per
  aggiungere `pam_faillock`/`pam_pwquality`. Se in futuro lanci
  `pam-auth-update` manualmente sul server, questi blocchi potrebbero
  essere sovrascritti: rilancia il playbook per ripristinarli.
- **Regole di audit immutabili**: `roles/common/templates/audit.rules.j2`
  ha la riga `-e 2` (modalità immutabile) commentata di proposito, perché
  altrimenti i run successivi del playbook fallirebbero nel ricaricare le
  regole finché non si riavvia il server. Abilitala manualmente quando la
  configurazione è stabile.
- **Firewall e accesso SSH**: se non imposti `firewall_allow_ssh_from`,
  la porta SSH resta raggiungibile da qualunque IP (con rate-limiting +
  fail2ban). Se il server ha un IP amministrativo fisso, valorizza quella
  variabile per restringere l'accesso.
- **MariaDB**: il ruolo `database` usa i moduli `community.mysql.*` con
  `login_unix_socket`, quindi funziona anche prima di aver impostato una
  password per root (autenticazione locale via socket, comportamento di
  default su Debian). Se in futuro imposti una password per root, dovrai
  passare `login_password` ai task o creare un file `.my.cnf`.
