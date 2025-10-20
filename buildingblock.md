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
* **DB (Data/Backend TCP):** *Internal TCP Load Balancer* (regionale, L4 passthrough) – **IP Privato**

| Caratteristica             | FE (External HTTPS)                                         | AL (Internal HTTPS)                                                                               | DB (Internal TCP)                                                                                                                                                          |
| :------------------------- | :---------------------------------------------------------- | :------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Scopo**                  | Pubblicazione servizi web verso Internet                    | Bilanciamento intra‑VPC per servizi applicativi HTTP(S)                                           | Bilanciamento TCP per servizi DB (es. 5432/3306/1521)                                                                                                                      |
| **Frontend IP**            | **Statico pubblico** (Global)                               | **Statico privato** (Regional, subnet interna/proxy‑only)                                         | **Statico privato** (Regional, subnet interna)                                                                                                                             |
| **Certificati**            | **Managed SSL** (auto‑renew)                                | Certificati tramite **Certificate Manager** (managed o self‑managed, DNS privato)                 | N/D (livello TCP)                                                                                                                                                          |
| **Backend**                | MIG/IG delle VM del cluster                                 | MIG/IG delle VM del cluster                                                                       | MIG/IG delle VM DB                                                                                                                                                         |
| **Health Check**           | HTTP su `/healthz`, timeout ≤ 5s, 3 tentativi               | HTTP su `/healthz`, timeout ≤ 5s, 3 tentativi                                                     | TCP sulla porta del DB (es. 5432), timeout ≤ 5s, 3 tentativi                                                                                                               |
| **Autoscaling**            | MIG con target CPU ≤ 60%, max 5                             | MIG con target CPU ≤ 60%, max 5                                                                   | **Facoltativo** (tipicamente fisso per DB)                                                                                                                                 |
| **Firewall**               | Allow 80/443 *da LB*; allow ranges **health‑checker** GCP   | Allow 80/443 dal VPC; allow ranges **health‑checker** GCP                                         | Allow `db_port` dal VPC; allow ranges **health‑checker** GCP                                                                                                               |
| **Logging/Monitoring**     | Abilitati (Cloud Logging/Monitoring)                        | Abilitati                                                                                         | Abilitati                                                                                                                                                                  |
| **Scope**                  | **Globale** (Application LB esterno)                        | **Regionale** (tutti i componenti **devono** essere regionali: URL Map, Target HTTPS Proxy, FR)   | **Regionale** (TCP LB interno)                                                                                                                                             |
| **Regola cluster**         | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto      | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto                                            | 1 LB per `Cluster` o 1 LB shared per ambiente/progetto                                                                                                                     |
| **load_balancing_scheme**  | `EXTERNAL_MANAGED`                                          | `INTERNAL_MANAGED`                                                                                | `INTERNAL`                                                                                                                                                                 |
| **protocol**               | `HTTP` *(oppure HTTPS/HTTP2 se richiesto)*                  | `HTTP` *(oppure HTTPS/HTTP2/H2C)*                                                                 | `TCP`                                                                                                                                                                      |
| **backend.balancing_mode** | `UTILIZATION`                                               | `UTILIZATION`                                                                                     | `CONNECTION`                                                                                                                                                               |
| **Target capacity**        | Se `UTILIZATION`: `max_utilization` in [0.0,1.0] (es. 0.6)  | Se `UTILIZATION`: `max_utilization` in [0.0,1.0] (es. 0.6)                                        | **Obbligatorio** con `CONNECTION`: impostare `max_connections` *oppure* `max_connections_per_instance` (o `per_endpoint`). Impostare anche `capacity_scaler > 0` se usato. |
| **Note tecniche**          | Named port del MIG coerente con `port_name` (es. `http:80`) | Tutti gli oggetti **regionali**; proxy‑only subnet richiesta; named port coerente con `port_name` | Passthrough L4; non usare `max_utilization`/`max_rate*` con `CONNECTION`                                                                                                   |

> **Note operative GCP**
>
> * **FE:** External Managed HTTPS LB richiede risorse globali (URL map, target HTTPS proxy, global forwarding rule).
> * **AL:** Internal Managed HTTPS LB richiede **proxy‑only subnet** nella regione e **tutti** i componenti in **scope regionale** (region URL map, region target HTTPS proxy, regional forwarding rule).
> * **DB:** Internal TCP LB usa **regional forwarding rule**, **region backend service (protocol = TCP)**, **health check TCP** e `balancing_mode = CONNECTION` con capacità esplicita.

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
| `fw-allow-<tipo>-...`       | Regole Firewall                 | Ingress dal LB e dagli health‑checker GCP                             |

---

## 💡 **Regole di conformità (riassunto)**

