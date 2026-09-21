# AZ-104 — Fiche 6/6 · Surveiller et sauvegarder des ressources Azure

> Parcours MS Learn : *AZ-104: Monitor and back up Azure resources*
> Modules : **Introduction to Azure Backup** · **Protect your virtual machines by using Azure Backup** · **Monitor your Azure virtual machines with Azure Monitor**
> Skills measured : **Monitor resources in Azure** + **Implement backup and recovery** (dont **Azure Site Recovery**)
> Poids à l'examen : **10–15 %** (le plus petit domaine, mais des questions très "procédurales")

---

# PARTIE A — AZURE MONITOR

## 1. Vue d'ensemble

Azure Monitor = plateforme unique de **collecte → analyse → action**.

```
SOURCES (Azure resources, OS invité, apps, on-prem)
        │
        ▼
  DONNÉES :  Metrics  |  Logs  |  Traces / Changes
        │
        ▼
  ANALYSE :  Metrics Explorer · Log Analytics (KQL) · Insights · Workbooks · Dashboards
        │
        ▼
  ACTION  :  Alerts → Action groups (mail, SMS, webhook, Logic App, Function, ITSM…) · Autoscale
```

## 2. Metrics vs Logs (⭐)

| | **Metrics** | **Logs** |
|---|---|---|
| Nature | **Valeurs numériques** (séries temporelles) | **Événements** / enregistrements structurés |
| Stockage | Base de **métriques** Azure Monitor | **Log Analytics workspace** (tables) |
| Latence | **Quasi temps réel** | Quelques minutes |
| Rétention | **93 jours** (gratuit, plateforme) | **Configurable** (30 j par défaut, jusqu'à 2 ans interactif + archive jusqu'à 12 ans) |
| Langage | **Metrics Explorer** (agrégations, filtres, splitting) | **KQL** (Kusto Query Language) |
| Exemple | `Percentage CPU` d'une VM | *Tous les échecs de connexion de la dernière heure* |
| Idéal pour | **Alertes rapides**, autoscale, tendances | Investigation, corrélation, audit |

### Les types de données à connaître

| Source | Contenu | Collecte | Où consulter |
|---|---|---|---|
| **Platform metrics** | Métriques **hôte** (CPU %, disk ops, network in/out) | **Automatique**, gratuit | Metrics Explorer |
| **Activity log** | Opérations **control plane** de la **subscription** (*qui a fait quoi, quand* : create, delete, RBAC, policy, **Service Health**) | **Automatique** | Portail (**90 jours**) ; exporter vers workspace pour + |
| **Resource logs** (diagnostic logs) | Opérations **internes** d'une ressource (ex. lecture blob, règle NSG matchée) | ⚠️ **Il faut créer un *diagnostic setting*** | Log Analytics / Storage / Event Hub |
| **Guest OS metrics / logs** (mémoire, disque logique, Event log, Syslog) | Dans la VM | **Azure Monitor Agent (AMA)** + **Data Collection Rule (DCR)** | Log Analytics / Metrics |
| **Entra ID logs** | Sign-in, audit | Diagnostic settings | Log Analytics |

> ⚠️ Les **métriques hôte** de VM **n'incluent pas la mémoire disponible** → il faut l'**AMA** (via **VM insights**).

### Diagnostic settings — destinations

| Destination | Usage |
|---|---|
| **Log Analytics workspace** | Requêtes KQL, alertes, workbooks |
| **Storage account** | **Archivage** bon marché long terme |
| **Event Hub** | Streaming vers **SIEM / outil tiers** |
| **Partner solutions** | Datadog, Elastic, etc. |

### Agents (à jour)
- **Azure Monitor Agent (AMA)** ✅ = agent actuel, piloté par **Data Collection Rules** (quoi collecter, où envoyer).
- **Log Analytics agent (MMA/OMS)** et **Diagnostics extensions (WAD/LAD)** = **legacy / retirés**.
- **Boot diagnostics** : capture d'écran + **serial console/log** pour dépanner un boot (stockage managé disponible).

## 3. Log Analytics & KQL

- **Workspace** = base de données de logs ; **retention** configurable ; accès via **RBAC** (*Log Analytics Reader/Contributor*, *Monitoring Reader/Contributor*).
- Modes d'accès : **workspace-context** (tout le workspace) vs **resource-context** (uniquement les logs des ressources auxquelles on a accès).

