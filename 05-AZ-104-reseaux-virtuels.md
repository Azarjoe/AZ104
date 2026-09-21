# AZ-104 — Fiche 5/6 · Configurer et gérer des réseaux virtuels

> Parcours MS Learn : *AZ-104: Configure and manage virtual networks for Azure administrators*
> Modules : **Configure virtual networks** · **Configure network security groups** · **Host your domain on Azure DNS** · **Configure Azure Virtual Network peering** · **Manage and control traffic flow with routes** · **Introduction to Azure Load Balancer** · **Introduction to Azure Application Gateway** · **Introduction to Azure Network Watcher**
> Poids à l'examen : **15–20 %**

---

## 1. Virtual Network (VNet) & sous-réseaux

- **VNet** = ton réseau privé dans Azure. **Régional** (couvre toutes les AZ de la région), **gratuit**, isolé par défaut.
- **Address space** en **CIDR**, plages **privées RFC 1918** recommandées : `10.0.0.0/8` · `172.16.0.0/12` · `192.168.0.0/16`.
- On peut **ajouter** des address spaces plus tard. **Pas de chevauchement** entre subnets (ni entre VNets à peerer ou à relier en VPN).
- Un VNet ne traverse **pas** les régions ni les subscriptions (→ **peering** / VPN pour relier).

### Plan d'adressage — exemple concret (Contoso)

| Subnet | CIDR | Rôle |
|---|---|---|
| VNet | `10.10.0.0/16` | Espace global |
| `snet-web` | `10.10.1.0/24` | Serveurs web |
| `snet-app` | `10.10.2.0/24` | Applications |
| `snet-db` | `10.10.3.0/24` | Bases de données |
| `AzureBastionSubnet` | `10.10.250.0/26` | **Bastion** (nom imposé, **/26 min**) |
| `GatewaySubnet` | `10.10.255.0/27` | **VPN / ExpressRoute gateway** (nom imposé) |

### Azure réserve 5 adresses par subnet ⭐
`x.x.x.0` (réseau) · `x.x.x.1` (**default gateway**) · `x.x.x.2` et `x.x.x.3` (**DNS Azure**) · dernière (**broadcast**).

| CIDR | Adresses | **Utilisables** (−5) |
|---|---|---|
| **/29** (plus petit subnet) | 8 | **3** |
| /28 | 16 | 11 |
| /27 | 32 | 27 |
| /26 | 64 | 59 |
| /25 | 128 | 123 |
| **/24** | 256 | **251** |

- **Subnets spéciaux** (noms **exacts**) : `GatewaySubnet`, `AzureBastionSubnet`, `AzureFirewallSubnet` (/26), `AzureFirewallManagementSubnet`.
- **Subnet delegation** : réserve un subnet à un service PaaS (ex. `Microsoft.Web/serverFarms`, `Microsoft.ContainerInstance/containerGroups`).
- Une **NIC** appartient à **1 subnet** ; une VM peut avoir **plusieurs NIC** (selon sa taille).

### IP publiques

| | **Standard** (à utiliser) | Basic (retirée) |
|---|---|---|
| Allocation | **Static** uniquement | Static / Dynamic |
| Zones | **Zone-redundant** / zonal / non-zonal | Non |
| Sécurité | **Fermée par défaut** (il faut une règle **NSG** pour autoriser) | Ouverte par défaut |
| Compatible avec | **Standard** Load Balancer, NAT Gateway, Bastion, App GW v2 | Basic LB |

- **IP privée** : *Dynamic* (DHCP Azure, défaut) ou *Static* (réservée dans le subnet).
- **Public IP prefix** : bloc contigu d'IP publiques.
- **NAT Gateway** : **sortie Internet** partagée et prévisible (SNAT) pour un subnet.
- ℹ️ Les **Basic** SKUs (IP publique, Load Balancer) sont **retirés** ; l'**accès sortant par défaut** est en cours de retrait → prévoir une **sortie explicite** (NAT Gateway, LB, IP publique).

---

