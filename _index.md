---
title: "OpenTelemetry Workshop: Osservabilità Moderna"
description: "Workshop pratico su OpenTelemetry: dal logging centralizzato al distributed tracing, con esempi reali in Node.js, Python e Go"
menu:
  sidebar:
    name: OpenTelemetry Workshop
    identifier: otel-workshop
    weight: 55
---

Workshop pratico su **OpenTelemetry** e osservabilità moderna. Dalla teoria del perché ci serve, all'implementazione di un sistema completo con logging, tracing, metriche e sampling intelligente.

## Cosa imparerai

- Perché JSON/HTTP non bastano per l'osservabilità distribuita
- Come OTel connette servizi in linguaggi diversi (Node.js, Python, Go)
- Implementazione pratica di distributed tracing
- Riduzione costi con tail sampling (10x risparmio)
- Routing intelligente per compliance (GDPR, SOC2)

## Articoli della serie

1. **Perché OpenTelemetry** - Il problema dell'osservabilità disconnessa
2. **Lo Standard è Importante** - Come OTel connette servizi diversi
3. **Il Tuo Primo Servizio** - Implementazione pratica con Node.js
4. **Il Costo Nascosto** - Tail sampling per ridurre storage e costi
5. **I Dati Dove Servono** - Routing intelligente per compliance

## Prerequisiti

- Conoscenza base di Node.js (Express)
- Docker e Docker Compose installati
- Familiarità con concetti di microservizi

## Repository Demo

Il codice completo del workshop è organizzato in moduli progressivi:

👉 **[otel-demo](https://github.com/monte97/otel-demo)**

### Struttura moduli

```text
otel-demo/
├── module-01/    # Logging centralizzato
├── module-02/    # Distributed tracing
├── module-03/    # Metriche
├── module-04/    # Tail sampling
├── module-05/    # Audit e sampling avanzato
└── module-06/    # Routing intelligente
```

### Quick Start

```bash
# Clone del repository
git clone https://github.com/monte97/otel-demo.git
cd otel-demo

# Avvia un modulo
cd module-03
docker compose up

# Testa
curl http://localhost:3003/buy?userId=test&productId=123

# Visualizza tracce in Jaeger
open http://localhost:16686
```

## Stack tecnologico

| Tecnologia | Utilizzo |
|------------|----------|
| OpenTelemetry SDK | Instrumentazione servizi |
| OTel Collector | Ricezione e routing dati |
| Jaeger | Visualizzazione tracce |
| Loki | Storage log |
| Grafana | Dashboard |
