🔴 Audit de Sécurité Active Directory — Red Teaming
> 🔧 Work in Progress
Projet académique d'audit de sécurité Active Directory basé sur le lab GOAD d'Orange Cyberdefense.  
Objectif : Devenir Domain Admin via des techniques d'attaque réelles sur un environnement Windows multi-domaine.
---
---
## 🎯 Objectifs

- Énumérer et cartographier un domaine Active Directory
- Exploiter les vulnérabilités Kerberos (Kerberoasting, AS-REP Roasting)
- Effectuer des mouvements latéraux (Pass-the-Hash, Pass-the-Ticket)
- Atteindre le niveau Domain Admin via DCSync
- Proposer des recommandations de sécurité (Tiered Admin Model)
---
---
🏗️ Environnement
Lab utilisé : GOAD Mini Lab — Orange Cyberdefense  
🔗 https://github.com/Orange-Cyberdefense/GOAD
```

- chemin d'architacture 


```
Domaine AD : `mini.lab`  
Chemin du lab :
```bash
cd ~/GOAD/workspace/1bdda2-minilab-virtualbox/provider
vagrant up
```
---
## 🛠️ Outils utilisés

| Outil | Rôle |
|-------|------|
| WSL1 (Ubuntu) | Environnement Linux sur Windows |
| Vagrant | Déploiement des VMs Windows |
| VirtualBox | Hyperviseur |
| Ansible (.venv) | Configuration automatique du lab AD |
| Kali Linux | Machine attaquante |
| BloodHound | Énumération et cartographie AD |
| SharpHound | Collecte des données AD (depuis Windows) |
| Impacket | Exploitation (DCSync, Pass-the-Hash) |
| Rubeus | Kerberoasting / AS-REP Roasting / Pass-the-Ticket |
| Nmap | Scan réseau |
---
## 📋 Méthodologie (6 étapes)

| Étape | Description | Statut |
|-------|-------------|--------|
| 1. Reconnaissance | AD enumeration avec BloodHound | ✅ Fait |
| 2. Scan & Énumération | Kerberoasting / AS-REP Roasting / Delegation | ✅ Fait |
| 3. Analyse vulnérabilités | Delegation abuse exploits | 🔧 En cours |
| 4. Exploitation | Pass-the-Hash / Pass-the-Ticket | 🔧 En cours |
| 5. Post-Exploitation | NTDS exfiltration (DCSync) | ⏳ À faire |
| 6. Rapport | Tiered Admin Model | ⏳ À faire |

---
## ✅ Résultats obtenus
## 🔍 Étape 1 — Reconnaissance Active Directory

Après le déploiement du lab, BloodHound a été utilisé pour cartographier 
le domaine `MINI.LAB` et identifier les chemins d'attaque vers **Domain Admins**.

### Requêtes Cypher exécutées

**Lister tous les utilisateurs du domaine :**
```cypher
MATCH (n:User) RETURN n
```

**Trouver les chemins vers Domain Admins :**
```cypher
MATCH p=shortestPath((n:User)-[*1 ..]->
(m:Group {name:"DOMAIN ADMINS@MINI.LAB"})) RETURN p
```

### Résultats

**10 utilisateurs découverts** dans le domaine `MINI.LAB` :

| Utilisateur | Tier Zero |
|-------------|-----------|
| ADMINISTRATOR@MINI.LAB | ✅ |
| ALICE@MINI.LAB | ✅ |
| VAGRANT@MINI.LAB | ✅ |
| CAROL@MINI.LAB | ✅ |
| KRBTGT@MINI.LAB | ✅ |
| BOB@MINI.LAB | ❌ |
| DAVE@MINI.LAB | ❌ |
| GUEST@MINI.LAB | ❌ |
```
-capture 
```


### Chemins d'attaque identifiés

BloodHound a révélé les relations suivantes vers **DOMAIN ADMINS@MINI.LAB** :

- `ADMINISTRATOR` → **MemberOf** → Domain Admins
- `ALICE` → **MemberOf** → Domain Admins  
- `CAROL` → **WriteDacl** → Domain Admins ⚠️
- `VAGRANT` → **MemberOf** → Administrators → Domain Admins

