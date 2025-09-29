+++
date = '2025-09-25T15:37:51+02:00'
draft = false
title = 'Creare un server email personale'

translationKey = 'email-server'
+++

Ho posseduto il dominio `tste.dev` per anni e ho sempre considerato la possibilità di hostare un server email personale.
Non si tratta di un'impresa banale, ma significa avere più controllo su come i propri dati vengono immagazzinati e gestiti,
specialmente considerando quante informazioni sensibili sono sfortunatamente trasmesse via email.

## Comprendere le email

Quando componi ed invii un'email, il tuo client trasferisce il messaggio ad un Mail Transfer Agent (MTA)
utilizzando il protocollo SMTP (Simple Mail Transfer Protocol). L'MTA cerca dunque il dominio del destinatario tramite i record MX del DNS
in modo tale da determinare quale server email dovrebbe ricevere il messaggio.

Il tuo MTA si connette dunque a tale server SMTP e trasmette l'email. Il server del destinatario memorizza il messaggio
in una casella di posta finché il client del destinatario non lo recupera. Tale operazione avviene solitamente attraverso IMAP
(Internet Message Access Protocol) o il più vecchio POP3 (Post Office Protocol).

> **Lo sapevi?** Un acronimo ridondante è un acronimo utilizzato ripetendo una parola facente parte della formazione dell'acronimo stesso.
In inglese ci si riferisce a ciò con un termine autologico: RAS syndrome (Redundant Acronym Syndrome).

### Standard di sicurezza moderni

#### Sender Policy Framework ([RFC7208](https://datatracker.ietf.org/doc/html/rfc7208))

SPF è essenzialmente un elenco di server autorizzati ad inviare email per conto di un dominio.
È rappresentato da un record DNS TXT. Quando un server di posta in arrivo riceve un messaggio, confronta il dominio del mittente
con il record SPF. Tuttavia, SPF da solo convalida solamente il percorso seguito dall'email, ma non garantisce che il corpo
o gli header del messaggio non siano stati alterati.

#### DomainKeys Identified Mail ([RFC6376](https://datatracker.ietf.org/doc/html/rfc6376))