### Cheat-sheet KQL

| Opérateur | Rôle | Exemple |
|---|---|---|
| `where` | Filtrer | `where TimeGenerated > ago(1h)` |
| `project` | Choisir des colonnes | `project Computer, TimeGenerated` |
| `extend` | Ajouter une colonne calculée | `extend GB = Size / 1024` |
| `summarize` | **Agréger** | `summarize count() by Computer` |
| `bin()` | Découper le temps | `summarize avg(CounterValue) by bin(TimeGenerated, 5m)` |
| `top` / `take` / `limit` | N premières lignes | `top 10 by TimeGenerated desc` |
| `order by` / `sort by` | Trier | `order by TimeGenerated desc` |
| `distinct` | Valeurs uniques | `distinct Computer` |
| `render` | Graphique | `render timechart` |
| `join` / `union` | Combiner des tables | `join kind=inner (...) on Computer` |

```kusto
// 1. VMs qui n'ont plus envoyé de heartbeat depuis 5 minutes
Heartbeat
| summarize LastBeat = max(TimeGenerated) by Computer
| where LastBeat < ago(5m)

// 2. CPU moyen par 5 minutes
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time" and TimeGenerated > ago(4h)
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart

// 3. Qui a supprimé une VM ?
AzureActivity
| where OperationNameValue =~ "MICROSOFT.COMPUTE/VIRTUALMACHINES/DELETE"
| project TimeGenerated, Caller, ResourceGroup, ActivityStatusValue
```
- Tables courantes : `Heartbeat`, `Perf`, `InsightsMetrics` (VM insights), `AzureActivity`, `AzureMetrics`, `AzureDiagnostics`, `Syslog`, `Event`, `StorageBlobLogs`.

## 4. Insights (surveillance "clé en main")

| Insight | Surveille | Prérequis / notes |
|---|---|---|
| **VM insights** | Performance (CPU, mémoire, disque, réseau) + **Map** (dépendances) | **AMA** + **DCR** ; agent de dépendances pour la Map |
| **Storage insights** | Latence, disponibilité, capacité, erreurs par storage account | Diagnostic/metrics |
| **Network insights** | Santé, topologie, connectivité de ressources réseau | Intégré à **Network Watcher** |
| **Container insights** | AKS / conteneurs | — |
| **Application Insights** | Apps (requêtes, exceptions, dépendances) | SDK / auto-instrumentation |

## 5. Alertes (⭐)

**Une règle d'alerte = Scope + Condition + Action group**

| Type d'alerte | Basée sur | Notes |
|---|---|---|
| **Metric alert** | Métriques | Quasi temps réel ; **seuil statique** ou **dynamique** (apprentissage) ; peut cibler **plusieurs ressources** |
| **Log search alert** | Requête **KQL** | Fréquence 1 min → 1 jour ; plus flexible, un peu plus coûteuse |
| **Activity log alert** | Événements **Activity log** | Ex. *Delete VM*, **Service Health**, **Resource Health** |
| **Smart detection** | Application Insights | Anomalies auto |

- **Severity** : Sev 0 (critique) → Sev 4 (verbose).
- **Alert state** (New / Acknowledged / Closed) ≠ **monitor condition** (Fired / Resolved).
- Réglages metric : **aggregation** (Avg/Max…), **window size** (période évaluée), **evaluation frequency**.

### Action groups
- **Réutilisables** entre alertes ; combinent **notifications** + **actions**.
- **Notifications** : e-mail, SMS, push (app Azure), voix, e-mail vers rôle RBAC.
- **Actions** : **Azure Function**, **Logic App**, **Webhook** (+ Secure webhook), **Automation runbook**, **Event Hub**, **ITSM**.

### Alert processing rules (⭐ ex-"action rules")
- Modifient **ce qui se passe quand une alerte se déclenche**, **sans modifier la règle**.
- 2 usages : **Suppress notifications** (ex. **fenêtre de maintenance** samedi 22h–06h) · **Add action groups** à **grande échelle** (par scope/filtres).
- N'empêchent **pas** l'alerte de se **déclencher** ; elles agissent sur les **notifications / actions**.

> 📟 Exemple : *"Si CPU > 80 % pendant 5 min sur `vm-web-*`, envoyer un e-mail à l'équipe Ops et appeler un webhook Teams."* → **Metric alert** (scope = RG, condition = CPU > 80 %, agrégation Avg sur 5 min) + **Action group** (e-mail + webhook). Pour éviter le bruit pendant la maintenance → **alert processing rule** *suppress*.

