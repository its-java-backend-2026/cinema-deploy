# cinema-deploy

Il montaggio del sistema: **un `docker-compose.yml` e nient'altro**.

Qui non c'è codice Java. Questo repository risponde a una domanda sola —
*come si mettono insieme i servizi per farli girare* — e cambia quando si
aggiunge un servizio o si sposta una porta, non quando cambia una riga di
dominio.

---

## Perché esiste

Ogni servizio ha il suo repository, e ognuno si clona, si costruisce e si
rilascia **da solo**. Se il compose che avvia tutto stesse dentro uno dei
cinque, quel repository smetterebbe di essere autonomo: chi lo clona senza
avere anche gli altri accanto si troverebbe un file che non funziona, e sarebbe
tornato l'accoppiamento che la separazione serviva a togliere.

In azienda questa cartella si chiama `deploy`, `platform` o `infra`. Con
Kubernetes diventa il repository **GitOps** che Argo CD o Flux sorvegliano. Il
nome cambia, il principio no: *il codice di un servizio e il montaggio del
sistema sono due artefatti con cicli di vita diversi.*

---

## Prerequisito: i sette repository accanto

I `build:` del compose puntano ai repository sorella con un percorso relativo.
Vanno clonati nella **stessa cartella**:

```
una-cartella-qualsiasi/
├── api-gateway/        (repo)   8080   <- l'unica porta di ingresso (dal G9)
├── shows-service/      (repo)   8081
├── pricing-service/    (repo)   8082
├── booking-service/    (repo)   8083   <- l'orchestratore della saga
├── loyalty-service/    (repo)   8084   (dal G8)
├── payment-service/    (repo)   8085   (dal G8)
└── cinema-deploy/      (repo)          <- si lancia da qui
```

Se ne manca uno, `docker compose up` si ferma subito dicendo quale percorso non
trova: è un errore chiaro, e va bene così.

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
| **api-gateway** | **8080** | — (instrada e basta) | — |
| shows-service | 8081 | http://localhost:8081/swagger-ui.html | `shows_db` (5432) |
| pricing-service | 8082 | http://localhost:8082/swagger-ui.html | — |
| booking-service | 8083 | http://localhost:8083/swagger-ui.html | `booking_db` (5433) |
| loyalty-service | 8084 | http://localhost:8084/swagger-ui.html | `loyalty_db` (5435) |
| payment-service | 8085 | http://localhost:8085/swagger-ui.html | `payment_db` (5434) |
| catalog-provider | 8090 | — (nginx con un JSON) | — |
| **zipkin** | **9411** | <http://localhost:9411> | in memoria |

Le porte fra parentesi sono quelle **sull'host**, per entrare con `psql` da
fuori. Dentro la rete di compose i database stanno tutti sulla 5432: a
distinguerli è il nome del servizio, non la porta.

**Dal G9 l'unico indirizzo che conta è `http://localhost:8080/api/...`.** Le
porte 8081-8085 restano pubblicate perché in aula serve poterle confrontare; in
esercizio non lo sarebbero — resterebbero sulla rete interna, e il gateway
sarebbe l'unica cosa esposta.

---

## La consegna del G6, in un comando

> Una `POST /bookings` crea la prenotazione con il prezzo corretto e i posti
> scalati, attraversando più processi. Dal G8 sono cinque.

```bash
# i posti disponibili PRIMA
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'

# la prenotazione: uno studente, due posti
curl -s -X POST localhost:8083/bookings \
     -H 'Content-Type: application/json' \
     -H 'Idempotency-Key: prova-1' \
     -d '{"showId":1,"customerId":"mario.rossi","customerType":"STUDENT","quantity":2}'

# i posti disponibili DOPO: due in meno
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'
```

Nella risposta c'è il `sagaId`. È la stringa che ricuce i log di **tutti** i
processi coinvolti:

```bash
docker compose logs | grep <il-sagaId-della-risposta>
```

