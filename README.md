# LAB 15 : Analyse Dynamique Android — Inspection TLS/HTTPS et Gestion du SSL Pinning

## 1. Introduction

Ce laboratoire porte sur l’analyse dynamique du trafic HTTPS d’une application Android à l’aide de Frida, d’un script JavaScript de bypass SSL Pinning et d’un proxy TLS tel que Burp Suite.

L’objectif est de comprendre comment une application Android peut empêcher l’interception HTTPS grâce au SSL Pinning, puis comment des hooks Frida peuvent être utilisés dans un environnement autorisé pour neutraliser ces vérifications et observer le trafic dans Burp Suite.

L’application de test utilisée est :

```text
HTTP Toolkit SSL Pinning Demo
Package : tech.httptoolkit.pinning_demo
```

Ce lab est réalisé uniquement dans un environnement de test contrôlé.

---

## 2. Objectifs du lab

Les objectifs principaux sont :

- installer et vérifier Frida côté PC ;
- déployer et lancer `frida-server` sur Android ;
- configurer Burp Suite comme proxy HTTPS ;
- installer le certificat CA de Burp sur l’émulateur ;
- lancer une application cible sous Frida ;
- injecter un script JavaScript de bypass SSL Pinning ;
- observer les logs Frida indiquant l’activation des hooks ;
- valider l’apparition des requêtes HTTPS dans Burp Suite.

---

## 3. Environnement de travail

| Élément | Valeur |
|---|---|
| Système hôte | Windows |
| Terminal | PowerShell |
| Appareil Android | Émulateur Android |
| Version Android utilisée | Android 11 |
| Proxy HTTPS | Burp Suite |
| Outil d’instrumentation | Frida |
| Script utilisé | `sslpin_bypass_universal.js` |
| Application cible | HTTP Toolkit SSL Pinning Demo |
| Package cible | `tech.httptoolkit.pinning_demo` |
| Version Frida observée | 17.9.11 |

---

## 4. Prérequis

Les éléments suivants sont nécessaires :

- Python 3.8 ou supérieur ;
- pip ;
- Android Platform Tools ;
- ADB ;
- Frida côté PC ;
- frida-server côté Android ;
- Burp Suite ;
- une application Android de test ;
- un émulateur Android avec débogage USB activé.

Vérification rapide :

```powershell
python --version
pip --version
adb version
frida --version
```
<img width="959" height="348" alt="Capture d&#39;écran 2026-05-28 232801" src="https://github.com/user-attachments/assets/ce3265ef-ddc3-4ab2-9273-6a4a028cf291" />

---

## 5. Vérification d’ADB et de l’émulateur

L’outil ADB est utilisé pour communiquer avec l’émulateur Android.

