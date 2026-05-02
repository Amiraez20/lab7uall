# 🔐 Lab : Analyse Dynamique Android avec MobSF & DIVA
### Environnement : Windows 11 + Android Studio + Docker Desktop

> **Auteur :** EZBIRI Amira  
> **Contexte :** Lab de sécurité mobile — Analyse runtime d'une APK vulnérable  
> **Outil principal :** MobSF (Mobile Security Framework) en mode dynamique  
> **Cible :** DIVA APK (Damn Insecure and Vulnerable Android App)

---

## 🎯 Objectifs du Lab

Ce laboratoire a pour but de maîtriser l'analyse dynamique (runtime) d'une application Android en conditions réelles simulées. À la fin de ce lab, tu seras capable de :

- Créer et configurer un émulateur AVD propre (sans Play Store) via Android Studio
- Déployer MobSF avec Docker et le connecter à l'émulateur via ADB
- Lancer une session d'analyse dynamique sur l'APK DIVA
- Intercepter le trafic HTTPS, lire les logs Android en direct et utiliser Frida pour hooker des méthodes Java
- Identifier les principales classes de vulnérabilités mobiles : stockage non sécurisé, intents exposés, secrets codés en dur, etc.

---

## 🧠 Pourquoi un AVD sans Play Store ?

L'image Android **sans Play Store** est privilégiée pour les raisons suivantes :

| Critère | Image sans Play Store | Image avec Play Store |
|---|---|---|
| Bruit réseau | ✅ Aucun | ❌ Services Google en fond |
| Compatibilité proxy MobSF | ✅ Totale | ⚠️ Conflits possibles |
| Accès root / Frida | ✅ Natif | ❌ Restreint |
| Analyse des logs | ✅ Propre | ❌ Polluée |
| API recommandée | API 29 / 30 | N/A |

> ⚠️ MobSF ne supporte pas les API > 30 car `/system` n'est plus accessible en écriture sur les versions récentes d'Android.

---

## 🖥️ Prérequis (Windows)

Avant de commencer, vérifier que les éléments suivants sont installés et fonctionnels :

- [x] **Android Studio** (dernière version stable) + SDK Platform-Tools (ADB inclus)
- [x] **Docker Desktop** pour Windows (WSL2 backend recommandé)
- [x] **Git for Windows**
- [x] Minimum **8 Go de RAM** + CPU 64-bit avec virtualisation activée (VT-x dans le BIOS)
- [x] Connexion internet active

> 💡 **Tip Windows :** Vérifier que la virtualisation est activée dans le BIOS/UEFI. Sur Intel : VT-x, sur AMD : AMD-V. Docker Desktop affiche une alerte si ce n'est pas le cas.

---

## 📋 Étapes du Lab

---

### Étape 1 — Création de l'AVD sans Play Store

**Durée estimée : 10 à 15 minutes**

1. Ouvrir **Android Studio** → menu `Tools` → `Device Manager` (ou `AVD Manager` selon la version)
2. Cliquer sur **Create Virtual Device**
3. Choisir un profil matériel : recommandé **Pixel 4** ou **Pixel 5**
4. Dans l'écran **System Image** :
   - Aller sur l'onglet `x86 Images`
   - Sélectionner une image **Android API 29 ou 30**, architecture **x86_64**
   - ⚠️ **IMPORTANT :** Choisir une image **sans mention "Google Play"** — la mention "Google APIs" seule est acceptable
   - Cliquer sur `Download` si l'image n'est pas encore présente, puis `Next`
5. Nommer l'AVD : `MobSF_Lab_API30` (ou un nom de ton choix)
6. Cliquer sur `Finish`

<img width="959" height="499" alt="AVDCREATION1" src="https://github.com/user-attachments/assets/6ba5d3f5-b1f5-4cc7-ae0c-7c81ca82ae3b" />

<img width="677" height="506" alt="AVDCREATION2" src="https://github.com/user-attachments/assets/204c3332-0e96-47ae-841d-19e7029296a3" />

<img width="680" height="502" alt="AVDCREATION3" src="https://github.com/user-attachments/assets/d8a9e86d-1304-476c-a1f4-f9dcd6c29c9e" />

<img width="674" height="497" alt="AVDCREATION4" src="https://github.com/user-attachments/assets/c6dbe0e8-1331-4a86-84ea-27ea8fff6739" />

