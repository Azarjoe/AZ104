# AZ-104 — Fiche 1/6 · Prérequis pour les administrateurs Azure

> Parcours MS Learn : *AZ-104: Prerequisites for Azure administrators*
> Modules actuels : **Introduction to Azure Cloud Shell** · **Deploy Azure infrastructure by using JSON ARM templates**
> Pré-requis officiels : AZ-900 (cloud concepts, architecture & services, management & governance)
> Poids à l'examen : pas de domaine dédié, mais **ARM/Bicep** tombe dans *Compute (20–25 %)* et les outils (Portal/CLI/PowerShell) sont utilisés partout.

---

## 1. Les 5 façons d'administrer Azure

| Outil | Style | Où / comment | Quand l'utiliser |
|---|---|---|---|
| **Azure portal** | GUI | portal.azure.com | Découverte, ponctuel, visualiser |
| **Cloud Shell** | Terminal navigateur | Portal, shell.azure.com, VS Code, app mobile | Aucune installation, déjà authentifié |
| **Azure CLI** (`az`) | Impératif | Bash / cmd / PowerShell, multiplateforme | Scripts, syntaxe courte, sortie JSON/JMESPath |
| **Azure PowerShell** (`Az` module) | Impératif | PowerShell 7+ / Windows PowerShell | Admins Windows, objets .NET, pipeline |
| **ARM template / Bicep** | **Déclaratif** | Fichier `.json` / `.bicep` | Déploiements **répétables**, IaC, CI/CD |

> 🧠 **Impératif** = "fais ces actions dans cet ordre". **Déclaratif** = "voici l'état final voulu, débrouille-toi".
> Exemple parlant : impératif = recette de cuisine pas à pas ; déclaratif = commander un plat au restaurant.

---

## 2. Azure Resource Manager (ARM) — le chef d'orchestre

- **ARM = la couche de gestion unique** : portal, CLI, PowerShell, REST, SDK, templates passent **tous** par l'API ARM.
- Conséquence : RBAC, Policy, Locks, Tags s'appliquent **quel que soit l'outil**.
- **Resource provider** = service qui fournit un type de ressource (`Microsoft.Compute`, `Microsoft.Storage`, `Microsoft.Network`…).
  - Un provider doit être **Registered** dans la subscription pour déployer ses ressources (erreur classique : *"MissingSubscriptionRegistration"*).
  - `az provider register --namespace Microsoft.Compute`
- **Control plane** (gérer la ressource : créer, supprimer, configurer) ≠ **Data plane** (utiliser la ressource : lire un blob, se connecter à la VM).
- Une ressource appartient à **un seul** resource group ; un RG appartient à **une seule** subscription.
- Idempotence : redéployer le même template = même résultat (pas de doublons).

### Hiérarchie de scopes (rappel express)

```
Tenant Root Group
 └─ Management group   (ex. MG-Contoso-Prod)
     └─ Subscription   (ex. Sub-Prod-France)
         └─ Resource group  (ex. rg-web-prod-frc)
             └─ Resource     (ex. vm-web-01)
```
> Détails et exemples complets dans la **fiche 2**.

---

## 3. Azure Cloud Shell

- Shell **dans le navigateur**, authentifié automatiquement avec ton compte.
- **2 expériences** : **Bash** ou **PowerShell** (on peut basculer sans perdre les fichiers).
- Outils préinstallés : `az`, Azure PowerShell, `azcopy`, `git`, `kubectl`, `terraform`, `vim`/`nano`, éditeur intégré (`code .`), etc.
- Session **temporaire** : machine éphémère, **timeout après ~20 min d'inactivité**.

### Persistance des fichiers (question fréquente)

