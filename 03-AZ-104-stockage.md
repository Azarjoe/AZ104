# AZ-104 — Fiche 3/6 · Implémenter et gérer le stockage dans Azure

> Parcours MS Learn : *AZ-104: Implement and manage storage in Azure*
> Modules : **Configure storage accounts** · **Configure Azure Blob Storage** · **Configure Azure Storage security** · **Configure Azure Files** (et Azure File Sync)
> Poids à l'examen : **15–20 %**

---

## 1. Storage account — les fondamentaux

- Conteneur qui regroupe les services : **Blob**, **Files**, **Queue**, **Table** (+ Data Lake Gen2 avec *hierarchical namespace*).
- **Nom** : **3–24 caractères**, **minuscules + chiffres** uniquement, **unique mondialement** (fait partie de l'URL).
- Endpoints : `https://<account>.blob.core.windows.net` · `.file.` · `.queue.` · `.table.` · `.dfs.`
- **Redundancy, encryption, network rules, access keys = au niveau du compte** (partagés par tous les services du compte).

### Types de comptes

| Type | Services | Performance | Redondances |
|---|---|---|---|
| **Standard general-purpose v2** (recommandé) | Blob, Files, Queue, Table | HDD | LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS |
| **Premium block blobs** | Block/append blobs | SSD, faible latence | **LRS, ZRS** |
| **Premium file shares** | Azure Files (SMB **et NFS**) | SSD | **LRS, ZRS** |
| **Premium page blobs** | Page blobs (disques non managés) | SSD | **LRS, ZRS** |

### Choisir un service de stockage

| Besoin | Service |
|---|---|
| Fichiers non structurés (images, backups, logs, vidéos) | **Blob Storage** |
| Partage de fichiers SMB/NFS "lift & shift" | **Azure Files** |
| Messages asynchrones entre composants | **Queue Storage** |
| NoSQL clé-valeur simple | **Table Storage** |
| Disque de VM | **Managed disks** |

### Custom domain (nom perso pour le endpoint)
- **CNAME direct** (`blob.contoso.com` → `contosostore.blob.core.windows.net`) : simple mais avec **court downtime** à la migration.
- **CNAME intermédiaire `asverify.`** : **zéro downtime** (validation avant bascule).

---

## 2. Redundancy — l'échelle de protection (⭐ incontournable)

### L'échelle "rayon de la panne"

```
Disque / rack / serveur  <  Datacenter  <  Availability Zone  <  Region  <  Region + Zone
        LRS protège ────────┘                                          
                              ZRS protège ──────────┘
                                          GRS protège ──────────────┘
                                                         GZRS protège ─────────────┘
```
> Analogie "coffre de banque" : **LRS** = 3 exemplaires du contrat dans **le même immeuble** · **ZRS** = 3 exemplaires dans **3 quartiers** de la ville · **GRS** = 3 dans l'immeuble + 3 dans **une autre ville** · **GZRS** = 3 quartiers + 3 dans **une autre ville**.

### Tableau comparatif

| | **LRS** | **ZRS** | **GRS** | **GZRS** |
|---|---|---|---|---|
| Copies | **3** | **3** | **6** (3 + 3) | **6** (3 + 3) |
| Primary region | 1 datacenter (LRS) | **3 AZ** (sync) | **LRS** (1 datacenter) | **ZRS** (3 AZ) |
| Secondary region | ❌ | ❌ | **LRS** (async) | **LRS** (async) |
| Durabilité (≥) | **11 nines** | **12 nines** | **16 nines** | **16 nines** |
| Survit à | Rack / disque / serveur | + panne d'**AZ** | + panne de **région** | + AZ **et** région |
| Panne datacenter (feu) | ❌ perdu | ✅ | ❌ (primaire perdue → **failover**) | ✅ |
| Lecture secondaire | ❌ | ❌ | **RA-GRS** | **RA-GZRS** |
| Coût | 💲 | 💲💲 | 💲💲 | 💲💲💲 |
| Exemple | Données recréables (thumbnails) | App critique dans **1 région** avec **résidence des données** | Sauvegardes, données à protéger d'un sinistre régional | Données **critiques** (finance, santé) |

**Points à retenir**
- **Réplication synchrone** dans la région primaire ; **asynchrone** vers la secondaire (RPO typique **< 15 min**, **sans SLA**).
- **Région secondaire = région pair, non modifiable**.
- Sans **RA-** : la secondaire **n'est lisible qu'après failover**.
- **RA-GRS / RA-GZRS** : endpoint lecture `https://<account>-secondary.blob.core.windows.net`.
- **GRS = LRS dans la primaire** (⚠️ **pas** de protection contre une panne de zone ; c'est GZRS qui l'apporte).
- **Premium** : **LRS ou ZRS** seulement (pas de géo-réplication).
- **Customer-managed failover** (GRS/RA-GRS/GZRS/RA-GZRS) : possible, **perte de données possible** (voir *Last Sync Time*). Après failover : le compte devient **LRS** (GRS) ou **ZRS** (GZRS) dans la nouvelle primaire → **reconfigurer** la géo-redondance.
- Changer de redondance : depuis le portail pour certaines conversions (ex. LRS ↔ GRS) ; pour d'autres (vers/depuis **ZRS**) → **conversion / live migration** (demande ou opération dédiée).

---

## 3. Sécuriser l'accès au stockage

### Vue d'ensemble : 4 méthodes d'autorisation

| Méthode | Principe | Niveau de sécurité | Quand |
|---|---|---|---|
| **Microsoft Entra ID + RBAC** | Identité + rôle (ex. *Storage Blob Data Contributor*) | ⭐⭐⭐ **Recommandé** | Apps, users, managed identities |
| **SAS** | URL signée, **limitée** (droits, durée, IP) | ⭐⭐ | Accès **délégué temporaire** à un tiers |
| **Access keys** | 2 clés = **accès total** au compte | ⭐ | À éviter / **rotation** ; peut être désactivé |
| **Anonymous read** | Public (niveau container) | ❌ | Sites statiques ; désactivé par défaut |

### Access keys
- **2 clés** (key1/key2) → **rotation sans interruption** (basculer les apps sur key2, régénérer key1).
- **Régénérer une clé invalide** tous les SAS **signés avec elle** (account SAS et service SAS **sans** stored access policy).
- Stocker dans **Azure Key Vault** ; option **"Allow storage account key access" = Disabled** → force Entra ID.

### SAS (Shared Access Signature)

| Type | Signé par | Portée | À retenir |
|---|---|---|---|
| **User delegation SAS** | **Credentials Entra ID** | **Blob / ADLS** uniquement | ⭐ **Le plus sûr** |
| **Service SAS** | Account key | **1 service** (blob, file, queue, table) | Peut utiliser une **stored access policy** |
| **Account SAS** | Account key | **Un ou plusieurs services** du compte | Le plus large |

- Paramètres d'un SAS : **permissions** (`sp`), **début/fin** (`st`/`se`), **IP** autorisées (`sip`), **protocole** (`spr` = HTTPS only), ressource (`sr`), signature (`sig`).
- **Stored access policy** (server-side) :
  - Uniquement pour **service SAS** (containers, file shares, queues, tables).
  - Définit start / expiry / permissions **côté serveur** → **révoquer** = **modifier ou supprimer** la policy (sans régénérer la clé).
  - **Max 5** policies par container/share/queue/table.
- ⚠️ Un **SAS ad-hoc** (sans policy) ne se révoque qu'en **régénérant la clé** (ou en expirant).

### Storage firewalls & virtual networks

| Option réseau | Comportement |
|---|---|
| **Enabled from all networks** | Accessible depuis Internet (défaut) |
| **Enabled from selected virtual networks and IP addresses** | **Deny par défaut** + règles d'autorisation |
| **Disabled** | Uniquement via **private endpoints** |

- Règles : **subnets** (via **service endpoint** `Microsoft.Storage`) · **plages IP publiques** · **resource instances** · exception **Trusted Azure services**.
- ⚠️ Règles IP : **IP publiques uniquement** (pas de plages privées) ; `/31` et `/32` non supportés → utiliser des IP individuelles.
- ⚠️ **Les règles IP ne s'appliquent pas** aux requêtes venant de la **même région** Azure (trafic via IP privée) → utiliser une **règle VNet**.
- S'applique à **tous les protocoles** (REST + SMB).
- **Service endpoint** vs **Private endpoint** → tableau détaillé dans la **fiche 5**.

### Identity-based access pour Azure Files (SMB)

| Source d'identité | Cas d'usage |
|---|---|
| **On-premises AD DS** | Serveurs joints au domaine, synchro avec Entra Connect |
| **Microsoft Entra Domain Services** | Identités cloud + domaine managé |
| **Microsoft Entra Kerberos** | Identités **hybrides** sans ligne de vue vers un DC |

- **2 niveaux d'autorisation** :
  1. **Share-level** → **rôles RBAC** : *Storage File Data SMB Share Reader / Contributor / Elevated Contributor*.
  2. **Directory/file-level** → **ACL NTFS** (Windows).
- Mount : `net use Z: \\<account>.file.core.windows.net\<share>` (**port TCP 445**).

### Chiffrement

| Niveau | Détail |
|---|---|
| **SSE** (Storage Service Encryption) | **Toujours actif**, **AES-256**, non désactivable |
| Clés | **Microsoft-managed** (défaut) ou **customer-managed** (Key Vault / HSM) |
| **Infrastructure encryption** | **Double chiffrement** (2 couches) — à activer **à la création** |
| **Encryption scopes** | Clés différentes par container/blob |
| En transit | **HTTPS** (option *Secure transfer required*) + **TLS minimum** |

---

## 4. Blob Storage

### Structure et types
`Storage account → Container → Blob`
- **Block blob** (fichiers, images, documents) · **Append blob** (logs) · **Page blob** (disques VM).
- **Niveaux d'accès d'un container** : **Private** (défaut) · **Blob** (lecture anonyme des blobs si URL connue) · **Container** (lecture anonyme + **listing**).
- L'accès anonyme doit être **autorisé au niveau du compte** (*Allow Blob anonymous access*).

### Access tiers (⭐ tableau à connaître par cœur)

| Tier | Usage | Coût stockage | Coût accès | **Durée min.** | Disponibilité |
|---|---|---|---|---|---|
| **Hot** | Accès fréquent | 💲💲💲 | 💲 | — | Online (ms) |
| **Cool** | Accès rare | 💲💲 | 💲💲 | **30 jours** | Online (ms) |
| **Cold** | Accès très rare | 💲 | 💲💲💲 | **90 jours** | Online (ms) |
| **Archive** | Archivage long terme | ¢ | 💲💲💲💲 | **180 jours** | **Offline** (heures) |

- **Default account tier** : **Hot ou Cool** seulement ; **Archive = par blob uniquement**.
- **Early deletion fee** si supprimé/déplacé avant la durée minimale.
- **Archive → rehydrate** : *Set blob tier* (vers Hot/Cool/Cold) ou *Copy blob* ; priorité **Standard** (jusqu'à ~15 h) ou **High** (< 1 h pour petits blobs).
- Blob Archive : **métadonnées lisibles**, contenu **non lisible** tant que non réhydraté.

> 📦 Exemple Contoso : photos produit = **Hot** · rapports mensuels = **Cool** après 30 j · logs d'audit = **Cold** · archives légales (10 ans) = **Archive**.

### Lifecycle management

- Règles **JSON** (filtres + actions), exécutées **1 fois par jour**.
- Actions : `tierToCool`, `tierToCold`, `tierToArchive`, `delete` (sur blob de base, snapshots, versions).
- Conditions : `daysAfterModificationGreaterThan`, `daysAfterCreationGreaterThan`, `daysAfterLastAccessTimeGreaterThan` (⚠️ requiert **Last access time tracking** activé).

```json
{
  "rules": [{
    "name": "archive-old-invoices",
    "enabled": true,
    "type": "Lifecycle",
    "definition": {
      "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["invoices/"] },
      "actions": { "baseBlob": {
        "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
        "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
        "delete":        { "daysAfterModificationGreaterThan": 3650 }
      }}
    }
  }]
}
```

### Protection des données : versioning / snapshot / soft delete

| Fonction | Automatique ? | Protège contre | Détails |
|---|---|---|---|
| **Blob versioning** | ✅ auto à chaque écriture/suppression | Écrasement / suppression | Chaque version a un **Version ID** ; coûts de stockage |
| **Blob snapshot** | ❌ **manuel** | Modifications | Copie **lecture seule** à un instant T |
| **Blob soft delete** | ✅ | Suppression / écrasement | Rétention **1–365 j** |
| **Container soft delete** | ✅ | Suppression d'un container | Rétention **1–365 j** |
| **Point-in-time restore** (block blobs) | ✅ | Corruption / suppression massive | Requiert **soft delete + change feed + versioning** |
| **Immutable storage (WORM)** | Politique | Modification/suppression (même admin) | **Time-based retention** ou **legal hold** ; policy *locked* = irréversible |

> ✅ Bonnes pratiques : **soft delete + versioning** activés ; **lifecycle** pour supprimer les anciennes versions.

### Object replication (⭐ ≠ redondance)

| | **Redundancy (GRS…)** | **Object replication** |
|---|---|---|
| Niveau | Tout le **compte** | **Container → container** (règles, filtres par préfixe) |
| Cible | Région pair **imposée** | **N'importe quel** compte (autre région / subscription / tenant) |
| Lecture cible | Après failover (ou RA-) | **Immédiate** (compte indépendant) |
| Type | Tous | **Block blobs** |
| Prérequis | — | **Versioning** (source **et** destination) + **Change feed** (source) |
| Sync | Async | **Async** |

> Usage : rapprocher des données de **utilisateurs** dans une autre région, **réduire la latence** de calcul, distribuer des contenus.

---

## 5. Azure Files & Azure File Sync

### Azure Files

| | **Standard** (HDD) | **Premium** (SSD) |
|---|---|---|
| Tiers | Transaction optimized · Hot · Cool | Provisioned (IOPS/throughput garantis) |
| Protocoles | **SMB** (2.1, 3.x) · NFS ❌ | **SMB** et **NFS 4.1** (Linux) |
| Redondance | LRS, ZRS, GRS, GZRS | **LRS, ZRS** |
| Usage | Partages généraux, profils | DB, apps sensibles à la latence |

- **SMB** = Windows/Linux/macOS (port **445** — parfois bloqué par les FAI → **VPN / ExpressRoute**). **NFS** = Linux uniquement (Premium).
- **Share snapshots** : copies **incrémentales lecture seule** (jusqu'à **200** par share) ; restauration via *Previous Versions* Windows ou le portail.
- **Soft delete des shares** : rétention **1–365 j**.
- **Azure Backup** peut sauvegarder les file shares (snapshots managés).

### Azure File Sync (partage on-prem "cache" du cloud)

```
Windows Server (agent) ──► Storage Sync Service ──► Azure file share
   server endpoint          sync group               cloud endpoint
```

| Composant | Rôle |
|---|---|
| **Storage Sync Service** | Ressource Azure racine |
| **Sync group** | Définit la topologie : **1 cloud endpoint** + **N server endpoints** |
| **Cloud endpoint** | Le **file share Azure** |
| **Server endpoint** | Un **dossier** sur un Windows Server **enregistré** (agent installé) |
| **Cloud tiering** | Garde les fichiers **chauds** en local ; les **froids** = pointeurs (stub) rappelés à la demande |

> Exemple : siège à Paris + agence à Lyon = 2 server endpoints, **1** cloud endpoint ; Lyon voit les fichiers de Paris.

---

## 6. Outils : Storage Explorer & AzCopy

| | **Storage Explorer** | **AzCopy** |
|---|---|---|
| Type | **GUI** (Windows/Mac/Linux) | **CLI** |
| Usage | Parcourir, glisser-déposer, générer des **SAS**, gérer policies | **Copie en masse**, scripts, **sync** |
| Auth | Entra ID, **key**, **SAS**, connection string | **`azcopy login`** (Entra) ou **SAS** dans l'URL |

```bash
azcopy login
azcopy make  "https://contosostore.blob.core.windows.net/backups"
azcopy copy  "C:\data\*" "https://contosostore.blob.core.windows.net/backups?<SAS>" --recursive
azcopy copy  "https://src.blob.core.windows.net/c?<SAS>" "https://dst.blob.core.windows.net/c?<SAS>" --recursive
azcopy sync  "C:\data" "https://contosostore.blob.core.windows.net/backups?<SAS>" --recursive   # source → destination, à sens unique
azcopy list  "https://contosostore.blob.core.windows.net/backups?<SAS>"
```
- ⚠️ `azcopy sync` = **unidirectionnel** (source → destination).
- Copie **compte à compte** = **côté serveur** (pas de passage par ta machine).
- Avec Entra ID, il faut le rôle **Storage Blob Data Contributor** (ou équivalent) sur la cible.

---

## 7. Commandes utiles

```bash
az storage account create -n contosostore -g rg-demo -l francecentral --sku Standard_GRS --kind StorageV2
az storage account update -n contosostore -g rg-demo --access-tier Cool
az storage container create --account-name contosostore -n backups --auth-mode login
az storage account keys list -n contosostore -g rg-demo
az storage container generate-sas --account-name contosostore -n backups --permissions rl --expiry 2026-12-31T00:00Z --https-only
az storage container policy create --account-name contosostore -c backups -n readonly1 --permissions rl
az storage account failover -n contosostore -g rg-demo
```
```powershell
New-AzStorageAccount -ResourceGroupName rg-demo -Name contosostore -Location francecentral -SkuName Standard_GRS -Kind StorageV2
```

---

## 8. Pièges d'examen ⚠️

- "Survivre à la perte d'une **zone** dans la région primaire" → **ZRS** (ou GZRS) — **pas GRS**.
- "Survivre à la perte d'une **région** **et** d'une zone" → **GZRS**.
- "Lire les données **en permanence** depuis la région secondaire" → **RA-GRS / RA-GZRS**.
- "Le moins cher, données recréables" → **LRS**.
- "Accès **temporaire**, **limité**, à un tiers sans donner de clé" → **SAS** (idéalement **user delegation**).
- "**Révoquer** un SAS sans régénérer la clé" → **stored access policy**.
- "Archiver après 180 jours puis supprimer après 10 ans" → **lifecycle management**.
- "Répliquer des blobs vers un **autre compte**" → **object replication** (versioning + change feed).
- "Récupérer un blob **supprimé** / une version précédente" → **soft delete** / **versioning**.
- "Accès à Azure Files par **identité AD**" → **identity-based auth** (AD DS / Entra DS / Entra Kerberos) + **RBAC share-level** + **NTFS**.
- "Serveur on-prem doit garder un cache local d'un share Azure" → **Azure File Sync**.
- "Copier des TB vers Azure via script" → **AzCopy**.
- "Règle IP ne marche pas depuis une VM de la même région" → utiliser **VNet rule / service endpoint**.
- "Blob en **Archive**, lecture impossible" → **rehydrate** d'abord.

## 9. Checklist finale ✅

- [ ] Je remplis de mémoire le tableau LRS/ZRS/GRS/GZRS (copies, durabilité, panne couverte, lecture secondaire)
- [ ] Je connais Hot/Cool/Cold/Archive et leurs durées minimales (**30 / 90 / 180**)
- [ ] Je distingue Access key / SAS (3 types) / stored access policy / Entra ID
- [ ] Je sais configurer un firewall de storage (VNet, IP, private endpoint)
- [ ] Je sais expliquer object replication, versioning, snapshot, soft delete
- [ ] Je sais décrire Azure File Sync (sync group, cloud/server endpoint, tiering)
- [ ] Je connais AzCopy copy/sync/login et Storage Explorer
