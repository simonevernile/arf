# 🧱 Building Block – Requisito Architetturale per VM su GCP

## 🔹 Descrizione

Ogni volta che viene creata una **VM Compute Engine** su **GCP**, è obbligatoria l’associazione di una **componente PaaS di bilanciamento gestito** per garantire *alta disponibilità*, *scalabilità* e *resilienza*.

Il presente documento descrive le configurazioni **verificate e funzionanti** in Terraform per i tre casi architetturali principali:

- **FE:** External Managed HTTP(S) Load Balancer – pubblico, globale  
- **AL:** Internal Managed HTTP(S) Load Balancer – privato, regionale  
- **DB:** Internal TCP Load Balancer – privato, regionale (L4)

---

## ✅ Riepilogo Architetturale

| Caratteristica | FE | AL | DB |
|----------------|----|----|----|
| Scope | Global | Regional | Regional |
| Tipo LB | HTTP(S) pubblico | HTTP(S) interno | TCP interno |
| IP | Statico pubblico | Statico privato | Statico privato |
| Scheme | EXTERNAL_MANAGED | INTERNAL_MANAGED | INTERNAL |
| Protocollo | HTTP/HTTPS | HTTP | TCP |
| Balancing Mode | UTILIZATION | UTILIZATION | CONNECTION |
| Named Port | http:80 | http:80 | N/A |
| Health Check | HTTP (global) | HTTP (regional) | TCP (regional) |
| Subnet Proxy | No | Sì (`REGIONAL_MANAGED_PROXY`) | No |
| Certificati | Managed SSL | Certificate Manager | N/A |
| Firewall | 80/443 | 80/443 | 5432 |
| Dipendenze | Nessuna | proxy_only_subnet | Nessuna |

---

## 💡 Best Practice Operative

1. **Scope coerente:** Global → tutte le risorse globali; Regional → tutte le risorse regionali.  
2. **Proxy-only subnet:** obbligatoria per ALB (`purpose = REGIONAL_MANAGED_PROXY`).  
3. **Balancing mode:** sempre specificato (`UTILIZATION` o `CONNECTION`).  
4. **Capacità:** definire `max_utilization` o `max_connections*`.  
5. **Dipendenze esplicite:** usare `depends_on` per evitare race condition.  
6. **Named Port:** allineato a `port_name`.  
7. **Health Check:** coerente con il backend.  
8. **Firewall:** includere i range health-checker `130.211.0.0/22` e `35.191.0.0/16`.  

---
