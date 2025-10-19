## 🧱 **Building Block – Requisito Architetturale per VM su GCP**

### 🔹 Descrizione

Ogni volta che viene creata una **VM Compute Engine** su **GCP**, è obbligatoria l’associazione di una **componente PaaS di bilanciamento gestito** per garantire *alta disponibilità*, *scalabilità* e *resilienza* in linea con gli standard architetturali.

Il Load Balancer deve essere gestito secondo la seguente regola:

* se nel campo **Cluster** è valorizzato, viene creato **un Load Balancer dedicato per cluster**;
* se il campo **Cluster** non è valorizzato, viene creato **un unico Load Balancer condiviso** per tutte le VM del medesimo ambiente o progetto.

---

### 🔹 Componente PaaS richiesta

**Servizio:** Google Cloud Load Balancing (gestito)
**Tipologie supportate:**

* **FE (Front-End):** *External Managed HTTP(S) Load Balancer* (globale) – **IP Pubblico**
* **AL (App Layer):** *Internal Managed HTTP(S) Load Balancer* (regionale) – **IP Privato**
* **DB (Data/Backend TCP):** *Internal TCP Load Balancer* (regionale) – **IP Privato**

| Caratteristica              | FE (External HTTPS)                                       | AL (Internal HTTPS)                                                               | DB (Internal TCP)                                                     |
| :-------------------------- | :-------------------------------------------------------- | :-------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| **Scopo**                   | Pubblicazione servizi web verso Internet                  | Bilanciamento intra-VPC per servizi applicativi HTTP(S)                           | Bilanciamento TCP per servizi DB (es. 5432/3306/1521)                 |
| **Frontend IP**             | **Statico pubblico** (Global)                             | **Statico privato** (Regional, subnet interna/proxy-only)                         | **Statico privato** (Regional, subnet interna)                        |
| **Certificati**             | **Managed SSL** (auto-renew)                              | Certificati tramite **Certificate Manager** (managed o self-managed, DNS privato) | N/D (livello TCP)                                                     |
| **Backend**                 | MIG/IG delle VM del cluster                               | MIG/IG delle VM del cluster                                                       | MIG/IG delle VM DB                                                    |
| **Health Check**            | HTTP su `/healthz`, timeout ≤ 5s, 3 tentativi             | HTTP su `/healthz`, timeout ≤ 5s, 3 tentativi                                     | TCP sulla porta del DB (es. `var.db_port`), timeout ≤ 5s, 3 tentativi |
| **Autoscaling**             | MIG con target CPU ≤ 60%, max 5                           | MIG con target CPU ≤ 60%, max 5                                                   | **Facoltativo** (tipicamente fisso per DB)                            |
| **Firewall**                | Allow 80/443 *da LB*; allow ranges **health-checker** GCP | Allow 80/443 dal VPC; allow ranges **health-checker** GCP                         | Allow `db_port` dal VPC; allow ranges **health-checker** GCP          |
| **Logging/Monitoring**      | Abilitati (Cloud Logging/Monitoring)                      | Abilitati                                                                         | Abilitati                                                             |
| **Scope**                   | **Globale** (Application LB esterno)                      | **Regionale** (Application LB interno, richiede **proxy-only subnet**)            | **Regionale** (TCP LB interno)                                        |
| **Regola cluster**          | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto    | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto                            | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto                |
| **load_balancing_scheme**   | `EXTERNAL_MANAGED`                                        | `INTERNAL_MANAGED`                                                                | `INTERNAL` *(protocol omesso)*                                        |
| **protocol**                | `HTTP` *(oppure HTTPS/HTTP2 se richiesto)*                | `HTTP` *(oppure HTTPS/HTTP2/H2C)*                                                 | **Non impostare** (obbligatorio omettere per passthrough)             |
| **backend.balancing_mode**  | `UTILIZATION`                                             | `UTILIZATION`                                                                     | `CONNECTION`                                                          |
| **backend.max_utilization** | Valido se `UTILIZATION`, es. `0.6`                        | Valido se `UTILIZATION`, es. `0.6`                                                | **Non applicabile**                                                   |
| **Note tecniche**           | Supporta `UTILIZATION`, `RATE`, `CUSTOM_METRICS`          | Supporta `UTILIZATION`, `RATE`, `CUSTOM_METRICS`                                  | Passthrough L4, niente target capacity                                |

> **Note operative GCP**
>
> * **FE:** External Managed HTTPS LB richiede risorse globali (URL map, target HTTPS proxy, global forwarding rule).
> * **AL:** Internal Managed HTTPS LB richiede **proxy-only subnet** nella regione e un **internal managed certificate** o self-managed in **Certificate Manager** + DNS privato.
> * **DB:** Internal TCP LB usa **regional forwarding rule**, **backend service TCP**, **health check TCP**.

---

### 🔹 Motivazione

* Garantire *High Availability* e *Fault Tolerance* senza configurazioni manuali.
* Centralizzare il traffico e semplificare la gestione di certificati e health check.
* Evitare proliferazione di LB non necessari, ottimizzando costi e manutenzione.
* Uniformare la configurazione delle architetture su GCP secondo lo standard enterprise.

---

### 🔹 Output atteso

| Risorsa                     | Tipo                            | Note                                                                  |
| :-------------------------- | :------------------------------ | :-------------------------------------------------------------------- |
| `lb-<tipo>-<cluster>-<env>` | Load Balancer (FE/AL/DB)        | dedicato per cluster; se `Cluster` assente → `lb-<tipo>-shared-<env>` |
| `hc-<tipo>-<servizio>`      | Health Check                    | HTTP (`/healthz`) per FE/AL, TCP per DB                               |
| `cert-<servizio>`           | Managed/Self SSL (FE/AL)        | Gestito tramite Certificate Manager                                   |
| `ip-<tipo>-<env>`           | Indirizzo IP (pubblico/privato) | Statico: pubblico solo per **FE**, privato per **AL/DB**              |
| `fw-allow-<tipo>-...`       | Regole Firewall                 | Ingress dal LB e dagli health-checker GCP                             |

---

## 💡 **Regole di conformità (riassunto)**

* **FE:** External HTTPS LB con **IP pubblico** statico globale, cert **managed**, URL Map/Proxy/FR globali, `load_balancing_scheme = EXTERNAL_MANAGED`, `protocol = HTTP`, `balancing_mode = UTILIZATION`, `max_utilization = 0.6`.
* **AL:** Internal HTTPS LB con **IP privato** regionale, **proxy-only subnet** obbligatoria, cert in **Certificate Manager**, `load_balancing_scheme = INTERNAL_MANAGED`, `protocol = HTTP`, `balancing_mode = UTILIZATION`, `max_utilization = 0.6`.
* **DB:** Internal TCP LB con **IP privato** regionale, health check **TCP** sulla porta DB, `load_balancing_scheme = INTERNAL`, **senza protocol**, `balancing_mode = CONNECTION`.
* **Cluster management:** naming `lb-<tipo>-<cluster>-<env>` oppure `lb-<tipo>-shared-<env>`.
* **Logging/Monitoring:** abilitati su backend service.
* **Firewall:** consentire **health-checker** GCP e porte servizio; 80/443 per FE/AL, `db_port` per DB.
