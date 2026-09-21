# AZ-104 — Fiche 7 · Glossaire des définitions

> Toutes les notions importantes des fiches 01 à 06, en **une phrase** chacune, avec un exemple quand il aide.
> Les termes restent en anglais (ceux de l'examen), les explications sont en français.
> Classement : **notions transversales**, puis un chapitre par fiche.

---

## 0. Notions transversales

- **SLA** (Service Level Agreement) — engagement contractuel de **disponibilité** d'un service, exprimé en pourcentage. *Ex. 99,99 % ≈ 52 min d'indisponibilité max par an.*
- **Disponibilité** — capacité à **accéder** au service maintenant. *Une VM qui redémarre est indisponible mais n'a rien perdu.*
- **Durabilité** — probabilité que les données **ne soient pas perdues** sur la durée. *LRS = 11 nines.*
- **Nines** — nombre de **9** dans un pourcentage de fiabilité ; chaque 9 en plus divise le risque par 10. *99,9 % = 3 nines.*
- **RPO** (Recovery Point Objective) — **durée maximale de données** que tu acceptes de perdre. *RPO 15 min = tu peux perdre les 15 dernières minutes.*
- **RTO** (Recovery Time Objective) — **durée maximale** pour que le service soit de nouveau opérationnel après une panne.
- **Réplication synchrone** — l'écriture n'est confirmée qu'**après** l'enregistrement de toutes les copies (pas de perte, un peu plus lent).
- **Réplication asynchrone** — l'écriture est confirmée **avant** la copie vers l'autre site (plus rapide, mais un décalage est possible).
- **Control plane** — le plan de **gestion** d'une ressource (la créer, la configurer, la supprimer) via ARM.
- **Data plane** — le plan d'**utilisation** d'une ressource (lire un blob, se connecter à une VM).
- **Stateful** — qui **mémorise** l'état d'une connexion : si l'aller est autorisé, le retour l'est aussi. *NSG, Azure Firewall.*
- **Idempotent** — qui donne **le même résultat** même exécuté plusieurs fois. *Redéployer un template ne crée pas de doublons.*
- **Scale up / down (vertical)** — changer la **taille** d'une ressource. *Passer d'une VM D2 à D8.*
- **Scale out / in (horizontal)** — changer le **nombre d'instances**. *Passer de 2 à 10 serveurs web.*
- **IaaS / PaaS / SaaS** — niveaux de service cloud : tu gères l'OS et l'app (IaaS, *VM*), seulement l'app (PaaS, *App Service*), ou rien (SaaS, *Microsoft 365*).
- **Responsabilité partagée** — Microsoft gère le physique ; **tu es toujours responsable** de tes données, de tes identités et de tes accès.

---

## 1. Prérequis pour les administrateurs Azure

### Outils
- **Azure portal** — interface web graphique pour gérer Azure. *portal.azure.com*
- **Azure Cloud Shell** — terminal **dans le navigateur**, déjà authentifié, en Bash ou PowerShell.
- **clouddrive** — dossier de Cloud Shell relié à un **Azure Files share**, qui conserve tes fichiers entre les sessions.
- **Azure CLI (`az`)** — outil en ligne de commande multiplateforme pour gérer Azure. *`az group create ...`*
- **Azure PowerShell (module Az)** — commandes PowerShell au format **Verbe-Nom** pour gérer Azure. *`New-AzResourceGroup`*
- **Cmdlet** — une commande PowerShell. *`Get-AzVM`*
- **JMESPath (`--query`)** — langage de **filtre** des résultats JSON de la CLI.

### Azure Resource Manager
- **ARM** (Azure Resource Manager) — la **couche de gestion** unique par laquelle passent portail, CLI, PowerShell et templates.
- **Resource provider** — service qui fournit un type de ressource, à **enregistrer** dans la subscription. *`Microsoft.Compute`*
- **Resource group (RG)** — conteneur logique de ressources qui partagent le même cycle de vie.

### Infrastructure as Code
- **IaC** (Infrastructure as Code) — décrire l'infrastructure **dans des fichiers** plutôt que par des clics.
- **Déclaratif** — tu décris **l'état final voulu**, Azure trouve comment y arriver. *Template ARM, Bicep.*
- **Impératif** — tu décris **les actions**, dans l'ordre. *Script CLI ou PowerShell.*
- **ARM template** — fichier **JSON** qui décrit des ressources à déployer.
- **Bicep** — langage **plus lisible** qui se compile en template ARM.
- **`parameters`** — valeurs fournies **au moment du déploiement**.
- **`variables`** — valeurs calculées et réutilisées dans le template.
- **`resources`** — la liste des ressources à déployer (**seule section obligatoire** avec `$schema` et `contentVersion`).
- **`outputs`** — valeurs renvoyées **après** le déploiement. *L'URL d'un site.*
- **`dependsOn`** — indique qu'une ressource doit être créée **après** une autre.
- **`uniqueString()`** — fonction qui génère un texte **unique mais reproductible**. *Utile pour un nom de storage account.*
- **Module (Bicep)** — un fichier Bicep **réutilisable**, appelé depuis un autre.
- **Mode Incremental** — le déploiement **ajoute/met à jour** ce qui est dans le template et **ne touche pas** au reste du RG (**défaut**).
- **Mode Complete** — le RG devient **identique au template** : ce qui n'y est pas est **supprimé**.
- **What-if** — **simulation** d'un déploiement qui montre ce qui sera créé, modifié ou supprimé.
- **Scope de déploiement** — niveau où le template est déployé : RG, subscription, management group ou tenant.
- **Template spec** — template **versionné** stocké dans Azure et partageable via RBAC.
- **Decompile** — convertir un template ARM JSON **en Bicep** (`az bicep decompile`).

---

## 2. Identités et gouvernance

### Identités
- **Microsoft Entra ID** (ex-Azure AD) — service d'**identité cloud** de Microsoft (connexion, SSO, MFA).
- **Tenant** — une instance d'Entra ID, c'est-à-dire **l'annuaire** d'une organisation.
- **AD DS** (Active Directory Domain Services) — annuaire Windows Server **on-premises**.
- **Microsoft Entra Domain Services** — **domaine AD managé** dans Azure, pour les applications qui ont besoin de Kerberos, NTLM ou LDAP.
- **Entra Connect** — outil qui **synchronise** l'AD on-premises avec Entra ID.
- **Entra ID Free / P1 / P2** — éditions : P1 apporte **groupes dynamiques, Conditional Access, group-based licensing, writeback** ; P2 ajoute **PIM** et Identity Protection.
- **MFA** (Multi-Factor Authentication) — vérification d'identité par **plusieurs preuves**. *Mot de passe + code de l'app.*
- **Conditional Access** — règles qui décident **selon le contexte** (lieu, appareil, risque) d'autoriser, bloquer ou exiger la MFA.
- **PIM** (Privileged Identity Management) — accès **privilégié temporaire** : le rôle s'active seulement quand il est nécessaire (P2).
- **Service principal** — identité d'une **application**.
- **Managed identity** — identité **gérée par Azure** pour une ressource, sans mot de passe à stocker.
  - **System-assigned** — liée à **une** ressource et supprimée avec elle.
  - **User-assigned** — ressource indépendante, **réutilisable** sur plusieurs ressources.

### Utilisateurs et groupes
- **Security group** — groupe utilisé pour **donner des accès** (RBAC, applications).
- **Microsoft 365 group** — groupe de **collaboration** (boîte mail, Teams, SharePoint) ; uniquement des utilisateurs.
- **Dynamic group** — groupe dont les membres sont ajoutés **automatiquement** par une règle. *`user.department -eq "Sales"`*
- **Group-based licensing** — la licence **suit l'appartenance** au groupe.
- **Usage location** — pays de l'utilisateur, **obligatoire** pour lui attribuer une licence.
- **Guest / B2B** — utilisateur **externe** invité qui garde sa propre identité.
- **SSPR** (Self-Service Password Reset) — l'utilisateur **réinitialise seul son mot de passe** après vérification de son identité.
- **Password writeback** — renvoie le nouveau mot de passe **du cloud vers l'AD on-premises** (hybride, via Entra Connect, P1 minimum).

### RBAC
- **Azure RBAC** (Role-Based Access Control) — système d'**autorisation** des ressources Azure : *qui* peut faire *quoi* et *où*.
- **Security principal** — **qui** reçoit le rôle : utilisateur, groupe, service principal ou managed identity.
- **Role definition** — la liste de permissions d'un rôle.
- **Role assignment** — l'association **principal + rôle + scope**.
- **Scope** — **où** s'applique le rôle : management group, subscription, RG ou ressource ; il **descend** vers les niveaux inférieurs.
- **Owner** — tous les droits **et** le droit d'attribuer des rôles.
- **Contributor** — tous les droits sur les ressources, **sauf** attribuer des rôles.
- **Reader** — **lecture seule**.
- **User Access Administrator** — **gère les accès** uniquement, pas les ressources.
- **Custom role** — rôle **sur mesure** défini en JSON (`Actions`, `NotActions`, `DataActions`, `AssignableScopes`).
- **`NotActions`** — permissions **retirées** d'un rôle ; ce n'est **pas** un deny.
- **Deny assignment** — blocage explicite d'un accès, **créé par Azure** (Blueprints, managed apps), non créable à la main.
- **Entra role** — rôle qui gère **l'annuaire** (utilisateurs, groupes) ; ≠ rôle Azure qui gère les **ressources**.
- **Global Administrator** — administrateur suprême d'Entra ID, **sans accès aux subscriptions** tant qu'il n'a pas fait l'**Elevate access**.

### Gouvernance
- **Management group** — regroupement de **subscriptions** pour appliquer RBAC et Policy en masse (6 niveaux max).
- **Subscription** — frontière de **facturation, d'accès et de quotas**.
- **Tag** — paire **nom:valeur** pour organiser et ventiler les coûts ; **non hérité** des RG vers les ressources.
- **Resource lock** — protection contre la suppression ou la modification, **même pour un Owner**.
  - **CanNotDelete** — modification autorisée, **suppression interdite**.
  - **ReadOnly** — **lecture seule** : plus aucune modification.
- **Azure Policy** — vérifie et impose que les **ressources respectent des règles**. *Interdire toute région hors France.*
- **Policy definition** — la règle (condition + effet).
- **Initiative** — **ensemble** de policies pour un objectif. *Conformité ISO 27001.*
- **Policy assignment** — application d'une policy ou initiative à un **scope**.
- **Exemption** — exclusion **justifiée** d'un scope ou d'une ressource.
- **Remediation task** — corrige les ressources **déjà** non conformes.
- **Effets de Policy :**
  - **Deny** — **bloque** ;
  - **Audit** — **signale** sans bloquer ;
  - **Append / Modify** — **ajoute ou modifie** des propriétés ou tags ;
  - **DeployIfNotExists** — **déploie** une ressource associée si elle manque ;
  - **AuditIfNotExists** — signale l'**absence** d'une ressource liée ;
  - **Disabled** — règle désactivée.
- **Budget** — seuil de dépense qui **envoie une alerte** (n'arrête pas les ressources).
- **Cost analysis** — visualisation des dépenses par RG, tag ou service.
- **Azure Advisor** — recommandations sur 5 axes : fiabilité, sécurité, performance, coût, excellence opérationnelle.

### Architecture Azure
- **Geography** — zone de **résidence des données** contenant au moins 2 régions. *France, Europe, US.*
- **Region** — ensemble de datacenters reliés par un réseau à faible latence. *France Central.*
- **Region pair** — deux régions d'une même geography, associées pour la reprise et les mises à jour. *France Central ↔ France South.*
- **Availability Zone (AZ)** — emplacement physique **séparé** dans une région (alimentation, refroidissement et réseau indépendants) ; **3 par région compatible**.
- **Datacenter** — le bâtiment qui contient les serveurs.
- **Zonal** — ressource que **tu places** dans une zone précise. *Une VM en Zone 1.*
- **Zone-redundant** — ressource qu'Azure **répartit automatiquement** sur plusieurs zones. *ZRS.*
- **Sovereign cloud** — instance d'Azure **isolée** pour des exigences légales. *Azure Government, Azure China.*

---

## 3. Stockage

### Bases
- **Storage account** — conteneur qui regroupe Blob, Files, Queue et Table ; nom **unique**, 3-24 caractères, minuscules et chiffres.
- **Blob Storage** — stockage d'**objets** (fichiers non structurés).
- **Container** — dossier de premier niveau **d'un compte Blob**.
- **Blob** — fichiers classiques / journaux à ajouter en fin / disques de VM.
- **Azure Files** — **partage de fichiers** SMB ou NFS hébergé dans Azure.
- **Queue Storage** — file de **messages** entre composants.
- **Table Storage** — base **NoSQL** clé-valeur.
- **Hierarchical namespace** — organisation en vrais **dossiers** (Data Lake Gen2).
- **Custom domain** — nom perso pour l'endpoint, via un **CNAME**.

### Redondance
- **LRS** (Locally Redundant Storage) — **3 copies dans un datacenter** de la région ; le moins cher.
- **ZRS** (Zone-Redundant Storage) — **3 copies dans 3 zones** de la région.
- **GRS** (Geo-Redundant Storage) — **LRS** dans la région principale **+ copie LRS** dans la région pair.
- **GZRS** (Geo-Zone-Redundant Storage) — **ZRS** dans la région principale **+ copie LRS** dans la région pair.
- **RA-GRS / RA-GZRS** — comme GRS/GZRS, avec la **lecture** de la région secondaire **en permanence**.
- **Failover** — bascule vers la région secondaire ; le compte devient LRS (ou ZRS pour GZRS) dans la nouvelle région principale.

### Sécurité
- **Access key** — clé qui donne un **accès total** au compte ; il y en a **2** pour permettre la rotation.
- **SAS** (Shared Access Signature) — **URL signée** qui donne un accès **limité** (droits, durée, IP). *Un comptable dépose un fichier pendant 24 h.*
- **User delegation SAS** — SAS signé avec **Entra ID** ; le plus sûr, réservé à Blob.
- **Service SAS** — SAS signé avec la clé du compte, pour **un service**.
- **Account SAS** — SAS signé avec la clé du compte, pour **un ou plusieurs services**.
- **Stored access policy** — paramètres d'un SAS **enregistrés côté serveur**, pour le **révoquer** sans changer la clé.
- **Storage firewall** — règles qui limitent l'accès **par réseau** (subnets, IP publiques).
- **SSE** (Storage Service Encryption) — **chiffrement au repos**, toujours actif, AES-256.
- **Customer-managed key (CMK)** — clé de chiffrement **que tu gères** dans Key Vault.
- **Infrastructure encryption** — **double chiffrement**, à activer à la création du compte.
- **Encryption scope** — clé de chiffrement **différente par container ou blob**.
- **Identity-based access (Azure Files)** — accès SMB par **identité** (AD DS, Entra Domain Services ou Entra Kerberos) : **RBAC** pour le partage, **NTFS** pour dossiers et fichiers.
- **ACL NTFS** — permissions **Windows** sur les dossiers et fichiers.

### Blob : niveaux et protection
- **Hot** — accès **fréquent** ; stockage plus cher, accès moins cher.
- **Cool** — accès **rare** ; durée minimale **30 jours**.
- **Cold** — accès **très rare** ; durée minimale **90 jours**.
- **Archive** — **archivage long terme**, contenu **hors ligne** ; durée minimale **180 jours**.
- **Rehydration** — remettre un blob Archive **en ligne** pour le lire.
- **Early deletion fee** — frais si un blob est supprimé **avant** la durée minimale de son niveau.
- **Lifecycle management** — règles **automatiques** qui changent de niveau ou suppriment les blobs selon leur âge.
- **Blob versioning** — conserve **automatiquement** les anciennes versions d'un blob.
- **Snapshot** — copie **en lecture seule**, prise **manuellement**, d'un blob à un instant T.
- **Soft delete** — les éléments supprimés restent **récupérables** pendant 1 à 365 jours.
- **Change feed** — journal des **modifications** des blobs.
- **Point-in-time restore** — remettre des block blobs **dans leur état passé**.
- **Immutable storage (WORM)** — données **non modifiables et non supprimables** pendant une durée. *Write Once, Read Many.*
- **Legal hold** — verrou d'immuabilité **sans date de fin** tant qu'il n'est pas levé.
- **Object replication** — copie **asynchrone de blobs** entre deux comptes, container par container.

### Azure Files et outils
- **SMB** — protocole de partage Windows (port **445**).
- **NFS** — protocole de partage **Linux** (Premium).
- **Share snapshot** — copie **incrémentale** en lecture seule d'un partage Azure Files.
- **Azure File Sync** — **synchronise** un serveur Windows on-premises avec un partage Azure Files.
  - **Storage Sync Service** — ressource Azure **racine**.
  - **Sync group** — définit ce qui est synchronisé : **1 cloud endpoint + N server endpoints**.
  - **Cloud endpoint** — le partage Azure Files.
  - **Server endpoint** — un **dossier** d'un Windows Server enregistré.
  - **Cloud tiering** — ne garde en local que les fichiers **récents ou fréquents**, les autres deviennent des raccourcis.
- **Storage Explorer** — application **graphique** pour parcourir et gérer le stockage.
- **AzCopy** — outil en **ligne de commande** pour copier ou synchroniser de gros volumes (`azcopy sync` est à **sens unique**).

---

## 4. Compute

### Machines virtuelles
- **VM** (Virtual Machine) — serveur virtuel **IaaS** dont tu gères l'OS.
- **Taille de VM** — combinaison CPU / mémoire / disques. *B = burstable, D = généraliste, E = mémoire, F = calcul, N = GPU.*
- **OS disk** — disque du système ; **persistant**.
- **Data disk** — disque de données ; **persistant**.
- **Temporary disk** — disque local de l'hôte ; **perdu** lors d'un redeploy ou d'un resize.
- **Managed disk** — disque **géré par Azure** ; types : Standard HDD, Standard SSD, Premium SSD, Premium SSD v2, Ultra Disk.
- **Snapshot (disque)** — copie d'un **disque** à un instant T.
- **Image** — **modèle** de VM pour en créer d'autres.
- **Azure Disk Encryption (ADE)** — chiffrement **dans l'OS** (BitLocker ou dm-crypt) avec Key Vault.
- **Encryption at host** — chiffrement sur l'**hôte** : inclut **disque temporaire et caches** ; incompatible avec ADE sur la même VM.
- **Trusted Launch** — option de sécurité de démarrage (**Secure Boot**, vTPM).
- **Azure Resource Mover** — outil pour **déplacer** des ressources vers une autre région.

### Disponibilité
- **Availability set** — répartit les VMs sur des **racks** différents dans un même datacenter (**99,95 %**).
- **Fault domain** — un **rack** (alimentation et réseau communs) ; 3 maximum.
- **Update domain** — groupe de VMs **redémarré ensemble** lors d'une maintenance ; 20 maximum.
- **Availability Zones (VM)** — VMs réparties sur plusieurs **zones** (**99,99 %**).
- **Proximity placement group** — rapproche les VMs pour réduire la **latence**.
- **VMSS** (Virtual Machine Scale Set) — groupe de VMs **identiques**, dont le nombre change **automatiquement** selon la charge.
  - **Flexible** — mode recommandé, VMs pouvant **différer**.
  - **Uniform** — VMs strictement **identiques**.
- **Autoscale** — ajout et retrait **automatiques** d'instances, selon une métrique ou un planning.
- **Cooldown** — délai d'**attente** après un scale avant une nouvelle action.
- **Upgrade policy** — manière de mettre à jour les instances d'un VMSS : **Manual, Automatic, Rolling**.

### Conteneurs
- **Container** — application **empaquetée** avec ce dont elle a besoin pour tourner.
- **ACR** (Azure Container Registry) — **registre privé** d'images de conteneurs. *`contosoacr.azurecr.io`*
- **ACR Tasks / `az acr build`** — construit l'image **dans Azure**, sans Docker local.
- **ACI** (Azure Container Instances) — lance des conteneurs **sans serveur** à gérer, sans autoscale ; idéal pour les tâches courtes.
- **Container group** — ensemble de conteneurs ACI qui **partagent** réseau, stockage et cycle de vie (comme un pod).
- **Restart policy** — quand relancer un conteneur : **Always, OnFailure, Never**.
- **ACA** (Azure Container Apps) — plateforme de conteneurs **serverless** avec autoscale.
  - **Environment** — frontière partagée (réseau, logs) de plusieurs apps.
  - **Revision** — **version immuable** d'une app, utile pour répartir le trafic.
  - **Ingress** — accès entrant à l'app (HTTP/TCP, externe ou interne).
  - **KEDA** — moteur qui déclenche le scale selon des **événements** (file de messages, etc.).
  - **Scale-to-zero** — l'app **s'arrête complètement** sans demande et ne coûte rien.
- **AKS** (Azure Kubernetes Service) — **Kubernetes** managé, pour les plateformes complexes.

### App Service
- **App Service** — plateforme **PaaS** pour héberger des applications web et API.
- **App Service plan** — les **serveurs** (région, OS, taille, tier) qui font tourner une ou plusieurs apps.
- **Tiers** — Free/Shared (dev), Basic (manuel), **Standard** (autoscale, slots, backup), Premium (performance), Isolated (réseau dédié).
- **Deployment slot** — **copie live** de l'app avec son propre nom, pour tester avant la mise en production.
- **Swap** — **échange** entre le slot de staging et la production, sans coupure.
- **Slot setting (sticky)** — paramètre qui **reste sur son slot** lors d'un swap.
- **Always On** — empêche l'app de **s'endormir** après une période d'inactivité.
- **Managed certificate** — certificat TLS **gratuit** géré par App Service.
- **SNI SSL** — liaison TLS **courante**, plusieurs sites sur une même IP.
- **IP-based SSL** — liaison TLS avec une **IP dédiée**, pour les anciens clients.
- **Enregistrement `asuid`** — TXT DNS qui **prouve** que tu possèdes le domaine.
- **VNet integration** — permet à l'app d'**appeler** des ressources privées (**sortant**).
- **Access restrictions** — filtres **entrants** par IP ou service tag.
- **Hybrid Connections** — relais pour joindre une ressource **on-premises**.
- **ASE** (App Service Environment) — App Service **isolé** dans ton VNet.

---

## 5. Réseaux virtuels

### Réseau de base
- **VNet** (Virtual Network) — ton **réseau privé** dans Azure, régional et gratuit.
- **Subnet** — **sous-division** d'un VNet.
- **CIDR** — notation d'une plage d'adresses IP. *`10.10.1.0/24` = 256 adresses.*
- **RFC 1918** — plages d'adresses **privées** : `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
- **Adresses réservées** — Azure retire **5 adresses** par subnet (un /24 offre donc **251** IP utilisables).
- **GatewaySubnet** — subnet au **nom imposé** pour les gateways VPN et ExpressRoute.
- **AzureBastionSubnet** — subnet au nom imposé pour Bastion (**/26 minimum**).
- **Subnet delegation** — subnet **réservé** à un service PaaS.
- **NIC** (Network Interface Card) — carte réseau virtuelle d'une VM, rattachée à **un subnet**.
- **Public IP (Standard)** — IP publique **statique**, fermée par défaut tant qu'un NSG ne l'autorise pas.
- **NAT Gateway** — fournit une **sortie Internet** partagée et prévisible à un subnet.

### Sécurité réseau
- **NSG** (Network Security Group) — **pare-feu L3/L4** à règles Allow/Deny, appliqué à un subnet ou une NIC.
- **Priorité (NSG)** — de **100 à 4096** ; la règle au **plus petit numéro** est évaluée en premier.
- **Règles par défaut (NSG)** — autorisent le VNet et le load balancer, **refusent** le reste en entrée (`DenyAllInBound`).
- **Service tag** — groupe d'adresses IP **géré par Microsoft**. *`Internet`, `Storage`, `AzureLoadBalancer`.*
- **ASG** (Application Security Group) — regroupe des NICs **par rôle** pour écrire des règles NSG sans IP. *`asg-web`*
- **Effective security rules** — vue **fusionnée** des règles NSG qui s'appliquent à une NIC.
- **Azure Bastion** — accès **RDP/SSH via le navigateur**, sans IP publique sur la VM.
- **Service endpoint** — autorise **un subnet** à joindre un service PaaS ; le service garde son **IP publique** ; gratuit.
- **Private endpoint / Private Link** — donne à un service PaaS **une IP privée** dans ton VNet.
- **Azure Firewall** — pare-feu **managé** L3-L7 (filtrage par FQDN, threat intelligence).

### Connectivité et routage
- **VNet peering** — relie **deux VNets** par le réseau Microsoft ; **non transitif**.
- **Non transitif** — A↔B et B↔C **n'implique pas** A↔C.
- **Gateway transit** — un VNet **partage** sa gateway avec les VNets peerés.
- **VPN Gateway** — passerelle de **connexion VPN chiffrée** vers Azure.
- **Site-to-Site (S2S)** — VPN entre un **site** on-premises et Azure.
- **Point-to-Site (P2S)** — VPN d'un **poste individuel** vers Azure.
- **ExpressRoute** — **lien privé** dédié vers Azure, sans passer par Internet.
- **Virtual WAN** — service de connectivité **en étoile** géré, pour de nombreux sites.
- **System route** — route créée **automatiquement** par Azure.
- **UDR** (User-Defined Route) — route **personnalisée** qui remplace une route système.
- **Route table** — ensemble d'UDR **associé à un ou plusieurs subnets**.
- **Next hop** — **prochain saut** d'un paquet : Virtual network gateway, Virtual network, Internet, Virtual appliance ou None.
- **NVA** (Network Virtual Appliance) — **appliance** réseau tierce dans une VM (firewall, routeur).
- **IP forwarding** — autorise une NIC à transférer du trafic **qui ne lui est pas destiné** ; requis pour un NVA.
- **Longest prefix match** — la route **la plus spécifique** l'emporte.

### DNS
- **Azure DNS** — service d'**hébergement de zones DNS**.
- **DNS zone** — regroupe les **enregistrements** d'un domaine. *`contoso.com`*
- **Délégation** — remplacer les **name servers** chez le registrar par ceux d'Azure.
- **Enregistrements** — **A** (nom → IPv4), **AAAA** (IPv6), **CNAME** (alias de nom), **MX** (mail), **TXT** (texte), **NS** (serveurs de noms).
- **Alias record** — enregistrement qui **pointe vers une ressource Azure**, utilisable à la racine du domaine.
- **Private DNS zone** — zone DNS visible **seulement** des VNets liés. *`contoso.internal`*
- **Auto-registration** — les VMs d'un VNet **s'enregistrent seules** dans la zone privée.

### Load balancing
- **Azure Load Balancer** — répartit le trafic **TCP/UDP** (couche 4) dans une région.
- **Frontend IP** — l'adresse que **voient les clients**.
- **Backend pool** — les **machines** qui reçoivent le trafic.
- **Health probe** — test **régulier** qui vérifie qu'un serveur répond ; s'il échoue, le trafic n'est plus envoyé.
- **Load balancing rule** — lie un port frontend à un port backend.
- **Inbound NAT rule** — redirige un port précis vers **une VM** donnée.
- **Session persistence** — renvoyer un même client vers **la même VM**. *Client IP = 2-tuple.*
- **HA ports** — équilibre **tous les ports** à la fois, sur un LB interne.
- **Application Gateway** — load balancer **HTTP/HTTPS** (couche 7) régional, avec routage par URL et WAF.
- **Listener** — composant qui **écoute** une adresse, un port et un nom d'hôte.
- **WAF** (Web Application Firewall) — protège contre les **attaques web** (mode Detection ou Prevention).
- **SSL termination** — le déchiffrement TLS se fait **sur la passerelle**.
- **Traffic Manager** — répartition **par DNS**, globale, sans passer par le flux de données.
- **Azure Front Door** — service **global** HTTP/HTTPS avec CDN et WAF.

### Diagnostic
- **Network Watcher** — ensemble d'**outils de diagnostic** réseau, activé par région.
- **IP flow verify** — indique si un paquet est **autorisé ou bloqué** par un NSG, et par quelle règle.
- **Next hop (outil)** — indique **où part** le trafic d'une VM.
- **Connection troubleshoot** — test de connectivité **ponctuel**.
- **Packet capture** — **capture de paquets** sur une VM.
- **Flow logs** — **journaux de trafic** réseau.

---

## 6. Surveillance et sauvegarde

### Azure Monitor
- **Azure Monitor** — plateforme de **collecte, analyse et alerte** sur les ressources Azure.
- **Metrics** — valeurs **numériques** en quasi temps réel, conservées **93 jours**. *Percentage CPU.*
- **Logs** — **événements** stockés dans un workspace et interrogés avec KQL.
- **Log Analytics workspace** — **base de données** de logs d'Azure Monitor.
- **KQL** (Kusto Query Language) — langage de **requête** des logs. *`where`, `summarize`, `render`.*
- **Activity log** — journal **automatique** des opérations de gestion d'une subscription (*qui a fait quoi*), conservé **90 jours**.
- **Resource logs** — journaux **internes** d'une ressource, collectés seulement avec un **diagnostic setting**.
- **Diagnostic setting** — envoie les logs et métriques d'une ressource vers un **workspace, un storage account ou un Event Hub**.
- **Event Hub** — service de **streaming** de données, utilisé pour envoyer les logs vers un outil tiers.
- **AMA** (Azure Monitor Agent) — **agent actuel** qui collecte les données **à l'intérieur** de la VM.
- **DCR** (Data Collection Rule) — règle qui définit **quoi collecter** et **où l'envoyer**.
- **Boot diagnostics** — capture d'écran et journal série pour **dépanner un démarrage**.
- **VM insights** — supervision clé en main des VMs (performances et carte des dépendances).
- **Storage insights** — supervision clé en main des comptes de stockage.
- **Network insights** — supervision clé en main des ressources réseau.
- **Application Insights** — supervision des **applications** (requêtes, exceptions).

### Alertes
- **Alert rule** — **scope + condition + action group**.
- **Metric alert** — alerte sur une **métrique**, quasi temps réel.
- **Log search alert** — alerte sur le résultat d'une **requête KQL**.
- **Activity log alert** — alerte sur un **événement** de gestion. *Suppression d'une VM.*
- **Action group** — liste **réutilisable** de notifications et d'actions (e-mail, SMS, webhook, Logic App, Function).
- **Alert processing rule** — modifie **ce qui se passe** quand une alerte se déclenche (supprimer les notifications, ajouter des action groups), sans modifier la règle.
- **Severity** — gravité de l'alerte, de **Sev 0** (critique) à **Sev 4** (information).
- **Connection Monitor** — surveillance **continue** de la connectivité entre deux points.
- **Service Health / Resource Health** — état des **services Azure** / état d'**une de tes ressources**.

### Azure Backup
- **Azure Backup** — service de **sauvegarde** : protège les **données** contre la suppression ou la corruption.
- **Recovery Services vault** — coffre pour **Azure Backup** (VMs, Files, SQL en VM, MARS, DPM) **et** Site Recovery.
- **Backup vault** — coffre **Azure Backup** pour les workloads plus récents (disques, blobs, PostgreSQL, AKS).
- **Backup policy** — définit **quand** sauvegarder et **combien de temps** conserver.
- **Retention** — durée de conservation : quotidienne, hebdomadaire, mensuelle, annuelle.
- **Instant Restore** — snapshots gardés **localement** pour une restauration rapide (1 à 5 jours).
- **Application-consistent** — sauvegarde qui **respecte l'état des applications** (via VSS sous Windows).
- **Crash-consistent** — sauvegarde équivalente à une **coupure de courant** : les données sont là, sans garantie applicative.
- **Restore : Create new VM** — crée une **nouvelle VM** depuis un point de restauration.
- **Restore : Restore disks** — restaure les **disques** ; tu crées la VM toi-même.
- **Restore : Replace existing** — **remplace les disques** de la VM actuelle.
- **File recovery** — monte un point de restauration pour **récupérer des fichiers** seuls.
- **Cross Region Restore (CRR)** — restaure dans la **région pair** (vault en GRS).
- **Soft delete (vault)** — conserve les sauvegardes supprimées **quelques jours**.
- **Immutable vault** — empêche la **suppression ou la réduction** des sauvegardes.
- **MARS agent** — agent Windows qui sauvegarde des **fichiers et dossiers on-premises** ; garde sa **passphrase**.
- **DPM / MABS** — serveurs de sauvegarde pour protéger des workloads **on-premises**.
- **Backup reports** — rapports fondés sur Log Analytics (taux de succès, utilisation).
- **Business Continuity Center** — vue **centralisée** de la protection (backup et DR).

### Azure Site Recovery (ASR)
- **Azure Site Recovery** — service de **reprise après sinistre** : réplique des VMs vers une autre région pour les redémarrer en cas de panne.
- **Replication policy** — durée de rétention des points de récupération et fréquence des snapshots app-consistent.
- **Mobility service** — extension installée sur la VM pour la **répliquer**.
- **Cache storage account** — compte **tampon** dans la région source, avant l'envoi vers la cible.
- **Network mapping** — correspondance entre les réseaux **source et cible**.
- **Recovery plan** — **plan d'orchestration** : ordre de démarrage des VMs, scripts, actions manuelles.
- **Recovery point** — **état de la VM** à un instant précis, utilisable pour le failover.
- **Test failover** — essai de basculement dans un réseau **isolé**, **sans impact** sur la production.
- **Failover** — **bascule réelle** vers la région cible.
- **Commit** — **valide** le failover.
- **Re-protect** — **inverse la réplication** (cible vers source) après un failover.
- **Failback** — **retour** vers la région d'origine.

---

## Distinctions à ne pas confondre

| Notion A | Notion B | Différence en une phrase |
|---|---|---|
| **RBAC** | **Policy** | RBAC dit **qui** peut agir ; Policy dit **quelles ressources** sont autorisées. |
| **Policy** | **Lock** | Policy contrôle la **conformité** ; Lock empêche la **suppression ou la modification**. |
| **Azure Files** | **Azure File Sync** | Files **stocke** les fichiers dans le cloud ; File Sync les **synchronise** avec un serveur on-premises. |
| **Service endpoint** | **Private endpoint** | Endpoint de service : le subnet accède à l'**IP publique** du service ; Private endpoint : le service a une **IP privée** chez toi. |
| **Backup** | **Site Recovery** | Backup protège les **données** ; Site Recovery protège la **disponibilité**. |
| **Metrics** | **Logs** | Metrics = **chiffres** en temps réel ; Logs = **événements** détaillés interrogeables. |
| **Incremental** | **Complete** | Incremental **n'efface rien** ; Complete **supprime** ce qui n'est pas dans le template. |
| **Scale up** | **Scale out** | Up = **plus gros** ; Out = **plus nombreux**. |
| **Availability set** | **Availability Zone** | Availability set protège des pannes de **rack** ; la zone protège d'un **datacenter entier**. |
| **Durabilité** | **Disponibilité** | Durabilité = les données **existent encore** ; Disponibilité = j'y **accède maintenant**. |
| **Synchrone** | **Asynchrone** | Synchrone attend **toutes les copies** ; asynchrone confirme **avant** la copie lointaine. |
| **VNet integration** | **Private endpoint** (App Service) | VNet integration = l'app **sort** vers le réseau privé ; Private endpoint = on **entre** dans l'app en privé. |
| **Connection troubleshoot** | **Connection Monitor** | L'un est un test **ponctuel**, l'autre une surveillance **continue**. |
