---

## 🧱 **Building Block – Requisito Architetturale per VM su GCP**

### 🔹 Descrizione

Ogni volta che viene creata una **VM Compute Engine** su **GCP**, è obbligatoria l’associazione di una **componente PaaS di bilanciamento gestito** per garantire *alta disponibilità*, *scalabilità* e *resilienza* in linea con gli standard architetturali.

---

### 🔹 Componente PaaS richiesta

**Servizio:** Google Cloud HTTP(S) Load Balancing
**Tipologia:** *External Managed Load Balancer*

| Caratteristica           | Valore richiesto                                                      |
| :----------------------- | :-------------------------------------------------------------------- |
| **Backend**              | La VM o il Managed Instance Group contenente la VM                    |
| **Health Check**         | HTTP su path `/healthz`, timeout ≤ 5 s, max 3 tentativi               |
| **SSL**                  | Certificato gestito GCP (auto-renew)                                  |
| **Frontend IP**          | Statico (pubblico o privato, in base all’ambiente)                    |
| **Logging e Monitoring** | Obbligatori – integrati con Cloud Logging e Monitoring                |
| **Autoscaling policy**   | Abilitata – target CPU ≤ 60%, max 5 istanze                           |
| **Firewall**             | Devono essere consentite le porte 80 e 443 dal load balancer          |
| **Billing model**        | Pay-as-you-go – costo per GB di traffico e per forwarding rule attiva |

---

### 🔹 Motivazione

* Garantire *High Availability* e *Fault Tolerance* senza configurazioni manuali.
* Centralizzare il traffico e semplificare la gestione di certificati e health check.
* Uniformare la configurazione delle architetture su GCP secondo lo standard enterprise.

---

### 🔹 Output atteso

| Risorsa | Tipo | Note |
|:--|:--|
| `lb-<servizio>-<ambiente>` | HTTP(S) Load Balancer | entry point PaaS per la VM |
| `hc-<servizio>` | Health Check | per bilanciamento automatico |
| `cert-<servizio>` | Managed SSL Certificate | per HTTPS |

---
