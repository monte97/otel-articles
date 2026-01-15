---
title: "Quando l'Osservabilità Non È Connessa: Il Perché del Workshop"
date: 2025-01-05T10:00:00+01:00
description: "Perché l'osservabilità diventa un problema quando log, metriche e tracce non sono connesse, e come OpenTelemetry risolve questo"
menu:
  sidebar:
    name: "1. Perché OTel"
    identifier: otel-1
    weight: 10
    parent: otel-workshop
tags: ["OpenTelemetry", "Observability", "Logging", "Tracing"]
categories: ["Observability", "DevOps", "Workshop"]
draft: false
---

*Tempo di lettura: ~8 minuti*

L'osservabilità nei sistemi distribuiti presenta un problema fondamentale: log, metriche e tracce provengono da strumenti diversi e non sono correlati.

Un servizio Node.js risulta lento. Tre tool mostrano informazioni separate:
- **Datadog**: "CPU 70%, memoria OK"
- **Loki**: "Error: timeout"
- **Jaeger**: "Traccia timeout fra servizio A e B"

La metrica indica **che** c'è un problema. Il log indica **cosa** è successo. La traccia mostra **dove**. Ma nessuno di questi tool collega le informazioni alla stessa richiesta.

```text
┌─────────────────┐
│   Datadog       │ ← "Qualcosa è lento"
├─────────────────┤
│   Loki          │ ← "Timeout a database"
├─────────────────┤
│   Jaeger        │ ← "Request timeout fra servizi"
└─────────────────┘
      Stessa richiesta?
      Non correlabile.
```

Il debugging richiede di correlare manualmente informazioni da fonti diverse.

---

## OpenTelemetry: La Soluzione

OpenTelemetry risolve questo problema con un approccio standard:

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

Quando una richiesta arriva nel sistema, OTel crea un trace ID che viaggia:
- Nel log
- Nella metrica
- Nella traccia
- Fra servizi diversi

I tre tool possono ora correlare i dati tramite lo stesso trace ID.

---

## La Struttura del Workshop

La documentazione ufficiale di OTel è completa ma orientata al riferimento. Questo workshop offre un percorso progressivo:

- Logging centralizzato → poi cosa?
- Tracce distribuite → cosa cambia nel codice?
- Metriche → influisce sul codice precedente?
- Sampling → come configurarlo?

Ogni modulo aggiunge un componente mantenendo compatibilità con i precedenti.

---

## Come È Strutturato il Workshop

6 moduli. Ognuno aggiunge un pezzo:

**Modulo 1: Logging Centralizzato**

Il cambio nel codice per logging centralizzato:

```javascript
// PRIMA: Log manuale con correlation ID
logger.info({ id: uuid.v4(), msg: 'Purchase started' });

// DOPO: OTel lo fa per me
logger.info('Purchase started'); // OTel gestisce l'ID
```

**Modulo 2: Tracce Distribuite**

Visualizzazione di una singola traccia che attraversa Node + Python + Go.

**Modulo 3: Metriche**

Aggiunta di metriche (rate, errors, latency) senza modificare il codice di logging.

**Modulo 4: Tail Sampling**

Configurazione del sampling per ridurre i costi mantenendo le tracce importanti.

**Modulo 5: Routing Intelligente**

Separazione dell'audit log dai debug log con destinazioni diverse.

**Modulo 6: Integrazione Completa**

Composizione di tutti i componenti in un sistema funzionante.

---

## Struttura dei Moduli

Ogni modulo contiene:

```text
module-01/
├── before/     ← Codice senza OTel
├── after/      ← Codice con OTel
├── current/    ← Ambiente di sperimentazione
├── script.md   ← Descrizione delle modifiche
└── docker-compose.yml  ← Setup completo
```

Esecuzione:

```bash
docker-compose up
curl http://localhost:3003/api/test
```

Gli strumenti sono immediatamente operativi.

---

## Prerequisiti

- Conoscenza base di Node.js (Express)
- Familiarità con Docker e `docker-compose`
- Nessuna esperienza precedente con OpenTelemetry richiesta

**Target:**
- Backend engineer che vogliono implementare OTel
- DevOps che devono strumentare applicazioni esistenti

---

## Contenuti

Il workshop copre:
- Strumentazione di un servizio Node con OTel
- Collegamento di tracce fra servizi in linguaggi diversi
- Aggiunta di metriche senza modificare il codice esistente
- Riduzione dei costi con tail sampling
- Routing di dati sensibili verso destinazioni separate

---

## Repository

Il codice completo del workshop è disponibile:

👉 **[otel-demo](https://github.com/monte97/otel-demo)**

---

*Serie OpenTelemetry Workshop:*
1. **Quando l'Osservabilità Non È Connessa** (questo articolo)
2. [Lo Standard È Importante: Come OTel Connette Servizi Diversi]({{< relref "/posts/otel-workshop/02-standard-otel" >}})
3. [Il Tuo Primo Servizio con OpenTelemetry]({{< relref "/posts/otel-workshop/03-primo-servizio" >}})
4. [Il Costo Nascosto dell'Osservabilità]({{< relref "/posts/otel-workshop/04-tail-sampling" >}})
5. [I Dati Dove Servono: Routing Intelligente]({{< relref "/posts/otel-workshop/05-routing" >}})
