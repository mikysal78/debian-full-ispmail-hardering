# debian-full-ispmail-hardering

Playbook Ansible modulare per l'hardening di server Debian 13 (trixie),
con ruoli opzionali per web server (nginx), mail server completo via
ispmail.sh (Postfix + Dovecot + Roundcube + Rspamd) e database (MariaDB).

## Struttura

```
01-hardening.yml                   # orchestrazione: common -> firewall/fail2ban -> ruoli di servizio
02-mailserver.yml                  # playbook standalone: installa Postfix+Dovecot+Roundcube via ispmail.sh
03-mx-backup.yml                   # playbook standalone opzionale: MX secondario (backup) relay-only
group_vars/all.yml.example         # tutte le opzioni di hardening, con commenti (copia -> all.yml)
group_vars/mailservers.yml.example # role_mailserver: true per il gruppo [mailservers]
group_vars/mx_backup.yml.example   # variabili del gruppo [mx_backup] (opzionale)
inventory/hosts.ini.example         # inventario di esempio
roles/
  common/      utenti, SSH, sysctl, moduli kernel, PAM, auditd, AppArmor,
               aggiornamenti automatici + needrestart, NTP, journald,
               banner legale, MOTD dinamico; opzionali /etc/hosts e
               /etc/network/interfaces
  firewall/    nftables (default-deny in ingresso)
  fail2ban/    ban automatico su SSH e, se attivi, nginx/postfix/dovecot
  webserver/   nginx: header di sicurezza, TLS moderno, no listing
  mailserver/  Postfix + Dovecot: TLS obbligatorio, anti-relay, SASL
  database/    MariaDB: bind locale, niente utenti anonimi/test/root remoto
  ispmail/     scarica ed esegue ispmail.sh (usato da 02-mailserver.yml)
  ispmail_admin/ pannello web opzionale per domini/mailbox/alias (usato da 02-mailserver.yml)
  ispmail_sni/ certificati TLS dedicati per domini extra via SNI (usato da 02-mailserver.yml)
  ispmail_autoconfig/  autoconfig Thunderbird, autodiscover Outlook, profilo iPhone per ogni dominio SNI
  ispmail_webmail_sni/ Roundcube anche su mail.<dominio> e webmail.<dominio> per ogni dominio SNI
  ispmail_dkim/        genera le chiavi DKIM per rspamd
  ispmail_postscreen/  postscreen + DNSBL sulla porta 25, unbound come resolver locale
  ispmail_carddav/     Radicale (CardDAV/CalDAV) + plugin RCMCardDAV in Roundcube
  ispmail_submission_privacy/ niente Received con l'IP del client nella posta autenticata
  mx_backup/   Postfix relay-only per un MX secondario (usato da 03-mx-backup.yml, opzionale)
```

## Setup iniziale (file privati)

I file con i dati reali della tua infrastruttura (chiave pubblica SSH,
IP/reti whitelistate, domini mail/web, host dell'inventario) sono in
`.gitignore` e NON vengono committati: nel repo trovi solo i template
`*.example`. Al primo utilizzo copiali senza suffisso e personalizzali:

```bash
cp group_vars/all.yml.example group_vars/all.yml
cp group_vars/mailservers.yml.example group_vars/mailservers.yml
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
   ansible-playbook 01-hardening.yml -u root -k -e ansible_ssh_pass=xxx -e ansible_port=22
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

`group_vars/mailservers.yml` attiva già `role_mailserver` per quel
gruppo. Il ruolo `webserver` (nginx) non ha più un `group_vars/` dedicato
(rimosso perché finora inutilizzato, per tenere puliti i file): se ti
serve un web server nginx, crea `group_vars/webservers.yml` con
`role_webserver: true` (stesso pattern di `mailservers.yml`), o imposta
il flag direttamente in `host_vars/<nome-host>.yml`. Stesso discorso per
`role_database` (ruolo `database`, MariaDB) e `group_vars/dbservers.yml`.

Il ruolo `firewall` legge questi stessi flag per aprire automaticamente
le porte giuste (80/443 per il web, 25/465/587/143/993 per la mail,
995 solo con `mail_enable_pop3: true`).

### Installare la posta con ispmail.sh (02-mailserver.yml)

In alternativa al ruolo `mailserver` di questo playbook, `02-mailserver.yml` è
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
   `ispmail_fqdn` in `/etc/ssl/acme/`, poi `01-hardening.yml` (hardening +
   firewall), poi:
   ```bash
   ansible-playbook 02-mailserver.yml -e ispmail_fqdn=mail.tuodominio.it
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
- Se il certificato manca in `/etc/ssl/acme/` quando lanci `02-mailserver.yml`,
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
  download) prima di lanciare `02-mailserver.yml` su un server di produzione.

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