**Vérification :** Dans le Device Manager, l'AVD ne doit pas afficher l'icône Play Store.

---

### Étape 2 — Clonage du dépôt MobSF

Ouvrir **PowerShell** ou **Git Bash** et exécuter :

```powershell
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
```
<img width="586" height="156" alt="Clonage" src="https://github.com/user-attachments/assets/18a2b2eb-86d6-49e8-99b9-bc91ca1cf881" />

<img width="239" height="25" alt="MOBSF" src="https://github.com/user-attachments/assets/014781be-fef1-4496-9ad0-b23d96e7efcd" />

Ce dépôt contient les scripts officiels de lancement d'émulateur optimisés pour MobSF.

---

### Étape 3 — Démarrage de l'émulateur via le script MobSF

Toujours dans le dossier cloné, sous **PowerShell** :

```powershell
.\scripts\start_avd.ps1
```

<img width="482" height="109" alt="MOBSF" src="https://github.com/user-attachments/assets/6fdc8ae6-e2ef-44b8-95f1-3316093102f8" />


Le script affiche la liste de vos AVD disponibles. Sélectionner `MobSF_Lab_API30`.

> 🔄 Le premier démarrage peut prendre 60 à 90 secondes.

<img width="959" height="503" alt="commandstarting" src="https://github.com/user-attachments/assets/6c783628-9247-4aec-a1a3-ffabdb0d5dcf" />

<img width="959" height="505" alt="Startingeml" src="https://github.com/user-attachments/assets/289af636-570b-4cbd-9f37-5bdadbafb912" />


**Vérification ADB :** Ouvrir un second terminal et taper :

```powershell
adb devices
```

<img width="959" height="502" alt="adbdevices" src="https://github.com/user-attachments/assets/1da157f7-5c2a-4b18-921f-40e8f017d236" />

Résultat :
```
List of devices attached
emulator-5554   device
```

> 📝 **Noter l'identifiant** (`emulator-5554`) — il sera utilisé dans la commande Docker.

---

### Étape 4 — Lancement de MobSF via Docker

⚠️ **L'émulateur doit être démarré avant MobSF.**

Dans un nouveau terminal PowerShell :

```powershell
docker pull opensecurity/mobile-security-framework-mobsf:latest
```

<img width="874" height="434" alt="pulldocker" src="https://github.com/user-attachments/assets/0ac904a6-11f3-4a99-b24c-9be60d18b8f9" />


Puis lancer MobSF en passant l'identifiant de l'émulateur :

```powershell
docker run -it --rm `
  -p 8000:8000 `
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 `
  opensecurity/mobile-security-framework-mobsf:latest
```

<img width="1798" height="890" alt="lancer MobSF" src="https://github.com/user-attachments/assets/5cb8cd1c-92e0-4018-96d9-7f2576c5f162" />


Une fois MobSF démarré, ouvrir le navigateur à l'adresse :

```
http://127.0.0.1:8000
```

<img width="959" height="475" alt="Interfacegraphique" src="https://github.com/user-attachments/assets/c1d178c4-01fb-4798-aa5f-d68cfbbd474b" />


**Identifiants par défaut :**
- Login : `mobsf`
- Mot de passe : `mobsf`

---

<img width="959" height="468" alt="mobsfinterface" src="https://github.com/user-attachments/assets/92a3b8ad-fa6b-4034-a17a-00294b90ef64" />


### Étape 5 — Obtention de l'APK DIVA

DIVA (Damn Insecure and Vulnerable Android App) est une application intentionnellement vulnérable conçue pour l'apprentissage de la sécurité mobile.

**Sources officielles :**
- Site Payatu : `http://www.payatu.com/damn-insecure-and-vulnerable-app/`
- GitHub : `https://github.com/payatu/diva-android`

Télécharger le fichier `diva-beta.apk` (ou compiler depuis les sources si tu veux explorer le code).

---

### Étape 6 — Analyse Statique dans MobSF

1. Dans MobSF → cliquer sur **Upload & Analyze**
3. Glisser-déposer ou sélectionner `diva-beta.apk`

<img width="959" height="476" alt="uploadandanalyze" src="https://github.com/user-attachments/assets/78b4f693-33e9-49e9-8385-264ba416d694" />


