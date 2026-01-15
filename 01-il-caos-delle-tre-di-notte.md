# Quando l'Osservabilità Non È Connessa: Il Perché del Workshop

## Il Problema Che Ho Trovato

Quando cominci a strumentare sistemi complessi, scopri in fretta che **l'osservabilità diventa un casino se i dati non sono connessi**.

Non sto parlando di una situazione drammatica di 3 di notte. Parlo della realtà quotidiana.

Hai un servizio Node.js. È lento. Accendi tre tool diversi:
- **Datadog**: "CPU 70%, memoria OK"
- **Loki**: "Error: timeout"
- **Jaeger**: "Traccia timeout fra servizio A e B"

Ora il problema: come connetto questi tre pezzi di informazione?

La metrica mi dice **che** c'è un problema. Il log mi dice **cosa** è successo. La traccia mi mostra **dove**. Ma **nessuno di questi tool sa che stanno parlando della stessa richiesta**.

```
┌─────────────────┐
│   Datadog       │ ← "Qualcosa è lento"
├─────────────────┤
│   Loki          │ ← "Timeout a database"
├─────────────────┤
│   Jaeger        │ ← "Request timeout fra servizi"
└─────────────────┘
      Stanno parlando
      della STESSA richiesta?
      Non lo so.
```

È come leggere tre giornali diversi che raccontano lo stesso fatto ma in "lingue" diverse. Capisco il fatto, ma ci metto il triplo del tempo.

---

## Quando Ho Scoperto OpenTelemetry

Ho scoperto OpenTelemetry leggendo blog e guardando conferenze.

La prima cosa che mi ha colpito non era "wow, rivoluzionario" ma: **"Ah, ecco come si risolve questo problema."**

L'idea è semplice:
- Uno stesso SDK per log, trace, metriche
- Una singola ID (trace ID) che lega tutto insieme
- Uno standard che tutti i servizi capiscono (Node, Python, Go, Java, ecc.)

```javascript
const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter(),
  serviceName: 'my-service',
});

sdk.start();
```

Questo non è complicato. Ma **cosa cambia?**

Quando una richiesta arriva nel tuo sistema, OTel crea un trace ID. Quel trace ID viaggia:
- Nel log
- Nella metrica
- Nella traccia
- Fra servizi diversi

Adesso quando apri i tre tool diversi, **sanno tutti che stanno parlando della stessa richiesta**.

---

## Il Problema del "Percorso Progressivo"

Quando ho deciso di usare OTel, ho letto la documentazione ufficiale. È ottima. Ma è **riferimento**, non **tutorial progressivo**.

Mi ricordo di aver pensato:
- "Ok, ho aggiunto logging centralizzato. Ora cosa?"
- "Ora aggiungo le tracce. Ma cosa cambio esattamente?"
- "Ora le metriche. Questo influisce sul codice che ho scritto prima?"
- "E il sampling? Come lo configuro senza rompere il resto?"

Non è che la documentazione non risponda a queste domande. È che la risposta è sparsa.

Così ho deciso: **creo un workshop che mostra passo dopo passo**.

---

## Come È Strutturato il Workshop

6 moduli. Ognuno aggiunge un pezzo:

**Modulo 1: Logging Centralizzato**
Qual è il cambio che faccio quando aggiungo logging centralizzato?
```javascript
// PRIMA: Log manuale con correlation ID
logger.info({ id: uuid.v4(), msg: 'Purchase started' });

// DOPO: OTel lo fa per me
logger.info('Purchase started'); // OTel gestisce l'ID
```

**Modulo 2: Tracce Distribuite**
E se ho Node + Python + Go? Come faccio a vedere una singola traccia che attraversa tutti e tre?

**Modulo 3: Metriche**
Come aggiungo metriche (rate, errors, latency) senza cambiare il codice di logging?

**Modulo 4: Tail Sampling**
Se ho 10,000 richieste al secondo, non posso salvare tutte. Come decido quali salvare?

**Modulo 5: Routing Intelligente**
L'audit log deve andare in un posto diverso dal debug log. Come lo faccio?

**Modulo 6: Tutto Insieme**
Come metto tutto assieme in un sistema che funziona davvero?

---

## Cosa Diverso da Altri Tutorial?

Ogni modulo ha tre cartelle:

```
module-01/
├── before/     ← Il codice SENZA OTel
├── after/      ← Il codice CON OTel
├── current/    ← Dove fai i tuoi esperimenti
├── script.md   ← Cosa è cambiato
└── docker-compose.yml  ← Copia-incolla e funziona
```

Puoi letteralmente fare:
```bash
docker-compose up
curl http://localhost:3003/api/test
```

E vedi gli strumenti funzionare live.

Non è "guarda il codice", è "**avvia e vedi**".

---

## Per Chi È Questo?

- Backend engineer che vuole capire OTel da zero
- DevOps che deve strumentare applicazioni
- Chiunque sia frustrato di saltare fra 3 tool diversi quando debugga
- Chiunque abbia paura che "osservabilità moderna" significhi "imparare un nuovo linguaggio per ogni tool"

**Non hai bisogno di esperienza precedente con OTel.**

Hai bisogno di:
- Node.js base (sai cos'è Express)
- Docker (sai fare `docker-compose up`)
- Curiosità (e sei qui, quindi sì)

---

## Cosa Otterrai

Alla fine del workshop capirai:
- Come strumentare un servizio Node con OTel
- Come collegare tracce fra servizi diversi
- Come misurare metriche senza complicare il codice
- Come ridurre i costi salvando solo quello che importa
- Come instradare dati sensibili dove servono

**E avrai visto come funziona davvero, non solo letto la teoria.**

---

## Il Prossimo Step

Nel prossimo articolo: **come aggiungere OTel a un servizio Node.js in pratica**.

Non "ecco come funziona OTel."

Ma "**ecco il codice prima, ecco il codice dopo, ecco cosa è cambiato.**"

👉 Leggi il prossimo articolo.

---

**Repository**: [Link GitHub]
**Domande?** Apri una Issue.

#OpenTelemetry #Observability #BackendDevelopment #NodeJS