Si vedono, in ordine, la lettura dello spettacolo, il preventivo, la riserva
dei posti, l'autorizzazione del pagamento, l'accredito dei punti e la conferma
— in cinque servizi diversi, con lo stesso identificativo.

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

### 4. La consegna del G8: un pagamento rifiutato non lascia posti bloccati

> `payment-service` rifiuta sopra `CINEMA_PAYMENT_SOGLIA` (default `100.00`).
> 20 posti da 10.00 euro fanno 200.00.

```bash
# PRIMA
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'

curl -i -X POST localhost:8083/bookings \
     -H 'Content-Type: application/json' \
     -H "Idempotency-Key: $(uuidgen)" \
     -d '{"showId":1,"customerId":"mario.rossi","customerType":"STUDENT","quantity":20}'
# -> HTTP/1.1 402 Payment Required
#    "detail": "Pagamento rifiutato: Importo 200.00 oltre la soglia ..."
#    "compensata": true

# DOPO: gli stessi posti di prima
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'
```

**I due numeri devono coincidere.** È tutta la giornata in due `curl`.

E la saga racconta dove si è fermata:

```bash
docker exec -it booking-db psql -U cinema -d booking_db \
  -c "SELECT booking_id, passo_raggiunto, stato, ultimo_errore
        FROM saga_state ORDER BY id DESC LIMIT 5;"
```

`POSTI_RISERVATI` e non `PAGATO`: si è fermata **prima** del pagamento, quindi
la compensazione ha rilasciato i posti e non ha stornato niente — non c'era
niente da stornare.

Per vederla fallire anche su una prenotazione piccola, si abbassa la soglia:

```bash
CINEMA_PAYMENT_SOGLIA=5 docker compose up -d payment-service
```

### 5. L'idempotenza dei partecipanti (passo 8.3)

Ripetere un passo non lo esegue due volte, ed è ciò che permette ai retry di
esistere:

```bash
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'

# la stessa riserva, due volte, con lo stesso sagaId
for i in 1 2; do
  curl -s -o /dev/null -X POST localhost:8081/shows/1/reserve \
       -H 'Content-Type: application/json' \
       -d '{"sagaId":"prova-idempotenza","quantity":2}'
done

# due posti in meno, non quattro
curl -s localhost:8081/shows/1 | grep -o '"availableSeats":[0-9]*'
```

Fino al G7 lo stesso comando ne toglieva quattro. Il cambiamento non è in
`booking-service`: è `show_operations`, in `shows_db`.

---

### 6. La consegna del G9: una prenotazione è UNA traccia su Zipkin

```bash
curl -s -X POST localhost:8080/api/bookings \
     -H 'Content-Type: application/json' \
     -H "Idempotency-Key: $(uuidgen)" \
     -d '{"showId":1,"customerId":"mario.rossi","customerType":"STUDENT","quantity":2}'
```

Poi <http://localhost:9411> → *Run Query* → la traccia più recente. Sono **13
span** che attraversano **sei** servizi:

```
api-gateway      SERVER  http post /api/bookings/**            1101 ms
api-gateway      CLIENT  http post                             1096 ms
booking-service  SERVER  http post /bookings                   1091 ms
booking-service  CLIENT  http get                                27 ms
shows-service    SERVER  http get /shows/{id}                    21 ms
booking-service  CLIENT  http post                              911 ms
pricing-service  SERVER  http post /prices/quote                902 ms
booking-service  CLIENT  http post                               20 ms
shows-service    SERVER  http post /shows/{id}/reserve           17 ms
booking-service  CLIENT  http post                               52 ms
payment-service  SERVER  http post /payments/authorize           48 ms
booking-service  CLIENT  http post                               28 ms
loyalty-service  SERVER  http post /loyalty/{customerid}/credit   25 ms
```