## 6. Network Watcher & Connection Monitor

| | **Connection troubleshoot** | **Connection Monitor** |
|---|---|---|
| Mode | **Ponctuel** (test instantané) | **Continu** (surveillance) |
| Mesure | Accessible ? latence, hops, blocage | **Reachability, latence (RTT), % checks échoués** dans le temps |
| Protocoles | TCP, ICMP, HTTP | TCP, ICMP, HTTP |
| Sources | Une VM | VMs, VMSS, machines Arc / on-prem (avec agent) |
| Alertes | — | ✅ (métriques intégrées) |

- Prérequis : **Network Watcher activé** dans la région ; **agent / extension Network Watcher** sur les sources.

---

# PARTIE B — SAUVEGARDE & REPRISE APRÈS SINISTRE

## 7. Backup vs Site Recovery (⭐ ne pas confondre)

| | **Azure Backup** | **Azure Site Recovery (ASR)** |
|---|---|---|
| Objectif | **Protection des données** (restaurer après suppression, corruption, ransomware) | **Continuité d'activité / DR** (redémarrer les workloads ailleurs) |
| Mécanisme | **Points de restauration** (snapshots + copie dans le vault) | **Réplication continue** vers une autre région/zone |
| RPO | Heures (selon la policy) | **Minutes** (crash-consistent toutes les ~5 min) |
| RTO | Restore (minutes → heures) | Failover (minutes) |
| Qu'est-ce qui redémarre ? | Rien : tu **restaures** | **Les VMs répliquées** dans la région cible |
| Exemple | "Un stagiaire a supprimé le serveur de fichiers" | "La région **France Central** est indisponible" |

> ➕ Les deux se complètent : Backup protège **les données**, ASR protège **la disponibilité**.

## 8. Vaults : Recovery Services vault vs Backup vault (⭐)

| | **Recovery Services vault** | **Backup vault** |
|---|---|---|
| Sert à | **Azure Backup** (VMs, etc.) **+ Site Recovery** | **Azure Backup** uniquement (workloads plus récents) |
| Workloads | **Azure VMs**, SQL/SAP HANA **dans VM**, **Azure Files**, **MARS agent** (fichiers/dossiers/system state on-prem), **DPM / MABS** | **Azure Disks**, **Azure Blobs**, PostgreSQL, Kubernetes (AKS)… |
| Site Recovery | ✅ | ❌ |

