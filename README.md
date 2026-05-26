# lab12-Localisation-temps-r-el-via-GPS-et-Google-Maps
LAB 12 – LOCALISATION TEMPS RÉEL VIA GPS ET GOOGLE MAPS

Cours : Programmation Mobile – Android avec Java


Objectif du laboratoire

Ce laboratoire a pour but de développer une application Android capable de récupérer les coordonnées GPS d’un smartphone, de les envoyer à un serveur distant via une API PHP, et de les afficher sur une carte Google Maps. L’architecture repose sur une base de données MySQL, un backend PHP structuré (modèle, DAO, service) et un client Android utilisant Volley pour les requêtes réseau.


Prérequis avant de commencer

Avant de démarrer, assurez-vous de disposer des éléments suivants :

- Un serveur web local avec Apache, PHP et MySQL (XAMPP, WAMP ou LAMP).
- Android Studio installé sur votre poste de développement.
- Un téléphone Android avec le GPS activé, ou un émulateur configuré pour simuler la localisation.
- Le téléphone et le serveur doivent être connectés au même réseau Wi-Fi (ou utiliser l’adresse 10.0.2.2 pour l’émulateur).
- Pour la partie Google Maps, vous devez créer une activité "Google Map Activity" dans Android Studio et configurer votre clé API (google_maps_key) comme indiqué dans l’énoncé.


Architecture générale

Le projet se divise en trois grandes parties :

1. Base de données MySQL – stockage des positions.
2. Backend PHP – API REST pour l’insertion et la récupération des données.
3. Application Android – collecte du GPS, envoi périodique des positions, affichage sur Google Maps.


PARTIE 1 – MYSQL (BASE DE DONNÉES)

Étape 1.1 – Créer la base de données

Dans phpMyAdmin ou en ligne de commande, créez une base nommée : localisation

Étape 1.2 – Créer la table "position"

Exécutez le script SQL suivant :

