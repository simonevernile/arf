# 🧱 Building Block – Requisito Architetturale per VM su GCP (Load Balancer L4)

## 🔹 Descrizione

Se è presente un cluster di **VM Compute Engine** su **GCP** all'interno della SA, è obbligatoria l’associazione di una **componente di Load Balancer (L4)** per garantire *alta disponibilità*, *scalabilità* e *fault tolerance*.

Questa versione del building block definisce i tre casi architetturali standard basati su **TCP/UDP Load Balancer**, senza livelli HTTP(S):

- **FE:** External TCP/UDP Load Balancer – pubblico (accesso esterno)
- **AL:** Internal TCP/UDP Load Balancer – privato (livello applicativo)
- **DB:** Internal TCP Load Balancer – privato (livello database)

Tutti i bilanciatori operano a **livello 4**, utilizzando la regola `load_balancing_scheme` appropriata.

---

## ✅ Riepilogo Architetturale

| Caratteristica | FE | AL | DB |
|----------------|----|----|----|
| Scope | Global | Regional | Regional |
| Tipo LB | TCP/UDP pubblico | TCP/UDP interno | TCP interno |
| IP | Statico pubblico | Statico privato | Statico privato |
| Scheme | EXTERNAL | INTERNAL | INTERNAL |
| Protocollo | TCP/UDP | TCP/UDP | TCP |
| Balancing Mode | CONNECTION | CONNECTION | CONNECTION |
| Named Port | tcp:80 | tcp:8080 | tcp:5432 |
| Health Check | TCP (global) | TCP (regional) | TCP (regional) |
| Subnet Proxy | No | Sì (subnet interna dedicata) | Sì (subnet DB) |
| Certificati | N/A (L4) | N/A (L4) | N/A (L4) |
| Firewall | Porta applicativa (80/8080) | Porta applicativa (8080) | Porta DB (5432) |
| Dipendenze | Nessuna | Subnet interna | Subnet DB |

---
