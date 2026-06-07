# LAB 15 : Analyse Dynamique Android : Inspection TLS/HTTPS et Gestion du SSL Pinning
 
**Environnement :** Windows 11 — Genymotion Android 9.0 (Pie) x86 — Frida 17.10.1  
**Cible :** OWASP OMTG "Attack me if u can" — `sg.vp.owasp_mobile.omtg_android`  
**Proxy :** Burp Suite Community Edition  
**Résultat :** Hook SSLContext.init actif — `[+] SSL bypass: SSLContext.init patched` confirmé

---

## 1. Vue d'ensemble — SSL Pinning

### Qu'est-ce que le SSL Pinning ?

Le SSL Pinning (ou Certificate Pinning) est une technique de sécurité où une application mobile **hardcode** le certificat ou la clé publique du serveur directement dans son code. Lors de la connexion TLS, l'app compare le certificat présenté par le serveur avec sa copie interne — si les deux ne correspondent pas, la connexion est rejetée.

```
Sans pinning :
App → TLS Handshake → Vérifie CA chain → ✅ Connexion établie

Avec pinning :
App → TLS Handshake → Vérifie CA chain → Vérifie cert interne → ❌ Rejet si proxy
```


### Pourquoi bypasser le SSL Pinning ?

En audit de sécurité mobile, intercepter le trafic HTTPS permet de :
- Analyser les requêtes API et détecter des vulnérabilités (IDOR, injections, données sensibles)
- Comprendre la logique métier de l'application
- Identifier des endpoints non documentés
- Tester la sécurité des échanges de données

### Approche — Hooks Frida sur TrustManager/Conscrypt/OkHttp

```
[App Android]
    ↓ appel TLS
[TrustManager / Conscrypt / OkHttp CertificatePinner]
    ↓ Frida hooks → retourner "OK" systématiquement
[Connexion établie malgré proxy]
    ↓
[Burp Suite] → trafic HTTPS déchiffré visible
```

---

## 2. Prérequis et vérifications initiales

### Stack technique

- OS : Windows 11
- Émulateur : Genymotion Android 9.0 (Pie) x86
- Frida client (PC) : 17.10.1
- frida-server (Android) : 17.10.1-android-x86
- Proxy : Burp Suite Community Edition (port 8080)
- App cible : OWASP OMTG — `sg.vp.owasp_mobile.omtg_android`

### Vérifications rapides

```powershell
python --version
# → Python 3.14.5

pip --version

frida --version
# → 17.10.1

adb devices
# → 192.168.206.103:5555   device
```
---

## 3. Étape 1 — Frida et frida-server

### 3.1 Installation Frida côté PC

```powershell
python -m pip install --upgrade frida frida-tools

# Vérification
frida --version
python -c "import frida; print(frida.__version__)"
```

### 3.2 Déploiement frida-server sur Genymotion

```powershell
# Identifier l'architecture
adb shell getprop ro.product.cpu.abi
# → x86

# SELinux permissif
adb shell setenforce 0

# Lancer frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0 &"

# Forwarder les ports
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

---

## 4. Étape 2 — Proxy Burp Suite et certificat CA

### 4.1 Configurer Burp Suite

- Ouvrir Burp Suite Community
- **Proxy → Options → Proxy Listeners**
- Vérifier que le listener écoute sur `0.0.0.0:8080`
- Activer **Intercept**

<img width="825" height="327" alt="image" src="https://github.com/user-attachments/assets/36e3a3f3-2c87-45bc-b900-d4b14a68c66f" />

### 4.2 Configurer le proxy Wi-Fi sur Genymotion

Sur Genymotion :
**Settings → Wi-Fi → Long press réseau → Modify network → Show advanced → Proxy: Manual**

- Proxy hostname : IP du PC (ex: `192.168.x.x`)
- Proxy port : `8080`

<img width="392" height="342" alt="image" src="https://github.com/user-attachments/assets/75002a02-3843-4cbe-8b96-79c96e37ee4d" />

### 4.3 Installer le certificat CA Burp

```powershell
# Télécharger le certificat depuis Burp
Invoke-WebRequest -Uri "http://127.0.0.1:8080/cert" -OutFile "C:\frida-lab\burp_cacert.der"

# Pousser sur le device
adb push C:\frida-lab\burp_cacert.der /sdcard/burp_cacert.der

