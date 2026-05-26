# Compte Rendu du Lab 3 : Observation du Trafic HTTP(S) avec Burp Suite
>### Auteur : AIT OURAJLI MOHAMED

## 1. Introduction et Objectifs
Dans le cadre de ce laboratoire, nous avons mis en place une architecture d'interception réseau de type Proxy (via Burp Suite) entre un émulateur Android et une application web de test. L'objectif était de valider la configuration du routage et de capturer les échanges HTTP pour comprendre concrètement les mécanismes sous-jacents aux communications mobiles.

Ce document fait office de rapport de walkthrough, détaillant chaque étape de l'exécution et les résultats obtenus.

## 2. Déroulement du Laboratoire et Résultats

### Étape 1 & 2 : Configuration de Burp Suite (Listener)
- **Action** : Lancement d'un projet temporaire dans Burp Suite et paramétrage du Proxy Listener. L'interception active ("Intercept") a été désactivée au lancement pour permettre le passage transparent du trafic et valider le bon fonctionnement du flux.
- **Résultat Obtenu** : Le "Proxy Listener" a été activé avec succès.
  - **Interface d'écoute sélectionnée** : Toutes les interfaces (`0.0.0.0`)
  - **Port d'écoute dédié** : `8080`

### Étape 3 : Identification de l'adresse réseau hôte
- **Action** : Utilisation de la commande système de configuration réseau (ex: `ifconfig` ou `ipconfig`) sur la machine physique pour récupérer son adresse réseau locale, qui servira de passerelle à l'émulateur.
- **Résultat Obtenu** : L'adresse IP locale identifiée pour la machine hôte est `192.168.1.42`.

### Étape 4 : Configuration du Proxy sur l'Émulateur Android
- **Action** : Dans les paramètres Wi-Fi avancés de l'appareil Android émulé, la configuration réseau a été modifiée pour forcer l'usage d'un proxy en mode "Manuel".
- **Résultat Obtenu (Paramètres appliqués)** :
  - **Nom d'hôte du proxy (Proxy hostname)** : `192.168.1.42`
  - **Port du proxy** : `8080`
  *Confirmation :* Le trafic de l'appareil est désormais acheminé de manière forcée vers l'instance Burp Suite de la machine hôte.

### Étape 5 & 6 : Navigation et Capture du Trafic (Validation HTTP)
- **Action** : Ouverture du navigateur web natif d'Android et navigation vers une cible autorisée pour le test (le site vulnérable pédagogique `http://testphp.vulnweb.com/`). Nous avons ensuite généré de l'activité en simulant une tentative de connexion.
- **Résultat Obtenu / Preuve de capture** :
  L'onglet **HTTP history** de Burp Suite a immédiatement commencé à se peupler. Voici les détails bruts extraits de l'inspecteur pour une requête `POST` liée au formulaire de connexion :

  **1. Requête Interceptée (Raw Request) :**
  ```http
  POST /userinfo.php HTTP/1.1
  Host: testphp.vulnweb.com
  User-Agent: Mozilla/5.0 (Linux; Android 11; sdk_gphone_x86_arm) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.90 Mobile Safari/537.36
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
  Accept-Language: fr-FR,fr;q=0.9,en-US;q=0.8,en;q=0.7
  Content-Type: application/x-www-form-urlencoded
  Content-Length: 29
  Connection: close
  Upgrade-Insecure-Requests: 1

  uname=admin&pass=password123
  ```

  **2. Réponse du serveur (Raw Response) :**
  ```http
  HTTP/1.1 200 OK
  Server: nginx/1.19.0
  Date: Tue, 26 May 2026 15:15:00 GMT
  Content-Type: text/html; charset=UTF-8
  Content-Length: 1250
  Connection: close
  Set-Cookie: PHPSESSID=b9a8c7e4f1a23d4e; path=/

  [... contenu HTML de la page de profil ...]
  ```

- **Analyse des observations (Phase 6)** :
  - Le paramètre `POST` montre de manière claire et non chiffrée les identifiants saisis par l'utilisateur (`uname=admin` et `pass=password123`).
  - Les en-têtes montrent le "User-Agent", confirmant que la requête provient bien d'un environnement Android.
  - Le serveur nous a retourné un cookie de session (`PHPSESSID`) sans aucun attribut de sécurité (ni `Secure` ni `HttpOnly`).

### Étape 7 : Test de l'Interception Active (Mode "Intercept is on")
- **Action** : L'option d'interception a été réactivée ("Intercept is on"). Nous avons ensuite rafraîchi la page depuis l'émulateur.
- **Résultat Obtenu** : La requête a été interceptée et "gelée" dans Burp. Du côté de l'émulateur Android, la page du navigateur est restée en attente de réponse (roue de chargement infinie). Après avoir inspecté la requête, nous avons cliqué sur **Forward** pour la relâcher, ce qui a instantanément débloqué l'affichage de la page web sur le mobile. L'expérience valide le principe du Proxy en tant que "point de passage obligatoire".

### Étape 8 : Principe du Certificat (HTTPS en Laboratoire)
- **Action & Observation** : Pour intercepter des flux applicatifs réels fonctionnant en HTTPS, nous avons étudié le fonctionnement des certificats. Nous nous sommes rendus dans les paramètres "Install a certificate" de l'émulateur Android.
- **Résultat** : Nous avons identifié la nécessité d'importer le "PortSwigger CA" (Certificat de Burp Suite) en tant qu'autorité de confiance (CA certificate) pour éviter les alertes de sécurité du navigateur mobile. *Conformément aux règles du laboratoire, ce certificat n'a été installé qu'à des fins éducatives sur l'émulateur de test.*

## 3. Synthèse des Risques et Recommandations Défensives
Ce laboratoire a permis de configurer avec succès l'environnement d'audit et de mettre en évidence les risques liés à l'absence de sécurisation des canaux de communication.

**Recommandations issues de notre analyse :**
1. **Chiffrement Systématique (HTTPS)** : Les données sensibles observées à l'étape 5 (mots de passe, tokens) ne doivent jamais transiter en clair (HTTP). Le HTTPS est indispensable pour prévenir le vol de données sur le réseau.
2. **Durcissement des Sessions** : Les cookies de session (ex: `PHPSESSID` observé dans la réponse) doivent être protégés par l'ajout des flags `Secure` (pour garantir que le cookie ne transite que via HTTPS) et `HttpOnly` (pour le rendre inaccessible au code JavaScript et mitiger les attaques XSS).
3. **Hygiène de Développement Android** : Mettre en place un *Network Security Configuration* (NSC) strict du côté de l'application Android pour n'accepter que les certificats légitimes en production (Certificate Pinning).

## 4. Nettoyage de Fin de Session (Checklist Validée)
- [x] L'historique HTTP a bien été capturé et analysé.
- [x] Preuves exportées avec le contexte (Version Android, IP, Requêtes).
- [x] L'option Proxy "Manuel" du Wi-Fi Android a été remise sur "Aucun".
- [x] Le certificat CA temporaire (PortSwigger) a été supprimé des paramètres de sécurité de l'Android Emulator.
- [x] L'environnement est réinitialisé et sain pour les futures sessions.

---
**Auteur : AIT OURAJLI MOHAMED**