| Élément | Détail |
|---|---|
| Premier lancement | Propose de créer un **storage account** + un **Azure Files share** |
| RG créé | `cloud-shell-storage-<region>` |
| Storage account | Standard **LRS** |
| Share | Monté en `$HOME/clouddrive` |
| `$HOME` | Persisté via une **image de 5 GB** stockée dans le file share |
| Option | **"No storage account required"** → session **éphémère** (rien n'est conservé) |
| Upload / Download | Boutons dans la barre d'outils Cloud Shell |

- Cloud Shell est **gratuit** ; tu ne paies que le storage account.
- Le **storage account est lié à une région** (choisie au 1er lancement).

---

## 4. Azure CLI vs Azure PowerShell

| Action | Azure CLI | Azure PowerShell |
|---|---|---|
| Se connecter | `az login` | `Connect-AzAccount` |
| Lister subscriptions | `az account list -o table` | `Get-AzSubscription` |
| Choisir subscription | `az account set --subscription "Sub-Prod"` | `Set-AzContext -Subscription "Sub-Prod"` |
| Créer un RG | `az group create -n rg-demo -l francecentral` | `New-AzResourceGroup -Name rg-demo -Location francecentral` |
| Créer une VM | `az vm create -g rg-demo -n vm01 --image Ubuntu2204 ...` | `New-AzVM -ResourceGroupName rg-demo -Name vm01 ...` |
| Lister des VMs | `az vm list -g rg-demo -o table` | `Get-AzVM -ResourceGroupName rg-demo` |
| Supprimer un RG | `az group delete -n rg-demo --yes --no-wait` | `Remove-AzResourceGroup -Name rg-demo -Force -AsJob` |
| Filtrer | `--query "[?location=='francecentral'].name"` (JMESPath) | `\| Where-Object Location -eq 'francecentral'` |
| Format de sortie | `-o json\|table\|tsv\|yaml` | `\| Format-Table` / `\| ConvertTo-Json` |

**Repères de syntaxe**

- CLI : `az <groupe> <sous-groupe> <action> --paramètres` (ex. `az storage account create`).
- PowerShell : **Verbe-Nom** (`Get-`, `New-`, `Set-`, `Remove-`) + préfixe **Az** (`Get-AzVM`).
- `-WhatIf` (PowerShell) / `--dry-run` (rare en CLI) → simuler ; `--no-wait` / `-AsJob` → asynchrone.
- Aide : `az vm create --help` / `Get-Help New-AzVM -Examples`.
- Trouver une commande PS : `Get-Command *AzVM*` ; explorer un objet : `$vm | Get-Member`.

---

## 5. ARM templates (JSON)

### Structure d'un template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters":  { },
  "variables":   { },
  "functions":   [ ],
  "resources":   [ ],
  "outputs":     { }
}
```

| Section | Obligatoire ? | Rôle |
|---|---|---|
| `$schema` | ✅ | Définit la version du langage |
| `contentVersion` | ✅ | Version **du template** (à toi de la gérer) |
| `parameters` | ❌ | Valeurs fournies **au déploiement** (types : `string`, `securestring`, `int`, `bool`, `object`, `secureObject`, `array`) |
| `variables` | ❌ | Valeurs calculées, réutilisées |
| `functions` | ❌ | Fonctions utilisateur |
| `resources` | ✅ | Ressources à déployer |
| `outputs` | ❌ | Valeurs renvoyées après déploiement (ex. endpoint) |

### Une ressource typique

```json
{
  "type": "Microsoft.Storage/storageAccounts",
  "apiVersion": "2023-05-01",
  "name": "[parameters('storageName')]",
  "location": "[resourceGroup().location]",
  "sku": { "name": "Standard_LRS" },
  "kind": "StorageV2",
  "dependsOn": [ ]
}
```

### Fonctions utiles (syntaxe `[ ... ]`)

- `[parameters('x')]`, `[variables('x')]`
- `[resourceGroup().location]`, `[subscription().subscriptionId]`
- `[uniqueString(resourceGroup().id)]` → nom unique **déterministe** (parfait pour storage accounts)
- `[concat('st', uniqueString(resourceGroup().id))]`
- `[resourceId('Microsoft.Network/virtualNetworks', 'vnet1')]`
- `[reference('name').property]` → lire une propriété d'une ressource déployée (crée une dépendance implicite)
- `copy` → boucle (créer N ressources)

### Modes de déploiement (⭐ très demandé)

| Mode | Comportement | Risque |
|---|---|---|
| **Incremental** (défaut) | Ajoute / met à jour ce qui est dans le template ; **ne touche pas** au reste du RG | Faible |
| **Complete** | Le RG devient **identique** au template : les ressources **absentes sont supprimées** | ⚠️ Suppression |

> 💡 Exemple : RG contient VM1 + VM2 + Storage. Template = VM1 seulement.
> **Incremental** → VM2 et Storage restent. **Complete** → VM2 et Storage **supprimés**.

### Scopes de déploiement

| Scope | Commande CLI | Usage |
|---|---|---|
| Resource group | `az deployment group create` | Le plus courant |
| Subscription | `az deployment sub create` | Créer des RG, assigner des policies |
| Management group | `az deployment mg create` | Gouvernance à grande échelle |
| Tenant | `az deployment tenant create` | Très rare |

### Commandes essentielles

```bash
# Valider / prévisualiser / déployer
az deployment group validate  -g rg-demo --template-file main.json --parameters @params.json
az deployment group what-if   -g rg-demo --template-file main.json --parameters @params.json
az deployment group create    -g rg-demo --template-file main.json --parameters @params.json
az deployment group create    -g rg-demo --template-file main.json --mode Complete

