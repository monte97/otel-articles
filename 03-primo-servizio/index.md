---
title: "Il Tuo Primo Servizio con OpenTelemetry: Non È Magia, È Codice"
date: 2025-01-07T10:00:00+01:00
description: "Guida pratica per aggiungere OpenTelemetry a un servizio Node.js: setup, Collector, Jaeger e logging correlato in 25 righe di codice"
menu:
  sidebar:
    name: "3. Primo Servizio"
    identifier: otel-3
    weight: 30
    parent: otel-workshop
tags: ["OpenTelemetry", "Node.js", "Jaeger", "Docker", "Tutorial"]
categories: ["Observability", "Tutorial", "Workshop"]
draft: false
---

*Tempo di lettura: ~12 minuti*

Questa sezione mostra l'implementazione pratica di OpenTelemetry su un servizio Node.js, partendo da un'applicazione senza strumentazione.

---

## Il Punto di Partenza: Un Servizio Node Normale

Un servizio e-commerce con endpoint `/buy`:

```javascript
// index.js - il tuo servizio (BEFORE)
const express = require('express');
const axios = require('axios');

const app = express();
const PORT = 3003;

app.get('/buy', async (req, res) => {
  const userId = req.query.userId;
  const productId = req.query.productId;

  console.log('Processing purchase for', userId);

  try {
    // Chiama servizio inventory (Python)
    const inventory = await axios.get('http://localhost:3007/data');

    // Chiama servizio shipping (Go)
    const shipping = await axios.get('http://localhost:3008/process');

    console.log('Purchase completed');
    res.json({ inventory: inventory.data, shipping: shipping.data });
  } catch (error) {
    console.error('Purchase failed', error.message);
    res.status(500).json({ error: 'Purchase failed' });
  }
});

app.listen(PORT, () => {
  console.log(`Server on port ${PORT}`);
});
```

**Limitazioni:**

Senza strumentazione:
- Console log (sparisce quando il processo muore)
- No trace di cosa è successo fra i servizi
- No metriche
- Se la chiamata a inventory è lenta, non lo vedi

---

## Step 1: Aggiungere OpenTelemetry (20 Righe di Codice)

Crei un file `instrumentation.js` (nota il nome, importante):

```javascript
// instrumentation.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4317'  // Dove mandare le tracce
  }),
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: 'shop-service',
});

sdk.start();
console.log('✓ OpenTelemetry initialized');
```

Questo codice:
- Configura OpenTelemetry
- Auto-instrumenta tutte le librerie Node.js (axios, express, database, ecc.)
- Manda le tracce al Collector su localhost:4317

---

## Step 2: Cambia Come Avvii il Servizio

Nel tuo `package.json`, cambia solo la riga `start`:

```json
{
  "scripts": {
    "start": "node --require ./instrumentation.js index.js"
  }
}
```

Il flag `--require` carica `instrumentation.js` **prima** di tutto il resto. Fondamentale, perché OTel deve interceptare le librerie prima che le usi.

Avvia:

```bash
npm start
```

Il servizio è ora instrumentato senza modifiche al codice applicativo.

Test:

```bash
curl http://localhost:3003/buy?userId=alice&productId=123
```

OTel automaticamente ha creato una traccia con:
- ✓ Request HTTP ricevuta
- ✓ Call a inventory (axios.get)
- ✓ Call a shipping (axios.get)
- ✓ Tempo totale

Per visualizzare le tracce serve un backend di raccolta.

---

## Step 3: Il Collector

Il servizio OTel genera tracce che devono essere raccolte e inviate a un backend di visualizzazione.

Il **OTel Collector** riceve tracce dai servizi e le inoltra alle destinazioni configurate.

File `collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # Ascolta qui le tracce

exporters:
  jaeger:
    endpoint: jaeger:14250      # Manda a Jaeger (visualization)
  logging:
    loglevel: debug             # Stampa anche a console

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [logging, jaeger]
```

Setup con docker-compose:

```yaml
# docker-compose.yml
version: '3'
services:
  otel-collector:
    image: otel/opentelemetry-collector:latest
    volumes:
      - ./collector-config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317"
    command: ["--config=/etc/otel/config.yaml"]

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI per visualizzare tracce
```

