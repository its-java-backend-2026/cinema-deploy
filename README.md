# cinema-deploy

Il montaggio del sistema: **un `docker-compose.yml` e nient'altro**.

Qui non c'è codice Java. Questo repository risponde a una domanda sola —
*come si mettono insieme i servizi per farli girare* — e cambia quando si
aggiunge un servizio o si sposta una porta, non quando cambia una riga di
dominio.

---

## Perché esiste

Ogni servizio ha il suo repository, e ognuno si clona, si costruisce e si
rilascia **da solo**. Se il compose che avvia tutto stesse dentro uno dei tre,
quel repository smetterebbe di essere autonomo: chi lo clona senza avere anche
gli altri due accanto si troverebbe un file che non funziona, e sarebbe tornato
l'accoppiamento che la separazione serviva a togliere.

In azienda questa cartella si chiama `deploy`, `platform` o `infra`. Con
Kubernetes diventa il repository **GitOps** che Argo CD o Flux sorvegliano. Il
nome cambia, il principio no: *il codice di un servizio e il montaggio del
sistema sono due artefatti con cicli di vita diversi.*

---

## Prerequisito: i quattro repository accanto

I `build:` del compose puntano ai repository sorella con un percorso relativo.
Vanno clonati nella **stessa cartella**:

```
una-cartella-qualsiasi/
├── shows-service/      (repo)
├── pricing-service/    (repo)
├── booking-service/    (repo)
└── cinema-deploy/      (repo)  <- si lancia da qui
```

Se manca uno dei tre, `docker compose up` si ferma subito dicendo quale
percorso non trova: è un errore chiaro, e va bene così.

---

## Avvio

```bash
cp .env.example .env        # opzionale: i default bastano
docker compose up --build
```

La prima volta ci vogliono alcuni minuti: Maven scarica le dipendenze dentro
ogni immagine. Le volte successive Docker riusa lo strato delle dipendenze,
a patto che il `pom.xml` non sia cambiato.

Quando tutto è su:

| Servizio | Porta | Swagger | Database |
|---|---|---|---|
| shows-service | 8081 | http://localhost:8081/swagger-ui.html | `shows_db` (5432) |
| pricing-service | 8082 | http://localhost:8082/swagger-ui.html | — |
| booking-service | 8083 | http://localhost:8083/swagger-ui.html | `booking_db` (5433) |
| catalog-provider | 8090 | — (nginx con un JSON) | — |

---

## La consegna del G6, in un comando

> Una `POST /bookings` crea la prenotazione con il prezzo corretto e i posti
> scalati, attraversando tre processi.

```bash
# i posti disponibili PRIMA
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'

# la prenotazione: uno studente, due posti
curl -s -X POST localhost:8083/bookings \
     -H 'Content-Type: application/json' \
     -d '{"showId":1,"customerType":"STUDENT","quantity":2}'

# i posti disponibili DOPO: due in meno
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'
```

Nella risposta c'è il `sagaId`. È la stringa che ricuce i log dei tre processi:

```bash
docker compose logs | grep <il-sagaId-della-risposta>
```

Si vedono, in ordine, la lettura dello spettacolo, il preventivo, la riserva
dei posti e il salvataggio — in tre servizi diversi, con lo stesso
identificativo.

---

## Le prove che vale la pena fare in aula

### 1. Un servizio a valle giù è un 503, non un 500

```bash
docker compose stop pricing-service

curl -i -X POST localhost:8083/bookings \
     -H 'Content-Type: application/json' \
     -d '{"showId":1,"customerType":"STUDENT","quantity":1}'
```

Deve rispondere **503** con l'header `Retry-After`, e deve farlo **in circa due
secondi**: sono i timeout del passo 6.6. Senza timeout espliciti la stessa
richiesta resterebbe appesa, e con abbastanza richieste insieme finirebbero i
thread e smetterebbe di rispondere anche il resto.

```bash
docker compose start pricing-service
```

### 2. Un guasto a valle non spegne cosa non c'entra

Con `pricing-service` ancora fermo:

```bash
curl -s localhost:8083/bookings          # risponde: non chiama nessuno
curl -s localhost:8083/actuator/health/readiness   # UP
```

La readiness di `booking-service` **non** dipende dalla salute degli altri due,
ed è voluto: se ci dipendesse, un guasto a valle toglierebbe dal bilanciatore
anche le nostre istanze sane.

### 3. Il prezzo non si accetta dal client

```bash
curl -s -X POST localhost:8083/bookings \
     -H 'Content-Type: application/json' \
     -d '{"showId":1,"customerType":"STUDENT","quantity":1,"unitPrice":0.01}'
```

`unitPrice` viene ignorato: il prezzo lo dice `pricing-service`, sempre.

### 4. Il buco che resta, ed è il G8

Il passo 3 della saga (riserva) e il passo 4 (salvataggio) non sono nella
stessa transazione, e non possono esserlo: una transazione locale non annulla
un POST già arrivato a destinazione. Se il salvataggio fallisce, **i posti
restano riservati**.

Non è un difetto da nascondere: è il problema che il G8 risolve con le
compensazioni. `ShowsClient.rilascia()` è già scritto, e oggi non lo chiama
nessuno di proposito.

---

## Comandi utili

```bash
docker compose up --build            # costruisce e avvia
docker compose down                  # ferma, i dati restano
docker compose down -v               # ferma E CANCELLA i dati
docker compose logs -f booking-service
docker compose ps                    # chi e' su, e chi e' "healthy"
docker compose stop pricing-service  # per le prove qui sopra

# entrare nei database
docker exec -it shows-db   psql -U cinema -d shows_db
docker exec -it booking-db psql -U cinema -d booking_db
```

---

## Due database, non due schemi

È la decisione strutturale del G6 ed è quella che rende vero tutto il resto.

Se `bookings` e `shows` stessero nello stesso PostgreSQL, prima o poi qualcuno
scriverebbe una JOIN fra le due, e da quel momento nessuno dei due servizi
potrebbe più cambiare la propria tabella senza rompere l'altro. Con due
database la JOIN non è possibile: l'unico modo di leggere i dati altrui è
passare dalla sua API.

Il prezzo si vede nel compose: due container, due volumi, due healthcheck. E
nella `V1` di `booking-service` non c'è nessuna `FOREIGN KEY` verso `shows`.
L'integrità referenziale fra servizi non esiste — al suo posto ci sono la saga
e la consapevolezza che i dati sono coerenti *alla fine*, non *sempre*.
