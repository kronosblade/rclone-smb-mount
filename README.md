# rclone-smb-mount
=======
# rclone SMB Mount con systemd

Mount automatico di una share Samba remota tramite rclone, gestito come servizio systemd user-level con ottimizzazioni per stabilita e performance.

## Stack

| Componente | Ruolo |
|------------|-------|
| **rclone** | Client per montare la share SMB come filesystem locale (FUSE) |
| **systemd (user)** | Gestione del ciclo di vita del mount: avvio, stop, restart automatico |
| **SMB/Samba** | Protocollo di condivisione file in rete (server remoto) |
| **FUSE** | Permette a rclone di esporre la share come directory locale |

## Struttura del progetto

```
.
├── README.md
├── rclone.conf.example          # Configurazione rclone (da copiare in ~/.config/rclone/)
└── rclone-mount.service.example # Unit systemd (da copiare in ~/.config/systemd/user/)
```

## Prerequisiti

- Linux con systemd
- `rclone` installato (`>= 1.50`)
- `fuse` o `fuse3` installato
- Server Samba raggiungibile in rete
- Utente e password validi sul server Samba

### Installazione dipendenze

**Arch Linux / CachyOS:**

```bash
sudo pacman -S rclone fuse3
```

**Debian / Ubuntu:**

```bash
sudo apt install rclone fuse3
```

**Fedora:**

```bash
sudo dnf install rclone fuse3
```

## Installazione

### 1. Configurare rclone

Copiare il file di configurazione example e personalizzarlo:

```bash
mkdir -p ~/.config/rclone
cp rclone.conf.example ~/.config/rclone/rclone.conf
```

Editare `~/.config/rclone/rclone.conf` sostituendo i valori placeholder:

- `myremote` -> nome del remote (a scelta)
- `192.168.1.100` -> IP del server Samba
- `myuser` -> nome utente Samba

Impostare la password in modo sicuro (viene cifrata automaticamente da rclone):

```bash
rclone config password myremote pass LA_TUA_PASSWORD
```

### 2. Creare il mount point

```bash
sudo mkdir -p /local/mount
sudo chown $USER:$USER /local/mount
```

### 3. Testare la connessione

```bash
rclone lsd myremote:/percorso/remoto
```

### 4. Installare il servizio systemd

Copiare il file service e personalizzarlo:

```bash
mkdir -p ~/.config/systemd/user
cp rclone-mount.service.example ~/.config/systemd/user/rclone-mount.service
```

Editare il file sostituendo:

- `REMOTE_NAME` -> nome del remote configurato in rclone.conf
- `/remote/path` -> percorso della share sul server
- `/local/mount` -> percorso locale del mount point
- `YOUR_USER` -> il proprio nome utente

### 5. Abilitare e avviare il servizio

```bash
systemctl --user daemon-reload
systemctl --user enable --now rclone-mount.service
```

### 6. Verificare

```bash
systemctl --user status rclone-mount.service
ls /local/mount
```

## Parametri di ottimizzazione

| Flag | Valore | Effetto |
|------|--------|---------|
| `--vfs-cache-mode` | `full` | Cache completa in locale (lettura + scrittura) |
| `--vfs-cache-max-age` | `72h` | Mantiene la cache per 72 ore |
| `--vfs-cache-max-size` | `10G` | Limita la cache a 10GB su disco |
| `--vfs-read-chunk-size` | `16M` | Legge a blocchi di 16MB |
| `--vfs-read-chunk-size-limit` | `256M` | Crescita massima dei chunk per file grandi |
| `--vfs-read-ahead` | `128M` | Pre-carica 128MB (utile per streaming video) |
| `--buffer-size` | `32M` | Buffer in memoria per ogni file aperto |
| `--transfers` | `4` | Trasferimenti paralleli |
| `--dir-cache-time` | `10m` | Cache listing directory per 10 minuti |
| `--poll-interval` | `1m` | Polling modifiche remote ogni minuto |
| `--attr-timeout` | `1s` | Timeout attributi breve (evita blocchi del file manager) |
| `--no-modtime` | - | Non sincronizza i tempi di modifica (riduce latenza) |

## Gestione errori e disconnessioni

Il servizio e progettato per gestire interruzioni di rete senza bloccare il sistema:

| Meccanismo | Effetto |
|------------|---------|
| `Restart=on-failure` | Riavvia automaticamente il mount in caso di crash |
| `RestartSec=10` | Attende 10 secondi prima di ritentare |
| `StartLimitBurst=5` | Massimo 5 riavvii in 5 minuti (evita loop infiniti) |
| `fusermount -uz` | Smontaggio lazy allo stop: sblocca subito il mount point |
| `--timeout 30s` | Timeout I/O: le operazioni falliscono dopo 30s invece di bloccarsi |
| `--contimeout 15s` | Timeout connessione: fallisce rapidamente se il server e irraggiungibile |
| `--low-level-retries 10` | Ritentativi automatici a basso livello |
| `--retries 5` | Ritentativi ad alto livello |

## Pro

- **Trasparente:** la share remota appare come una cartella locale, utilizzabile da qualsiasi applicazione
- **Automatico:** systemd gestisce avvio al login, riavvio dopo errori e stop allo shutdown
- **Resiliente:** timeout e retry evitano che disconnessioni temporanee blocchino terminale o file manager
- **Cache locale:** riduce il traffico di rete e migliora la velocita di accesso ai file gia letti
- **User-level:** non richiede permessi root per gestire il servizio (solo per creare il mount point)
- **Portabile:** la configurazione e replicabile su qualsiasi macchina Linux con rclone

## Contro

- **Dipendenza da FUSE:** le performance sono inferiori a un mount nativo CIFS/SMB (`mount.cifs`)
- **Cache su disco:** la VFS cache puo occupare fino a 10GB di spazio locale
- **Latenza:** le operazioni su file non in cache hanno la latenza della rete
- **Non real-time:** con `--dir-cache-time 10m` le modifiche fatte sul server possono impiegare fino a 10 minuti per apparire
- **Polling:** il polling periodico genera traffico di rete anche quando non ci sono modifiche
- **Nessun lock SMB:** rclone non supporta il file locking nativo di SMB, potenziali conflitti in scrittura concorrente
- **Debugging:** in caso di problemi la diagnostica richiede di incrociare log di rclone, systemd e rete

## Comandi utili

```bash
# Stato del servizio
systemctl --user status rclone-mount.service

# Log in tempo reale
journalctl --user -u rclone-mount.service -f

# Riavviare il servizio
systemctl --user restart rclone-mount.service

# Fermare il servizio
systemctl --user stop rclone-mount.service

# Smontaggio manuale d'emergenza
fusermount -uz /local/mount
```

## Licenza

MIT