Gli span **CLIENT** e **SERVER** sono la stessa chiamata vista dalle due
parti, e la differenza fra i due tempi è la rete più l'attesa: è il numero che
distingue *«l'altro è lento»* da *«la rete fra noi è lenta»*.

### 7. L'esercizio che vale la giornata (passo 9.9)

Trovare il collo di bottiglia **guardando solo la traccia**, senza aprire un
log.

```bash
CINEMA_PRICING_RITARDO_MS=800 docker compose up -d pricing-service
```

Poi si rifà la prenotazione qui sopra e si guarda Zipkin. Nella cascata degli
span uno diventa lungo quanto tutti gli altri messi insieme, e ha il nome del
servizio scritto sopra — l'output riportato al punto 6 è proprio con il ritardo
acceso: `pricing-service` da solo si prende 902 ms su 1091.

Per spegnerlo:

```bash
docker compose up -d pricing-service    # senza la variabile: torna a zero
```

Che il difetto sia **finto** non cambia niente del metodo: un servizio lento
vero — una query senza indice, un pool esaurito, un GC che non respira — in una
traccia si presenta esattamente così.

### 8. Le metriche (passi 9.4 e 9.5)

```bash
curl -s localhost:8083/actuator/prometheus | grep http_server_requests_seconds_count
curl -s localhost:8083/actuator/prometheus | grep hikaricp_connections_active
curl -s localhost:8083/actuator/prometheus | grep resilience4j_circuitbreaker_state
```

Nessuno le ha scritte: arrivano dagli stessi filtri che gestiscono le
richieste, ed è il motivo per cui non possono disallinearsi dal codice come
farebbe una strumentazione scritta a mano.

`hikaricp_connections_active` è quella da tenere d'occhio: è il numero che
spiega i blocchi del passo 6.6, e dal G8 anche il motivo per cui le
transazioni di `SagaStore` devono restare corte.

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
docker exec -it payment-db psql -U cinema -d payment_db
docker exec -it loyalty-db psql -U cinema -d loyalty_db

# le tracce (passo 9.7)
open http://localhost:9411

# le metriche di un servizio, in formato Prometheus (passo 9.5)
curl -s localhost:8083/actuator/prometheus | head -40

# lo stato delle saghe (passo 8.4)
docker exec -it booking-db psql -U cinema -d booking_db \
  -c "SELECT booking_id, passo_raggiunto, stato FROM saga_state ORDER BY id DESC LIMIT 10;"

# le saghe rimaste per strada, e quelle che nessuno sistemerà
docker exec -it booking-db psql -U cinema -d booking_db \
  -c "SELECT * FROM saga_state
       WHERE stato = 'IN_CORSO' AND aggiornata_il < now() - interval '5 minutes'
          OR stato = 'COMPENSAZIONE_PARZIALE';"
```

---

## Quattro database, non quattro schemi

È la decisione strutturale del G6 ed è quella che rende vero tutto il resto.

Se `bookings` e `shows` stessero nello stesso PostgreSQL, prima o poi qualcuno
scriverebbe una JOIN fra le due, e da quel momento nessuno dei due servizi
potrebbe più cambiare la propria tabella senza rompere l'altro. Con database
separati la JOIN non è possibile: l'unico modo di leggere i dati altrui è
passare dalla sua API.

Il prezzo si vede nel compose, e dal G8 è raddoppiato: quattro container,
quattro volumi, quattro healthcheck, quattro porte diverse sull'host. Vale la
pena guardarlo in faccia — è il prezzo vero dei microservizi, e si paga in
memoria del portatile prima ancora che in complessità. È anche il motivo per
cui la saga esiste: senza un database solo non c'è nessuna transazione che
tenga insieme i cinque servizi. E
nella `V1` di `booking-service` non c'è nessuna `FOREIGN KEY` verso `shows`.
L'integrità referenziale fra servizi non esiste — al suo posto ci sono la saga
e la consapevolezza che i dati sono coerenti *alla fine*, non *sempre*.
