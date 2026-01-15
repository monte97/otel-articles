# Il Costo Nascosto dell'Osservabilità: Come Ridurlo Senza Perdere i Dati Importanti

## Il Problema Reale: Tracciare Tutto Costa

Quando inizi con OTel, potrebbe sembrare: "Perfetto! Ora traccio tutto!"

Ma c'è un problema.

Se tracciassi **ogni richiesta**, **ogni call**, **ogni evento**, il tuo storage esploderebbe. E i costi con lui.

```
1000 richieste/secondo × 8 span per richiesta
= 8000 span/secondo
= 3.11 MILIARDI di span al mese
= 10+ TB di storage
= $3000+/mese in costi
```

E tutto per tracciare anche cose inutili:
- Health check ogni 5 secondi? Tracciato.
- Request di monitoraggio? Tracciato.
- Call velocissime che non hanno mai problemi? Tracciato.

---

## Il Problema Reale: Non Tutte le Tracce Sono Uguali

```
GET /health → 2ms (inutile, traccia ogni 5 secondi)
GET /api/data → 150ms (utile, ma ce ne sono milioni)
POST /checkout → 500ms (IMPORTANTE, richiesta rara ma critica)
ERROR: Timeout → 5s (IMPORTANTE, voglio sapere cosa è successo)
```

Ma stavo pagando per **tutte** allo stesso modo.

Era come pagare l'assicurazione auto al massimo della copertura anche per i parcheggi in città.

---

## Tail Sampling: La Soluzione

La logica è semplice:

**Senza Tail Sampling (il mio modo vecchio):**
```
Ogni richiesta
  ↓
Decidi se tracciare (random 10% di tutte)
  ↓
Invia al backend

Problema: Traccia il 10% dei health check
          Perdi il 90% dei POST /checkout che vanno male
```

**Con Tail Sampling (intelligente):**
```
Ogni richiesta
  ↓
Crea la traccia (temporaneamente)
  ↓
Analizza: è un errore? È un'operazione critica? È lenta?
  ↓
Decidi:
  - Errore? SALVA 100%
  - Operazione di audit (checkout, login)? SALVA 100%
  - Richiesta lenta (>1s)? SALVA 100%
  - Health check? SCARTA (0%)
  - Request normale? SALVA il 10%

Beneficio: Salvi TUTTE le cose importanti
           Scardi il rumore inutile
           Riduci costi di 10x
```

---

## I Numeri: Scenario Reale

Scenario: E-commerce con **1000 richieste/secondo**.

Breakdown:
- 400/sec: Health checks (nessuno le vuole tracciare)
- 500/sec: Request normali di shopping
- 80/sec: Checkout (importante, salva sempre)
- 15/sec: Errori (importante, salva sempre)
- 5/sec: Admin operations (importante, salva sempre)

---

## Senza Tail Sampling (Tracciare Tutto)

```
Tracce/sec: 1000 (tutte!)
Span per traccia: 8 (media)
Span/sec: 8000

Per mese:
8000 span/sec × 86400 sec/day × 30 days = 20.736 MILIARDI span

Storage per span: ~500 bytes
20.736B × 500 bytes = 10.4 TB

Costo (a ~$0.023/GB):
10,400 GB × $0.023 = $240/mese solo per storage

Con query, retention, alerting: +$2000-3000/mese per Datadog
```

**Totale: ~$3000/mese per il tracing.**

---

## Con Tail Sampling (Il Modo Intelligente)

```
Calcolo per ogni categoria:

Health checks (400/sec): SCARTA 100%
  → 0 tracce salvate

Normal requests (500/sec): SALVA 10%
  → 50 tracce/sec

Checkout (80/sec): SALVA 100%
  → 80 tracce/sec

Errors (15/sec): SALVA 100%
  → 15 tracce/sec

Admin (5/sec): SALVA 100%
  → 5 tracce/sec

TOTALE salvate: 150 tracce/sec (vs 1000!)

Span salvati: 150 × 8 = 1200 span/sec (vs 8000!)

Per mese:
1200 span/sec × 86400 × 30 = 3.11 MILIARDI span

Storage:
3.11B × 500 bytes = 1.55 TB

Costo:
1550 GB × $0.023 = $35/mese per storage
Con Datadog light tier: ~$300/mese

TOTALE: ~$300/mese
```