Avvia:

```bash
docker-compose up
```

---

## Step 4: Visualizza le Tracce

Apri il browser su **http://localhost:16686** (Jaeger UI).

Seleziona "shop-service" dal dropdown.

Premi il bottone Search.

**Output:**

```text
Service: shop-service
Operation: GET /buy
Duration: 547ms

├─ GET http://localhost:3007/data - 150ms
├─ GET http://localhost:3008/process - 320ms
└─ Response - 77ms
```

Singola timeline con tutte le operazioni correlate.

---

## Step 5: Aggiungere Logging Correlato (Opzionale)

È possibile collegare i log alla traccia tramite il trace ID.

Esempio con Pino:

```javascript
// index.js - con logging
const pino = require('pino');
const { trace, context } = require('@opentelemetry/api');

const logger = pino({
  level: 'info',
  mixin: () => {
    // Aggiungi il traceId automaticamente ai log
    const currentSpan = trace.getSpan(context.active());
    if (currentSpan) {
      return { traceId: currentSpan.spanContext().traceId };
    }
    return {};
  }
});

app.get('/buy', async (req, res) => {
  const userId = req.query.userId;

  logger.info('Starting purchase');  // ← Automaticamente avrà traceId

  try {
    const inventory = await axios.get('http://localhost:3007/data');
    logger.info('Inventory checked');

    const shipping = await axios.get('http://localhost:3008/process');
    logger.info('Shipping arranged');

    res.json({ ok: true });
  } catch (error) {
    logger.error(error, 'Purchase failed');
    res.status(500).json({ error: 'Failed' });
  }
});
```

Ogni log include il `traceId`, permettendo la correlazione con le tracce in Jaeger.

---

## Quello Che È Successo

Prima:

```text
console.log('Purchase started')
  → stdout → scompare
```

Dopo:

```text
logger.info('Purchase started')
  → {traceId: "abc123...", message: "Purchase started"}
  → OTel Collector
  → Loki (log storage)
  → Grep correlato con Jaeger (trace)
```

**Una singola traccia collega:**
- ✓ HTTP request
- ✓ Latenze di ogni hop
- ✓ Log da ogni servizio
- ✓ Errori

---

## I File Che Hai Creato

```text
my-service/
├── instrumentation.js      (20 righe)
├── index.js               (il tuo codice, UNCHANGED)
├── package.json           (una riga cambia nello "start")
├── collector-config.yaml  (configurazione Collector)
└── docker-compose.yml     (avvia Collector + Jaeger)
```

Codice aggiunto: **~25 righe.**

Funzionalità ottenute:
- ✓ Distributed tracing
- ✓ Cross-service correlation
- ✓ Timeline visualizzazione
- ✓ No vendor lock-in

---

## Prossimi Passi

Aspetti non ancora coperti:

1. **Alto volume di richieste** — Costi di storage (→ Tail Sampling)
2. **Misurazione performance** — Rate, Errors, Duration (→ Metriche)
3. **Separazione log** — Audit log vs debug log (→ Routing)

---

## Repository

Il codice completo è disponibile nel repository:

```bash
git clone https://github.com/monte97/otel-demo
cd otel-demo/module-03
docker-compose up
# Apri http://localhost:5003/buy?userId=test&productId=123
# Poi guarda Jaeger su http://localhost:16686
```

---

*Serie OpenTelemetry Workshop:*
1. [Quando l'Osservabilità Non È Connessa]({{< relref "/posts/otel-workshop/01-perche-otel" >}})
2. [Lo Standard È Importante: Come OTel Connette Servizi Diversi]({{< relref "/posts/otel-workshop/02-standard-otel" >}})
3. **Il Tuo Primo Servizio con OpenTelemetry** (questo articolo)
4. [Il Costo Nascosto dell'Osservabilità]({{< relref "/posts/otel-workshop/04-tail-sampling" >}})
5. [I Dati Dove Servono: Routing Intelligente]({{< relref "/posts/otel-workshop/05-routing" >}})