### Ospitare altri domini con un certificato TLS proprio (SNI)

Per ospitare *altri* domini di posta sullo stesso server (es.
`utente@altrodominio.it` accanto a quelle di `ispmail_fqdn`) **non serve
un certificato per dominio**: i client di posta si collegano comunque
all'hostname del server (`ispmail_fqdn`), che presenta sempre lo stesso
certificato. Basta aggiungere il dominio in ISPmail Admin (o via SQL) e
puntarci l'MX via DNS.

Serve un certificato dedicato solo se vuoi che, connettendosi a un
hostname *specifico* di quel dominio (es. `mail.altrodominio.it`), il
server presenti il certificato di quel dominio invece di quello di
`ispmail_fqdn` (TLS SNI - Server Name Indication). In tal caso:

1. Fai emettere un certificato per quel dominio dal ruolo `ansible-dns`
   (aggiungilo ad `acme_domains`), che lo deposita in `/etc/ssl/acme/`
   con la stessa convenzione "piatta" già usata per `ispmail_fqdn`.
2. Punta un record DNS dell'hostname scelto (es. `mail.altrodominio.it`)
   a questo host.
3. Configura:
   ```yaml
   ispmail_sni_domains:
     - domain: "altrodominio.it"
       sni_hostname: "mail.altrodominio.it"
       cert_name: "altrodominio.it"   # default: = domain
   ```

Il ruolo `ispmail_sni` configura sia Postfix (`tls_server_sni_maps`) sia
Dovecot (blocchi `local_name`) per presentare il certificato giusto in
base all'hostname richiesto dal client. Se il certificato manca ancora in
`/etc/ssl/acme/`, il ruolo si ferma con un errore esplicito invece di
proseguire con una configurazione rotta.

Oltre a `sni_hostname`, ricevono il certificato del dominio anche
`imap.`, `smtp.`, `pop3.` e `pop.<dominio>` più il dominio stesso
(`ispmail_sni_alias_prefixes`); con `sni_aliases` su una voce indichi
invece la lista esatta. Il certificato deve coprire questi nomi
(wildcard o SAN).

**Rinnovi**: Dovecot legge i certificati tramite link, che seguono da
soli i rinnovi. Postfix invece vuole un unico file chiave+catena per
dominio, che `postmap -F` copia dentro la mappa: lo riallinea lo script
`/usr/local/sbin/sni-postfix-rinnova`, lanciato da
`sni-postfix-rinnova.path` quando cambia un certificato in
`/etc/ssl/acme/` e da `sni-postfix-rinnova.timer` ogni giorno alle 4:40
come rete di sicurezza. Se chiave e certificato non corrispondono
(rinnovo a metà) salta il dominio e riprova al file successivo. Log:
`journalctl -t sni-postfix-rinnova`.

I domini di `ispmail_sni_domains` sono anche quelli per cui i ruoli
`ispmail_autoconfig` e `ispmail_webmail_sni` (vedi sotto) creano i vhost
dedicati.

### Chiavi DKIM per rspamd

`ispmail.sh` predispone l'infrastruttura DKIM di rspamd (milter verso
Postfix, path e mappa dei selettori in
`/etc/rspamd/local.d/dkim_signing.conf`) ma **non genera le chiavi**: è
un `# TODO: create a key for $FQDN and test it` mai completato nello
script stesso. Il ruolo `ispmail_dkim` genera la chiave per ogni dominio
in `ispmail_dkim_domains`:

```yaml
ispmail_dkim_domains:
  - domain: "example.com"
    selector: "mail"
```

