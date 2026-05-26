# Laboratoire 3 : Interception et Analyse du Trafic HTTP/HTTPS sous Android

## Introduction
Ce guide pratique explique comment positionner un proxy d'écoute (Burp Suite) entre un émulateur Android et une application cible. L'enjeu est de visualiser concrètement :
- Le fonctionnement d'un proxy réseau (Man-in-the-Middle)
- Les données échangées (requêtes, en-têtes, paramètres, cookies)
- Les bases du protocole HTTP/HTTPS et la gestion des certificats
- La méthodologie pour rédiger des preuves de concept (PoC) lors d'un audit.

Ressource vidéo de référence : [Lien YouTube](https://www.youtube.com/watch?v=_nPy5MhtNk0)

## Compétences Acquises
À l'issue de ce TP, vous saurez :
- Forcer le trafic d'un smartphone Android à transiter par un proxy d'analyse.
- Décortiquer les requêtes web (méthodes, URI, données envoyées).
- Comprendre les mécanismes de chiffrement HTTPS et la nécessité d'une Autorité de Certification (CA) dédiée aux tests.
- Rédiger des rapports techniques clairs avec contexte et captures.

## Conditions Préalables
1. Avoir **Burp Suite Community Edition** installé et opérationnel.
2. Disposer d'un environnement virtualisé Android (via Android Studio, par exemple un Pixel ou Nexus).
3. Définir une application ou un site d'entraînement dont vous avez l'autorisation d'analyser le trafic.

> **⚠️ Avertissements de Sécurité (Strictes)**
> - N'interceptez que les flux de vos cibles de test.
> - Ne manipulez aucune donnée sensible ou personnelle.
> - Pensez impérativement à désinstaller les certificats root ajoutés à la fin du laboratoire.

---

## Mode d'Opération (Pas-à-Pas)

### Phase 1 : Initialisation de l'outil d'interception
1. Lancez Burp Suite et créez un projet temporaire (Temporary Project).
2. Rendez-vous dans l'onglet **Proxy** > **Intercept**.
3. Assurez-vous que le bouton indique **Intercept is off**.
*Explication* : En phase de configuration, il ne faut pas bloquer la navigation. Le mode passif est préférable pour s'assurer que le flux passe bien avant d'essayer de le modifier.
*Piège classique* : Laisser "Intercept is on" et croire que l'émulateur n'a pas accès à internet.

### Phase 2 : Configuration du point d'écoute (Listener)
1. Allez dans les paramètres du Proxy (**Proxy settings** ou **Proxy listeners**).
2. Vérifiez qu'une interface d'écoute est configurée et activée (Enabled).
3. Relevez ces deux informations essentielles :
   - Le port d'écoute (ex: `8080`), que nous appellerons `<PORT_PROXY>`
   - L'interface réseau associée ("Loopback only" ou "All interfaces")

### Phase 3 : Repérage de l'adresse IP de l'hôte
Pour que le téléphone virtuel puisse communiquer avec votre ordinateur, il a besoin de l'IP locale de ce dernier.
1. Ouvrez un terminal sur votre machine physique (hôte) et utilisez `ipconfig` (Windows) ou `ifconfig` / `ip a` (Linux/Mac) pour trouver votre adresse réseau locale (IPv4).
2. Notez cette adresse : `<IP_HOTE>`.
*Attention* : Ne confondez pas votre IP publique avec votre IP réseau locale (du type 192.168.X.X ou 10.X.X.X).

### Phase 4 : Routage du trafic de l'émulateur
1. Démarrez votre émulateur Android.
2. Allez dans les réglages **Réseau & Internet**, puis cliquez sur la connexion **Wi-Fi** actuelle.
3. Modifiez les paramètres avancés de ce réseau et basculez le **Proxy** en mode **Manuel**.
4. Saisissez :
   - Hostname / Nom d'hôte : `<IP_HOTE>`
   - Port : `<PORT_PROXY>`
5. Sauvegardez la configuration.

### Phase 5 : Validation de la capture en clair (HTTP)
1. Ouvrez le navigateur intégré à l'émulateur Android.
2. Naviguez vers une page web HTTP classique (ou votre cible d'entraînement).
3. Basculez sur Burp Suite et ouvrez l'onglet **HTTP history**.
4. Confirmez que de nouvelles lignes apparaissent (requêtes GET/POST).
*Si l'historique reste vide* : Vérifiez que l'IP et le port sont corrects et que votre pare-feu local (sur la machine hôte) ne bloque pas les connexions entrantes sur le `<PORT_PROXY>`.

### Phase 6 : Examen approfondi des requêtes
1. Dans l'historique de Burp, cliquez sur une des requêtes.
2. Observez la section **Raw** (Brut) pour visualiser :
   - La méthode HTTP (GET, POST, PUT...)
   - Les en-têtes envoyés par le navigateur (User-Agent, Accept-Language...)
3. Utilisez le panneau **Inspector** pour décoder plus facilement les paramètres d'URL (Query parameters) et les Cookies.
*Compétence clé* : La plus-value de l'auditeur réside dans sa capacité à comprendre le contexte de ces champs (qui envoie quoi, et pourquoi ?).

### Phase 7 : Test de l'interception active
1. Dans l'onglet **Intercept**, cliquez pour afficher **Intercept is on**.
2. Rechargez la page web sur le mobile.
3. Observez que la page charge indéfiniment : la requête est bloquée dans Burp, en attente de votre validation (Forward) ou rejet (Drop).
4. Désactivez l'interception (**Intercept is off**) pour laisser passer le trafic.

### Phase 8 : Comprendre la problématique du HTTPS
*Note : Cette section est théorique pour ce TP, l'ajout effectif d'un certificat doit être fait avec précaution.*
Afin que Burp puisse déchiffrer les requêtes HTTPS, il agit comme une attaque de l'homme du milieu. Pour que l'appareil Android accepte de communiquer, il faut lui injecter un certificat d'Autorité (CA) généré par Burp.
- Naviguez dans les paramètres de sécurité Android ("Install a certificate").
- Remarquez la différence entre les certificats Wi-Fi, VPN et CA.
*Avertissement* : N'installez jamais de certificats de labo sur vos appareils personnels de tous les jours.

### Phase 9 : Rédaction du compte-rendu
Un bon audit s'accompagne d'un rapport irréprochable. Créez un document comprenant :
- **Périmètre** : Cible testée et environnement (Émulateur Android).
- **Setup** : Version de Burp, IP `<IP_HOTE>` et port `<PORT_PROXY>`, date du test.
- **Preuves Techniques** : Extraits des logs HTTP (Headers, URL de la requête).
- **Analyse** : Les informations sensibles qui transitent, les observations sur les mécanismes de session (Cookies).
- **Recommandations** : Suggestions d'amélioration de la sécurité (attributs de cookies, limitation des données envoyées).

---

## Liste de Vérification (Checklist)
- [ ] Le trafic HTTP s'affiche bien dans `HTTP history`.
- [ ] Le port et l'IP du Proxy listener sont correctement relevés.
- [ ] Le paramétrage proxy "Manuel" d'Android correspond à l'hôte.
- [ ] Le mode "Intercept" a été testé avec succès puis éteint.
- [ ] Le mini-rapport (contexte + preuve) est rédigé.
- [ ] Le nettoyage de fin de session a été effectué.

## Nettoyage Post-Labo
Pour éviter des problèmes de connexion lors de vos prochaines utilisations :
1. Repassez le Proxy du Wi-Fi Android sur "None" (Aucun).
2. Supprimez tout certificat CA Burp éventuellement installé.
3. Fermez votre projet Burp Suite.
