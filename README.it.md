# ![WireGuard](images/wireguard_logo.png) wgc - WireGuard Connection Manager

![Made with Bash](https://img.shields.io/badge/Made%20with-Bash-1f425f.svg)

**[🇬🇧 English](README.md) | [🇮🇹 Italiano](README.it.md)**

> Esegui e monitora più tunnel WireGuard isolati utilizzando i network namespace di Linux.

`wgc` è uno script bash per gestire connessioni WireGuard multiple e simultanee su un sistema Linux. La sua funzionalità principale è la capacità di eseguire VPN in **network namespace isolati** (`ip netns`) o nel namespace predefinito con routing basato su policy.

Ogni connessione VPN può essere attivata in un namespace isolato con la propria interfaccia di rete, tabella di routing e configurazione DNS, oppure nel namespace predefinito utilizzando regole di routing basate sulla sorgente. Questo permette a più VPN di essere attive contemporaneamente senza conflitti di route.

---

## Caratteristiche

* **Modalità Duale di Funzionamento:** 
  * **Modalità namespace (`nup`):** Isolamento completo con namespace dedicato, perfetto per eseguire applicazioni specifiche attraverso la VPN.
  * **Modalità predefinita (`up`):** Utilizza il namespace predefinito del sistema con routing basato su policy, ideale per l'utilizzo della VPN a livello di sistema.
* **Isolamento Totale:** Esegui più VPN simultaneamente in modalità namespace. Il traffico di ogni VPN è completamente separato dall'host e dalle altre VPN.
* **Esecuzione Mirata:** Esegui comandi o applicazioni specifiche (come `curl`, `ssh`, `bash`, un browser, un servizio) *all'interno* di un namespace VPN. Questo instrada solo il traffico di quella applicazione attraverso il tunnel.
* **DNS Automatico:** Configura automaticamente i server DNS specificati nel file `.conf` (tramite la chiave `DNS =`).
* **Gestione Intelligente dei Processi:** Terminazione controllata dei processi in esecuzione nei namespace con timeout e indicatore di progresso.
* **Arresto Flessibile:** Opzione per arrestare forzatamente le VPN anche quando ci sono processi in esecuzione nel loro namespace.
* **Interfaccia Semplice:** Un singolo script con comandi chiari per avviare, fermare, elencare e monitorare i tunnel.

## Requisiti

* `bash`
* Accesso `sudo` (lo script si auto-eleva se non eseguito come root) 
* `wireguard-tools` (fornisce il comando `wg`) 
* `iproute2` (fornisce il comando `ip`)
* `systemd` (fornisce il comando `resolvectl` per la gestione DNS)

## Installazione

1. Scarica il file [wgc](https://github.com/colemar/wgc/raw/refs/heads/main/wgc)

2. Rendi lo script eseguibile:
   
   ```bash
   chmod +x wgc
   ```

3. Sposta lo script in una directory nel tuo `$PATH`, come `/usr/local/bin`:
   
   ```bash
   sudo cp wgc /usr/local/bin/wgc
   ```

## Configurazione

I file di configurazione WireGuard (`.conf`) devono essere posizionati in `/etc/wireguard/` o in un'altra directory indicata dalla variabile d'ambiente `WGC_CFGDIR`.

```bash
export WGC_CFGDIR=/percorso/delle/tue/configurazioni
wgc list
```

Lo script utilizza il nome del file (senza l'estensione `.conf`) come identificatore della VPN. Ad esempio, un file in `$WGC_CFGDIR/proton-it.conf` sarà gestito come VPN `proton-it`.

**Nota:** I file di configurazione denominati secondo il pattern `wg[0-9]+` (es. `wg0.conf`, `wg1.conf`) sono esclusi dalla gestione per evitare conflitti con le interfacce di sistema.

Lo script analizza il file e si aspetta le chiavi standard di WireGuard:

* **Chiavi Obbligatorie:** Lo script terminerà se manca una qualsiasi di queste:
  * `Address` 
  * `PrivateKey` 
  * `PublicKey` 
  * `Endpoint` 
  * `AllowedIPs` 
* **Chiavi Opzionali:** Lo script supporta anche:
  * `ListenPort`
  * `DNS`
  * `MTU` 
  * `PresharedKey` 
  * `PersistentKeepalive` 

---

## Utilizzo

La sintassi generale è `wgc [comando] <nome_vpn>`.

Lo script richiede accesso `sudo` o root perché manipola interfacce di rete e namespace.

  ![](images/wgc.png)

### Comandi Principali

* **`up <vpn>`**
  Avvia la connessione VPN nel **namespace predefinito** utilizzando routing basato su policy. Le route della VPN sono applicate in base all'indirizzo sorgente, permettendo alla VPN di coesistere con la tua normale connessione di rete.
  
  Questo significa che un'applicazione (es. qBittorrent) deve essere associata all'interfaccia VPN o all'indirizzo ip per avere il suo traffico di rete reindirizzato attraverso il tunnel.
  
  Questo significa anche che se sei connesso via ssh, la tua sessione non si chiuderà quando avvii la VPN.
  
  ```bash
  wgc up proton-it
  ```
  
  ![](images/start.png)

* **`upd <vpn>`**
  Avvia la connessione VPN nel **namespace predefinito** utilizzando routing basato su policy **e** aggiungendo una ***route divisa*** per il tunnel (0.0.0.0/1, 128.0.0.0/1).
  
  Questo significa che qualsiasi applicazione avrà il suo traffico di rete instradato attraverso il tunnel VPN.
  
  Questo significa anche che se sei connesso via ssh la tua sessione verrà terminata, a meno che prima di avviare la VPN non aggiungi manualmente una route specifica per il tuo indirizzo ip.
  
  ```bash
  wgc upd proton-it
  ```

* **`nup <vpn>`**
  Avvia la connessione VPN nel suo **namespace isolato**. Questo fornisce un isolamento di rete completo ed è la modalità consigliata per eseguire applicazioni specifiche attraverso la VPN.
  
  Una route predefinita (0.0.0.0/0) verrà aggiunta per il tunnel VPN, quindi qualsiasi applicazione avrà il suo traffico di rete instradato attraverso di essa.
  
  **Nota:** Le applicazioni con listener di rete (es. interfacce web su porte TCP) saranno irraggiungibili dall'esterno del namespace VPN a meno che non configuri il port forwarding (supporto integrato in arrivo) o una *veth* (ethernet virtuale) che collega il namespace predefinito con il namespace VPN. Le applicazioni con interfacce GUI o terminale rimangono completamente accessibili.
  
  ```bash
  wgc nup proton-it
  ```
  
  ![](images/nup.png)

* **`down <vpn> [force]`**
  Arresta la connessione VPN.
  
  * Se la VPN è attiva nel namespace predefinito (avviata con `up/upd`), la arresta.
  * Se la VPN è attiva nel suo namespace (avviata con `nup`):
    * Se non ci sono processi in esecuzione nello stesso namespace, arresta la VPN.
    * Se ci sono processi in esecuzione nello stesso namespace, mostra l'elenco dei processi e rifiuta di arrestare la VPN. Se specifichi `force`, termina in maniera controllata tutti i processi nel namespace (SIGTERM), attende fino a 10 secondi mostrando una barra di progresso, poi uccide forzatamente i processi rimanenti (SIGKILL). Infine, arresta la VPN.
  
  ```bash
  wgc down proton-it
  wgc down proton-it force
  ```
  
  ![](images/stop.png)

* **`status <vpn>`**
  Mostra lo stato dettagliato della connessione, includendo:
  
  * Stato della connessione e namespace
  * Dettagli dell'interfaccia WireGuard
  * Indirizzi IP e route
  * Configurazione DNS
  * Processi in esecuzione (per la modalità namespace)
  
  ```bash
  wgc status proton-it
  ```
  
  ![](images/status.png)

* **`exec <vpn> <comando...>`**
  Esegue un comando *all'interno* del namespace della VPN. Funziona solo per le VPN avviate con `nup`.
  
    *Esempio: Controlla il tuo IP pubblico come visto dalla VPN.*
  
  ```bash
  wgc exec proton-it curl ipinfo.io
  ```
  
    *Esempio: Avvia una shell interattiva che vede solamente la rete VPN.*
  
  ```bash
  wgc exec proton-it bash
  ```
  
  ![](images/exec.png)

* **`list`**
  Elenca tutti i file `.conf` disponibili trovati nella directory di configurazione con i loro dettagli chiave (Address, AllowedIPs, Endpoint). 
  
  ```bash
  wgc list
  ```
  
  ![](images/list.png)

* **`active`**
  Elenca tutte le VPN *attualmente attive*, mostrando sia quelle nel namespace predefinito che quelle nei namespace isolati.
  
  ```bash
  wgc active
  ```
  
  ![](images/active.png)

### Scorciatoie dei Comandi

Alcuni comandi supportano la corrispondenza per prefisso. Ad esempio:

* `nup`, `nu`, `n` → `nup`
* `down`, `dow`, `do`, `d` → `down`
* `status`, `stat`, `st`, `s` → `status`
* `exec`, `exe`, `ex`, `e` → `exec`
* `active`, `activ`, `act`, `ac`, `a` → `active`
* `list`, `lis`, `li`, `l` → `list`

### Completamento Bash

Lo script può installare il proprio file di completamento bash con suggerimenti intelligenti:

1. Esegui il seguente comando:
   
   ```bash
   wgc completion
   ```

2. Questo creerà il file `/etc/bash_completion.d/wgc`. 

3. Carica il file o avvia una nuova shell per utilizzare il completamento:
   
   ```bash
   source /etc/bash_completion.d/wgc
   ```

4. Il completamento fornisce:
   
   * Completamento dei nomi dei comandi
   * Completamento dei nomi VPN basato sui file di configurazione disponibili (per `up`/`nup`)
   * Completamento delle VPN attive (per `down`/`status`/`exec`)
   * Completamento dei comandi di sistema dopo `exec <vpn>` (nota: inviare doppio TAB, senza avere fornito almeno un carattere per restringere la ricerca dei comandi, causa un ritardo notevole)

5. Per evitare richieste di password sudo durante il completamento, l'installatore fornisce regole `sudoers` opzionali.

6. Se `WGC_CFGDIR` viene modificato, il comando `completion` deve essere eseguito nuovamente per aggiornare la conoscenza del percorso della directory di configurazione.

## Confronto Modalità di Funzionamento

| Caratteristica        | Namespace Predefinito (`up`)             | Namespace Isolato (`nup`) |
| --------------------- | ---------------------------------------- | ------------------------- |
| Isolamento di rete    | Parziale (regole di routing).            | Completo.                 |
| Configurazione DNS    | A livello di sistema.                    | Specifica del namespace.  |
| Esecuzione processi   | Diretta.                                 | Tramite `wgc exec`.       |
| VPN multiple          | Possibile, con indirizzi ip distinti.    | Semplice e pulito.        |
| Caso d'uso            | VPN a livello di sistema.                | VPN specifica per app.    |

## Esempi

### Eseguire un browser attraverso la VPN

```bash
# Avvia la VPN nel namespace
wgc nup proton-it

# Lancia Firefox nel namespace VPN
wgc exec proton-it firefox

# Quando hai finito, arresta la VPN (chiederà di forzare se Firefox è in esecuzione)
wgc down proton-it
```

### Verificare la connettività VPN

```bash
# Il tuo IP reale
curl ipinfo.io

# IP come visto attraverso la VPN, se avviata con 'up'
curl --interface proton-it ipinfo.io
curl --interface 10.2.0.2 ipinfo.io

# IP come visto attraverso la VPN, se avviata con 'nup'
wgc exec proton-it curl ipinfo.io
```

### VPN simultanee multiple

```bash
# Avvia più VPN nei loro namespace
wgc nup vpn1
wgc nup vpn2
wgc nup vpn3

# Elenca tutte le VPN attive
wgc active

# Usa VPN diverse per applicazioni diverse
wgc exec vpn1 firefox
wgc exec vpn2 bash # poi qualsiasi comando ti serva
```

## Risoluzione Problemi

### Directory di configurazione personalizzata

Se vuoi utilizzare una directory di configurazione diversa da `/etc/wireguard/`, imposta la variabile d'ambiente `WGC_CFGDIR`:

```bash
export WGC_CFGDIR=$HOME/.config/wireguard
wgc list
```

Assicurati che la directory:

- Esista e sia leggibile
- Contenga i tuoi file `.conf` con i permessi appropriati
- Abbia gli stessi diritti di accesso che avrebbe `/etc/wireguard/`

**Importante:** Quando si usa `sudo`, le variabili d'ambiente non sono sempre preservate. Puoi:

- Aggiungere `WGC_CFGDIR` alla tua configurazione sudoers
- Oppure lo script la passerà automaticamente quando eleva i privilegi

### Analisi dei file di configurazione

Lo script valida i file di configurazione e segnala:

* Chiavi obbligatorie mancanti
* Sezioni o chiavi sconosciute (come avvertimenti)
* Righe malformate
* Indirizzi server DNS non validi

### Gestione dei processi

Quando si arresta una VPN in namespace con processi in esecuzione:

1. Il primo tentativo mostra l'elenco dei processi
2. Usa il parametro `force` per terminarli
3. Lo script prova la terminazione controllata (SIGTERM)
4. La barra di progresso mostra il timeout di 10 secondi
5. I processi rimanenti vengono uccisi forzatamente (SIGKILL)

### Problemi DNS

* In modalità namespace: il DNS è configurato in `/etc/netns/<nome_vpn>/resolv.conf`
* In modalità predefinita: il DNS è impostato tramite `resolvectl`
* Gli indirizzi DNS malformati vengono rilevati e saltati con avvertimenti

## Licenza

Questo progetto è rilasciato sotto la licenza GNU General Public License v3.0 (GPL-3.0).
Consulta il file [LICENSE](LICENSE) per i dettagli.