Déclaration du chemin ADB sous Windows :

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
```

Vérification de l’appareil connecté :

```powershell
& $ADB devices
```

Résultat attendu :

```text
List of devices attached
emulator-5554    device
```

Cette étape confirme que l’émulateur est correctement détecté par ADB.


## 6. Installation et vérification de Frida côté PC

Frida et ses outils sont installés avec pip :

```powershell
python -m pip install --upgrade frida frida-tools
```

Vérification de la version Frida :

```powershell
frida --version
python -c "import frida; print(frida.__version__)"
```

Dans ce lab, la version utilisée est :

```text
Frida 17.9.11
```


## 7. Déploiement de frida-server sur Android

L’architecture de l’émulateur est identifiée avec la commande :

```powershell
& $ADB shell getprop ro.product.cpu.abi
```

Dans notre cas, l’architecture est :

```text
x86_64
```

Le binaire `frida-server` correspondant à la même version que Frida côté PC est copié sur Android :

```powershell
& $ADB push .\frida-server /data/local/tmp/frida-server
& $ADB shell chmod 755 /data/local/tmp/frida-server
```

Lancement de `frida-server` :

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

Dans un deuxième terminal PowerShell, la connexion est vérifiée :

```powershell
frida-ps -Uai
```

Cette commande liste les processus et applications disponibles sur l’appareil Android.

<img width="939" height="450" alt="Capture d&#39;écran 2026-05-28 232837" src="https://github.com/user-attachments/assets/3e28db52-e644-49ef-81e7-a7eafd73cdce" />

---

## 8. Configuration du proxy Burp Suite

Burp Suite est configuré comme proxy HTTPS sur le PC.

Configuration utilisée :

```text
Adresse côté émulateur : 10.0.2.2
Port : 8080
```

Activation du proxy Android avec ADB :

```powershell
& $ADB shell settings put global http_proxy 10.0.2.2:8080
```

Vérification :

```powershell
& $ADB shell settings get global http_proxy
```

Résultat attendu :

```text
10.0.2.2:8080
```

Dans Burp Suite, l’interception doit être désactivée pour laisser passer les requêtes :

```text
Proxy → Intercept → Intercept is off
```

---

## 9. Installation du certificat CA Burp

Le certificat CA de Burp Suite doit être installé sur l’émulateur Android afin que le trafic HTTPS puisse être déchiffré dans Burp.

Installation depuis Android :

```text
Settings
→ Security
→ Encryption & credentials
→ Install a certificate
→ CA certificate
```


## 10. Installation et identification de l’application cible

L’application utilisée est :

```text
HTTP Toolkit SSL Pinning Demo
```

Installation de l’APK :

```powershell
& $ADB install -r .\android-ssl-pinning-demo.apk
```

Package cible :

```text
tech.httptoolkit.pinning_demo
```

Vérification avec Frida :

```powershell
frida-ps -Uai | Select-String -Pattern "pinning"
```

Résultat attendu :

```text
SSL Pinning Demo    tech.httptoolkit.pinning_demo
```

---

## 11. Test avant bypass SSL Pinning

Avant l’injection du script Frida, certaines requêtes HTTPS pinnées peuvent échouer.

Exemples de requêtes concernées :

```text
CONFIG-PINNED REQUEST
CONTEXT-PINNED REQUEST
OKHTTP PINNED REQUEST
VOLLEY PINNED REQUEST
TRUSTKIT PINNED REQUEST
```

Avant bypass, l’application peut afficher une erreur de type :

```text
javax.net.ssl.SSLHandshakeException
```

Cette erreur indique que l’application refuse le certificat présenté par Burp Suite.

Capture recommandée :

<img width="307" height="232" alt="image" src="https://github.com/user-attachments/assets/aad5314e-71e0-4e91-8619-d8a10c4310d9" />

---

## 12. Création du script `sslpin_bypass_universal.js`

Le script Frida utilisé permet de neutraliser plusieurs mécanismes de validation TLS courants :

- `SSLContext.init` ;
- `TrustManagerImpl` ;
- `okhttp3.CertificatePinner` ;
- TrustKit ;
- `WebViewClient.onReceivedSslError`.

Création du fichier :

```powershell
notepad .\sslpin_bypass_universal.js
```

Contenu utilisé :

```javascript
// sslpin_bypass_universal.js
// LAB 15 - Bypass SSL Pinning avec Frida
// À utiliser uniquement sur une application de test ou dans un cadre autorisé.

