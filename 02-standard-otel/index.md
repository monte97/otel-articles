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

Quando inizi a lavorare con microservizi in linguaggi diversi, scopri in fretta il problema.

Hai:
- Un servizio **Node.js** (API principale)
- Un servizio **Python** (inventory)
- Un servizio **Go** (payment)

Tutto su stessa richiesta HTTP.

Domanda semplice: **come faccio a vedere dove ci impiega più tempo?**

---

## Il Problema: Linguaggi Diversi, Tool Diversi

Senza OTel:
- Il team Node ha il suo strumento di tracing (tipo APM di Node)
- Il team Python ha il suo strumento
- Il team Go ha il suo strumento

Sono tre "lingue" diverse. Tre tool diversi. Tre modi diversi di scrivere ID di traccia.

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

Non lo so.
```

Il problema? Ognuno di questi tool ha il suo schema, il suo formato, le sue convenzioni.

Quando la richiesta passa da Node a Python, non c'è uno standard che dice "passa questo ID in questo modo". Ognuno fa il suo.

---

## Quando Ho Capito Che Uno Standard Era Necessario

Potevo fare:
1. **Passare manualmente l'ID**: Ogni servizio copia-incolla l'ID dal precedente nei log. Funziona ma è manuale e fragile.
2. **Usare uno standard condiviso**: Tutti leggono e scrivono con lo stesso formato.

Mi sembrò ovvio che la soluzione era la 2.

Cominciai a cercare se esisteva uno standard così. E scoprii che sì: **OpenTelemetry**.

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

Non ho scritto nulla di questo. OTel lo ha fatto.

Quando il servizio Python riceve la richiesta, OTel Python **sa come leggere il trace ID** dal header (perché è uno standard).

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

Non devo aprire tre tool diversi. Non devo copiare manualmente gli ID. Non devo sperare che i timestamp corrispondano.

È tutto automatico.

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

## Nel Workshop

Nel workshop mostro esattamente questa cosa.

- Modulo 1: Un servizio Node con logging
- Modulo 2: Aggiungi due servizi (Python e Go) e guarda una singola traccia
- Modulo 3: Aggiungi metriche

**Non è "impara OTel".** È "**vedi come uno standard connette sistemi diversi**."

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