## 2. Network Security Groups (NSG) & ASG

### Principe
- **Pare-feu L3/L4 stateful** : règles **Allow/Deny** filtrées par **5-tuple** : *source, port source, destination, port destination, protocole*.
- Associé à un **subnet** et/ou à une **NIC**.
- **Priorité 100–4096** : le **plus petit numéro** est évalué **en premier** ; la 1re règle qui matche **s'applique** (stop).
- **Stateful** : si l'entrant est autorisé, la **réponse** l'est aussi (pas besoin de règle retour).

### Règles par défaut (non supprimables, mais **surchargeables** par priorité plus basse)

| Direction | Nom | Priorité | Effet |
|---|---|---|---|
| Inbound | `AllowVNetInBound` | 65000 | Autorise VNet → VNet |
| Inbound | `AllowAzureLoadBalancerInBound` | 65001 | Autorise le **probe** du LB |
| Inbound | `DenyAllInBound` | 65500 | **Refuse tout le reste** |
| Outbound | `AllowVnetOutBound` | 65000 | Autorise VNet |
| Outbound | `AllowInternetOutBound` | 65001 | Autorise Internet |
| Outbound | `DenyAllOutBound` | 65500 | Refuse le reste |

### Ordre d'évaluation (⭐ NSG subnet + NSG NIC)
```
INBOUND :   Internet → [NSG du SUBNET] → [NSG de la NIC] → VM
OUTBOUND :  VM → [NSG de la NIC] → [NSG du SUBNET] → Internet
```
- Le trafic doit être **autorisé par les deux** NSG (s'il y en a deux).
- **Effective security rules** (NIC → *Effective security rules*, ou **Network Watcher**) = **vue fusionnée** de toutes les règles applicables → **le bon outil pour diagnostiquer**.

### Service tags & ASG
- **Service tag** : groupe d'IP gérés par Microsoft : `Internet`, `VirtualNetwork`, `AzureLoadBalancer`, `Storage`, `Sql`, `AzureActiveDirectory`…
- **ASG (Application Security Group)** : regrouper des **NICs par rôle** (ex. `asg-web`, `asg-db`) et écrire des règles **par rôle** plutôt que par IP.
  - Ex. règle : *Allow TCP 1433 de `asg-app` vers `asg-db`*. Les VMs d'un ASG doivent être dans le **même VNet**.

### Exemple de règles (3 tiers)

| Prio | Nom | Source | Dest | Port | Action |
|---|---|---|---|---|---|
| 100 | Allow-HTTPS-Internet | Internet | asg-web | 443 | **Allow** |
| 110 | Allow-App | asg-web | asg-app | 8080 | **Allow** |
| 120 | Allow-SQL | asg-app | asg-db | 1433 | **Allow** |
| 4000 | Deny-RDP-Internet | Internet | * | 3389 | **Deny** |

---

## 3. VNet Peering

- Relie **2 VNets** via le **backbone Microsoft** (privé, faible latence, **haut débit**).
- **Régional** (même région) ou **Global** (régions différentes) ; même/différente **subscription** ou **tenant**.
- Exigences : **address spaces sans chevauchement** · peering **dans les 2 sens** (le portail crée les 2) · statut **Connected**.
- ⚠️ **NON transitif** : A↔B et B↔C **≠** A↔C.

```
Spoke1 ⇄ Hub ⇄ Spoke2    →  Spoke1 ne parle PAS à Spoke2 sans :
                              • peering direct Spoke1⇄Spoke2, ou
                              • hub avec Azure Firewall / NVA + UDR, ou
                              • gateway transit / Virtual WAN
```

| Option | Sens |
|---|---|
| **Allow virtual network access** | Communication entre les 2 VNets (par défaut ON) |
| **Allow forwarded traffic** | Accepte le trafic **transféré** (ex. via un NVA) |
| **Allow gateway transit** | (côté **hub**, qui a la gateway) le VNet **partage** sa VPN/ER gateway |
| **Use remote gateways** | (côté **spoke**) utilise la gateway du **hub** |

- Ajouter un address space à un VNet peeré → **resynchroniser** le peering.
- Peering vs VPN : peering = **moins cher, plus rapide, pas de gateway** ; VPN Gateway VNet-to-VNet = chiffré IPsec, supporte des cas particuliers.

### Options de connectivité (aperçu)

| Solution | Relie | Via | Notes |
|---|---|---|---|
| **VNet peering** | Azure ↔ Azure | Backbone Microsoft | Non transitif |
| **VPN Gateway – Site-to-Site** | On-prem ↔ Azure | **Internet** (IPsec/IKE) | Nécessite `GatewaySubnet`, gateway **route-based** |
| **VPN Gateway – Point-to-Site** | **Poste individuel** ↔ Azure | Internet (OpenVPN, IKEv2, SSTP) | Télétravail |
| **ExpressRoute** | On-prem ↔ Azure/M365 | **Lien privé** via opérateur (pas Internet) | Débit/latence prévisibles |
| **Virtual WAN** | Hub managé multi-sites | Mix | Grande échelle |

### Alternatives au peering : relier des VNets autrement

> ℹ️ Les **skills measured** ne testent que le **peering** (+ gateway transit, UDR). VPN Gateway, Virtual WAN et Virtual Network Manager sont du **contexte** : à connaître de façon générale, sans les détails.

| Solution | Relie | Trafic | À retenir |
|---|---|---|---|
| **VNet peering** | VNet ↔ VNet | Backbone Microsoft (privé) | Simple, rapide, sans gateway, **non transitif** |
| **VPN Gateway (VNet-to-VNet)** | VNet ↔ VNet | Tunnel **IPsec** chiffré | Une **gateway par VNet** (`GatewaySubnet`), débit limité par le **SKU** |
| **Hub-and-spoke + Azure Firewall / NVA + UDR** | Spoke ↔ Spoke **via un hub** | Peerings + routage forcé | Rend le peering **"transitif"** et permet de **filtrer** entre spokes |
| **Virtual WAN** | Nombreux VNets **et** sites on-prem | Hub **géré** par Microsoft | Routage **transitif inclus**, pour les grandes topologies |
| **Azure Virtual Network Manager** | Groupes de VNets | Peerings **gérés de façon centralisée** | **Automatise** la topologie (hub-and-spoke ou mesh) et des règles de sécurité à grande échelle |
| **ExpressRoute** | On-prem ↔ Azure | Lien privé opérateur | Relie ton **datacenter**, pas deux VNets entre eux |

#### Peering vs VPN Gateway VNet-to-VNet (comparaison la plus probable)

| | **Peering** | **VPN Gateway VNet-to-VNet** |
|---|---|---|
| Gateway nécessaire | ❌ | ✅ (une par VNet) |
| Tunnel chiffré IPsec | ❌ (trafic privé sur le backbone Microsoft) | ✅ |
| Débit / latence | **Élevé / faible** | Limité par le **SKU** de la gateway |
| Coût | Plus faible (trafic facturé) | Plus élevé (**gateway facturée en continu**) |
| Mise en place | Rapide | Plus longue (déploiement de la gateway) |
| Address spaces qui se chevauchent | ❌ interdit | ❌ interdit |
| Inter-régions / subscriptions | ✅ | ✅ |

> **Repère :** peering **par défaut** ; VPN Gateway quand on **exige un tunnel chiffré** entre les VNets ou qu'une gateway existe déjà pour des sites on-prem.

#### Exemple concret : faire parler deux spokes via un hub

```
Spoke-Web (10.20.0.0/16) ⇄ Hub (10.10.0.0/16) ⇄ Spoke-DB (10.30.0.0/16)
                              │
                     Azure Firewall 10.10.100.4
```
- Le peering étant **non transitif**, `Spoke-Web` ne joint pas `Spoke-DB` tout seul.
- Solution : dans chaque spoke, une **route table** (UDR) qui envoie le trafic de l'autre spoke vers le firewall.

| Route table sur… | Prefix | Next hop type | Next hop address |
|---|---|---|---|
| Subnets de `Spoke-Web` | `10.30.0.0/16` | **Virtual appliance** | `10.10.100.4` |
| Subnets de `Spoke-DB` | `10.20.0.0/16` | **Virtual appliance** | `10.10.100.4` |

- Activer **Allow forwarded traffic** sur les peerings concernés.
- Le firewall du hub a aussi des **règles** qui autorisent ce trafic (bonus : tu **filtres** ce qui passe entre spokes).
- Alternative sans firewall : un **peering direct** Spoke-Web ⇄ Spoke-DB (simple, mais ça ne passe plus à l'échelle avec beaucoup de spokes).

#### Comment choisir

```
Relier deux VNets Azure ?
 ├─ Simple, sans chiffrement IPsec exigé ─────────────► Peering
 ├─ Tunnel chiffré IPsec exigé ───────────────────────► VPN Gateway VNet-to-VNet
 ├─ Spokes à faire communiquer via un hub (filtrage) ─► Hub + Azure Firewall/NVA + UDR
 ├─ Des dizaines de VNets + sites on-prem ────────────► Virtual WAN
 └─ Gérer centralement de nombreux peerings ──────────► Virtual Network Manager

Relier on-prem à Azure ?
 ├─ Via Internet, chiffré ────────────────────────────► VPN Gateway (Site-to-Site)
 └─ Lien privé, débit stable ─────────────────────────► ExpressRoute
```

**Ne remplacent pas le peering** : *service endpoint* et *private endpoint* (accès à un **service PaaS**), *Bastion* (accès RDP/SSH).

---

## 4. Routage (system routes & UDR)

- **System routes** : créées automatiquement (VNet local, Internet `0.0.0.0/0`, peering, etc.).
- **User-defined routes (UDR)** : dans une **route table** **associée à un subnet** → **remplace** les routes système.
- **Next hop types** : **Virtual network gateway** · **Virtual network** · **Internet** · **Virtual appliance** · **None** (= *black hole*).
- **Sélection de route** : **longest prefix match** (le plus spécifique gagne) ; à préfixe égal → **UDR > BGP > System**.

**Exemple : forcer tout le trafic Internet via un firewall**

| Route | Prefix | Next hop type | Next hop address |
|---|---|---|---|
| `to-internet-via-fw` | `0.0.0.0/0` | **Virtual appliance** | `10.10.100.4` (Azure Firewall / NVA) |

- NVA : activer **IP forwarding** sur la **NIC Azure** **et** dans l'**OS**.
- Une route table s'associe à **N subnets** ; un subnet n'a qu'**1** route table.
- Diagnostic : **Next hop** (Network Watcher) · **Effective routes** (NIC).

---

## 5. Accès sécurisé aux VNets

### Azure Bastion (⭐)
- **RDP/SSH via le navigateur** (TLS **443**) → **la VM n'a pas besoin d'IP publique**, pas de port 3389/22 exposé.
- Nécessite le subnet **`AzureBastionSubnet`** (**/26 minimum** pour Basic/Standard).
- **SKUs** : **Developer** (gratuit, 1 VM à la fois, sans subnet dédié) · **Basic** · **Standard** (client natif, **transfert de fichiers**, **IP-based connection**, liens partageables, scale units) · **Premium** (enregistrement de sessions, déploiement **privé**).
- Fonctionne aussi avec les VMs des **VNets peerés**.

### Service endpoints vs Private endpoints (⭐ classique)

| | **Service endpoint** | **Private endpoint** (Private Link) |
|---|---|---|
| Principe | Le **subnet** est autorisé à joindre le service PaaS | Le service reçoit une **NIC privée** (IP privée) dans **ton subnet** |
| Adresse du service | **IP publique** du service (route optimisée) | **IP privée** du VNet |
| Portée | **Tout le service** (ex. tous les Storage de la région) | **Une ressource précise** (+ sous-ressource : blob, file…) |
| Accès on-premises | ❌ (via VPN/ER pas supporté nativement) | ✅ (via VPN/ExpressRoute) |
| Désactiver l'accès public | ❌ (on filtre par firewall) | ✅ possible |
| DNS | Pas de changement | **Private DNS zone** requise (`privatelink.blob.core.windows.net`) |
| Coût | **Gratuit** | **Payant** |
| Config | Sur le **subnet** (`Microsoft.Storage`) + règle VNet côté service | Ressource dédiée + DNS |

> 🏦 Analogie : **service endpoint** = un badge qui ouvre **la porte de service** d'un immeuble public · **private endpoint** = **un bureau annexe privé** installé chez toi, relié à **un seul** service.

### NSG vs Azure Firewall

| | **NSG** | **Azure Firewall** |
|---|---|---|
| Niveau | L3/L4 | **L3–L7** (FQDN, URL, threat intelligence, TLS inspection en Premium) |
| Portée | Subnet / NIC | **VNet / hub** central |
| Coût | Gratuit | Payant (SKUs Basic / Standard / Premium) |
| Subnet | Existant | `AzureFirewallSubnet` (**/26**) |

---

## 6. Azure DNS

### Zones publiques
- **DNS zone** `contoso.com` hébergée dans Azure → Azure te donne **4 name servers** (`ns1-xx.azure-dns.com`, `.net`, `.org`, `.info`).
- **Étape clé** : chez ton **registrar**, **remplacer les NS** par ceux d'Azure (**délégation**). Azure DNS **n'est pas un registrar**.
- Enregistrements : **A**, **AAAA**, **CNAME**, **MX**, **NS**, **PTR**, **SOA**, **SRV**, **TXT**, **CAA** · **TTL** configurable.
- ⚠️ **CNAME impossible à l'apex** (`contoso.com`) → utiliser un **alias record**.
- **Alias record** : un A/AAAA/CNAME qui **pointe vers une ressource Azure** (IP publique, Traffic Manager, Front Door, CDN) → **suit** la ressource si l'IP change ; **fonctionne à l'apex**.

### Zones privées (résolution **interne**)
- **Private DNS zone** (ex. `contoso.internal`) **liée** à un ou plusieurs VNets (**virtual network links**).
- **Auto-registration** : les VMs du VNet s'enregistrent **automatiquement** (création/suppression d'enregistrements) — **1 seule** zone avec auto-registration par VNet.
- DNS **par défaut** Azure : `168.63.129.16` ; résout les noms **dans le même VNet**. Pour résoudre entre VNets peerés ou vers on-prem → **zone privée liée** ou **DNS custom / Azure DNS Private Resolver**.

| | **Zone publique** | **Zone privée** |
|---|---|---|
| Visible depuis | Internet | **VNets liés** uniquement |
| Usage | `www.contoso.com` | `sql01.contoso.internal` |
| Délégation | Registrar → NS Azure | Pas de registrar (liens VNet) |

---

## 7. Load balancing (⭐ le grand tableau de décision)

| Service | Couche | Portée | Protocoles | Fonctions clés |
|---|---|---|---|---|
| **Azure Load Balancer** | **L4** (TCP/UDP) | **Régional** (+ tier *global* cross-region) | Tout TCP/UDP | Ultra-rapide, **health probes**, zone-redundant |
| **Application Gateway** | **L7** | **Régional** | **HTTP/HTTPS**, WebSocket | **URL path routing**, multi-site, **SSL termination**, **WAF**, cookie affinity |
| **Traffic Manager** | **DNS** | **Global** | Tout (résolution DNS) | Routage **DNS** : **Priority, Weighted, Performance, Geographic, MultiValue, Subnet** |
| **Azure Front Door** | **L7** | **Global** | HTTP/HTTPS | Anycast, **CDN + WAF**, accélération |

**Arbre de décision**
```
Trafic HTTP(S) ?
 ├─ Oui ── Global ? ── Oui → Front Door
 │                └── Non → Application Gateway
 └─ Non (TCP/UDP) ── Global ? ── Oui → Traffic Manager (DNS) ou LB global
                              └── Non → Load Balancer
```
> Exemples : site e-commerce mondial avec WAF → **Front Door** · site régional avec `/images` et `/api` sur des pools distincts → **Application Gateway** · flotte de serveurs SQL/ports custom → **Load Balancer** · bascule **DNS** entre 2 régions → **Traffic Manager**.

### Azure Load Balancer — composants
- **Public** (IP publique) ou **Internal** (IP privée).
- **Frontend IP config** · **Backend pool** (NIC / IP de VMs / VMSS) · **Health probes** (**TCP, HTTP, HTTPS**) · **Load balancing rules** · **Inbound NAT rules** (port forwarding, ex. 50001 → 3389) · **Outbound rules** (SNAT).
- **Standard SKU** : **secure by default** (il faut un **NSG** qui autorise), **AZ**, **SLA 99,99 %**, backends dans **1 VNet**. (Basic retiré.)
- **Distribution modes** (**session persistence**) :

| Mode | Hash sur | Effet |
|---|---|---|
| **None** (défaut) | **5-tuple** (src IP/port, dst IP/port, protocole) | Répartition **équilibrée** ; chaque connexion peut aller ailleurs |
| **Client IP** | **2-tuple** (src IP + dst IP) | Même **client** → même VM |
| **Client IP and protocol** | **3-tuple** | Même client + protocole → même VM |

- **HA ports** : LB **interne** Standard équilibre **tous les ports** (souvent pour NVA).
- **Le probe vient de `168.63.129.16`** → le NSG doit **autoriser** `AzureLoadBalancer`.
- **Troubleshoot LB** : probe **échoue** (port/chemin/service arrêté) · **NSG/OS firewall** bloque le probe ou le trafic · **règle** mal configurée (port frontend/backend) · backend **pas dans le pool** · métriques **Health probe status** / **Data path availability**.

### Application Gateway — composants
- **Frontend IP** → **Listener** (basic ou **multi-site** par hostname) → **Routing rule** (basic ou **path-based**) → **HTTP settings** (port, protocole, cookie affinity) → **Backend pool** (VMs, VMSS, **App Service**, FQDN/IP) · **Health probes**.
- **Subnet dédié**. **v2** : **autoscale**, **zone-redundant**.
- **WAF** (OWASP) : modes **Detection** (log) / **Prevention** (bloque).
- **SSL termination** ou **end-to-end TLS** ; réécriture d'URL/headers ; redirection HTTP → HTTPS.

---

## 8. Network Watcher (diagnostic réseau)

- Activé **par région** (créé auto avec un VNet ; RG `NetworkWatcherRG`).

| Outil | Question à laquelle il répond |
|---|---|
| **IP flow verify** | "Ce **paquet** (5-tuple) est-il **autorisé ou bloqué** par un NSG, et par **quelle règle** ?" |
| **Next hop** | "Où part ce trafic ?" → **prochaine étape + route** appliquée |
| **Effective security rules** | Règles NSG **fusionnées** appliquées à une NIC |
| **Connection troubleshoot** | Test **ponctuel** de connectivité source → destination (latence, hops, blocage) |
| **Connection Monitor** | Surveillance **continue** de la connectivité (→ **fiche 6**) |
| **Packet capture** | Capture de paquets sur une VM (extension) |
| **VPN diagnostics** | Diagnostic des gateways/tunnels VPN |
| **Flow logs** (**VNet flow logs**, en remplacement progressif des NSG flow logs) | Journaux de trafic vers Storage / Log Analytics (**Traffic Analytics**) |
| **Topology / Network insights** | Vue graphique des ressources |

**Méthode de dépannage "ça ne se connecte pas"**
1. **NSG** ? → *IP flow verify* / *Effective security rules*
2. **Routage** ? → *Next hop* / *Effective routes* (UDR ?)
3. **Peering** en **Connected** ? address spaces OK ?
4. **DNS** résout-il le bon nom/IP ? (`nslookup`)
5. **Pare-feu de l'OS** et **service à l'écoute** sur le port ? (*Connection troubleshoot*)
6. **LB** : probe healthy ? **App GW** : backend health ?

---

## 9. Commandes utiles

```bash
az network vnet create -g rg -n vnet-contoso --address-prefixes 10.10.0.0/16 --subnet-name snet-web --subnet-prefixes 10.10.1.0/24
az network vnet subnet create -g rg --vnet-name vnet-contoso -n snet-app --address-prefixes 10.10.2.0/24
az network nsg create -g rg -n nsg-web
az network nsg rule create -g rg --nsg-name nsg-web -n Allow-HTTPS --priority 100 --access Allow --direction Inbound --protocol Tcp --destination-port-ranges 443
az network vnet peering create -g rg -n hub-to-spoke1 --vnet-name vnet-hub --remote-vnet vnet-spoke1 --allow-vnet-access
az network route-table create -g rg -n rt-spoke
az network route-table route create -g rg --route-table-name rt-spoke -n to-fw --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.10.100.4
az network dns zone create -g rg -n contoso.com
az network dns record-set a add-record -g rg -z contoso.com -n www -a 20.1.2.3
az network watcher test-ip-flow -g rg --vm vm-web-01 --direction Inbound --protocol TCP --local 10.10.1.4:443 --remote 203.0.113.10:50000
az network watcher show-next-hop -g rg --vm vm-web-01 --source-ip 10.10.1.4 --dest-ip 8.8.8.8
```

---

## 10. Pièges d'examen ⚠️

- "Combien d'IP utilisables dans un **/24**" → **251** (Azure réserve **5**).
- "Plus petit subnet possible" → **/29**.
- "Deux règles NSG en conflit" → la **priorité la plus basse (numéro le plus petit)** gagne.
- "Trafic entrant bloqué alors que la NIC autorise" → vérifier le **NSG du subnet** aussi.
- "A↔B, B↔C, A doit joindre C" → **peering non transitif** → peering A↔C / hub + firewall + UDR / gateway transit.
- "Relier 2 VNets **sans gateway**, faible latence" → **Peering** ; "tunnel **IPsec** entre 2 VNets" → **VPN Gateway VNet-to-VNet**.
- "Toute la sortie Internet via un firewall" → **UDR `0.0.0.0/0` → Virtual appliance** (+ IP forwarding).
- "Accéder aux VMs en RDP/SSH **sans IP publique**" → **Azure Bastion**.
- "Storage accessible **uniquement depuis mon VNet**, depuis **on-prem** aussi" → **Private endpoint** ; "gratuit, depuis un subnet" → **Service endpoint**.
- "Domaine racine vers une ressource Azure" → **Alias record** (pas de CNAME à l'apex).
- "Les VMs d'un VNet doivent se résoudre par nom" → **Private DNS zone** + **auto-registration**.
- "Routage par **URL** (`/images`, `/api`) ou **WAF**" → **Application Gateway**.
- "Répartir du trafic **DNS** entre régions" → **Traffic Manager**.
- "Répartition **TCP** ultra-rapide, non HTTP" → **Load Balancer**.
- "Même client toujours sur la même VM (LB)" → **Client IP** persistence.
- "Pourquoi le trafic est-il bloqué / quelle règle ?" → **IP flow verify** · "Où part le trafic ?" → **Next hop**.

## 11. Checklist finale ✅

- [ ] Je calcule les IP utilisables d'un subnet et je connais les subnets spéciaux
- [ ] Je sais lire/écrire des règles NSG (priorité, défaut, ordre subnet/NIC) + ASG
- [ ] Je configure un peering (2 sens, non transitif, gateway transit)
- [ ] Je sais créer un UDR (next hop virtual appliance) et l'associer à un subnet
- [ ] Je distingue service endpoint / private endpoint / Bastion
- [ ] Je sais déléguer une zone DNS, créer alias / private zone / auto-registration
- [ ] Je choisis LB / App GW / Traffic Manager / Front Door
- [ ] Je connais les outils Network Watcher et l'ordre de dépannage
