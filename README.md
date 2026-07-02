# Hardening server Debian trixie (Ansible)

Playbook Ansible modulare per l'hardening di server Debian 13 (trixie),
con ruoli opzionali per web server (nginx), mail server (Postfix +
Dovecot) e database (MariaDB).

## Struttura

```
playbook.yml              # orchestrazione: common -> firewall/fail2ban -> ruoli di servizio
group_vars/all.yml         # tutte le opzioni di hardening, con commenti
group_vars/webservers.yml  # role_webserver: true per il gruppo [webservers]
group_vars/mailservers.yml # role_mailserver: true per il gruppo [mailservers]
group_vars/dbservers.yml   # role_database: true per il gruppo [dbservers]
inventory/hosts.ini         # inventario di esempio
roles/
  common/      utenti, SSH, sysctl, moduli kernel, PAM, auditd, AppArmor,
               aggiornamenti automatici, NTP, journald, banner legale
  firewall/    nftables (default-deny in ingresso)
  fail2ban/    ban automatico su SSH e, se attivi, nginx/postfix/dovecot
  webserver/   nginx: header di sicurezza, TLS moderno, no listing
  mailserver/  Postfix + Dovecot: TLS obbligatorio, anti-relay, SASL
  database/    MariaDB: bind locale, niente utenti anonimi/test/root remoto
```

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
3. Esegui il primo run autenticandoti come root:
   ```bash
   ansible-playbook playbook.yml -u root -k -e ansible_ssh_pass=xxx
   ```
   (o con chiave se il provider l'ha già installata: `-u root`)
4. **Verifica in un'altra finestra di terminale, PRIMA di chiudere la sessione
   corrente**, che riesci a collegarti con il nuovo utente sulla nuova porta:
   ```bash
   ssh -p 2400 admin@<ip-server>
   ```
   Il playbook include un `assert` che blocca l'esecuzione se stai per
   disabilitare sia il login root che l'autenticazione a password senza aver
   configurato un utente admin con chiave — ma la prudenza (secondo terminale
   aperto) resta comunque consigliata quando si cambia la configurazione SSH
   da remoto.
5. Dai run successivi, aggiorna `ansible.cfg` (`remote_user`) o
   l'inventario per usare l'utente admin sulla porta 2400.

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
le porte giuste (80/443 per il web, 25/587/465/143/993 per la mail).

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
