# AZ-104 — Fiche 4/6 · Déployer et gérer les ressources de calcul Azure

> Parcours MS Learn : *AZ-104: Deploy and manage Azure compute resources*
> Modules : **Introduction to Azure virtual machines** · **Configure virtual machine availability** · **Configure Azure App Service plans** · **Configure Azure App Service** · **Configure Azure Container Instances**
> Skills measured associés : ARM/Bicep · VMs · **Containers (ACR, ACI, Container Apps)** · App Service
> Poids à l'examen : **20–25 %** (avec Identités, le plus gros domaine)

---

## 0. Quel service de calcul choisir ? (vue d'ensemble)

| Service | Modèle | Tu gères | Scale | Cas d'usage | Exemple Contoso |
|---|---|---|---|---|---|
| **Virtual Machines** | IaaS | OS, patchs, runtime, app | Manuel / **VMSS** | Contrôle total, legacy, lift & shift | Serveur ERP historique |
| **VM Scale Sets** | IaaS | Idem (flotte identique) | **Autoscale** | Web front stateless, batch | Front e-commerce en soldes |
| **App Service** | PaaS | Code + config | Scale up / out, autoscale | Web apps, API, sans gérer l'OS | Site vitrine + API |
| **Container Instances (ACI)** | Serverless containers | Image + ressources | ❌ pas d'autoscale (1 groupe = 1 instance) | Tâches ponctuelles, tests, batch | Job de conversion PDF |
| **Container Apps (ACA)** | Serverless (K8s masqué) | Image + règles de scale | **Auto, scale-to-zero** (KEDA) | Microservices, event-driven, APIs | Micro-service de notifications |
| **AKS** | Kubernetes managé | Workloads K8s | HPA / cluster autoscaler | Orchestration complexe | Plateforme de dizaines de services |

> 🧠 Règle de pouce : **contrôle total** → VM · **juste du code web** → App Service · **conteneur simple/court** → ACI · **conteneurs en prod avec autoscale/scale-to-zero** → ACA · **orchestration avancée** → AKS.

---

## 1. Automatiser avec ARM templates / Bicep (rappels ciblés compute)