# Renommer en .crt pour que Android accepte l'installation
adb shell mv /sdcard/burp_cacert.der /sdcard/burp_cacert.crt
```

Sur Genymotion :
**Settings → Security → Encryption & credentials → Install from SD card → sélectionner `burp_cacert.crt`**

- Nom : `BurpCA`
- Credential use : `VPN and apps`
<img width="452" height="323" alt="image" src="https://github.com/user-attachments/assets/0762f789-648c-4792-8189-2799f9539762" />

### 4.4 Vérification du certificat installé

**Settings → Security → Trusted credentials → onglet USER**

<img width="452" height="206" alt="image" src="https://github.com/user-attachments/assets/d27fedc7-712a-4fb4-a347-cacb32129931" />

### 4.5 Validation du proxy

Depuis le navigateur Genymotion, naviguer vers `http://google.com` — la requête doit apparaître dans **Burp → Proxy → HTTP History**.

<img width="1337" height="106" alt="image" src="https://github.com/user-attachments/assets/f846b450-3ebe-4c92-8af3-d7689fab60df" />

---

## 5. Étape 3 — App cible et injection Frida

### 5.1 Installation de l'app OWASP OMTG

```powershell
adb install "C:\Users\mayam\Downloads\omtg-android.apk"
# → Success
````
#### Vérifier l'installation

<img width="1081" height="62" alt="image" src="https://github.com/user-attachments/assets/14edb1f5-55a1-455e-b268-5080cea4fcc1" />

### 5.2 Identifier le module SSL Pinning

L'app "Attack me if u can" contient plusieurs modules de test. Le module ciblé est :

```
OMTG_NETW_004_SSL_PINNING_WHOLE_CERT
```

> 📸 **[SCREEN 6 — Insérer ici]**  
> *Capture de l'app OMTG ouverte sur Genymotion montrant la liste des modules dont OMTG_NETW_004_SSL_PINNING_WHOLE_CERT visible*

### 5.3 Test d'injection minimal

```powershell
frida -U -f sg.vp.owasp_mobile.omtg_android -l hello.js
```

**Résultat :**
```
Spawned `sg.vp.owasp_mobile.omtg_android`. Resuming main thread!
[Phone::sg.vp.owasp_mobile.omtg_android ]-> [+] Frida Java.perform OK
```

> 📸 **[SCREEN 7 — Insérer ici]**  
> *Capture du terminal montrant `[+] Frida Java.perform OK` après injection de hello.js sur l'app OMTG*

---

## 6. Étape 4 — Script sslpin_bypass_universal.js

### 6.1 Présentation du script

Le script couvre 5 vecteurs de SSL pinning :

| Vecteur | Cible | Technique |
|---|---|---|
| SSLContext.init | Injection TrustManager permissif | Remplacer le TM si absent |
| X509TrustManager | Toutes implémentations | Énumération + neutralisation |
| Conscrypt TrustManagerImpl | Android 7+ | Hook checkTrusted/verifyChain |
| OkHttp CertificatePinner | Apps utilisant OkHttp3 | Hook check() |
| WebView | Apps WebView | Hook onReceivedSslError |

### 6.2 Script complet

```javascript