CREATE TABLE `position` (
  `id` int(11) NOT NULL PRIMARY KEY AUTO_INCREMENT,
  `latitude` double NOT NULL,
  `longitude` double NOT NULL,
  `date` datetime NOT NULL,
  `imei` varchar(20) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

Vérification : Une requête SELECT * FROM position; doit retourner une table vide au départ.


PARTIE 2 – BACKEND PHP (API + CLASSES)

Organisation des fichiers

Créez un dossier nommé "localisation" dans la racine de votre serveur web (par exemple htdocs pour XAMPP). À l’intérieur, créez l’arborescence suivante :

localisation/
  classe/Position.php
  connexion/Connexion.php
  dao/IDao.php
  service/PositionService.php
  createPosition.php
  showPositions.php

Étape 2.1 – Modèle : classe/Position.php

Cette classe représente une ligne de la table position. Elle contient les attributs privés (id, latitude, longitude, date, imei) et leurs getters/setters. Le constructeur accepte tous les champs.

Étape 2.2 – Connexion : connexion/Connexion.php

Cette classe établit une connexion PDO à la base de données MySQL. Elle utilise les paramètres : host localhost, dbname localisation, login root, mot de passe vide (à adapter selon votre configuration). Le mode d’erreur PDO::ERRMODE_EXCEPTION est activé.

Étape 2.3 – Interface DAO : dao/IDao.php

Elle définit les méthodes standard du CRUD : create, update, delete, getById, getAll. Dans ce TP, seules create et getAll sont implémentées.

Étape 2.4 – Service : service/PositionService.php

Cette classe implémente IDao. Elle contient :

- create($position) : exécute une requête préparée INSERT INTO position (latitude, longitude, date, imei) VALUES (?,?,?,?)
- getAll() : exécute SELECT * FROM position et retourne le résultat sous forme de tableau associatif.

Étape 2.5 – Script d’insertion : createPosition.php

Ce script est appelé par l’application Android en méthode POST. Il :

- Vérifie que la requête est bien en POST.
- Récupère les paramètres latitude, longitude, date, imei depuis $_POST.
- Instancie un objet Position et appelle PositionService::create().
- Retourne une réponse JSON (ok true/false, éventuellement l’adresse IP du client).

Étape 2.6 – Script de récupération : showPositions.php

Ce script est appelé par l’application Android (pour afficher les marqueurs sur la carte). Il retourne un tableau JSON contenant toutes les positions enregistrées, sous la forme {"positions": [...]}.

Vérification : Testez ces deux scripts avec Postman ou un navigateur (pour GET, mais en POST pour create). showPositions.php doit renvoyer un JSON valide.


PARTIE 3 – APPLICATION ANDROID (GPS + VOLLEY)

Étape 3.1 – Créer le projet

Dans Android Studio, créez un nouveau projet avec une Empty Activity. Nommez-le "Localisation". Langage Java.

Étape 3.2 – Ajouter les permissions dans AndroidManifest.xml

Ajoutez les permissions suivantes :

<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

Pour autoriser les requêtes HTTP non sécurisées (car votre serveur est en http), ajoutez l’attribut dans la balise <application> :

android:usesCleartextTraffic="true"

Étape 3.3 – Ajouter la dépendance Volley

Dans le fichier build.gradle (module app), ajoutez :

implementation 'com.android.volley:volley:1.2.1'

Puis synchronisez le projet.

Étape 3.4 – Créer le layout activity_main.xml

Le layout contient deux TextView pour afficher la latitude et la longitude, et un Button pour ouvrir la carte. Utilisez un LinearLayout vertical avec un padding de 16dp.

Étape 3.5 – Écrire MainActivity.java

Cette activité :

- Initialise Volley (RequestQueue).
- Gère la demande de permission de localisation (ACCESS_FINE_LOCATION) au moment de l’exécution (Android 6+).
- Utilise LocationManager avec GPS_PROVIDER, un intervalle de 60000 ms et une distance de 150 mètres (comme spécifié dans l’énoncé).
- Dans onLocationChanged, affiche les coordonnées dans les TextView, envoie les données au serveur via une requête POST (StringRequest) et affiche un Toast.
- La méthode addPosition(lat, lon) construit les paramètres (latitude, longitude, date au format "yyyy-MM-dd HH:mm:ss", imei). Pour l’imei, utilisez soit TelephonyManager.getDeviceId() (si permission accordée) soit Settings.Secure.ANDROID_ID comme fallback.
- Le bouton "Afficher Map" lance MapsActivity.

Vérification : Lancez l’application, acceptez la permission, activez le GPS. En vous déplaçant ou en attendant, des lignes doivent s’insérer dans la table "position" (vérifiez avec phpMyAdmin).


PARTIE 4 – GOOGLE MAP ACTIVITY (AFFICHAGE DES POSITIONS)

Étape 4.1 – Créer une activité Google Map

Dans Android Studio, cliquez droit sur le package → New → Google → Google Maps Activity. Nommez-la "MapsActivity". Android Studio génère automatiquement un fichier activity_maps.xml, un fichier google_maps_api.xml (contenant votre clé API) et la classe MapsActivity.

Étape 4.2 – Configurer la clé API

Suivez les instructions dans google_maps_api.xml pour obtenir une clé API Google Maps et l’insérer.

Étape 4.3 – Modifier MapsActivity

Dans MapsActivity, implémentez OnMapReadyCallback. Dans onCreate, récupérez le SupportMapFragment et appelez getMapAsync(this). Dans onMapReady, stockez la référence à GoogleMap et appelez une méthode setUpMap().

La méthode setUpMap() envoie une requête POST (ou GET) à showPositions.php (par exemple via Volley JsonObjectRequest). L’URL est du type http://IP_SERVEUR/localisation/showPositions.php. Une fois la réponse JSON reçue, parcourez le tableau "positions" et pour chaque objet, créez un marker avec new MarkerOptions().position(new LatLng(lat, lon)).title("Position").

Ajoutez également la permission INTERNET (déjà présente) et vérifiez que le GPS est actif.

Étape 4.4 – Lancer la carte

Depuis MainActivity, le bouton "Afficher Map" démarre MapsActivity. Celle-ci doit afficher tous les points enregistrés sous forme de marqueurs.


Tests et validation

- Test d’insertion : Après quelques secondes (intervalle GPS de 60 secondes), de nouvelles lignes apparaissent dans la table position.
- Test d’affichage : Ouvrez la carte ; tous les points enregistrés pour cet appareil doivent être visibles.
- Test de robustesse : Si le GPS est désactivé, l’application affiche des Toasts de statut. Si le serveur est inaccessible, un message d’erreur réseau s’affiche (sans crash).


Bonnes pratiques mises en œuvre

- Utilisation de requêtes préparées côté PHP pour éviter les injections SQL.
- Gestion des permissions dynamiques pour Android 6+.
- Séparation des couches (modèle, DAO, service) pour une meilleure maintenabilité.
- Utilisation de Volley pour les communications réseau asynchrones.
- Libération des ressources (pas de fuite de mémoire).
- Identification unique de l’appareil avec fallback (ANDROID_ID si IMEI indisponible).


Améliorations possibles

- Ajouter un service d’arrière-plan (ForegroundService) pour continuer l’envoi même lorsque l’application n’est pas au premier plan.
- Stocker localement les positions non synchronisées (avec SQLite ou Room) et les renvoyer plus tard.
- Filtrer les positions par date ou par appareil.
- Améliorer l’interface utilisateur (graphique, bouton de centrage, zoom).


Conclusion

Ce laboratoire vous a permis de mettre en place une chaîne complète d’acquisition, de transmission et de visualisation de données géolocalisées. Vous avez manipulé le GPS Android, la bibliothèque Volley, une API PHP orientée objet et l’intégration de Google Maps. Ces compétences sont réutilisables dans de nombreux projets mobiles (suivi de flotte, journal de bord, applications de logistique, etc.).
