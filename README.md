# LAB-16 — Intercepter du trafic HTTPS sur Android (SSL Pinning Bypass)

**Auteur :** Mohamed Amine Znady  
**Filière :** Cycle Ingénieur — CIR (Systèmes d'Information Distribués), EMSI Marrakech  
**Module :** Sécurité Mobile & Analyse de Trafic Android  

---

## Objectif du lab

L'objectif est d'apprendre à intercepter les communications réseau d'une application Android utilisant **HTTPS**. Une application bien configurée refuse de communiquer si elle détecte qu'un proxy s'intercale entre elle et son serveur — c'est le mécanisme de **SSL Pinning**.

Dans ce lab, on va le désactiver à l'aide de **Frida** et **Objection**, puis observer le trafic déchiffré directement dans **Burp Suite**.

**Résultat attendu :** les requêtes de l'application apparaissent dans l'historique HTTP de Burp Suite.

---

## Environnement

| Élément | Détail |
|---|---|
| **Système hôte** | Windows |
| **Émulateur** | Android 11 |
| **Application cible** | InsecureBankv2 (`com.android.insecurebankv2`) |
| **Proxy** | Burp Suite Community Edition — port `8080` |
| **IP hôte** | `192.168.11.130` |
| **Outils** | ADB, Frida, frida-server, Objection, Burp Suite |

### Application cible

**InsecureBankv2** est une application Android intentionnellement vulnérable, conçue pour l'entraînement à la sécurité mobile. Package : `com.android.insecurebankv2`.

---

## Étape 1 — Configurer le proxy sur l'émulateur Android

Configurer un proxy manuel sur le réseau Wi-Fi **AndroidWifi** de l'émulateur :

```
Proxy       : Manual
Hostname    : 192.168.11.130
Port        : 8080
IP settings : DHCP
```

Tout le trafic réseau de l'app transite désormais par Burp Suite.

---

## Étape 2 — Configurer Burp Suite pour écouter

Dans Burp Suite, configurer le listener proxy :

```
Bind to port    : 8080
Bind to address : All interfaces
```

> Le choix **All interfaces** est essentiel — il permet à l'émulateur Android de se connecter à Burp via l'IP de la machine hôte, et non uniquement via `localhost`.

---

## Étape 3 — Télécharger le certificat CA de Burp

Pour qu'Android accepte les certificats générés par Burp, il faut lui faire confiance. Depuis le navigateur de l'émulateur, ouvrir :

```
http://burp
```

Si la page Burp Suite s'affiche → le proxy est bien actif. Cliquer sur **CA Certificate** pour télécharger `cacert.der`.

---

## Étape 4 — Installer le certificat CA sur Android

Chemin d'installation dans les paramètres Android :

```
Settings → Security → Encryption & credentials → Install a certificate → CA certificate
```

Confirmation attendue :
```
CA certificate installed
```

---

## Étape 5 — Attacher Objection à l'application

Lancer Objection pour s'attacher à l'application à chaud :

```bash
objection -g com.android.insecurebankv2 explore
```

Console attendue :
```
com.android.insecurebankv2 (run) on (Android: 11) [usb]
```

---

## Étape 6 — Désactiver le SSL Pinning

Depuis la console Objection :

```bash
android sslpinning disable
```

Objection injecte des hooks pour contourner toutes les vérifications SSL. Sortie attendue :

```
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job ... Name: android-sslpinning-disable
```

Ces lignes confirment que le SSL Pinning est **neutralisé** — l'application ne vérifiera plus si le certificat présenté correspond à celui attendu.

---

## Étape 7 — Vérifier l'interception dans Burp Suite

Utiliser InsecureBankv2 normalement (connexion, navigation...) et observer l'onglet **HTTP History** de Burp Suite.

Des requêtes doivent apparaître, notamment :

```
http://192.168.11.130:8888/login
```

Le SSL Pinning est contourné — Burp intercepte le trafic réel de l'application vers son backend.

---

## Résumé du flux d'attaque

```
Application Android (InsecureBankv2)
   │
   ├─ [Étape 1]  Proxy Android → 192.168.11.130:8080
   ├─ [Étape 2]  Burp écoute sur toutes les interfaces
   ├─ [Étape 3-4] Certificat CA Burp installé sur Android
   ├─ [Étape 5]  Objection attaché à com.android.insecurebankv2
   ├─ [Étape 6]  SSL Pinning désactivé via hooks Frida/Objection
   └─ [Étape 7]  Trafic HTTPS déchiffré visible dans Burp ✓
```

---

*Mohamed Amine Znady — EMSI Marrakech, 2024-2025*