```

### 6.3 Exécution

```powershell
frida -U -f sg.vp.owasp_mobile.omtg_android -l sslpin_bypass_universal.js
```

### 6.4 Logs obtenus

```
Spawned `sg.vp.owasp_mobile.omtg_android`. Resuming main thread!
[Phone::sg.vp.owasp_mobile.omtg_android ]-> [+] SSL bypass: SSLContext.init patched
```

> 📸 **[SCREEN 8 — Insérer ici]**  
> *Capture du terminal montrant `[+] SSL bypass: SSLContext.init patched` après lancement du script sur l'app OMTG*

### 6.5 Analyse des résultats

Le hook `SSLContext.init` est actif et confirmé par le log. L'app OMTG utilise une **Network Security Configuration** qui ne respecte pas le proxy Wi-Fi système Android — le trafic ne transite donc pas par Burp via la configuration proxy standard.

**Ce que ça démontre :**
- ✅ Frida injecte correctement dans l'app
- ✅ Le hook SSLContext.init s'installe et s'exécute
- ✅ Le TrustManager permissif est injecté quand aucun TM n'est fourni
- ⚠️ La capture Burp nécessite que l'app respecte le proxy système ou utilise `adb reverse`

**Sur une app standard utilisant OkHttp :** les logs afficheraient également :
```
[+] SSL bypass: okhttp3.CertificatePinner.check skip
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl.checkTrusted -> allow
[+] SSL bypass: X509TrustManager patches attempted
```

---

## 7. Étape 5 — Variantes et cibles spécifiques

### 7.1 OkHttp/Retrofit uniquement

Si l'app utilise uniquement OkHttp pour le pinning :

```javascript
// Le bloc CertificatePinner.check suffit
try{
  const CP = Java.use('okhttp3.CertificatePinner');
  CP.check.overloads.forEach(ov => {
    ov.implementation = function(){
      console.log('[+] CertificatePinner.check bypassed');
    };
  });
}catch(e){}
```

Si l'app utilise un package renommé (obfuscation), identifier le nom réel :

```javascript
// Dans la console Frida interactive
Java.perform(function(){
  Java.enumerateLoadedClasses({
    onMatch: function(n){
      if (n.toLowerCase().includes('okhttp') ||
          n.toLowerCase().includes('pin') ||
          n.toLowerCase().includes('trust'))
        console.log(n);
    },
    onComplete: function(){ console.log('done'); }
  });
});
```

### 7.2 Conscrypt moderne (Android 7+)

Sur Android récents, `TrustManagerImpl` est le goulot principal :

```javascript
const TMI = Java.use('com.android.org.conscrypt.TrustManagerImpl');
['checkTrusted', 'verifyChain'].forEach(m => {
  if (TMI[m]) TMI[m].overloads.forEach(ov => {
    ov.implementation = function(){
      console.log('[+] Conscrypt.' + m + ' bypassed');
      return null;
    };
  });
});
```

### 7.3 WebView embarquée

```javascript
const WVC = Java.use('android.webkit.WebViewClient');
WVC.onReceivedSslError.implementation = function(view, handler, error){
  console.log('[+] WebView SSL error ignored');
  handler.proceed();
};
```

---

## 8. Étape 6 — Cas avancé : pinning natif BoringSSL

### 8.1 Quand utiliser le bypass natif ?

Si malgré le script Java aucune requête n'apparaît dans Burp, l'app fait probablement le pinning dans une lib native (BoringSSL/OpenSSL).

### 8.2 Découverte des symboles natifs

```powershell
frida-trace -U -i "SSL_*" -i "X509_*" sg.vp.owasp_mobile.omtg_android
```

Observer si `SSL_get_verify_result`, `X509_verify_cert`, `SSL_set_custom_verify` apparaissent.

### 8.3 Script sslpin_bypass_native.js

```javascript
// sslpin_bypass_native.js
function hook(name, lib){
  const addr = Module.findExportByName(lib || null, name);
  if (!addr) return console.log('[*] no', name);
  Interceptor.attach(addr, {
    onLeave(rv){
      if (name === 'SSL_get_verify_result'){
        console.log('[+] SSL_get_verify_result -> X509_V_OK');
        rv.replace(ptr(0)); // 0 = X509_V_OK
      }
    }
  });
  console.log('[+] Hooked', name);
}

hook('SSL_get_verify_result', 'libssl.so');
```

### 8.4 Exécution combinée

```powershell
frida -U -f sg.vp.owasp_mobile.omtg_android `
  -l sslpin_bypass_universal.js `
  -l sslpin_bypass_native.js