La chiave viene generata **una sola volta** (mai rigenerata ai run
successivi: rigenerarla invaliderebbe il record DNS già pubblicato). A
fine run, il playbook mostra per ogni dominio il record DNS TXT esatto
da pubblicare (`<selector>._domainkey.<dominio>`) — copialo nella zona
DNS di quel dominio (via `ansible-dns` o a mano). Se lo perdi, lo trovi
di nuovo sul server in
`/var/lib/rspamd/dkim/<dominio>.<selettore>.dns.txt`.

### Caselle: quota, cartelle, POP3S, mail di benvenuto

Il ruolo `ispmail`, dopo lo script, completa la configurazione delle
caselle:
- **quota di default** `ispmail_quota_default` (sintassi Dovecot, es.
  `3G`) per tutte le caselle con `quota=0` nel DB; il valore in byte
  nella colonna `virtual_users.quota` (da ISPmail Admin) la sovrascrive
  per la singola casella;
- **cartelle standard** Drafts, Sent e Archive create e sottoscritte al
  primo login con il flag SPECIAL-USE, accanto a Junk e Trash di
  ispmail.sh (svuotate dopo 30 giorni);
- **POP3S** (995) in Dovecot solo se `mail_enable_pop3: true`; il POP3 in
  chiaro (110) resta sempre spento;
- **log di debug di Sieve** spento di default (`ispmail_sieve_debug`);
- **mail di benvenuto**: dopo aver creato una casella, sul server
  ```bash
  ispmail-benvenuto nuovo.utente@tuodominio.it
  ```
  invia il testo di `/etc/ispmail/benvenuto.txt`, personalizzato con
  indirizzo e dominio.

### Configurazione automatica dei client (ispmail_autoconfig)

Attivo di default (`ispmail_autoconfig_enabled`) per ogni dominio di
`ispmail_sni_domains`, con un vhost HTTPS `autoconfig.<dominio>`:
- `https://autoconfig.<dominio>/mail/config-v1.1.xml` per Thunderbird,
  K-9/Thunderbird per Android, Evolution, KMail;
- `autodiscover.<dominio>` per Outlook desktop (IMAP 993 + SMTP 465);
- `https://autoconfig.<dominio>/iphone` da aprire con Safari: scarica un
  profilo `.mobileconfig` già compilato per iPhone/iPad/Mac (non firmato,
  iOS lo segnala come "non verificato"), con rubrica e calendario se
  CardDAV è attivo.

In tutti i casi l'utente proposto è l'indirizzo completo, come vuole
Dovecot. Prerequisito non gestito qui: i record DNS `autoconfig.` e
`autodiscover.<dominio>`.

### Webmail su ogni dominio (ispmail_webmail_sni)

Attivo di default (`ispmail_webmail_sni_enabled`): Roundcube risponde
anche su `https://mail.<dominio>` e `https://webmail.<dominio>` per ogni
dominio di `ispmail_sni_domains`, con il certificato di quel dominio. È
la stessa installazione del vhost principale, cambia solo l'indirizzo
nel browser. Il dominio il cui `sni_hostname` coincide con `ispmail_fqdn`
è saltato (lo serve già il vhost di ispmail.sh). Servono i record DNS.

### Filtro DNSBL sulla porta 25 (ispmail_postscreen)

`ispmail_postscreen_enabled: true` mette postscreen davanti alla porta
25: ogni connessione viene confrontata con le DNSBL (Spamhaus zen,
Mailspike, SpamCop, con DNSWL come compensazione) e con il test del
banner, prima di occupare un processo smtpd e arrivare a rspamd. Le
porte submission (587/465) non passano da qui. Liste, pesi e soglia sono
in `roles/ispmail_postscreen/defaults/main.yml`; per un periodo di sola
osservazione metti le azioni a `"ignore"`.

Il ruolo installa anche **unbound** come resolver ricorsivo locale e
punta `/etc/resolv.conf` a `127.0.0.1`: Spamhaus non risponde ai
resolver pubblici (8.8.8.8, 1.1.1.1).

### Rubriche e calendari (ispmail_carddav)