Java.perform(function () {
    console.log("[*] Universal SSL Pinning Bypass started");

    function log(msg) {
        console.log("[+] SSL bypass: " + msg);
    }

    // 1. TrustManager permissif
    try {
        var X509TrustManager = Java.use("javax.net.ssl.X509TrustManager");
        var SSLContext = Java.use("javax.net.ssl.SSLContext");

        var TrustManager = Java.registerClass({
            name: "com.frida.FriendlyTrustManager",
            implements: [X509TrustManager],
            methods: {
                checkClientTrusted: function (chain, authType) {
                    log("checkClientTrusted allowed");
                },
                checkServerTrusted: function (chain, authType) {
                    log("checkServerTrusted allowed");
                },
                getAcceptedIssuers: function () {
                    return [];
                }
            }
        });

        var TrustManagers = [TrustManager.$new()];

        var SSLContext_init = SSLContext.init.overload(
            "[Ljavax.net.ssl.KeyManager;",
            "[Ljavax.net.ssl.TrustManager;",
            "java.security.SecureRandom"
        );

        SSLContext_init.implementation = function (keyManager, trustManager, secureRandom) {
            log("SSLContext.init patched");
            return SSLContext_init.call(this, keyManager, TrustManagers, secureRandom);
        };

        log("SSLContext.init hook installed");
    } catch (e) {
        console.log("[-] SSLContext hook failed: " + e);
    }

    // 2. Conscrypt TrustManagerImpl Android
    try {
        var TrustManagerImpl = Java.use("com.android.org.conscrypt.TrustManagerImpl");

        if (TrustManagerImpl.verifyChain) {
            TrustManagerImpl.verifyChain.implementation = function (
                untrustedChain,
                trustAnchorChain,
                host,
                clientAuth,
                ocspData,
                tlsSctData
            ) {
                log("TrustManagerImpl.verifyChain bypassed for host: " + host);
                return untrustedChain;
            };
        }

        if (TrustManagerImpl.checkTrustedRecursive) {
            TrustManagerImpl.checkTrustedRecursive.implementation = function (
                certs,
                host,
                clientAuth,
                untrustedChain,
                trustAnchorChain,
                used
            ) {
                log("TrustManagerImpl.checkTrustedRecursive bypassed for host: " + host);
                var ArrayList = Java.use("java.util.ArrayList");
                return ArrayList.$new();
            };
        }

        log("TrustManagerImpl hooks installed");
    } catch (e) {
        console.log("[-] TrustManagerImpl hook failed: " + e);
    }

    // 3. OkHttp 3/4 CertificatePinner
    try {
        var CertificatePinner = Java.use("okhttp3.CertificatePinner");

        CertificatePinner.check.overloads.forEach(function (overload) {
            overload.implementation = function () {
                log("okhttp3.CertificatePinner.check bypassed");
                return;
            };
        });

        log("OkHttp CertificatePinner hooks installed");
    } catch (e) {
        console.log("[-] OkHttp CertificatePinner hook failed or not present: " + e);
    }

    // 4. TrustKit
    try {
        var OkHostnameVerifier = Java.use("com.datatheorem.android.trustkit.pinning.OkHostnameVerifier");

        OkHostnameVerifier.verify.overloads.forEach(function (overload) {
            overload.implementation = function () {
                log("TrustKit OkHostnameVerifier.verify bypassed");
                return true;
            };
        });

        log("TrustKit hooks installed");
    } catch (e) {
        console.log("[-] TrustKit hook failed or not present: " + e);
    }

    // 5. WebView SSL errors
    try {
        var WebViewClient = Java.use("android.webkit.WebViewClient");

        WebViewClient.onReceivedSslError.implementation = function (view, handler, error) {
            log("WebView SSL error bypassed");
            handler.proceed();
        };

        log("WebViewClient onReceivedSslError hook installed");
    } catch (e) {
        console.log("[-] WebView hook failed: " + e);
    }

    console.log("[+] Universal SSL pinning bypass installed successfully");
});
```

Capture recommandée :

```md
![Script Frida universel](images/07-script-universal-js.png)
```

---

## 13. Lancement de l’application avec Frida

L’application est arrêtée avant le lancement en mode spawn :

```powershell
& $ADB shell am force-stop tech.httptoolkit.pinning_demo
```

Lancement de Frida avec injection du script :

```powershell
frida -U -f tech.httptoolkit.pinning_demo -l .\sslpin_bypass_universal.js
```

Résultat observé dans la console Frida :

```text
Frida 17.9.11 - A world-class dynamic instrumentation toolkit
Connected to Android Emulator 5554
Spawned `tech.httptoolkit.pinning_demo`. Resuming main thread!
[*] Universal SSL Pinning Bypass started
[+] SSL bypass: SSLContext.init hook installed
[+] SSL bypass: TrustManagerImpl hooks installed
[+] SSL bypass: OkHttp CertificatePinner hooks installed
[+] SSL bypass: TrustKit hooks installed
[+] SSL bypass: WebViewClient onReceivedSslError hook installed
[+] Universal SSL pinning bypass installed successfully
```

Des logs ont ensuite été affichés lors des connexions HTTPS :

```text
[+] SSL bypass: TrustManagerImpl.checkTrustedRecursive bypassed for host: null
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: checkServerTrusted allowed
```

<img width="1040" height="447" alt="image" src="https://github.com/user-attachments/assets/f97796cf-e7ba-46ef-81ed-36259328cdbe" />


---

## 14. Validation après bypass dans l’application

Après injection du script Frida, les requêtes HTTPS ont été relancées depuis l’application.

Les hooks installés permettent de contourner les mécanismes principaux de validation TLS.

Résultat attendu :

```text
CONFIG-PINNED REQUEST      Succès
CONTEXT-PINNED REQUEST     Succès
OKHTTP PINNED REQUEST      Succès
VOLLEY PINNED REQUEST      Succès
TRUSTKIT PINNED REQUEST    Succès
```

<img width="501" height="476" alt="Capture d&#39;écran 2026-05-28 234817" src="https://github.com/user-attachments/assets/caf084e8-8db7-4f44-9ca0-3400fd6077a0" />


## 15. Validation dans Burp Suite

Après l’injection du script Frida, Burp Suite affiche les requêtes HTTPS de l’application dans :

```text
Proxy → HTTP history
```

Requêtes observées :

```text
https://sha256.badssl.com
https://ecc384.badssl.com
```

Les réponses HTTP observées sont :

```text
Status : 200
Type   : HTML
Port   : 8080
```

<img width="1913" height="116" alt="Capture d&#39;écran 2026-05-28 234838" src="https://github.com/user-attachments/assets/1933cf10-e379-4ffe-8ef3-a9d974dd80ff" />


Interprétation :

```text
Le trafic HTTPS de l’application est visible dans Burp Suite.
Les requêtes vers sha256.badssl.com et ecc384.badssl.com retournent un code HTTP 200.
Cela confirme que le script Frida a permis de gérer le SSL Pinning et de rendre le trafic observable dans le proxy.
```

---

## 16. Analyse des résultats

La console Frida montre que plusieurs hooks ont été installés avec succès :

| Élément hooké | Rôle |
|---|---|
| `SSLContext.init` | Remplace ou modifie le TrustManager utilisé par l’application |
| `TrustManagerImpl` | Neutralise certaines validations TLS Android/Conscrypt |
| `okhttp3.CertificatePinner` | Contourne le pinning OkHttp/Retrofit |
| TrustKit | Contourne certaines vérifications TrustKit |
| `WebViewClient.onReceivedSslError` | Autorise la navigation WebView malgré une erreur SSL |

Les logs Frida suivants confirment l’activation du bypass :

```text
[+] SSL bypass: SSLContext.init hook installed
[+] SSL bypass: TrustManagerImpl hooks installed
[+] SSL bypass: OkHttp CertificatePinner hooks installed
[+] SSL bypass: TrustKit hooks installed
[+] Universal SSL pinning bypass installed successfully
```

---

## 17. Remarque sur les requêtes `unknown host`

Dans Burp Suite, certaines lignes peuvent apparaître avec :

```text
unknown host
```

Exemples :

```text
http://wuicdmiyhxooq
http://mwcfbkxwdjtm
```

Ces requêtes ne sont pas utilisées comme preuve principale du lab.  
Elles peuvent provenir de requêtes système, de tests réseau de l’émulateur ou de trafic parasite.

Les preuves importantes sont les requêtes HTTPS valides :

```text
https://sha256.badssl.com    200
https://ecc384.badssl.com    200
```

---

## 18. Résumé des résultats

| Test | Résultat |
|---|---|
| ADB détecte l’émulateur | Réussi |
| Frida détecte l’appareil | Réussi |
| frida-server fonctionne | Réussi |
| Burp Suite reçoit le trafic | Réussi |
| Certificat CA Burp installé | Réussi |
| Script Frida injecté | Réussi |
| Hooks SSL/TLS installés | Réussi |
| Logs `[+] SSL bypass` visibles | Réussi |
| Requêtes HTTPS visibles dans Burp | Réussi |
| Réponses HTTP 200 observées | Réussi |

---

## 19. Commandes principales utilisées

### Déclaration ADB

```powershell
$ADB = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
```

### Vérifier l’appareil

```powershell
& $ADB devices
```

### Configurer le proxy Android

```powershell
& $ADB shell settings put global http_proxy 10.0.2.2:8080
& $ADB shell settings get global http_proxy
```

### Vérifier Frida

```powershell
frida --version
frida-ps -Uai
```

### Lancer frida-server

```powershell
& $ADB shell "/data/local/tmp/frida-server -l 0.0.0.0:27042"
```

### Lancer l’application avec Frida

```powershell
frida -U -f tech.httptoolkit.pinning_demo -l .\sslpin_bypass_universal.js
```

### Nettoyer le proxy après le lab

```powershell
& $ADB shell settings put global http_proxy :0
```

---

## 20. Livrables

Les livrables attendus pour ce lab sont :

| Fichier | Description |
|---|---|
| `sslpin_bypass_universal.js` | Script Frida de bypass SSL Pinning |
| `images/01-adb-devices.png` | Vérification ADB |
| `images/02-frida-version.png` | Version Frida |
| `images/03-frida-ps.png` | Liste des applications avec Frida |
| `images/04-proxy-config.png` | Proxy Android configuré |
| `images/05-burp-ca-installed.png` | Certificat Burp installé |
| `images/06-app-before-bypass.png` | État avant bypass |
| `images/07-script-universal-js.png` | Script utilisé |
| `images/08-frida-script-loaded.png` | Logs Frida avec hooks |
| `images/09-app-after-bypass.png` | Application après bypass |
| `images/10-burp-https-traffic.png` | Requêtes HTTPS dans Burp |

---

## 21. Conclusion

Ce lab a permis de réaliser une analyse dynamique TLS/HTTPS d’une application Android à l’aide de Frida et Burp Suite.

Après configuration du proxy et installation du certificat CA Burp, un script Frida nommé `sslpin_bypass_universal.js` a été injecté dans le processus de l’application `tech.httptoolkit.pinning_demo`.

Les logs Frida montrent que plusieurs mécanismes TLS ont été hookés avec succès, notamment `SSLContext.init`, `TrustManagerImpl`, `okhttp3.CertificatePinner`, TrustKit et `WebViewClient.onReceivedSslError`.

Après injection du script, les requêtes HTTPS de l’application sont devenues visibles dans Burp Suite, notamment vers `sha256.badssl.com` et `ecc384.badssl.com`, avec des réponses HTTP 200.

Le lab est donc validé.

---

## 22. Remarque éthique

Les techniques présentées dans ce lab doivent être utilisées uniquement dans un cadre légal et autorisé, par exemple sur une application de test, dans un laboratoire pédagogique ou lors d’un audit de sécurité encadré.

Elles ne doivent jamais être appliquées à des applications tierces sans autorisation explicite.
