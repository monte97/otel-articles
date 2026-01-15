---
title: "I Dati Dove Servono: Routing Intelligente con OpenTelemetry"
date: 2025-01-09T10:00:00+01:00
description: "Come instradare log e tracce verso destinazioni diverse per compliance GDPR e SOC2 usando il Routing Processor di OTel Collector"
menu:
  sidebar:
    name: "5. Routing"
    identifier: otel-5
    weight: 50
    parent: otel-workshop
tags: ["OpenTelemetry", "Compliance", "GDPR", "Routing", "Logging"]
categories: ["Observability", "Security", "Workshop"]
draft: false
---

*Tempo di lettura: ~8 minuti*

Quando usi OTel e inizi a loggare, tutto finisce nel **OTel Collector**.

Il Collector riceve tutto e lo manda dove è configurato.

```text
├─ Log tecnico (debug, info, errors)
├─ Audit log (chi ha fatto cosa, quando)
└─ Log di sistema (health checks, internal operations)

Tutto → OTel Collector → Loki
```

Il problema: **non tutti questi log vanno nello stesso posto.**

Con GDPR e compliance requirements:
- **Audit log** deve essere in accesso ristretto (compliance team only)
- **Log tecnico** può andare a Loki (per il debugging)
- **Log di sistema** può essere scartato (è rumore)

Se tutto va nello stesso Loki, un developer vede **TUTTO**: audit log, log sensibili, tutto.

Questo non va bene per compliance.

```text
Loki (accesso al team tecnico):
  [Payment of €5000 by alice@example.com] ← AUDIT, sensibile!
  [Internal cache miss]                     ← Tecnico, ok
  [Debug info]                               ← Tecnico, ok
```

Soluzione: **fare il Collector intelligente**.

---

## La Soluzione: Routing nel Collector

Il Collector ha un **Routing Processor**.

La logica è semplice: il Collector **legge gli attributi dei log e decide dove mandarli.**

```yaml
processors:
  routing:
    default_exporters: [loki]
    from_attributes: [log.type]
    routes:
      audit:
        exporters: [audit_service]
      technical:
        exporters: [loki]
      debug:
        exporters: []  # Scarta (too noisy)

exporters:
  loki:
    endpoint: http://loki:3100
    tenant_id: "technical"

  audit_service:
    endpoint: http://audit-service:4317
    headers:
      Authorization: "Bearer audit-token"
```

---

## Nel Tuo Codice: Semplice Marcatura

Nel tuo `index.js`, aggiungi un attributo `log.type` ai tuoi log:

```javascript
// Log tecnico (va a Loki, per il debugging)
app.get('/api/data', (req, res) => {
  logger.info({
    message: 'API data fetched',
    'log.type': 'technical'
  });
  res.json({ data: [...] });
});

// Audit log (va a audit-service, protetto)
app.post('/checkout', (req, res) => {
  const user = req.user.id;
  const amount = req.body.amount;

  logger.info({
    message: 'Checkout completed',
    amount,
    user,
    'log.type': 'audit'  // ← Questo attributo
  });

  res.json({ ok: true });
});
```

Il Collector legge l'attributo `log.type`:
- Se vede `'audit'`, lo manda a un servizio protetto
- Se vede `'technical'`, lo manda a Loki
- Se vede `'debug'`, lo scarta

**È tutto quello che serve.**

---

## Il Risultato

**Prima del Routing:**

```text
Loki (everyone has access):
  Payment: €5000 by alice@example.com ← AUDIT, sensibile!
  Cache miss                           ← Tecnico
  Query slow                           ← Tecnico
```

Un developer vede i dati di pagamento. Non va bene.

**Dopo il Routing:**

```text
Loki (dev team):
  Cache miss
  Query slow

Audit Service (compliance only):
  Payment: €5000 by alice@example.com
  (Encrypted, restricted access)
```

Developer accede solo ai log tecnici. Audit è separato. Compliance è soddisfatta.

---

## Destinazioni Tipiche

Nel tuo Collector puoi configurare tre route principali:

