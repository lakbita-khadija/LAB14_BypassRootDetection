# LAB 14 : Bypass Root Detection sur Android

## Techniques Dynamiques avec Frida et Objection

**Application cible :** DIVA (jakhar.aseem.diva)  
**Plateforme :** Windows 11 + Émulateur Android Studio  
**Version Android :** 12 (API 31)  
**Architecture :** x86_64

---

## Table des matières

1. [Avertissement légal](#1-avertissement-légal)
2. [Préparation de l'environnement](#2-préparation-de-lenvironnement)
3. [Démarrage de frida-server](#3-démarrage-de-frida-server)
4. [Bypass Root avec Frida](#4-bypass-root-avec-frida)
5. [Bypass Root avec Objection](#5-bypass-root-avec-objection)
6. [Codes sources complets](#6-codes-sources-complets)
7. [Dépannage](#7-dépannage)


---

## 1. Avertissement légal

> ⚠️ **Ces techniques sont uniquement destinées à un usage éducatif et autorisé.**
>
> - Utilisez-les uniquement sur vos propres appareils ou avec une autorisation écrite.
> - Le contournement des mécanismes de sécurité peut violer les conditions d'utilisation.
> - L'auteur décline toute responsabilité en cas d'utilisation non autorisée.

---

## 2. Préparation de l'environnement

### 2.1 Vérification des outils

```powershell
# Vérifier Python
python --version

# Vérifier ADB
adb --version

# Vérifier Frida
frida --version
```
<img width="948" height="255" alt="Capture d&#39;écran 2026-05-20 175407" src="https://github.com/user-attachments/assets/4d560d0d-c84f-43f2-827b-01c32a7e35d8" />

**Configuration testée :**
- Python 3.11+
- Frida 17.9.10
- Android Emulator 5556 (x86_64)

### 2.2 Vérification de l'émulateur

```powershell
adb devices
```

**Résultat attendu :**
```text
List of devices attached
emulator-5556   device
```
<img width="960" height="151" alt="image" src="https://github.com/user-attachments/assets/1cf0d18c-f0ef-4d83-a840-da52999c072c" />

---

## 3. Démarrage de frida-server

### 3.1 Pousser frida-server sur l'émulateur

```powershell
adb -s emulator-5556 push frida-server /data/local/tmp/
adb -s emulator-5556 shell chmod 755 /data/local/tmp/frida-server
```

### 3.2 Lancer frida-server

```powershell
# Dans un terminal dédié (à laisser ouvert)
adb -s emulator-5556 shell "/data/local/tmp/frida-server -l 0.0.0.0"
```
<img width="951" height="158" alt="Capture d&#39;écran 2026-05-20 183657" src="https://github.com/user-attachments/assets/9a7c43d6-3ca8-442f-84a1-5d307029eb41" />

### 3.3 Vérifier que frida-server fonctionne

```powershell
# Dans un second terminal
frida-ps -Uai
```
<img width="944" height="437" alt="image" src="https://github.com/user-attachments/assets/27b97ff2-e48f-4049-b89e-dfb61fb9b7d7" />


---

## 4. Bypass Root avec Frida

### 4.1 Script de bypass complet

**Fichier :** `C:\Users\a\bypass_root.js`

```javascript
// ============================================
// BYPASS ROOT DETECTION - SCRIPT COMPLET
// ============================================

var suspiciousPaths = [
    "/system/bin/su", "/system/xbin/su", "/sbin/su",
    "/system/bin/busybox", "/system/xbin/busybox",
    "/system/app/Superuser.apk", "/system/app/SuperSU.apk"
];

var suspiciousCommands = ["su", "which su", "busybox"];

Java.perform(function() {
    console.log("[*] Initialisation du bypass root...");

    // 1. Patch Build.TAGS
    try {
        var Build = Java.use('android.os.Build');
        Object.defineProperty(Build, 'TAGS', {
            get: function() { return 'release-keys'; }
        });
        console.log("[+] Build.TAGS → release-keys");
    } catch(e) { console.log("[-] Build.TAGS:", e); }

    // 2. Patch RootBeer
    try {
        var RootBeer = Java.use('com.scottyab.rootbeer.RootBeer');
        RootBeer.isRooted.implementation = function() {
            console.log("[+] RootBeer.isRooted → false");
            return false;
        };
        console.log("[+] RootBeer patché");
    } catch(e) { console.log("[*] RootBeer non présent"); }

    // 3. Hook File.exists()
    try {
        var File = Java.use('java.io.File');
        var originalExists = File.exists;
        File.exists.implementation = function() {
            var path = this.getAbsolutePath();
            for (var i = 0; i < suspiciousPaths.length; i++) {
                if (path.indexOf(suspiciousPaths[i]) !== -1) {
                    console.log("[+] File.exists bloqué: " + path);
                    return false;
                }
            }
            return originalExists.call(this);
        };
        console.log("[+] File.exists hooké");
    } catch(e) { console.log("[-] File.exists:", e); }

    // 4. Hook Runtime.exec()
    try {
        var Runtime = Java.use('java.lang.Runtime');
        Runtime.exec.overload('java.lang.String').implementation = function(cmd) {
            var lowerCmd = cmd.toLowerCase();
            for (var i = 0; i < suspiciousCommands.length; i++) {
                if (lowerCmd.indexOf(suspiciousCommands[i]) !== -1) {
                    console.log("[+] Runtime.exec bloqué: " + cmd);
                    return this.exec("echo");
                }
            }
            return this.exec(cmd);
        };
        console.log("[+] Runtime.exec hooké");
    } catch(e) { console.log("[-] Runtime.exec:", e); }

    console.log("[✓] Bypass root activé !");
});
```

### 4.2 Exécution du script

```powershell
# Ouvrir DIVA sur l'émulateur, puis :
frida -U -n Diva -l C:\Users\a\bypass_root.js
```
<img width="951" height="647" alt="Capture d&#39;écran 2026-05-20 175718" src="https://github.com/user-attachments/assets/e1c77b1a-bb6c-43ef-a356-287d8181906f" />



### 4.3 Alternative en mode spawn

```powershell
# Démarre l'application automatiquement
frida -U -f jakhar.aseem.diva -l C:\Users\a\bypass_root.js
```

---

## 5. Bypass Root avec Objection

Objection automatise les hooks les plus courants.

### 5.1 Installation

```powershell
pip install objection
```

### 5.2 Lancement

```powershell
objection -g jakhar.aseem.diva explore
```
<img width="953" height="413" alt="Capture d&#39;écran 2026-05-20 180117" src="https://github.com/user-attachments/assets/6cedf193-46c8-4917-832a-d5d239f2b5d1" />

### 5.3 Activation du bypass

Dans la console Objection :
```bash
android root disable
```

**Résultat attendu :**
```text
(agent) Registering job 994774. Name: root-detection-disable
```

### 5.4 Commandes Objection utiles

| Commande | Description |
|---|---|
| `android root disable` | Désactive la détection root |
| `android sslpinning disable` | Désactive le SSL Pinning |
| `android hooking list classes` | Liste les classes chargées |
| `env` | Affiche les variables d'environnement |
| `exit` | Quitter Objection |

### 5.5 Syntaxe moderne (recommandée)

```powershell
objection -n jakhar.aseem.diva start
```
<img width="953" height="413" alt="Capture d&#39;écran 2026-05-20 180117" src="https://github.com/user-attachments/assets/f1414956-7a1c-407c-876d-ec6e420d7397" />

---

## 6. Codes sources complets

### Script Frida avec hooks natifs (`bypass_native.js`)

```javascript
// Hooks pour les fonctions C/C++
var suspiciousPaths = [
    "/system/bin/su", "/system/xbin/su", "/sbin/su"
];

function isSuspiciousPath(ptr) {
    try {
        var path = ptr.readCString();
        if (!path) return false;
        for (var i = 0; i < suspiciousPaths.length; i++) {
            if (path.indexOf(suspiciousPaths[i]) !== -1) return true;
        }
        return false;
    } catch(e) { return false; }
}

function hookNative(name, argIndex) {
    try {
        var addr = Module.findExportByName("libc.so", name);
        if (!addr) return;
        Interceptor.attach(addr, {
            onEnter: function(args) {
                if (isSuspiciousPath(args[argIndex])) {
                    this.block = true;
                    console.log("[+] Bloqué: " + name);
                }
            },
            onLeave: function(retval) {
                if (this.block) retval.replace(ptr(-1));
            }
        });
        console.log("[+] Hooké: " + name);
    } catch(e) {}
}

hookNative("open", 0);
hookNative("access", 0);
```

---

## 7. Dépannage

**Problème : `frida-ps -Uai` ne montre rien**
**Solution :** Vérifier que frida-server tourne.
```powershell
adb shell "ps | grep frida"
```

**Problème : `unable to locate setArgV0()`**
**Solution :** Utiliser le mode attach (`-n`) au lieu de spawn (`-f`).
```powershell
# OUVREZ D'ABORD L'APP MANUELLEMENT, PUIS :
frida -U -n Diva -l script.js
```

**Problème : `objection: command not found`**
**Solution :** Réinstaller ou ajouter au PATH.
```powershell
python -m pip install --upgrade objection
```

**Problème : L'application crash après injection**
**Solution :** Vérifier les versions Frida client/serveur.
```powershell
# Côté PC
frida --version

---