* **FE:** External HTTPS LB con **IP pubblico** statico globale, cert **managed**, URL Map/Proxy/FR globali; `load_balancing_scheme = EXTERNAL_MANAGED`, `protocol = HTTP`, `balancing_mode = UTILIZATION`, `max_utilization = 0.6`; **named port** del MIG coerente con `port_name`.
* **AL:** Internal HTTPS LB con **IP privato** regionale; **tutti i componenti regionali** (region URL map, region target HTTPS proxy, regional forwarding rule); `load_balancing_scheme = INTERNAL_MANAGED`, `protocol = HTTP`, `balancing_mode = UTILIZATION`, `max_utilization = 0.6`; **proxy‑only subnet** obbligatoria.
* **DB:** Internal TCP LB con **IP privato** regionale; `load_balancing_scheme = INTERNAL`, `protocol = TCP`, `balancing_mode = CONNECTION` **con** `max_connections` oppure `max_connections_per_instance` (o `per_endpoint`); health check **TCP** sulla porta DB.
* **Cluster management:** naming `lb-<tipo>-<cluster>-<env>` oppure `lb-<tipo>-shared-<env>`.
* **Logging/Monitoring:** abilitati su backend service.
* **Firewall:** consentire **health‑checker** GCP e porte servizio; 80/443 per FE/AL, `db_port` per DB.

## ⚙️ Parametri minimi consigliati

> Valori base per garantire conformità e funzionamento out‑of‑the‑box. Adattali in base a carico/ambiente.

### FE — External Managed HTTP(S) LB (pubblico)

* **Scope**: globale (Application LB esterno)
* **IP**: statico **pubblico**
* **Certificati**: Managed SSL (auto‑renew), TLS ≥ 1.2
* **Backend service**:

  * `load_balancing_scheme = EXTERNAL_MANAGED`
  * `protocol = HTTP` *(o `HTTPS`/`HTTP2`)*
  * `port_name = "https"` *(consigliato in prod)*
  * `session_affinity = NONE`
  * **Logging**: abilitato, `sample_rate = 1.0`
* **Backend (MIG/IG)**:

  * `balancing_mode = UTILIZATION`
  * `max_utilization = 0.6`
  * `capacity_scaler = 1.0`
  * **Named port MIG**: `https:443` *(oppure `http:80` se necessario)*
* **Health check (HTTP/S)**:

  * Path: `/healthz`
  * `timeout ≤ 5s`, `check_interval ≤ 5s`
  * `healthy_threshold = 2`, `unhealthy_threshold = 3`
  * `port_specification = USE_SERVING_PORT`
* **Firewall**:

  * Allow `80,443` dal LB e dai range health‑checker GCP `130.211.0.0/22`, `35.191.0.0/16`

### AL — Internal Managed HTTP(S) LB (privato)

* **Scope**: **regionale** (tutti i componenti nella **stessa region**)
* **IP**: statico **privato** (subnet interna)
* **Proxy‑only subnet**: dedicata (purpose **REGIONAL_MANAGED_PROXY**)
* **Certificati**: Certificate Manager (managed/self‑managed), DNS privato
* **Backend service (regionale)**:

  * `load_balancing_scheme = INTERNAL_MANAGED`
  * `protocol = HTTP` *(o `HTTPS`/`HTTP2`/`H2C`)*
  * `port_name = "https"`
  * `session_affinity = NONE`
  * **Logging**: abilitato, `sample_rate = 1.0`
* **Backend (MIG/IG)**:

  * `balancing_mode = UTILIZATION`
  * `max_utilization = 0.6`
  * `capacity_scaler = 1.0`
  * **Named port MIG**: `https:443`
* **Health check (HTTP/S)**:

  * Path: `/healthz`
  * `timeout ≤ 5s`, `check_interval ≤ 5s`
  * `healthy_threshold = 2`, `unhealthy_threshold = 3`
  * `port_specification = USE_SERVING_PORT`
* **Firewall**:

  * Allow `80,443` dal VPC e dai range health‑checker GCP `130.211.0.0/22`, `35.191.0.0/16`

### DB — Internal TCP LB (privato, L4 passthrough)

* **Scope**: **regionale**
* **IP**: statico **privato** (subnet DB)
* **Backend service (regionale)**:

  * `load_balancing_scheme = INTERNAL`
  * `protocol = TCP`
  * `session_affinity = CLIENT_IP` *(facoltativa)*
  * **Logging**: abilitato, `sample_rate = 1.0`
* **Backend (MIG/IG/NEG)**:

  * `balancing_mode = CONNECTION`
  * **Capacità** *(obbligatoria)*: impostare **uno** tra:

    * `max_connections_per_instance = 2000` *(valore iniziale consigliato)*, **oppure**
    * `max_connections = 10000` *(se vuoi una soglia totale)*
  * `capacity_scaler = 1.0`
* **Health check (TCP)**:

  * Porta: `db_port` (es. `5432`)
  * `timeout ≤ 5s`, `check_interval ≤ 5s`
  * `healthy_threshold = 2`, `unhealthy_threshold = 3`
* **Firewall**:

  * Allow `db_port` dal VPC e dai range health‑checker GCP `130.211.0.0/22`, `35.191.0.0/16`