```yaml
routes:
  technical:
    exporters: [loki]
    # Per debug, performance, errori tecnici
    # Accessibile a: dev team

  audit:
    exporters: [audit_service]
    # Per compliance, GDPR, SOC2
    # Accessibile a: compliance team only

  debug:
    exporters: []  # Scarta
    # No retention, it's noise
```

---

## Un Esempio Pratico

Quando un utente fa checkout:

```javascript
app.post('/checkout', async (req, res) => {
  const user = req.user.id;
  const amount = req.body.amount;

  // 1. Log di inizio (audit)
  logger.info({
    event: 'checkout_started',
    user,
    amount,
    'log.type': 'audit'  // → audit-service
  });

  try {
    const transactionId = `TXN-${Date.now()}`;

    // 2. Log di successo (audit)
    logger.info({
      event: 'payment_success',
      transactionId,
      'log.type': 'audit'  // → audit-service
    });

    res.json({ orderId: transactionId });
  } catch (error) {
    // 3. Log di errore (tecnico, per debug)
    logger.error({
      event: 'payment_failed',
      error: error.message,
      'log.type': 'technical'  // → Loki
    });

    res.status(500).json({ error: 'Failed' });
  }
});
```

**Il routing automatico:**
- Audit logs → servizio protetto (compliance vede chi ha pagato cosa)
- Error logs → Loki (dev team debugga)

---

## Benefici

1. **Compliance**: GDPR/SOC2 automaticamente soddisfatti
2. **Security**: Dati sensibili non visibili a tutti
3. **Performance**: Loki non pieno di audit log
4. **Cost**: Storage diverso per dati diversi

---

## La Configurazione Completa

Ecco come mettere tutto insieme:

```yaml
# collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  routing:
    default_exporters: [loki]
    from_attributes: [log.type]
    routes:
      audit:
        exporters: [audit_otlp]
      technical:
        exporters: [loki]
      debug:
        exporters: []  # discard

exporters:
  # Technical logs (fast, accessible)
  loki:
    endpoint: http://loki:3100
    tenant_id: "technical"

  # Audit logs (encrypted, ristretto)
  audit_otlp:
    endpoint: http://audit-service:4317
    headers:
      Authorization: "Bearer secret-token"

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [routing]
      exporters: []  # routing decide!
```

---

## Cosa Ho Imparato

1. **Non tutti i dati sono uguali.** Trattali diversamente.
2. **Il Routing Processor è potente e semplice.** Una riga di codice nel tuo log.
3. **Compliance non è "aggiunto dopo".** È baked-in dal primo giorno.

---

## Riepilogo della Serie

In questa serie abbiamo costruito un sistema di osservabilità completo:

- **Logging centralizzato** con trace ID correlati
- **Distributed tracing** fra servizi Node.js, Python e Go
- **Metriche** per misurare performance
- **Tail sampling** per ridurre costi del 90%
- **Routing intelligente** per compliance GDPR/SOC2

**Un singolo SDK, un singolo Collector, mille possibilità.**

---

## Repository

Tutto il codice è disponibile:

👉 **[otel-demo](https://github.com/monte97/otel-demo)**

```bash
cd otel-demo/module-06
docker-compose up

# Genera un checkout
curl -X POST http://localhost:8004/checkout \
  -H "Content-Type: application/json" \
  -d '{"user": "alice@example.com", "amount": 5000}'

# Vedi l'audit log andare a audit-service
docker logs module-06-audit-service-1

# Vedi i log tecnici su Loki via Grafana
```

---

*Serie OpenTelemetry Workshop:*
1. [Quando l'Osservabilità Non È Connessa]({{< relref "/posts/otel-workshop/01-perche-otel" >}})
2. [Lo Standard È Importante: Come OTel Connette Servizi Diversi]({{< relref "/posts/otel-workshop/02-standard-otel" >}})
3. [Il Tuo Primo Servizio con OpenTelemetry]({{< relref "/posts/otel-workshop/03-primo-servizio" >}})
4. [Il Costo Nascosto dell'Osservabilità]({{< relref "/posts/otel-workshop/04-tail-sampling" >}})
5. **I Dati Dove Servono: Routing Intelligente** (questo articolo)