> ⚠️ **CAROL** possède le droit `WriteDacl` sur Domain Admins — 
> vecteur d'escalade de privilèges critique.

```
capture 
```


---

---
## ⚠️ Problèmes rencontrés & Solutions

Problème 1 — Vagrant timeout au démarrage
- Symptôme : `Timed out while waiting for the machine to boot`  
- Cause : Windows Server 2019 met trop de temps à démarrer  
- Solution : Augmenter le timeout dans les fichiers Ansible :
```bash
sed -i 's/reboot_timeout: 900/reboot_timeout: 2000/g' ~/GOAD/ansible/roles/domain_controller/tasks/main.yml
sed -i 's/reboot_timeout: 600/reboot_timeout: 2000/g' ~/GOAD/ansible/roles/settings/hostname/tasks/main.yml
sed -i 's/reboot_timeout: 600/reboot_timeout: 2000/g' ~/GOAD/ansible/roles/settings/windows_defender/tasks/main.yml
```
---
Problème 2 — WSL2 ne communique pas avec VirtualBox
- Symptôme : `ping 192.168.56.30 → Destination Host Unreachable`  
- Cause : WSL2 a un réseau isolé (`172.17.x.x`) séparé de VirtualBox (`192.168.56.x`)  
- Solution : Migrer vers WSL1 :
```powershell
wsl --set-version Ubuntu 1
```
WSL1 partage le réseau Windows et peut voir les VMs VirtualBox directement.
---
Problème 3 — Service ADWS non démarré sur DC01
- Symptôme : `get-ADDomain` → commande non reconnue  
- ADWS = service Windows qui permet à PowerShell de communiquer avec l'AD  
- Solution :
```powershell
Start-Service ADWS
Import-Module ActiveDirectory
Get-ADDomain
```
---
Problème 4 — Ansible ne peut pas se connecter aux VMs (WinRM)
- Symptôme : `Connection refused port 5986`  
- WinRM = service qui permet à Ansible de se connecter aux machines Windows  
- Solution sur DC01 et WS01 :
```powershell
winrm quickconfig -force
$cert = New-SelfSignedCertificate -DnsName "DC01" -CertStoreLocation "cert:\LocalMachine\My"
$thumb = $cert.Thumbprint
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $thumb -Force
Stop-Service WinRM
Start-Service WinRM
netstat -an | findstr "5986"
# Résultat attendu : TCP 0.0.0.0:5986 LISTENING
```
---
Problème 5 — DC01 sans IP dans le réseau 192.168.56.x
- Symptôme : DC01 affiche `169.254.x.x` sur Ethernet 2  
- Solution :
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet 2" -IPAddress 192.168.56.30 -PrefixLength 24
```
---
Problème 6 — WS01 s'éteignait au lieu de redémarrer
- Symptôme : Ansible envoie un reboot → WS01 s'éteint complètement  
- Solution :
```powershell
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
```
---
---
Problème 7 — PowerShellGet bloqué sur WS01
- Symptôme : `The version of PackageManagement is currently in use`  
- Solution :
```powershell
Install-PackageProvider -Name NuGet -Force
Install-Module PowerShellGet -Force -AllowClobber -SkipPublisherCheck
```
---
Problème 8 — Kali ne voit pas DC01
- Symptôme : `ping 192.168.56.30 → no reply`  
- Solution :
```bash
sudo ip addr add 192.168.56.100/24 dev eth1
echo "nameserver 192.168.56.30" | sudo tee /etc/resolv.conf
echo "192.168.56.30 dc.mini.lab mini.lab" | sudo tee -a /etc/hosts
```
-Désactiver aussi le firewall sur DC01 :
```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```
---
📸 Screenshots
> 🔧 Coming soon — captures BloodHound, DCSync et résultats d'attaques à venir
---
📄 Rapport
> ⏳ À venir après complétion de toutes les étapes
---
👩‍💻 Auteur
Khouloud Bahri — Étudiante Ingénieure Cybersécurité @ TEK-UP  
📧 khouloud.bahri19@gmail.com