**Règles du vault**
- **Même région** que la **source** (pour Backup) : un vault en France Central protège des ressources de France Central.
- **Redondance du vault** : **LRS / ZRS / GRS** (défaut **GRS**) — modifiable **seulement avant** de protéger un 1er élément.
- **GRS + Cross Region Restore (CRR)** → **restaurer dans la région pair**.
- **Sécurité** : **soft delete** (données supprimées conservées ~**14 jours+**, *enhanced soft delete* jusqu'à 180 j), **immutable vault**, **Multi-user authorization** (Resource Guard), chiffrement, RBAC (*Backup Contributor / Operator / Reader*).
- **Suppression d'un vault** : impossible s'il reste des **éléments protégés** → **arrêter la protection + supprimer les données** (+ purger le soft-deleted) d'abord.

## 9. Azure Backup pour les VMs

### Fonctionnement
1. La **policy** déclenche la sauvegarde → l'**extension de backup** (`VMSnapshot`) prend un **snapshot** du VM.
2. Le snapshot est conservé **localement** (**Instant Restore**, **1–5 jours**, défaut 2) → restauration **rapide**.
3. Les données sont **transférées** vers le **vault** (rétention long terme).
- Aucun agent à installer à la main : **l'extension est installée** automatiquement (Windows / Linux).
- **Cohérence** : **application-consistent** (Windows via **VSS**), **file-system consistent** (Linux, sauf pré/post-scripts), **crash-consistent** (VM arrêtée).

### Backup policy

| Élément | Détail |
|---|---|
| **Schedule** | **Daily** ou **Weekly** (policy *Standard*) · **Hourly** (policy *Enhanced*) |
| **Retention** | **Daily / Weekly / Monthly / Yearly** (jusqu'à **9999 jours** ) |
| **Instant restore** | Rétention des snapshots locaux (1–5 j Standard ; jusqu'à 30 j Enhanced) |
| **Tiers** | **Snapshot → Vault-standard → Vault-archive** (archivage long terme moins cher) |

> 🗂️ Exemple GFS (Grand-père/Père/Fils) : **7** quotidiens + **4** hebdomadaires + **12** mensuels + **5** annuels.

### Restaurer (⭐ options)

| Option | Ce que ça fait | Quand |
|---|---|---|
| **Create new VM** | Crée une **nouvelle VM** depuis le point de restauration | Récupération **rapide**, test |
| **Restore disks** | Restaure les **disques** (+ template) ; **tu crées la VM** | **Personnaliser** la VM (taille, réseau, config) |
| **Replace existing** | **Remplace les disques** de la VM existante | Revenir en arrière **en place** |
| **File recovery** | Monte le point de restauration (script) pour **copier des fichiers** | Récupérer **un fichier** sans restaurer toute la VM |
| **Cross Region Restore** | Restaure dans la **région secondaire (pair)** | Nécessite vault **GRS + CRR** |

### Protéger autre chose
| Workload | Comment |
|---|---|
| **Azure Files** | Snapshots managés dans le Recovery Services vault |
| **On-prem fichiers/dossiers/system state** | **MARS agent** (Windows) — **vault credentials** + **passphrase de chiffrement** à **conserver** |
| **On-prem VMs / serveurs / SQL** | **DPM** ou **Azure Backup Server (MABS)** |
| **SQL Server / SAP HANA dans une VM Azure** | Backup **au niveau workload** |

### Rapports & alertes (skill "configure and interpret reports and alerts")
- **Alertes intégrées** (Azure Monitor) : échecs de **backup/restore**, suppression de données de sauvegarde… (**actives par défaut** pour les critiques) → **action group** pour notifier.
- **Backup reports** : nécessitent un **diagnostic setting** vers un **Log Analytics workspace** ; **workbooks** (taux de succès, utilisation du stockage, jobs).
- **Business Continuity Center** (successeur de *Backup Center*) : vue **centralisée** de la protection (backup + DR) sur plusieurs vaults/subscriptions.

### Commandes
```bash
az backup vault create -g rg -n rsv-contoso -l francecentral
az backup vault backup-properties set -g rg -n rsv-contoso --backup-storage-redundancy GeoRedundant --cross-region-restore-flag true
az backup protection enable-for-vm -g rg -v rsv-contoso --vm vm-web-01 --policy-name DefaultPolicy
az backup protection backup-now -g rg -v rsv-contoso -c vm-web-01 -i vm-web-01 --backup-management-type AzureIaasVM --retain-until 31-12-2026
az backup restore restore-disks -g rg -v rsv-contoso -c vm-web-01 -i vm-web-01 -r <recoveryPointName> --storage-account contosorestore
```
```powershell
New-AzRecoveryServicesVault -ResourceGroupName rg -Name rsv-contoso -Location francecentral
```

---

## 10. Azure Site Recovery (ASR) — DR des VMs Azure

### Composants

| Composant | Rôle |
|---|---|
| **Recovery Services vault** | Contient la config ASR (dans une **région ≠ source**, idéalement la **région cible**) |
| **Replication policy** | Rétention des points de récupération (**24 h** par défaut) + fréquence des **app-consistent snapshots** (**4 h** par défaut) |
| **Mobility service** | Extension installée **automatiquement** sur la VM |
| **Cache storage account** | Compte **dans la région source** : tampon des écritures avant envoi |
| **Target** | **RG, VNet, subnet, région (ou zone)** de destination (**network mapping**) |
| **Recovery plan** | Regroupe des VMs dans un **ordre** de démarrage (groupes), avec **scripts/runbooks** et **actions manuelles** |

> ⚠️ En pratique on utilise **2 vaults** différents : un **pour Backup** (même région que les VMs) et un **pour ASR** (région cible).

### Scénarios supportés
- **Azure → Azure** (région → région, ou **zone → zone**) · **VMware / Hyper-V / physique → Azure**.

### Types de failover (⭐ à connaître dans l'ordre)

| Étape | But | Impact production |
|---|---|---|
| **Test failover** | **Valider** le DR dans un **VNet isolé** | ❌ **Aucun** (à nettoyer ensuite : *Cleanup test failover*) |
| **Failover** (planifié / non planifié) | Basculer **réellement** vers la région cible | Bascule de prod |
| **Commit** | Valider le failover (supprime les autres points de récupération) | — |
| **Re-protect** | **Inverser la réplication** (cible → source) | — |
| **Failback** | Revenir vers la région d'origine (failover inverse + commit) | Retour à la normale |

- **Recovery points** : *crash-consistent* (~toutes les **5 min**), *app-consistent* (selon la policy). Au failover : **Latest processed**, **Latest**, **Latest app-consistent** ou **custom**.
- **Planned failover** = **sans perte de données** (disponible surtout pour VMware/Hyper-V) ; **Unplanned** = en cas de **sinistre**, **perte minimale** possible (RPO).
- **Après failover** : les VMs sont **créées** dans la région cible (**IP publique non conservée** → prévoir **Traffic Manager / LB / DNS**) ; les **IP privées** peuvent être **conservées ou définies** dans le network mapping.
- **Bonnes pratiques** : recovery plans, **test failover régulier** (sans impact prod), **NSG/UDR** équivalents côté cible, **quotas** dans la région cible.

### Pas à pas (configurer ASR Azure → Azure)
1. Créer un **Recovery Services vault** (région cible).
2. VM → **Disaster recovery** (ou vault → **Replicated items → Enable replication**).
3. Choisir **source** (région/RG/VMs) puis **cible** (région, RG, **VNet**, subnet, **cache storage**).
4. Choisir/créer la **replication policy**.
5. **Activer** ; attendre l'état **Protected**.
6. Lancer un **Test failover** → vérifier → **Cleanup**.
7. En cas de sinistre : **Failover** → **Commit** → **Re-protect** → (plus tard) **Failback**.

---

## 11. Pièges d'examen ⚠️

- "Alerte **quasi temps réel** sur un seuil de CPU" → **Metric alert**.
- "Alerter sur un **événement complexe** dans des logs" → **Log search alert (KQL)**.
- "Être prévenu quand **quelqu'un supprime** une VM" → **Activity log alert**.
- "Ne pas notifier pendant la **maintenance**" → **Alert processing rule (suppress)**.
- "Réutiliser la même liste de destinataires dans 20 alertes" → **Action group**.
- "Voir la **mémoire** d'une VM dans Azure Monitor" → **AMA + DCR / VM insights**.
- "Les **resource logs** ne sont pas collectés" → créer un **diagnostic setting**.
- "Conserver des logs **longtemps et pas cher**" → **Storage account** (ou archive/long retention).
- "Envoyer les logs vers un **SIEM tiers**" → **Event Hub**.
- "Test de connectivité **continu** entre 2 VMs" → **Connection Monitor** (≠ Connection troubleshoot).
- "Sauvegarder une VM Azure **sans installer d'agent**" → **Azure Backup (extension auto)**.
- "Vault et VMs dans des **régions différentes**" → ❌ **même région** pour Backup.
- "Restaurer **un seul fichier**" → **File recovery** ; "personnaliser la VM restaurée" → **Restore disks**.
- "Restaurer dans **l'autre région**" → **Cross Region Restore** (**GRS**).
- "Sauvegarder des dossiers d'un serveur **on-prem**" → **MARS agent** (garder la **passphrase**).
- "Impossible de **supprimer le vault**" → arrêter protection + supprimer les données de backup.
- "Basculer les VMs vers une autre région en cas de sinistre" → **Site Recovery** (**pas** Backup).
- "Tester le DR **sans impacter la prod**" → **Test failover**.
- "Après un failover, protéger de nouveau vers l'origine" → **Re-protect** puis **Failback**.

## 12. Checklist finale ✅

- [ ] Je distingue Metrics / Logs / Activity log / Resource logs et sais quand créer un diagnostic setting
- [ ] Je sais écrire 3 requêtes KQL simples (`where`, `summarize`, `render`)
- [ ] Je sais créer une alerte (scope, condition, action group) et une alert processing rule
- [ ] Je connais VM insights / Storage insights / Network insights + Connection Monitor
- [ ] Je distingue Recovery Services vault et Backup vault ; vault = même région que la source
- [ ] Je sais créer une policy, lancer un backup, choisir une option de restore (5 options)
- [ ] Je sais configurer ASR, faire un test failover, failover → commit → re-protect → failback
- [ ] Je ne confonds **jamais** Backup (données) et Site Recovery (disponibilité)