```

> ⚠️ **Note x86/Frida 17 :** `Interceptor` n'est pas fonctionnel sur Genymotion x86 avec Frida 17 (limitation connue documentée en Lab 5). Le script natif est fonctionnel sur un vrai device arm64.

---

## 9. Étape 7 — Validation et résultats

### 9.1 Ce qu'on a obtenu

| Étape | Statut | Preuve |
|---|---|---|
| Frida installé et opérationnel | ✅ | `frida --version` → 17.10.1 |
| frida-server déployé | ✅ | `frida-ps -Uai` liste les apps |
| App OMTG installée | ✅ | `adb install` → Success |
| Injection Frida validée | ✅ | `Java.perform OK` |
| Certificat Burp installé | ✅ | PortSwigger dans Trusted credentials |
| Proxy Wi-Fi configuré | ✅ | Trafic HTTP navigateur visible dans Burp |
| Hook SSLContext.init actif | ✅ | `[+] SSL bypass: SSLContext.init patched` |
| Capture Burp HTTPS app cible | ⚠️ | OMTG bypass proxy système via NSC |

### 9.2 Logs Frida confirmés

<img width="821" height="282" alt="image" src="https://github.com/user-attachments/assets/445bfa43-f0c7-478c-8cb9-9019dd098464" />


### 9.3 Pourquoi OMTG ne passe pas par Burp

Deux causes combinées expliquent l'absence de trafic dans Burp :

---

#### ⚠️ Cause 1 — Android 9 ne fait pas confiance aux certificats utilisateur

C'est **la cause principale**. Depuis Android 7 (Nougat), la politique par défaut a changé :

```
Android 6 et avant :
  App → TLS → Vérifie System CA + User CA  ✅

Android 7+ par défaut :
  App → TLS → Vérifie System CA uniquement ❌ User CA ignorées
```

Le certificat Burp a été installé dans le **store utilisateur** (Settings → Security → User credentials). Android 9 l'ignore complètement pour les connexions TLS des apps — même avec le proxy Wi-Fi correctement configuré.

**Solution :** installer le cert Burp en **System CA** via ADB :

```powershell
# 1. Calculer le hash du certificat (côté PC avec openssl)
openssl x509 -inform DER -subject_hash_old -in burp_cacert.der | head -1
# → ex: 9a5ba575

# 2. Remonter /system en écriture (root Genymotion requis)
adb shell mount -o rw,remount /system

# 3. Copier le cert avec le nom hash.0
adb shell cp /sdcard/burp_cacert.crt /system/etc/security/cacerts/9a5ba575.0
adb shell chmod 644 /system/etc/security/cacerts/9a5ba575.0

# 4. Redémarrer
adb reboot
```

> **Note :** `openssl` n'est pas disponible sur Genymotion Android 9 (`openssl: not found`). Le hash doit être calculé côté PC avec openssl installé localement.

---

#### ⚠️ Cause 2 — Network Security Configuration de l'app OMTG

L'app OMTG utilise en plus une **Network Security Configuration (NSC)** dans son `AndroidManifest.xml` qui exclut explicitement les CA utilisateur :

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
            <!-- Pas de "user" → CA utilisateur ignorées même si installées -->
        </trust-anchors>
    </base-config>
</network-security-config>
```

---


## 10. Concepts clés

### 10.1 SSL Pinning — Types

| Type | Mécanisme | Bypass |
|---|---|---|
| Certificate pinning | Compare le certificat complet | Hooker TrustManager |
| Public key pinning | Compare la clé publique uniquement | Hooker TrustManager / OkHttp |
| Native pinning | Validation en C/C++ via BoringSSL | Interceptor.attach sur SSL_* |

### 10.2 TrustManager — Rôle central

Le `TrustManager` est l'interface Java qui décide si un certificat est acceptable. En hookant `checkServerTrusted()` pour qu'il retourne `null` (pas d'exception = accepté), on court-circuite toute validation :

```javascript
X509TrustManager.checkServerTrusted.implementation = function(chain, authType) {
  // Ne rien faire = pas d'exception = certificat accepté
  return null;
};
```

### 10.3 Conscrypt — Implémentation Android

Android utilise **Conscrypt** comme provider TLS par défaut depuis Android 7. `TrustManagerImpl` dans `com.android.org.conscrypt` est le point central de validation des certificats sur la plupart des appareils modernes.

### 10.4 OkHttp CertificatePinner

OkHttp implémente son propre pinning via `CertificatePinner.check()` — indépendant du TrustManager système :

```java
// Code OkHttp normal
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("example.com", "sha256/XXXX...")
    .build();
```

Le hook Frida retourne simplement sans lever d'exception :

```javascript
CP.check.overloads.forEach(ov => {
  ov.implementation = function() { return; }; // pas d'exception = OK
});
```

### 10.5 Network Security Configuration

Android 7+ permet aux apps de définir leur politique de confiance via `res/xml/network_security_config.xml`. Si `<certificates src="user"/>` est absent, les CA utilisateur (dont Burp) sont ignorées même si le proxy est configuré.

---