DKIM aggiunge una firma crittografica alle email. Il server di invio utilizza una chiave privata per firmare header specifici e il corpo dell'email,
mentre la corrispondente chiave pubblica viene pubblicata nel DNS. Quando il server di ricezione riceve l'email, verifica la firma recuperando
la chiave pubblica. Ciò rende il contenuto dell'email praticamente impossibile da manomettere, purché la chiave privata sia archiviata in modo sicuro.
La coppia di chiavi è comunemente RSA-SHA256 a 1024-4096 bit; i setup più recenti potrebbero usare Ed25519 ([RFC8463](https://datatracker.ietf.org/doc/html/rfc8463)).

#### Domain-based Message Authentication, Reporting, and Conformance ([RFC7489](https://datatracker.ietf.org/doc/html/rfc7489))

DMARC è un livello di policy che coniuga SPF e DKIM. Indica ai server riceventi come gestire i messaggi che non superano i controlli SPF e/o DKIM.
Il proprietario di un dominio pubblica un record DMARC nel DNS specificando:
- Quali controlli devono essere superati affinché un'email sia considerata valida.
- Cosa fare con i messaggi che non superano i controlli (none, quarantine, reject).
- Dove inviare i rapporti sull'attività di autenticazione.

## Deliverability

A differenza di sistemi di messaggistica strettamente controllati, l'email si basa su un'infrastruttura aperta e decentralizzata
in cui chiunque può inviare messaggi a chiunque altro. Tale apertura, unita alla mancanza di una forte autenticazione del mittente
integrata nel protocollo SMTP originale, rende l'email particolarmente vulnerabile a spam, spoofing e abusi.

Di conseguenza, la consegna delle email moderne dipende fortemente da filtri, blacklist e sistemi di reputazione gestiti dai provider di ricezione.
La sfida è che anche i messaggi legittimi possono essere classificati erroneamente o rifiutati se provengono da un nuovo dominio,
da un server configurato in modo errato o se non dispongono di un'autenticazione adeguata.

Infine, è importante tenere presente che la posta elettronica non tollera necessariamente i tempi di downtime.
Se il server SMTP ricevente non funziona, alcuni server di invio potrebbero riprovare, anche per giorni,
mentre altri rinunceranno immediatamente all'invio del messaggio.

### Garantire deliverability

#### Autenticazione

Configura correttamente SPF, DKIM, DMARC. Una volta definita una policy DMARC solida, potresti prendere in considerazione [BIMI](https://datatracker.ietf.org/doc/draft-brand-indicators-for-message-identification/).
Usa TLS per la crittografia in transito.

#### Consistenza

Imposta un record PTR (DNS inverso) in modo che dall'IP si possa risalire al tuo dominio. Di solito questo comporta contattare il tuo ISP o provider cloud.
Inoltre, imposta un hostname HELO/EHLO ([RFC5321](https://datatracker.ietf.org/doc/html/rfc5321)) sul tuo server di posta.

#### Reputazione IP

La reputazione del tuo IP deve essere pulita. Se stai cercando di hostare da casa, è probabile che l'IP fornito dal tuo ISP sia già sporco o che questi rifiuti
di aprire la porta 25 in uscita o di creare un record PTR. Se ti trovi in questa situazione, puoi configurare un relay SMTP esterno. Potrebbe essere una tua VPS
che esegue il relay (utile se desideri che la maggior parte della tua architettura rimanga all'interno della tua abitazione) o uno fornito da molte compagnie specializzate.

#### Monitoraggio

Puoi controllare la reputazione del tuo dominio e del tuo IP utilizzando strumenti come Google Postmaster Tools e Microsoft SNDS.

## Il setup

> A quanto pare, provider come Microsoft bloccano gli IP di Oracle a prescindere, dunque sono andato sul sicuro utilizzando Mailjet come relay SMTP.
Ho anche intenzione di implementare regole Postfix per instradare le email inviate a domini sicuri noti attraverso il mio server SMTP,
al fine di massimizzare la sovranità sui miei dati.

Ho configurato il mio server in una Oracle Cloud VM instance utilizzando [mailcow](https://mailcow.email), una mailserver suite basata su Docker
che sfrutta molteplici componenti note e utilizzate da tempo. I due software fondamentali con cui dovresti familiarizzare sono:
- Dovecot: server IMAP/POP
- Postfix: Mail Transfer Agent

Questi possono ovviamente essere utilizzati anche singolarmente se desideri una configurazione più minimale o hai un caso d'uso più specifico.

## Porte

È necessario abilitare le seguenti porte in entrata nel firewall cloud o fisico:
| Servizio              | Protocollo    | Porta   |
|-----------------------|---------------|---------|
| Postfix SMTP(S)       | TCP           | 25/465  |
| Postfix Submission    | TCP           | 587     |
| Dovecot IMAP(S)       | TCP           | 143/993 |
| Dovecot POP3(S)       | TCP           | 110/995 |
| Dovecot ManageSieve   | TCP           | 4190    |
| HTTP(S)               | TCP           | 80/443  |

Tutte le porte in uscita sono probabilmente già aperte.
In caso contrario, fai riferimento a [questa tabella](https://docs.mailcow.email/getstarted/prerequisite-system/#outgoing-portshosts).

## Record DNS

### A/AAAA

Questo punta semplicemente all'IP del tuo server mail.
```
mail.tste.dev. IN A 130.110.6.94
```

### PTR

Un record PTR (DNS inverso) consente di eseguire un lookup del dominio dato l'IP, migliorando la fiducia dei server riceventi.
Di solito è necessario contattare il proprio provider per configurarlo.
```
94.6.110.130.in-addr.arpa. IN PTR mail.tste.dev
```
Nota come gli ottetti siano in ordine inverso.

### MX

Un record MX indica al server di posta in uscita quale server mail utilizzare per consegnare le email per un dominio specifico.
```
tste.dev. IN MX 10 mail.tste.dev.
```
Questo significa che le email inviate a `@tste.dev` saranno instradate verso `mail.tste.dev`.
Il numero `10` è la priorità del record, utile quando sono disponibili molteplici server.

### SPF

{{< tabs tabTotal="2" >}}

{{% tab tabName="Senza relay" %}}
```
tste.dev. IN TXT "v=spf1 mx ~all"
```
Ciò significa che l'unico server consentito è quello specificato nel tuo record MX.
Il parametro `~all` segnala al server ricevente di eseguire un soft fail se il controllo non ha esito positivo, il che significa che l'email viene consegnata ma contrassegnata come potenzialmente pericolosa.
{{% /tab %}}

{{% tab tabName="Relay SMTP" %}}
```
tste.dev. IN TXT "v=spf1 include:spf.mailjet.com mx ~all"
```
Ciò consente anche ad spf.mailjet.com di inviare email per tuo conto.
{{% /tab %}}

{{< /tabs >}}

### DKIM

```
dkim._domainkey. IN TXT "v=DKIM1;k=rsa;t=s;s=email;p=MIIBIjANBg..."
```
La parte prima di `._domainkey` è della label: è possibile elencare più chiavi pubbliche utilizzando diversi label.
Ad esempio, ho creato un record DKIM con l'etichetta `mailjet` e la chiave corrispondente per consentire
al relay di inviare posta per mio conto.
Il valore assegnato a `p=` è la chiave pubblica codificata in base64. Verrà generata in seguito durante la configurazione di Mailcow.

### DMARC

```
_dmarc.tste.dev. IN TXT "v=DMARC1; p=reject; sp=reject;
adkim=s; aspf=s; rua=mailto:dmarc@tste.dev;"
```

`p=reject` → Rifiuta le email che non superano i controlli DMARC.

`sp=reject` → Rifiuta anche la posta proveniente dai sottodomini che non superano i controlli.

`adkim=s` → La modalità di allineamento DKIM è rigorosa (il dominio di firma DKIM deve corrispondere esattamente al dominio From:).

`aspf=s` → La modalità di allineamento SPF è rigorosa (il dominio convalidato SPF deve corrispondere esattamente al dominio From:).

`rua=mailto:dmarc@tste.dev` → I rapporti aggregati sui risultati DMARC devono essere inviati a questo indirizzo email.

## Setup software

> Tieni presente che questi passaggi tengono conto delle sfide specifiche che ho affrontato sul mio sistema con la mia configurazione.
Per una guida completa e generica, consulta la [documentazione di mailcow](https://docs.mailcow.email).

### Propedeutico

Assicurati di avere installato questi pacchetti di base.

```sh
sudo dnf install -y git openssl curl gawk grep jq
```

Installa docker.

```sh
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Questo ha generato un errore sul mio sistema, ma
# dopo alcune ricerche sono riuscito a risolverlo con
# sudo dnf install -y kernel-modules-extra-$(uname -r)
# ed un riavvio
sudo systemctl enable --now docker
```

Controlla l'MTU della tua interfaccia di rete con `ip l`. Se si tratta di un valore non comune (diverso da 1500),
aggiungilo a `/etc/docker/daemon.json`.

```json
{
  "selinux-enabled": true,
  "mtu": 9000
}
```

Aggiungi selinux-enabled, se applicabile, e tieni a mente il tuo MTU poiché ti servirà in un passaggio successivo.

### Mailcow

```sh
sudo su
umask 0022
cd /opt

git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized

./generate_config.sh
vim mailcow.conf  # Imposta HTTP_REDIRECT=y se non hai intenzione di utilizzare un reverse proxy
```

Modifica `docker-compose.yml`.
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

Avvia i container.
```sh
docker compose pull
docker compose up -d
```

## Configurazione

### Primo login

Una volta messo in funzione mailcow, ho effettuato l'accesso all'interfaccia di amministrazione, ho generato una nuova password casuale e complessa
e abilitato la 2FA attraverso TOTP.
Ovviamente consiglio di utilizzare un password manager come [KeePassXC](https://github.com/keepassxreboot/keepassxc) ed un'app di autenticazione mobile FOSS.

### Chiave DKIM

L'UI fornisce un menu per generare chiavi DKIM sotto System > Configuration > Options > ARC/DCIM keys.
Tuttavia, restituiva l'errore "Access denied or incomplete / invalid data", e ho semplicemente deciso di generare
io stesso la coppia di chiavi ed importarla dallo stesso menu.

> È possibile utilizzare 3072 o 4096 bit, ma spesso ciò non è necessario e può causare problemi con i limiti di lunghezza dei record DNS.

```sh
openssl genrsa -out dkim.key 2048
cat dkim.key | xclip -c  # Copia negli appunti
```

Una volta fatto ciò, è possibile copiare e incollare il record DKIM visualizzato per completare la configurazione DNS.

### Domini, mailbox, alias

Per completare il setup, ho creato il domino `@tste.dev` e aggiunto alcune caselle di posta.
Ogni mailbox è essenzialmente un utente con una propria password.

Mailcow consente di creare alias per le tue caselle di posta. Ciò può tornare utile in diversi modi,
come per mitigare lo spam, categorizzare ed aggregare le email in una sola casella centrale.

Ogni casella di posta può anche autonomamente generare un indirizzo email temporaneo con una parte locale pseudocasuale, come `xiheha.jazi@tste.dev`.

## Considerazioni sulla privacy

Ciò che in molti non comprendono è che la privacy e la cybersicurezza richiedono la definizione di un threat model.
In assenza di questo, l'unico sistema sicuro è incapsulato nel calcestruzzo e posto sul fondo della Fossa delle Marianne.

Oracle non avrebbe alcun problema ad accedere al mio volume cloud. E la cifratura del disco non ha molto senso, poiché la chiave di decrittazione
risiederebbe in RAM finché la macchina è in esecuzione, ovvero sempre, e un dump della memoria è altrettanto banale quanto un dump del volume di storage.

Tuttavia, dubito che stiano attivamente analizzando ogni singola istanza cloud nell'eventualità che qualcuno mantenga email in chiaro sul disco.
Posso anche cancellare i messaggi subito dopo averli letti, pratica che potrebbe essere contrastata solo mediante una scansione immediata di ogni file
appena creato o modificato, oppure tramite tecniche forensi quali la scansione di inode liberi o il file carving basato sulle firme dei file.

Google, al contrario, effettua realmente una scansione sistematica di ogni singola email in entrata ed in uscita per fini di pubblicità mirata,
analisi predittiva del comportamento dei clienti, rivendita a data broker e per altri distopici scopi.

Alla luce di quanto esposto, è ragionevole affermare che la mia configurazione sia piuttosto sicura contro pratiche di data harvesting corporate.
Tuttavia, come già discusso, Oracle avrebbe la capacità di monitorare o persino alterare qualsiasi flusso attraverso il mio mail server; ciò significa che
se il mio threat model fosse rappresentato da un governo o da un'entità con risorse di livello governativo, potrei essere compromesso con relativa facilità.
Ciononostante, se fossi effettivamente bersaglio di tale antagonista, non hosterei certo su OCI né descriverei la mia infrastruttura su internet.

Infine, l'utilizzo di un relay SMTP di terze parti implica che l'azienda corrispondente abbia accesso al contenuto dei messaggi inviati.
Questo non rappresenta per me una criticità significativa, dal momento che invio email molto raramente; la mia preoccupazione principale era proteggere i messaggi ricevuti.
Inoltre, qualora dovessi trasmettere informazioni sensibili, ricorrerei a PGP. Se il destinatario non fosse in grado o non volesse utilizzare PGP, sarebbe evidente
la sua scarsa consapevolezza in materia di privacy; anche utilizzando un mio server SMTP, infatti, il suo provider e/o client di posta potrebbero comunque intercettare la comunicazione.
