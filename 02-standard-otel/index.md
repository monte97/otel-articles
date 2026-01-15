---
title: "Lo Standard È Importante: Come OTel Connette Servizi Diversi"
date: 2025-01-06T10:00:00+01:00
description: "Perché OpenTelemetry come standard permette a servizi in Node.js, Python e Go di comunicare tracce senza configurazione manuale"
menu:
  sidebar:
    name: "2. Standard OTel"
    identifier: otel-2
    weight: 20
    parent: otel-workshop
tags: ["OpenTelemetry", "Standards", "Microservices", "Distributed Tracing"]
categories: ["Observability", "DevOps", "Workshop"]
draft: false
---

*Tempo di lettura: ~10 minuti*

In un'architettura a microservizi con linguaggi diversi, il tracing presenta un problema di interoperabilità.

Scenario tipico:
- Servizio **Node.js** (API principale)
- Servizio **Python** (inventory)
- Servizio **Go** (payment)

Una singola richiesta HTTP attraversa tutti e tre. Per identificare dove viene speso il tempo è necessario correlare le tracce fra servizi.

---

## Il Problema: Linguaggi Diversi, Tool Diversi

Senza uno standard comune:
- Il team Node usa un APM Node-specifico
- Il team Python usa un proprio strumento di tracing
- Il team Go usa un'altra soluzione

Tre strumenti con formati diversi per rappresentare gli ID di traccia.

```text
Node.js APM:
  └─ Request ID: "abc-123"
     └─ Timestamp: 1692000000

Python APM:
  └─ Trace ID: "xyz-789"
     └─ Timestamp: 1692000000 (è lo STESSO momento?)

Go APM:
  └─ Span ID: "def-456"
     └─ Timestamp: 1692000000 (sono connessi?)

→ Non correlabile
```

Ogni tool ha schema, formato e convenzioni proprietarie. Quando la richiesta passa da Node a Python, non esiste uno standard per propagare il trace ID.

---

## Le Alternative

Due approcci possibili:
1. **Propagazione manuale**: Ogni servizio estrae e passa l'ID nei log. Funziona ma è fragile e richiede manutenzione.
2. **Standard condiviso**: Tutti i servizi usano lo stesso formato e protocollo.

OpenTelemetry implementa l'approccio 2.

---

## Cosa Significa "Uno Standard"

OpenTelemetry non è un tool. È uno **standard di come scrivere osservabilità**.

Definisce:
- **Come** rappresentare una traccia (trace ID, span ID, timestamp, attributi)
- **Come** un servizio in Node passa il trace ID a un servizio Python
- **Come** rappresentare metriche e log nello stesso modo

Tutti i linguaggi (Node, Python, Go, Java, ecc.) hanno la stessa interpretazione.

```javascript
// Node.js con OTel
const span = tracer.startSpan('fetch_inventory');
span.setAttribute('user.id', userId);
span.end();
```

```python
# Python con OTel - STESSO CONCETTO
span = tracer.start_span('fetch_inventory')
span.set_attribute('user.id', user_id)
span.end()
```

```go
// Go con OTel - STESSO CONCETTO
span := tracer.Start(ctx, "fetch_inventory")
span.SetAttributes(attribute.String("user.id", userID))
span.End()
```

**Stessi attributi, stessa struttura, stessa idea.**

Quando la richiesta passa da Node a Python, Python **sa esattamente come leggere il trace ID** che Node gli passa. Perché c'è uno standard.

---

## L'Auto-Instrumentation: La Parte Magica

OTel ha qualcosa che si chiama **auto-instrumentation**.

Significa: "Non devo scrivere codice manuale per tracciare ogni operazione. OTel lo fa per me."

Quando faccio:

```javascript
const response = await axios.get('http://inventory-service:3007/data');
```

OTel **automaticamente**:
- Crea uno span per la HTTP call
- Aggiunge il trace ID all'header della richiesta
- Registra la latenza
- Attacca la risposta allo span

L'auto-instrumentation gestisce questi aspetti automaticamente.

Quando il servizio Python riceve la richiesta, l'SDK OTel Python legge il trace ID dall'header standard.

```python
# Python automaticamente legge il trace ID dall'header
# Non devo scrivere code per questo
# OTel lo sa fare
response = requests.get('http://payment-service:3008/process')
# La nuova richiesta avrà lo STESSO trace ID di quella da Node
```

---

## Il Risultato: Una Singola Traccia

Quando una richiesta attraversa Node → Python → Go, il risultato è:

```text
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

GET /buy (Node)           [0ms - 470ms]
  └─ axios.get (Node)     [10ms - 160ms]
     └─ GET /data (Python) [15ms - 150ms]
        └─ SELECT (Python) [20ms - 140ms]
  └─ axios.get (Node)     [165ms - 465ms]
     └─ POST /process (Go) [170ms - 460ms]
        └─ Stripe API (Go) [180ms - 450ms]
```

**Una singola ID. Una singola timeline. Tutto correlato.**

Non è necessario aprire tool separati o correlare manualmente gli ID. La propagazione è automatica.

---

## Perché Uno Standard Conta

Quando non c'è uno standard, succede questo:

```text
Node dice: "trace_id=abc"
Python interpreta come: Niente. Cosa significa abc?
Go non sa nemmeno che Node ha mandato un ID.

Risultato: Tre tracce disconnesse.
```

Con uno standard:

```text
Node dice: "traceparent=4bf92f3577b34da6a3ce929d0e0e4736"
Python legge: "Ah, traceparent è lo standard OTEL. Lo conosco."
Go legge: "Lo conosco anche io."

Risultato: Una traccia.
```

Questo è il vero valore di OTel: **non è uno strumento nuovo, è un standard condiviso**.

---

## La Libertà Che Uno Standard Dà

C'è un'altra cosa importante.

Se usi uno standard come OTel, **non sei legato a un vendor**.

```text
Con Datadog APM:
- Scrivi il codice usando Datadog SDK
- Sei legato a Datadog
- Se domani cambio a Honeycomb, riscrivo tutto

Con OTel:
- Scrivi il codice usando OTel SDK
- OTel è uno standard, supportato da chiunque
- Se domani cambio a Honeycomb, cambio solo l'endpoint del Collector
- Il codice rimane lo stesso
```

È come HTTP. Non sei legato a un server web specifico. Usare HTTP significa poter usare Nginx, Apache, Node.js, Go, ecc.

OTel è lo stesso per l'osservabilità.

---

## Nella Pratica

Il workshop dimostra la correlazione cross-language:

- **Modulo 1**: Servizio Node con logging
- **Modulo 2**: Aggiunta di servizi Python e Go con traccia unificata
- **Modulo 3**: Integrazione delle metriche

L'obiettivo è mostrare come uno standard comune connette sistemi eterogenei.

---

## Repository

👉 **[otel-demo](https://github.com/monte97/otel-demo)**

---

*Serie OpenTelemetry Workshop:*
1. [Quando l'Osservabilità Non È Connessa]({{< relref "/posts/otel-workshop/01-perche-otel" >}})
2. **Lo Standard È Importante: Come OTel Connette Servizi Diversi** (questo articolo)
3. [Il Tuo Primo Servizio con OpenTelemetry]({{< relref "/posts/otel-workshop/03-primo-servizio" >}})
4. [Il Costo Nascosto dell'Osservabilità]({{< relref "/posts/otel-workshop/04-tail-sampling" >}})
5. [I Dati Dove Servono: Routing Intelligente]({{< relref "/posts/otel-workshop/05-routing" >}})