# Exporter
az group export -n rg-demo > exported.json
```

```powershell
New-AzResourceGroupDeployment -ResourceGroupName rg-demo -TemplateFile main.json -TemplateParameterFile params.json
New-AzResourceGroupDeployment -ResourceGroupName rg-demo -TemplateFile main.json -Mode Complete
Export-AzResourceGroup -ResourceGroupName rg-demo
```

- **What-if** = "dry run" : montre Create / Modify / Delete / NoChange **avant** de déployer.
- **Portail** : *Automation → Export template* (RG ou ressource) · *Deployments → Template* (historique) · *Deploy a custom template* (coller un JSON).
- **Template specs** : templates versionnés stockés dans Azure, partageables via RBAC.
- **Paramètres sensibles** : `securestring` ou référence à **Key Vault** dans le fichier de paramètres.

---

## 6. Bicep

| | ARM (JSON) | Bicep |
|---|---|---|
| Syntaxe | Verbeuse (JSON + `[ ]`) | **Concise**, lisible |
| Dépendances | `dependsOn` explicite parfois | **Implicites** via le nom symbolique |
| Modules | Linked / nested templates | `module` natif |
| Compilation | — | Compile vers un ARM JSON |
| Fonctionnalités | Référence | **Même** couverture Azure (day-0 support) |

```bicep
param location string = resourceGroup().location
param storageName string = 'st${uniqueString(resourceGroup().id)}'

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output blobEndpoint string = sa.properties.primaryEndpoints.blob
```

**Mots-clés Bicep à reconnaître**

- `param`, `var`, `resource`, `module`, `output`, `existing` (référencer une ressource déjà déployée), `targetScope = 'subscription'`
- Décorateurs : `@secure()`, `@allowed([...])`, `@minLength()`, `@description()`
- Boucle : `[for i in range(0, 3): { ... }]` · Condition : `if (deployX)`

**Conversions (⭐ dans les skills measured)**

```bash
az bicep decompile --file main.json     # ARM JSON  → Bicep  (best effort, à relire)
az bicep build     --file main.bicep    # Bicep     → ARM JSON
az bicep install / upgrade
```
- Déployer un `.bicep` : `az deployment group create -g rg -f main.bicep` (pas besoin de compiler avant).

---

## 7. Rappels AZ-900 indispensables (échelles)

### Échelle géographique

```
Geography  →  Region pair  →  Region  →  Availability Zone  →  Datacenter
 (France)     (France Central   (France      (Zone 1, 2, 3)      (bâtiment avec
              + France South)   Central)                          serveurs)
```
> Analogie : **Geography** = pays / cadre légal · **Region** = ville · **AZ** = quartier avec sa propre centrale électrique · **Datacenter** = immeuble.

### Échelle des modèles de service

| Modèle | Tu gères | Exemple Azure |
|---|---|---|
| **IaaS** | OS, runtime, app, données | Virtual Machines |
| **PaaS** | App + données | App Service, Azure SQL |
| **SaaS** | Uniquement l'usage | Microsoft 365 |

### Responsabilité partagée
- **Toujours à toi** : données, identités, endpoints (comptes, appareils).
- **Toujours à Microsoft** : datacenters physiques, réseau physique, hôtes.
- Varie selon le modèle : OS, applications, contrôles réseau.

---

## 8. Pièges d'examen ⚠️

- "Sans rien installer, exécuter une commande Azure depuis mon navigateur" → **Cloud Shell**.
- "Les fichiers Cloud Shell disparaissent" → session **sans storage** ou timeout ; persistance = **Azure Files share (`clouddrive`)**.
- "Ne pas supprimer les autres ressources du RG" → mode **Incremental**.
- "Aligner le RG **exactement** sur le template" → mode **Complete**.
- "Nom de storage account unique dans un template" → `uniqueString()`.
- "Vérifier ce qui va changer **avant** de déployer" → **what-if**.
- "Convertir un template ARM en Bicep" → `az bicep decompile`.
- "Récupérer le template d'une infra existante" → **Export template**.
- "Erreur *MissingSubscriptionRegistration*" → **register** le resource provider.

## 9. Checklist finale ✅

- [ ] Je sais citer CLI vs PowerShell + 3 équivalences de commandes
- [ ] Je connais les sections d'un template ARM et lesquelles sont obligatoires
- [ ] Je distingue **Incremental** et **Complete** avec un exemple
- [ ] Je sais exporter, valider, `what-if`, déployer (RG / subscription)
- [ ] Je sais lire un fichier Bicep simple et le convertir dans les 2 sens
- [ ] Je connais la persistance et le timeout de Cloud Shell
