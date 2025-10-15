## 🧱 **Building Block – Requisito Architetturale per VM su GCP**

### 🔹 Descrizione

Ogni volta che viene creata una **VM Compute Engine** su **GCP**, è obbligatoria l’associazione di una **componente PaaS di bilanciamento gestito** per garantire *alta disponibilità*, *scalabilità* e *resilienza* in linea con gli standard architetturali.

Il Load Balancer deve essere gestito secondo la seguente regola:

* se nel campo **Cluster** è valorizzato, viene creato **un Load Balancer dedicato per cluster**;
* se il campo **Cluster** non è valorizzato, viene creato **un unico Load Balancer condiviso** per tutte le VM del medesimo ambiente o progetto.

---

### 🔹 Componente PaaS richiesta

**Servizio:** Google Cloud HTTP(S) Load Balancing
**Tipologia:** *External Managed Load Balancer*

| Caratteristica           | Valore richiesto                                                                 |
| :----------------------- | :------------------------------------------------------------------------------- |
| **Backend**              | Le VM o i Managed Instance Group appartenenti allo stesso cluster                |
| **Health Check**         | HTTP su path `/healthz`, timeout ≤ 5 s, max 3 tentativi                          |
| **SSL**                  | Certificato gestito GCP (auto-renew)                                             |
| **Frontend IP**          | Statico (pubblico o privato, in base all’ambiente)                               |
| **Logging e Monitoring** | Obbligatori – integrati con Cloud Logging e Monitoring                           |
| **Autoscaling policy**   | Abilitata – target CPU ≤ 60%, max 5 istanze                                      |
| **Firewall**             | Devono essere consentite le porte 80 e 443 dal load balancer                     |
| **Cluster management**   | Un Load Balancer per cluster; se `Cluster` non specificato → uno per tutte le VM |
| **Billing model**        | Pay-as-you-go – costo per GB di traffico e per forwarding rule attiva            |

---

### 🔹 Motivazione

* Garantire *High Availability* e *Fault Tolerance* senza configurazioni manuali.
* Centralizzare il traffico e semplificare la gestione di certificati e health check.
* Evitare proliferazione di Load Balancer non necessari, ottimizzando costi e manutenzione.
* Uniformare la configurazione delle architetture su GCP secondo lo standard enterprise.

---

### 🔹 Output atteso

| Risorsa                   | Tipo                    | Note                                             |
| :------------------------ | :---------------------- | :----------------------------------------------- |
| `lb-<cluster>-<ambiente>` | HTTP(S) Load Balancer   | creato per cluster, se specificato               |
| `lb-shared-<ambiente>`    | HTTP(S) Load Balancer   | unico per tutte le VM se nessun cluster indicato |
| `hc-<servizio>`           | Health Check            | per bilanciamento automatico                     |
| `cert-<servizio>`         | Managed SSL Certificate | per HTTPS                                        |