**Sono passato da $3000 a $300.**

**10x di riduzione.**

---

## Come Funziona Tecnicamente?

Nel tuo `collector-config.yaml`:

```yaml
processors:
  tail_sampling:
    policies:
      # Policy 1: Salva TUTTI gli errori
      - name: error_policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Policy 2: Salva operazioni audit/checkout
      - name: audit_policy
        type: attribute
        attribute:
          key: audit.event
          values: [true]

      # Policy 3: Salva richieste lente
      - name: latency_policy
        type: latency
        latency:
          threshold_ms: 1000

      # Policy 4: Salva il 10% delle altre (normali)
      - name: probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

      # Policy 5: Scarta health checks
      - name: drop_health_checks
        type: regex_attribute
        regex_attribute:
          regex: "health|ping|status"
          attribute_key: "http.url"
          action: drop

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling]
      exporters: [jaeger]
```

---

## Nel Tuo Codice

Per markare una richiesta come "importante", aggiungi questo:

```javascript
// checkout endpoint - importante!
app.post('/checkout', (req, res) => {
  const span = trace.getActiveSpan();

  // Questo attributo fa sì che il Collector la SALVI
  if (span) {
    span.setAttribute('audit.event', true);
    span.setAttribute('audit.user', req.user.id);
    span.setAttribute('audit.amount', req.body.amount);
  }

  // ... rest of code ...
});

// health check - non importante, scartabile
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
  // Niente setAttribute, il Collector la scarta
});
```

---

## Le Metriche Che Contano

Quando implementi il tail sampling, devi misurare:

```
Prima:
- Storage usage: 10.4 TB/mese
- Costo: $3000/mese
- Query latency: 5-10s (troppi dati)
- Retention: max 7 giorni (costo)

Dopo:
- Storage usage: 1.55 TB/mese ← 85% meno
- Costo: $300/mese ← 90% risparmio
- Query latency: <500ms (meno dati!)
- Retention: 90 giorni (puoi permettertelo!)

Ma cosa è importante: NON hai perso NULLA di critico
- 100% degli errori: salvati
- 100% dei checkout: salvati
- 100% delle operazioni lente: salvate
```

---

## Cosa Devi Sapere

### 1. **Tail Sampling ha un trade-off: memoria vs intelligenza**

Il Collector deve **attendere che la traccia sia completa** prima di decidere se salvarla.

Significa: 100-500ms di latenza in memoria nel Collector.

Non è un problema per la maggior parte dei casi, ma è un'informazione importante.

### 2. **Devi decidere quali richieste sono importanti**

Non è "plug and play". Devi pensare al tuo caso:

- **E-commerce**: Checkout, payment, cart operations
- **SaaS**: Login, configuration changes, admin operations
- **API pubblica**: Errori, rate-limit violations, slow queries
- **Tutti**: Errors sono SEMPRE importanti

---

## Il Valore Reale

La cosa importante è capire questo:

**Più dati non significa migliore osservabilità.**

Significa storage pieno, query lente, costi alti, rumore.

Con Tail Sampling:
- 85% di riduzione dello storage
- 90% di riduzione dei costi
- Ma ZERO perdita di dati importanti (errori, audit, operazioni critiche)

---

## Nel Prossimo Articolo

Parlerò di come **instradare i tuoi dati dove servono davvero**.

Non tutte le tracce, log, metriche vanno nello stesso posto:

- I log tecnici vanno a Loki
- L'audit log va a un servizio separato (compliance)
- Le metriche vanno a Prometheus
- Gli errori critici vanno ad una chat Slack

**Un singolo SDK, mille destinazioni.**

👉 Leggi il prossimo articolo.

---

**Vuoi provare il Tail Sampling?**

Ho preparato il modulo 5 del workshop con tutto configurato:

```bash
cd module-05-auditing-sampling
docker-compose up
# Genera traffico
curl -X POST http://localhost:7003/checkout -d '{"amount": 5000}'
# Guarda Jaeger e vedi che QUESTA traccia viene salvata
```

#OpenTelemetry #CostOptimization #Observability #Sampling