5. Attendre la fin de l'analyse statique (manifest, permissions, code décompilé, etc.)

<img width="959" height="473" alt="resultat" src="https://github.com/user-attachments/assets/8f6c1a2a-4df1-4a6d-981d-8753b1946acf" />


**Ce que MobSF analyse automatiquement :**
- Le fichier `AndroidManifest.xml` (permissions, activités exportées, services)
- Le code Smali / Java décompilé (détection de patterns dangereux)
- Les bibliothèques natives (.so)
- Les certificats de signature

<img width="959" height="476" alt="resultat2" src="https://github.com/user-attachments/assets/631ca1d9-0bbb-45dd-8dc2-bfbe1087c3ad" />

<img width="959" height="475" alt="resultat3" src="https://github.com/user-attachments/assets/b2e66aef-083b-4639-bf47-bc4b564580a5" />


---

### Étape 7 — Lancement de l'Analyse Dynamique

Depuis le rapport statique de DIVA, cliquer sur le bouton **Dynamic Analyzer** (ou **Start Dynamic Analysis**).

MobSF effectue automatiquement les actions suivantes :
1. Installation de DIVA sur l'émulateur
2. Démarrage du serveur Frida
3. Configuration du proxy HTTPS global sur l'émulateur
4. Ouverture du tableau de bord Dynamic Analyzer

<img width="959" height="418" alt="dynamicanalyzer" src="https://github.com/user-attachments/assets/81ea858b-da65-4c50-b07f-37e303c35849" />

<img width="957" height="413" alt="STARTDYNAMIC" src="https://github.com/user-attachments/assets/bf64f441-a791-4a9d-baeb-4c9f9590c48f" />

---

## 🔧 Référence du Menu Dynamic Analyzer

| Bouton | Rôle | Quand l'utiliser |
|---|---|---|
| **Stop Screen** | Arrêter le mirroring de l'écran | Libérer des ressources |
| **Remove Root CA** | Supprimer le certificat MobSF | Après les tests HTTPS |
| **Unset HTTP(S) Proxy** | Désactiver le proxy réseau | Fin de session |
| **TLS/SSL Security Tester** | Auditer la validation des certificats | Tester le certificate pinning |
| **Exported Activity Tester** | Lancer les activités exportées | Tester l'access control |
| **Activity Tester** | Parcourir toutes les activités | Cartographier l'app |
| **Get Dependencies** | Lister les bibliothèques | Identifier les CVE connues |
| **Take a Screenshot** | Capturer l'écran | Documentation |
| **Logcat Stream** | Logs Android en temps réel | Détecter les fuites |
| **Generate Report** | Rapport final | Fin d'analyse |

---

## 🚨 Dépannage Windows

### Problème : "Dynamic Analysis Failed"
**Cause :** MobSF lancé avant l'émulateur, ou mauvais identifiant ADB  
**Solution :**
```powershell
adb devices  # Vérifier l'identifiant exact
# Relancer Docker avec le bon MOBSF_ANALYZER_IDENTIFIER
```

### Problème : Docker ne voit pas l'émulateur
**Cause :** Isolation réseau entre Docker et l'hôte Windows  
**Solution :** Utiliser l'adresse IP de l'hôte Windows depuis le conteneur

```powershell
# Trouver l'IP de l'interface vEthernet (WSL)
ipconfig | Select-String "WSL" -A 5

# Relancer avec --add-host
docker run -it --rm `
  -p 8000:8000 `
  --add-host=host.docker.internal:host-gateway `
  -e MOBSF_ANALYZER_IDENTIFIER=emulator-5554 `
  opensecurity/mobile-security-framework-mobsf:latest
```

### Problème : Émulateur très lent
**Cause :** Pas d'accélération matérielle ou image ARM  
**Solution :** Utiliser uniquement des images x86_64, vérifier que HAXM (Intel) ou WHPX (Windows) est activé dans Android Studio

### Problème : ADB ne détecte pas l'émulateur
```powershell
adb kill-server
adb start-server
adb devices
```

---

## 📚 Ressources

- Documentation MobSF : https://github.com/MobSF/docs
- Dépôt DIVA Android : https://github.com/payatu/diva-android
- OWASP Mobile Security Testing Guide (MASTG) : https://owasp.org/www-project-mobile-app-security/