`ispmail_carddav_enabled: true` installa **Radicale** (CardDAV/CalDAV)
dietro Apache su `https://<ispmail_fqdn>/radicale/`, con le stesse
credenziali della casella: rubrica e calendario personali per ogni
casella, più rubrica e calendario condivisi per ogni dominio. In
Roundcube il plugin RCMCardDAV mostra un'unica rubrica; il calendario si
usa solo da client esterni (smartphone, Thunderbird), perché Debian non
ha un plugin calendario per Roundcube.

### Privacy della posta inviata (ispmail_submission_privacy)

Attivo di default: la posta inviata da utenti autenticati (porte
587/465, client o Roundcube) esce senza l'intestazione `Received` con IP
e nome del PC del mittente. L'IP resta nei log di Postfix e rspamd lo
vede comunque, quindi antispam e DKIM non cambiano. La porta 25 non è
toccata. Altre intestazioni da togliere (es. `User-Agent`) si aggiungono
in `ispmail_submission_privacy_header_checks`.

## MX secondario / backup (03-mx-backup.yml, opzionale)

`03-mx-backup.yml` configura un **secondo server**, separato dal mail
server principale, come **backup MX**: riceve la posta per i domini
indicati quando il primario non è raggiungibile, e la mette in coda per
consegnarla lì appena torna su. È solo Postfix — **nessun Dovecot,
nessuna mailbox locale**: le caselle restano sul server primario.

Disattivato di default: finché non aggiungi un host al gruppo
`[mx_backup]` dell'inventario, questo playbook non fa nulla.

Per attivarlo:
1. Aggiungi l'host in `inventory/hosts.ini` sotto `[mx_backup]`.
2. Esegui prima `01-hardening.yml` su quell'host.
3. Fai emettere un certificato per il suo hostname dal ruolo
   `ansible-dns` (stessa convenzione di `roles/ispmail`).
4. In `group_vars/mx_backup.yml` (copiala da `.example`) imposta:
   ```yaml
   mx_backup_fqdn: "mx2.tuodominio.it"          # hostname di QUESTO server
   mx_backup_primary_host: "mail.tuodominio.it" # il server primario
   mx_backup_relay_domains:
     - "tuodominio.it"
   firewall_allowed_tcp_ports: [25]
   ```
5. Aggiungi un record MX secondario (priorità/numero più alto, quindi
   meno prioritario) su ogni dominio in `mx_backup_relay_domains`, che
   punti a `mx_backup_fqdn`.
6. `ansible-playbook 03-mx-backup.yml`

**Backscatter da conoscere**: questa configurazione valida i
destinatari solo a livello di **dominio** (`relay_domains`), non di
singola casella — accetta quindi temporaneamente mail anche per
indirizzi inesistenti, che il primario poi rifiuta in fase di relay,
generando un bounce da questo server verso il mittente (rischio
backscatter se il mittente è falsificato, come spesso accade con lo
spam). Per una validazione più stretta serve una `relay_recipient_maps`
con i soli recipient validi (mappa statica sincronizzata dal primario,
o query diretta al suo DB aperta solo a questo host) — non inclusa di
default per tenere il setup semplice; il commento in
`roles/mx_backup/templates/main.cf.j2` indica dove aggiungerla.

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
ansible-playbook 01-hardening.yml
ansible-playbook 01-hardening.yml --limit webservers   # solo i webserver
ansible-playbook 01-hardening.yml --check --diff       # dry-run
```

## Note importanti da leggere prima di andare in produzione

- **TLS/certificati**: i ruoli `webserver` e `mailserver` partono con
  certificati "snakeoil" self-signed. Per il web, genera i certificati
  reali (es. `certbot --nginx -d tuodominio.it`) prima di attivare i
  vhost con `webserver_enable_tls: true` con domini reali in
  `webserver_domains`. Per la mail, sostituisci i path
  `smtpd_tls_cert_file`/`smtpd_tls_key_file` in
  `roles/mailserver/templates/main.cf.j2` con i tuoi certificati.
- **DKIM/SPF/DMARC**: con ispmail.sh le chiavi DKIM le genera il ruolo
  `ispmail_dkim` (vedi sopra); i record SPF e DMARC vanno comunque
  pubblicati nel DNS di ogni dominio. Il ruolo `mailserver` alternativo
  installa invece `opendkim` senza configurarlo.
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