- **Interpréter** : identifier `parameters`, `variables`, `resources` (`type` + `apiVersion` + `name` + `location`), `dependsOn`, `outputs`.
- **Modifier un template existant** : ajouter un paramètre, changer `sku`/`vmSize`, ajouter une ressource, ajouter un `copy` (boucle).
- **Déployer** : `az deployment group create -g rg -f main.bicep` · `New-AzResourceGroupDeployment`.
- **Modes** : **Incremental** (défaut, ne supprime rien) vs **Complete** (supprime ce qui n'est pas dans le template).
- **Exporter** un déploiement / un RG → JSON (portail : *Automation → Export template*).
- **Convertir** : `az bicep decompile --file template.json` (JSON → Bicep) · `az bicep build --file main.bicep` (Bicep → JSON).
- **What-if** avant déploiement pour prévisualiser les changements.
> Détails complets dans la **fiche 1**.

---

## 2. Virtual Machines

### Checklist de création
Subscription → **RG** → nom → **région** → **availability option** (none / **AZ** / **availability set** / **VMSS**) → **security type** (Standard / **Trusted Launch** / Confidential) → **image** → **taille** → admin (mot de passe / **clé SSH**) → ports entrants → **disques** → **réseau (VNet/subnet/NSG/IP publique)** → management (boot diagnostics, identity, auto-shutdown) → **tags**.

### Familles de tailles

| Famille | Nom | Usage | Exemple |
|---|---|---|---|
| **B** | Burstable | Dev/test, charges faibles avec pics | `Standard_B2s` |
| **D** | General purpose | Web, apps, DB petite/moyenne | `Standard_D4s_v5` |
| **E** | Memory optimized | Bases de données, cache in-memory | `Standard_E8s_v5` |
| **F** | Compute optimized | Batch, gaming, calcul | `Standard_F8s_v2` |
| **L** | Storage optimized | Big data, DB NoSQL (I/O local) | `Standard_L8s_v3` |
| **N** | GPU | Rendu, IA/ML | `Standard_NC6s_v3` |
| **M** | Très grande mémoire | SAP HANA | `Standard_M128s` |

### Redimensionner (resize)
- Portail : VM → **Size**. CLI : `az vm resize -g rg -n vm --size Standard_D4s_v5` · PS : `Update-AzVM`.
- Si la taille est disponible **sur le cluster matériel actuel** → **redémarrage** simple ; sinon → **deallocate** requis puis resize.
- **Impact** : redémarrage = **coupure** · **IP publique dynamique** peut changer (les **statiques** restent) · **disque temporaire** (`D:` / `/dev/sdb`) **peut être perdu**.
- Scale **vertical** (**up/down**) = changer la taille · Scale **horizontal** (**out/in**) = ajouter/retirer des instances.

### Disques

| Disque | Rôle | Persistance |
|---|---|---|
| **OS disk** | Système (`C:` / `/`) | ✅ |
| **Data disk(s)** | Données | ✅ (nb max selon taille de VM) |
| **Temporary disk** | Swap/cache, stockage local de l'hôte | ❌ **Perdu** au redeploy / resize / maintenance |

| Type de managed disk | Média | Usage |
|---|---|---|
| **Standard HDD** | HDD | Sauvegardes, accès rare, dev/test |
| **Standard SSD** | SSD | Serveurs web, charges légères |
| **Premium SSD** | SSD | **Production**, DB (SLA 99,9 % single VM) |
| **Premium SSD v2** | SSD | IOPS/débit/capacité **réglables séparément** |
| **Ultra Disk** | SSD | I/O intensif, **latence sub-ms**, IOPS réglables à chaud |

- Un disque peut être **agrandi** (jamais **réduit**). Ajouter/détacher un data disk : portail, `az vm disk attach`, `Add-AzVMDataDisk`.
- **Snapshot** = copie point-in-time d'un disque · **Image** = modèle de VM (généralisée) pour cloner.
- **Caching** : OS = *ReadWrite* · data = *None / ReadOnly* (ReadOnly pour données de DB lecture-intensive).

### Chiffrement des VMs (⭐ "encryption at host" dans les skills measured)

| Option | Où | Ce qui est chiffré | Clés |
|---|---|---|---|
| **SSE at rest** (défaut) | Service Storage | Disques managés **au repos** | Microsoft-managed (PMK) ou **customer-managed (CMK)** via **Disk Encryption Set** |
| **Azure Disk Encryption (ADE)** | **Dans l'OS** (BitLocker / dm-crypt) | Volumes OS + data | **Key Vault** |
| **Encryption at host** | **Hôte** de la VM | **Disque temporaire**, **caches** OS/data + flux vers Storage → **chiffrement de bout en bout** | PMK ou CMK |
| **Confidential disk encryption** | Confidential VMs | Disque OS lié au TPM/hardware | — |

- **Encryption at host** : feature à **enregistrer** sur la subscription (`Microsoft.Compute/EncryptionAtHost`) ; `az vm create --encryption-at-host true` ; **VM à arrêter (deallocate)** pour l'activer sur une VM existante ; **incompatible avec ADE** sur la même VM.
- Question type : "chiffrer **aussi le disque temporaire et les caches**" → **Encryption at host**.

### Déplacer une VM

| Vers | Méthode | Notes |
|---|---|---|
| Autre **resource group** / **subscription** (même région) | **Move** (portail *Move → Move to another resource group/subscription*, `az resource move`) | Déplacer **ensemble** VM + disques + NIC + IP ; **l'ID de ressource change** ; valider avant |
| Autre **région** | **Azure Resource Mover** ou **Site Recovery** (ou recréer depuis snapshot/image) | La VM d'origine **reste** ; la nouvelle VM est créée dans la cible |
| Autre **zone** | Recréer depuis **snapshot/disque** ou Resource Mover | Pas de changement de zone "en place" |

### Availability (⭐ tableau SLA)

| Option | Protège contre | **SLA** | Notes |
|---|---|---|---|
| **VM unique** + Premium SSD/Ultra | — | **99,9 %** | (Standard SSD 99,5 %, HDD 95 %) |
| **Availability set** (≥ 2 VMs) | Panne rack / maintenance **dans 1 datacenter** | **99,95 %** | Défini **à la création** |
| **Availability Zones** (≥ 2 VMs dans ≥ 2 zones) | Panne **datacenter / zone** | **99,99 %** | La plus haute dispo VM |
| **VMSS** multi-zones | Idem + **scale** | 99,99 % | — |

**Availability set** : répartit les VMs en
- **Fault domains (FD)** = **rack** (alim + réseau communs), **max 3** → panne matérielle.
- **Update domains (UD)** = groupe redémarré **ensemble** lors d'une maintenance, **max 20** (défaut 5).
> Exemple : 6 VMs web dans un availability set (3 FD × 2 UD) → une panne de rack n'en tue que 2 sur 6.

**AZ vs Availability set** → **AZ** protège d'un **datacenter entier**, l'availability set **non**. **Proximity placement group** = rapprocher les VMs (**latence**) — ⚠️ peut **réduire** la dispo.

### VM Scale Sets (VMSS)

| | **Flexible** (recommandé) | **Uniform** |
|---|---|---|
| VMs | Peuvent **différer** (tailles, spot/regular) | **Identiques** (modèle unique) |
| Gestion | Comme des VMs normales | Via le scale set |
| Répartition FD / AZ | ✅ max spreading | ✅ |
| Usage | Nouvelles charges, HA | Charges homogènes stateless |

**Autoscale**
- **Metric-based** : ex. *CPU moyen > 70 % pendant 10 min → +2 instances* ; *CPU < 30 % → −1*.
- **Schedule-based** : ex. *lundi–vendredi 8h–18h → 10 instances, sinon 2*.
- Paramètres : **min / max / default**, **cooldown** (défaut 5 min), **scale-out et scale-in** (toujours prévoir les 2 !).
- **Scale-in policy** : Default (équilibrage zones/FD) · NewestVM · OldestVM.
- **Upgrade policy** (Uniform) : **Manual / Automatic / Rolling**.
- Placer un **Load Balancer** ou **Application Gateway** devant.

---

## 3. Conteneurs

### Azure Container Registry (ACR)

| SKU | Points clés |
|---|---|
| **Basic** | Dev/test, stockage/débit limités |
| **Standard** | Production, plus de stockage/débit |
| **Premium** | **Geo-replication**, **Private Link**, zone redundancy, customer-managed keys, content trust |

- URL : `<name>.azurecr.io` — nom **unique mondialement**.
- **Auth** : **Entra ID + RBAC** (`AcrPull`, `AcrPush`), **managed identity**, **repository-scoped tokens**, **admin user** (⚠️ à éviter en prod).
```bash
az acr create -g rg -n contosoacr --sku Standard
az acr login -n contosoacr
docker tag web:v1 contosoacr.azurecr.io/web:v1 && docker push contosoacr.azurecr.io/web:v1
az acr build -r contosoacr -t web:v1 .        # construit l'image DANS Azure (ACR Tasks), sans Docker local
az acr repository list -n contosoacr
```

### Azure Container Instances (ACI)

- **Container group** = ensemble de conteneurs qui **partagent** cycle de vie, réseau (même IP/ports), volumes → l'équivalent d'un **pod**.
- **Linux** : multi-conteneurs possible · **Windows** : **1 conteneur** par groupe.
- **Restart policy** : **Always** (défaut) · **OnFailure** · **Never** (⭐ pour **tâches batch / one-shot**).
- Réseau : **IP publique + DNS label** (`<label>.<region>.azurecontainer.io`) **ou** déploiement dans un **VNet** (subnet délégué).
- **Volumes** : **Azure Files**, **emptyDir**, secret, gitRepo · **Variables d'env sécurisées** (`--secure-environment-variables`).
- CPU/mémoire **définis par conteneur** ; **pas d'autoscale** natif.
- **Multi-conteneurs** : déploiement par **YAML / ARM / Bicep**.
```bash
az container create -g rg -n web --image mcr.microsoft.com/azuredocs/aci-helloworld --dns-name-label contoso-web --ports 80 --cpu 1 --memory 1.5
az container create -g rg -f group.yaml
az container logs -g rg -n web
az container show -g rg -n web --query ipAddress.fqdn
```

### Azure Container Apps (ACA)

| Concept | Description |
|---|---|
| **Environment** | Frontière sécurisée (VNet, logs) partagée par plusieurs apps |
| **Container app** | Ton conteneur/micro-service |
| **Revision** | Snapshot **immuable** d'une version → **traffic splitting** (blue/green, canary) |
| **Ingress** | Accès HTTP/TCP, **external** ou **internal** |
| **Scale rules** | **HTTP** (requêtes concurrentes), **TCP**, **custom (KEDA)** : file Service Bus, CPU/mémoire… |
| **Replicas** | **min** (peut être **0 = scale-to-zero**) et **max** |
| **Jobs** | Manuels, planifiés (cron) ou événementiels |

- **Sizing** : couples vCPU/mémoire (ex. **0,25 vCPU / 0,5 Gi** jusqu'à **4 vCPU / 8 Gi** par réplica en plan **Consumption**) ; **workload profiles dédiés** pour plus.
- Basé sur Kubernetes **sans l'exposer** ; intégration **Dapr**.

### ACI vs ACA vs AKS

| | **ACI** | **ACA** | **AKS** |
|---|---|---|---|
| Gestion | Minimale | Faible | **Élevée** (cluster) |
| Autoscale | ❌ | ✅ (scale-to-zero) | ✅ (HPA / cluster autoscaler) |
| Multi-conteneurs | Container group | Sidecars / plusieurs conteneurs | Pods |
| Traffic splitting / révisions | ❌ | ✅ | Via ingress/mesh |
| Idéal | Tâche courte, test | Microservices / event-driven | Plateforme complexe |

---

## 4. App Service

### Concept
- **App Service plan** = **les serveurs** (région + OS + taille + tier de prix). **App** = le programme qui tourne dessus.
- **Plusieurs apps peuvent partager 1 plan** (même région, **même OS** — Windows et Linux ne se mélangent pas dans un plan) et **se partagent CPU/RAM**.
- 💡 Analogie : le **plan** = l'immeuble de bureaux loué ; les **apps** = les entreprises qui y travaillent (elles se partagent l'ascenseur et la clim).

### Tiers de prix (⭐)

| Tier | Compute | Scale out | Autoscale | Deployment slots | Backup (custom) | Points clés |
|---|---|---|:-:|:-:|:-:|---|
| **Free / Shared** (F1, D1) | **Partagé** (quotas CPU) | ❌ | ❌ | ❌ | ❌ | Dev/test ; pas de SLA ; Free sans custom domain |
| **Basic** (B1–B3) | Dédié | **Manuel**, jusqu'à ~3 | ❌ | ❌ | ❌ | Custom domain + TLS |
| **Standard** (S1–S3) | Dédié | Jusqu'à ~10 | ✅ | **5** | ✅ | Entrée de gamme **production** |
| **Premium** (v3…) | Dédié (perf +) | Jusqu'à ~30 | ✅ | **20** | ✅ | Perf, **VNet integration**, plus de slots |
| **Isolated** (App Service Environment) | Dédié, **VNet dédié** | Jusqu'à ~100 | ✅ | 20 | ✅ | **Isolation réseau/conformité** |

### Scaling

| | **Scale up / down** (vertical) | **Scale out / in** (horizontal) |
|---|---|---|
| Action | **Changer de tier / taille** du plan | Changer le **nombre d'instances** |
| Effet | + CPU / RAM / fonctionnalités | + capacité / résilience |
| Interruption | Généralement **sans coupure** (l'app migre vers de nouvelles instances) | Non |
| Automatique ? | ❌ | ✅ **Autoscale** (Standard+) |

- Autoscale **au niveau du plan** : règles **métriques** (CPU %, mémoire %, **HTTP queue length**) ou **planning** ; **min / max / default** instances ; **cooldown**.
- *Premium* : **Automatic scaling** (sans règles manuelles) + **prewarmed instances**.
- **Always On** (Basic+) : évite la mise en veille de l'app après inactivité.

### Custom domain & TLS

**Mapper un nom DNS existant**

| Type de nom | Enregistrement DNS |
|---|---|
| **Sous-domaine** (`www.contoso.com`) | **CNAME** → `<app>.azurewebsites.net` **+** **TXT** `asuid.www` (vérification) |
| **Domaine racine / apex** (`contoso.com`) | **A** → IP de l'app **+** **TXT** `asuid` (CNAME impossible à l'apex) |

**Certificats & TLS**
- **App Service Managed Certificate** : **gratuit**, géré/renouvelé, **Basic+**, pas de wildcard, **non exportable**.
- **App Service Certificate** (acheté), **import depuis Key Vault**, ou **upload PFX**.
- **Binding TLS/SSL** : **SNI SSL** (courant, 1 IP partagée) vs **IP-based SSL** (IP dédiée, vieux clients).
- Toggles : **HTTPS Only**, **Minimum TLS version** (1.2 recommandé).
- Custom domain + TLS : **Basic et plus**.

### Networking d'une App Service

| Sens | Fonctionnalité | Rôle |
|---|---|---|
| **Entrant** | **Access restrictions** | Autoriser/refuser IP, service tags, subnets |
| **Entrant** | **Private endpoint** | IP privée dans ton VNet |
| **Sortant** | **VNet integration** (régionale, **subnet délégué** `Microsoft.Web/serverFarms`) | L'app **appelle** des ressources privées (DB, API interne) |
| **Sortant** | **Hybrid connections** | Joindre une ressource on-prem via relais |
| **Isolation totale** | **App Service Environment (ASE)** | Déploiement dans **ton** VNet |

> ⚠️ **VNet integration = sortant** ; **private endpoint = entrant**. Ne pas confondre.

### Deployment slots (⭐)
- **Slot** = **instance live** de l'app avec **son propre hostname** (`app-staging.azurewebsites.net`) dans le **même plan** (Standard+ : **5** ; Premium/Isolated : **20**).
- **Swap** : échange staging ↔ production après **warm-up** → **zéro downtime** ; **rollback** = re-swap.
- **Slot settings ("sticky")** : paramètres qui **restent** sur le slot (ex. connection string de test) au lieu de suivre l'app.
- **Traffic routing** : envoyer **x %** du trafic prod vers un slot (tests en prod).
- **Swap with preview** (multi-phase) · **Auto swap**.
- ⚠️ Les slots **partagent les ressources du plan** → un **test de charge sur staging** peut dégrader la **prod**.

```
Prod slot  ◄── SWAP ──►  Staging slot
(app.azurewebsites.net)   (app-staging.azurewebsites.net)
       ▲ 90 % trafic              ▲ 10 % trafic (test)
```

### Backup App Service
- **Custom backup** (Standard+) vers un **storage account** (container) : contenu de l'app + config + **DB liée** (option), **manuel ou planifié**, **restore** vers l'app, un slot ou une nouvelle app.
- Le **storage account** doit être accessible (pas bloqué par firewall), taille **limitée** (~10 GB app + DB).

### Commandes
```bash
az appservice plan create -g rg -n plan-web --sku S1 --is-linux
az webapp create -g rg -p plan-web -n contoso-web --runtime "NODE:20-lts"
az webapp deployment slot create -g rg -n contoso-web --slot staging
az webapp deployment slot swap   -g rg -n contoso-web --slot staging --target-slot production
az appservice plan update -g rg -n plan-web --sku P1V3            # scale up
az appservice plan update -g rg -n plan-web --number-of-workers 3 # scale out (manuel)
az webapp config hostname add -g rg --webapp-name contoso-web --hostname www.contoso.com
az webapp config ssl bind -g rg -n contoso-web --certificate-thumbprint <thumb> --ssl-type SNI
```

---

## 5. Pièges d'examen ⚠️

- "Modifier le template **sans** supprimer les autres ressources" → mode **Incremental**.
- "Chiffrer le **disque temporaire** et les caches" → **Encryption at host** (≠ ADE).
- "**99,99 %** de SLA pour des VMs" → **≥ 2 VMs dans ≥ 2 Availability Zones**.
- "Protéger contre panne de rack + maintenance" → **Availability set** (FD + UD) ; contre **datacenter** → **AZ**.
- "Ajouter des VMs **automatiquement** selon la charge" → **VMSS + autoscale** (règles **out ET in**).
- "Déplacer une VM vers une **autre région**" → **Resource Mover / Site Recovery** (pas le Move classique).
- "Conteneur ponctuel qui **s'arrête** à la fin" → **ACI** + restart policy **Never/OnFailure**.
- "Scale-to-zero, révisions, traffic splitting" → **Container Apps**.
- "Construire une image **sans Docker local**" → **`az acr build`** (ACR Tasks).
- "**Geo-replication** ou Private Link pour le registry" → **ACR Premium**.
- "Déploiements **sans downtime** / **rollback**" → **deployment slots** (Standard+).
- "Autoscale App Service" → **Standard+** ; "Slots" → **Standard+** ; "Backup custom" → **Standard+**.
- "Apex domain vers App Service" → **A record + TXT** (pas de CNAME).
- "Une app doit joindre une DB privée" → **VNet integration** (sortant).
- "Mettre Windows et Linux dans le même plan" → **impossible**.

## 6. Checklist finale ✅

- [ ] Je choisis entre VM / VMSS / App Service / ACI / ACA / AKS en 10 secondes
- [ ] Je connais les SLA VM (99,9 / 99,95 / 99,99 %) et FD vs UD
- [ ] Je sais dérouler resize, disques, snapshots, chiffrement (SSE / ADE / at host)
- [ ] Je configure un autoscale (metric + schedule, out + in, cooldown)
- [ ] Je connais ACR (SKU, `az acr build`), ACI (container group, restart policy), ACA (revisions, KEDA, scale-to-zero)
- [ ] Je sais expliquer plan vs app, tiers, scale up vs out, slots, domain custom, TLS, backup, networking
