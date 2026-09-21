# AZ-104 — Fiche 2/6 · Gérer les identités et la gouvernance dans Azure

> Parcours MS Learn : *AZ-104: Manage identities and governance in Azure*
> Modules actuels : **Understand Microsoft Entra ID** · **Create, configure, and manage identities** · **Describe the core architectural components of Azure** · **Azure Policy initiatives** · **Secure your Azure resources with Azure RBAC** · **Microsoft Entra self-service password reset (SSPR)**
> Poids à l'examen : **20–25 %** (avec Compute, le plus gros domaine)

---

## 1. Microsoft Entra ID (ex-Azure AD)

- Service d'**identité cloud** de Microsoft : authentification + autorisation, SSO, MFA, Conditional Access.
- **Tenant** = une instance (annuaire) d'Entra ID = "l'annuaire de Contoso".
- Une **subscription** fait confiance à **un seul** tenant ; un tenant peut avoir **plusieurs** subscriptions.
- Protocoles : OAuth 2.0, OpenID Connect, SAML (≠ Kerberos/NTLM/LDAP de l'AD DS).

### AD DS vs Entra ID vs Entra Domain Services

| | **AD DS** (on-prem) | **Microsoft Entra ID** | **Entra Domain Services** |
|---|---|---|---|
| Nature | Annuaire Windows Server | Identité cloud (IDaaS) | Domaine **managé** dans Azure |
| Protocoles | Kerberos, NTLM, LDAP | OAuth, OIDC, SAML, REST (Graph) | Kerberos, NTLM, LDAP, GPO |
| Structure | OU, GPO, forêts/domaines | **Flat** (pas d'OU/GPO), tenant | Domaine managé (OU + GPO limitées) |
| Usage | Serveurs, PC domain-joined | SaaS, web apps, Azure, M365 | **Lift & shift** d'apps legacy sans DC à gérer |
| Sync | — | **Entra Connect** (Sync / Cloud Sync) depuis AD DS | Synchro **unidirectionnelle** depuis Entra ID |

### Éditions

| Fonction | **Free** | **P1** | **P2** |
|---|:-:|:-:|:-:|
| Users/groups, SSO, B2B, MFA via *security defaults* | ✅ | ✅ | ✅ |
| **SSPR** cloud-only | ✅ | ✅ | ✅ |
| **Groupes dynamiques** | ❌ | ✅ | ✅ |
| **Group-based licensing** | ❌ | ✅ | ✅ |
| **Conditional Access** | ❌ | ✅ | ✅ |
| **SSPR avec writeback** (hybride) | ❌ | ✅ | ✅ |
| **PIM** (accès just-in-time), Identity Protection | ❌ | ❌ | ✅ |

### Types d'identités
- **User** (membre ou **guest**) · **Group** · **Service principal** (identité d'une app) · **Managed identity** (**system-assigned** = liée au cycle de vie de la ressource ; **user-assigned** = indépendante, réutilisable) · **Device**.

---

## 2. Users & groups

### Créer et gérer
- Portail / `az ad user create` / PowerShell (Microsoft Graph) / **Bulk create / invite / delete via CSV**.
- Propriétés : job title, department, manager, usage location, etc.
- **Usage location obligatoire** pour attribuer une licence.

### Groupes

| | **Security group** | **Microsoft 365 group** |
|---|---|---|
| Usage | Accès aux ressources (RBAC, apps) | Collaboration (mailbox, Teams, SharePoint) |
| Membres | Users, devices, groups, service principals | Users uniquement |

| Type d'appartenance | Fonctionnement |
|---|---|
| **Assigned** | Ajout manuel |
| **Dynamic User** | Règle → membres **automatiques** (P1) |
| **Dynamic Device** | Règle sur attributs d'appareil (P1) |

> Exemples de règles dynamiques :
> `user.department -eq "Sales"`
> `user.userType -eq "Guest"`
> `(user.country -eq "France") and (user.accountEnabled -eq true)`

- Un groupe **dynamique** ne peut pas avoir de membres ajoutés à la main.
- Bonne pratique : **assigner rôles et licences à des groupes**, pas à des individus.

### Licences
- **Group-based licensing** (P1) : la licence suit l'appartenance au groupe (arrivée/départ automatique).
- Conflits possibles (plans incompatibles) → erreurs de traitement à corriger.
- Pas de *usage location* → l'attribution **échoue**.

### Utilisateurs externes (guests, B2B)
- **B2B collaboration** : invitation par e-mail → l'invité garde **sa propre identité** (autre tenant, Google, MSA…).
- Créé comme `userType = Guest` ; permissions **limitées par défaut** dans l'annuaire.
- Paramètres : *External collaboration settings* (qui peut inviter), *Cross-tenant access settings*.
- Peut recevoir des **rôles RBAC** Azure comme un membre.

### SSPR — Self-Service Password Reset

| Paramètre | À retenir |
|---|---|
| **Scope** | **None / Selected (groupe) / All** — tester d'abord avec un **groupe pilote** |
| Méthodes | App mobile (notif/code), e-mail, SMS, appel, questions de sécurité |
| Nombre requis | **1 ou 2** méthodes (les **admins** = toujours **2**, sans questions de sécurité) |
| Enregistrement | Users doivent **s'enregistrer** (option : forcer à la connexion) |
| Hybride | **Password writeback** via **Entra Connect** (P1 minimum) |
| Bonus | Peut aussi **débloquer** un compte |

---

## 3. Azure RBAC — contrôle d'accès aux ressources

**Assignation = Security principal + Role definition + Scope**
> "**Lina** (principal) est **Contributor** (rôle) sur **rg-web-prod** (scope)."

### Héritage des scopes
```
Management group  →  Subscription  →  Resource group  →  Resource
   (hérite vers le bas ↓ ; on ne peut pas "retirer" un droit hérité en RBAC, sauf deny assignment)
```
- Permissions **cumulatives (union)** des rôles ; un **deny assignment** (ex. créé par Blueprints/managed apps) **prime**.
- Bonne pratique : **moindre privilège**, rôle sur le **scope le plus étroit** possible.

### Rôles built-in à connaître

| Rôle | Peut faire | Ne peut PAS |
|---|---|---|
| **Owner** | Tout **+ assigner des rôles** | — |
| **Contributor** | Tout créer/gérer | **Assigner des rôles** |
| **Reader** | Lire | Modifier |
| **User Access Administrator** | **Gérer les accès** (assigner des rôles) | Gérer les ressources |
| **Role Based Access Control Administrator** | Assigner des rôles (avec conditions possibles) | Gérer les ressources |
| **Virtual Machine Contributor** | Gérer les VMs | Accéder au VNet / storage connectés, ni se connecter |
| **Virtual Machine Administrator Login** | Se connecter à la VM en admin (Entra login) | Gérer la VM |
| **Network Contributor** | Gérer les réseaux | — |
| **Storage Account Contributor** | Gérer le storage account (control plane) | Lire les blobs (data plane) |
| **Storage Blob Data Reader / Contributor / Owner** | **Data plane** sur les blobs | — |
| **Monitoring Reader / Contributor** | Lire / configurer métriques et alertes | — |

### Entra roles vs Azure roles (⭐ piège classique)

| | **Microsoft Entra roles** | **Azure RBAC roles** |
|---|---|---|
| Gèrent | **Objets de l'annuaire** (users, groups, apps, domaines) | **Ressources Azure** |
| Scope | Tenant, administrative unit, app | Management group / subscription / RG / resource |
| Exemples | **Global Administrator**, User Administrator, Groups Administrator, License Administrator | Owner, Contributor, Reader |
| Custom | Oui | Oui |

> ⚠️ **Global Administrator n'a PAS accès aux subscriptions par défaut.** Il doit activer **"Access management for Azure resources"** (*Elevate access*) → devient **User Access Administrator** au **root scope**.

### Custom roles (JSON)

```json
{
  "Name": "VM Operator",
  "Description": "Start / restart / deallocate VMs",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": ["/subscriptions/<sub-id>"]
}
```
- `NotActions` = **soustraction** de permissions (**ce n'est pas un deny**).
- `DataActions` / `NotDataActions` = data plane.
- Créer : `az role definition create --role-definition role.json` · `New-AzRoleDefinition`.

### Interpréter les accès
- Ressource → **Access control (IAM)** : onglets **Check access**, **Role assignments** (montre aussi les **hérités**), **Roles**, **Deny assignments**.
- `az role assignment list --assignee user@contoso.com --all` · `Get-AzRoleAssignment`.
- Assigner : `az role assignment create --assignee <id> --role "Reader" --scope <scope>`.
- Délai de propagation : **quelques minutes** ; un utilisateur peut devoir se reconnecter.

---

## 4. Gouvernance Azure

### Hiérarchie — exemple concret (Contoso)

| Niveau | Exemple | Ce qu'on y attache |
|---|---|---|
| **Tenant Root Group** | Contoso | Politiques globales |
| **Management group** | `MG-Prod`, `MG-Dev` | RBAC / Policy hérités par toutes les subscriptions dessous |
| **Subscription** | `Sub-Prod-France` | Facturation, quotas, limites, frontière d'accès |
| **Resource group** | `rg-web-prod-frc` | Cycle de vie commun, RBAC, lock, tags |
| **Resource** | `vm-web-01` | Config propre |

### Management groups
- Regroupent des **subscriptions** pour appliquer **Policy / RBAC** en masse.
- Jusqu'à **6 niveaux** de profondeur (hors root et hors niveau subscription) · **10 000** MG par tenant.
- Chaque MG / subscription a **un seul parent**.
- Toutes les nouvelles subscriptions arrivent dans le **Root MG** par défaut (configurable).

### Subscriptions
- Frontière de **facturation**, de **contrôle d'accès** et de **limites/quotas**.
- Utile pour **séparer** : prod/dev, départements, environnements réglementés.
- Actions : changer de MG, **transférer** la facturation, **changer de tenant** (annuaire), renommer, annuler (→ *Disabled*).

### Resource groups
- Conteneur **logique** ; la **location du RG** = où sont stockées les **métadonnées** (les ressources peuvent être dans **d'autres régions**).
- **Pas d'imbrication** de RG · un RG **ne se renomme pas** · **supprimer le RG = supprimer tout son contenu**.
- **Move resources** : entre RG / subscriptions (même tenant) → **verrouille** temporairement source et cible en écriture ; **la location de la ressource ne change pas** ; tous les types ne sont pas déplaçables.

### Tags
- Paires **nom:valeur** (ex. `CostCenter:Finance`, `Env:Prod`) pour organiser, **facturer**, automatiser.
- **⚠️ Non hérités** : un tag posé sur le RG **ne descend pas** aux ressources.
  - Pour l'hériter → **Azure Policy** (effet *Modify* / policy built-in **"Inherit a tag from the resource group"**).
- Max **50 paires** par ressource / RG / subscription.
- Les tags **ne sont pas supprimés** au move d'une ressource.

### Resource locks

| Type | Effet | Exemple |
|---|---|---|
| **CanNotDelete** (*Delete*) | Lecture + modification OK, **suppression interdite** | Protéger un RG de prod |
| **ReadOnly** | **Lecture seule** (comme Reader pour tout le monde, **même Owner**) | Geler une config |

- **Hérité** par les ressources enfants ; s'applique à **tout le monde**, y compris **Owner**.
- Ne s'applique **qu'au control plane** (pas aux opérations *data plane*, ex. écrire un blob).
- Pour supprimer : **retirer le lock d'abord** (droit `Microsoft.Authorization/locks/*` → Owner / User Access Administrator).
- ⚠️ **ReadOnly** peut casser des choses inattendues : **démarrer/arrêter une VM**, **lister les clés** d'un storage account (`listKeys` est une action POST), ajouter une ressource dans le RG…

### Azure Policy

| Concept | Rôle |
|---|---|
| **Policy definition** | Règle (`if` condition → `then` effet), en JSON |
| **Initiative** (policy set) | **Groupe** de policies pour un objectif (ex. conformité ISO 27001) |
| **Assignment** | Applique une definition/initiative à un **scope** (avec *exclusions* possibles) |
| **Exemption** | Exclusion **temporaire/justifiée** d'une ressource/scope |
| **Remediation task** | Corrige les ressources **déjà** non conformes (DeployIfNotExists / Modify) |

**Effets à connaître**

| Effet | Ce qu'il fait | Exemple |
|---|---|---|
| **Deny** | **Bloque** la création/modif non conforme | "Interdire toute région ≠ France Central" |
| **Audit** | **Signale** (warning) sans bloquer | "Auditer les VMs sans backup" |
| **Append** | Ajoute des champs à la requête | Ajouter un tag / règle IP |
| **Modify** | Ajoute/modifie/supprime propriétés ou **tags** | Forcer le tag `CostCenter` |
| **DeployIfNotExists** | Déploie une ressource associée si absente | Installer l'extension monitoring |
| **AuditIfNotExists** | Audite l'absence d'une ressource liée | VM sans agent |
| **Disabled** | Désactive la règle | Tests |

- **Effet sur l'existant** : une policy **Deny** ne supprime **pas** ce qui existe → la ressource apparaît **non conforme** (Compliance).
- **DeployIfNotExists / Modify** utilisent une **managed identity** avec un rôle suffisant.
- **Évaluation** : à la création/modification, à l'assignation (~30 min), puis **toutes les 24 h** ; à la demande : `az policy state trigger-scan`.
- **Enforcement mode** : *Default* (applique) / *DoNotEnforce* (**simule** seulement).

### Policy vs RBAC vs Lock (⭐ question récurrente)

| | **RBAC** | **Azure Policy** | **Resource lock** |
|---|---|---|---|
| Question | **QUI** peut faire **quoi** ? | **QUELLES ressources** sont autorisées / conformes ? | **PROTÉGER** contre suppression / modif accidentelle |
| S'applique à | Identités | **Ressources** (propriétés) | Ressources |
| Un Owner peut contourner ? | — | Non (Deny s'applique à lui) | **Non** (jusqu'à retrait du lock) |
| Exemple | Lina = Contributor | Interdire les VMs `Standard_E64` | Delete lock sur `rg-prod` |

### Coûts

| Outil | Rôle |
|---|---|
| **Cost analysis** | Visualiser les dépenses (par RG, tag, service, ressource) |
| **Budgets** | Seuils d'**alerte** (**réel** ou **prévisionnel**) sur MG / subscription / RG |
| **Alertes de budget** | Notifient par e-mail / **action group** — **n'arrêtent PAS** les ressources (sauf automation via action group) |
| **Azure Advisor** | Recommandations **5 piliers** : Reliability, Security, Performance, **Cost**, Operational Excellence |
| Économies | Redimensionner/arrêter VMs sous-utilisées, **Reservations / Savings plans**, supprimer disques non attachés |
| **Tags** | Ventiler les coûts (ex. par `CostCenter`) |

---

## 5. Architecture Azure — les échelles à connaître

### Échelle physique (avec exemples)

| Niveau | Définition | Exemple concret |
|---|---|---|
| **Geography** | Marché/frontière de **résidence des données** (au moins 2 régions) | **France**, Europe, US |
| **Region pair** | 2 régions dans la même geography, **≥ 300 miles** si possible | **France Central** (Paris) ↔ **France South** (Marseille) · **West Europe** (Pays-Bas) ↔ **North Europe** (Irlande) · **East US** ↔ **West US** |
| **Region** | Ensemble de datacenters reliés par un réseau à faible latence | France Central |
| **Availability Zone** | **≥ 3 zones** par région compatible ; chaque zone = 1 ou +datacenters avec **alim, refroidissement, réseau indépendants** | France Central : Zone 1, 2, 3 |
| **Datacenter** | Bâtiment physique avec serveurs | — |

> 🗺️ Analogie : Geography = **pays** · Region = **ville** · AZ = **quartier avec sa propre centrale électrique** · Datacenter = **immeuble**.

**Ce que la région pair apporte**
- **Mises à jour planifiées séquentielles** (une région de la paire à la fois).
- **Récupération prioritaire** en cas de panne large.
- **Cible de réplication géo** (GRS, GZRS, Site Recovery).
- ⚠️ Certaines régions récentes **n'ont pas de paire**. Certaines paires (ex. France South) sont **à accès restreint** et **sans AZ**.

### Services : zonal vs zone-redundant

| Type | Sens | Exemples |
|---|---|---|
| **Zonal** | Tu **choisis** la zone | VM (Zone 1), disque managé |
| **Zone-redundant** | Azure **répartit** sur plusieurs zones automatiquement | ZRS, Standard Load Balancer, App Gateway v2, VPN Gateway zone-redundant |
| **Non-regional / global** | Pas lié à une région | Entra ID, Azure DNS, Traffic Manager |

- Le **numéro de zone est logique** par subscription (Zone 1 chez toi ≠ forcément Zone 1 physique chez un autre client).
- **Clouds souverains** : Azure Government (US), Azure China (opéré par 21Vianet).

### SLA composite (rappel)
- Deux services en **série** → SLA global = **produit** (ex. 99,95 % × 99,9 % ≈ 99,85 %).
- Ajouter des **zones / redondance** augmente la disponibilité.

---

## 6. Pièges d'examen ⚠️

- "Appliquer une règle à **toutes** les subscriptions d'un département" → **Management group** (+ Policy/RBAC).
- "Empêcher de créer des ressources hors France" → **Policy (Allowed locations, Deny)**, pas RBAC.
- "Empêcher la **suppression** accidentelle, même par un Owner" → **Lock CanNotDelete**.
- "Le tag du RG n'est pas sur les ressources" → **non hérité** → Policy *Modify*.
- "Lina peut créer des VMs mais pas donner des accès" → **Contributor**.
- "Quelqu'un doit **déléguer des accès** sans gérer les ressources" → **User Access Administrator**.
- "Global admin ne voit pas les subscriptions" → **Elevate access**.
- "Dynamic group / group-based licensing" → nécessite **P1**.
- "Un budget doit **arrêter** les VMs" → budget seul ne suffit pas → **action group + automation**.
- "Ressources non conformes existantes après Deny" → **remediation** (DeployIfNotExists/Modify) ou correction manuelle.
- "Utilisateur sans *usage location*" → **licence impossible**.
- "Résilience à la panne d'un **datacenter/zone**" → **AZ** ; "panne de **région**" → **région pair / géo-réplication**.

## 7. Checklist finale ✅

- [ ] Je place Entra ID vs AD DS vs Entra Domain Services et les éditions Free/P1/P2
- [ ] Je fais la différence Entra roles / Azure RBAC + Elevate access
- [ ] Je sais lire l'héritage des rôles (MG → Sub → RG → Resource)
- [ ] Je connais les effets de Policy (Deny / Audit / Modify / DINE) et l'ordre Policy vs RBAC vs Lock
- [ ] Je sais que les tags ne sont pas hérités et comment corriger
- [ ] Je récite Geography → Region pair → Region → AZ → Datacenter avec un exemple
- [ ] Je sais paramétrer SSPR (scope, méthodes, writeback)
