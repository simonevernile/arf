# 🧱 Building Block – Requisito Architetturale per VM su GCP

## 🔹 Descrizione

Ogni volta che viene creata una **VM Compute Engine** su **GCP**, è obbligatoria l’associazione di una **componente PaaS di bilanciamento gestito** per garantire *alta disponibilità*, *scalabilità* e *resilienza*.

Il presente documento descrive le configurazioni **verificate e funzionanti** in Terraform per i tre casi architetturali principali:

- **FE:** External Managed HTTP(S) Load Balancer – pubblico, globale  
- **AL:** Internal Managed HTTP(S) Load Balancer – privato, regionale  
- **DB:** Internal TCP Load Balancer – privato, regionale (L4)

---

## 🧩 Struttura dei file Terraform

| File | Contenuto | Scope | Note |
|------|------------|--------|------|
| `load_balancers.tf` | Definizione dei tre LB, backend services, health checks e instance groups | Global + Regional | File principale |
| `network.tf` | Subnet proxy-only dedicata | Regional | Necessaria per ALB (`purpose = REGIONAL_MANAGED_PROXY`) |

---

## ⚙️ Schema generale (chiavi standard)

```yaml
schema:
  scope: [GLOBAL, REGIONAL]
  load_balancing_scheme: [EXTERNAL_MANAGED, INTERNAL_MANAGED, INTERNAL, INTERNAL_SELF_MANAGED]
  protocol: [HTTP, HTTPS, TCP, HTTP2, H2C]
  balancing_mode: [UTILIZATION, RATE, CONNECTION]
  ip_type: [PUBLIC_STATIC, PRIVATE_STATIC]
  health_check_type: [HTTP, HTTPS, TCP]
```

---

## 1️⃣ External Managed HTTP(S) Load Balancer (FE)

```yaml
lb: FE
scope: GLOBAL
ip_type: PUBLIC_STATIC
resources:
  backend_service: google_compute_backend_service
  url_map: google_compute_url_map
  target_proxy: google_compute_target_http_proxy
  forwarding_rule: google_compute_global_forwarding_rule
  health_check: google_compute_health_check
configuration:
  load_balancing_scheme: EXTERNAL_MANAGED
  protocol: HTTP
  port_name: http
  balancing_mode: UTILIZATION
  max_utilization: 0.6
  capacity_scaler: 1.0
  health_check:
    type: HTTP
    port: 80
  firewall:
    allow_ports: [80, 443]
    allow_sources: [130.211.0.0/22, 35.191.0.0/16]
notes:
  - Tutte le risorse sono **globali**
  - Nessuna subnet richiesta
  - Named Port MIG: `http:80`
  - Certificati: Managed SSL
  - Health check coerente con backend globale
```

---

## 2️⃣ Internal Managed HTTP(S) Load Balancer (AL)

```yaml
lb: AL
scope: REGIONAL
ip_type: PRIVATE_STATIC
resources:
  backend_service: google_compute_region_backend_service
  url_map: google_compute_region_url_map
  target_proxy: google_compute_region_target_http_proxy
  forwarding_rule: google_compute_forwarding_rule
  health_check: google_compute_region_health_check
  proxy_only_subnet: google_compute_subnetwork.proxy_only_subnet
configuration:
  load_balancing_scheme: INTERNAL_MANAGED
  protocol: HTTP
  port_name: http
  balancing_mode: UTILIZATION
  max_utilization: 0.6
  capacity_scaler: 1.0
  health_check:
    type: HTTP
    port: 80
  forwarding_rule:
    network: default
    subnetwork: proxy_only_subnet
    depends_on: [google_compute_subnetwork.proxy_only_subnet]
  proxy_only_subnet:
    purpose: REGIONAL_MANAGED_PROXY
    role: ACTIVE
    region: var.region
  firewall:
    allow_ports: [80, 443]
    allow_sources: [130.211.0.0/22, 35.191.0.0/16]
notes:
  - Tutte le risorse devono essere **REGIONAL**
  - `proxy_only_subnet` obbligatoria
  - Dipendenza esplicita `depends_on` per evitare race condition
  - Named port MIG: `http:80`
  - Certificati: Certificate Manager (managed/self)
  - Health check coerente con backend regionale
```

---

## 3️⃣ Internal TCP Load Balancer (DB)

```yaml
lb: DB
scope: REGIONAL
ip_type: PRIVATE_STATIC
resources:
  backend_service: google_compute_region_backend_service
  forwarding_rule: google_compute_forwarding_rule
  health_check: google_compute_region_health_check
configuration:
  load_balancing_scheme: INTERNAL
  protocol: TCP
  balancing_mode: CONNECTION
  max_connections_per_instance: 2000
  capacity_scaler: 1.0
  health_check:
    type: TCP
    port: 5432
  forwarding_rule:
    network: default
    subnetwork: default-subnet-new
    ports: [5432]
  firewall:
    allow_ports: [5432]
    allow_sources: [130.211.0.0/22, 35.191.0.0/16]
notes:
  - Tutti i componenti devono essere **REGIONAL**
  - `balancing_mode = CONNECTION` obbligatorio
  - Specificare sempre `max_connections*` per evitare errori di capacità
  - Nessuna URL map o target proxy
```

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

## 🧠 Validazioni automatiche consigliate

```yaml
validations:
  - check: scope_mismatch
    rule: forwarding_rule.scope == target_proxy.scope == backend_service.scope
  - check: missing_balancing_mode
    rule: backend_service.backend.balancing_mode != null
  - check: invalid_capacity
    rule: if balancing_mode == CONNECTION then max_connections* must exist
  - check: proxy_subnet_presence
    rule: if load_balancing_scheme == INTERNAL_MANAGED then proxy_only_subnet must exist
  - check: healthcheck_scope
    rule: health_check.scope == backend_service.scope
```

---

## 📘 Conclusioni

> Con questa configurazione:
> - Tutti i **Load Balancer** vengono creati correttamente al **primo `terraform apply`**.  
> - Nessun errore di **scope mismatch** o **capacità non valida**.  
> - Allineamento completo con **GCP API**, **Terraform Provider** e **Gemini CLI**.
