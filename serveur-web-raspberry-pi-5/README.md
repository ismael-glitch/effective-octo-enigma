# Serveur web sur Raspberry Pi 5

Guide pratique en français — préparé le 23 octobre 2026.

Objectif : héberger d'abord un site HTML sur le réseau local avec Raspberry Pi OS Lite 64 bits et Nginx, puis, si souhaité, le publier sur Internet avec HTTPS. Les commandes sont à exécuter manuellement : ce dépôt ne configure pas automatiquement le Raspberry Pi.

## Sommaire

1. [Matériel et préparation](#1-matériel-et-préparation)
2. [Installer le système](#2-installer-le-système)
3. [Connexion et mises à jour](#3-connexion-et-mises-à-jour)
4. [Installer le serveur web](#4-installer-le-serveur-web)
5. [Créer le site](#5-créer-le-site)
6. [Pare-feu et SSH](#6-pare-feu-et-ssh)
7. [Publication et HTTPS](#7-publication-et-https)
8. [Déployer ses fichiers](#8-déployer-ses-fichiers)
9. [Sauvegarde et restauration](#9-sauvegarde-et-restauration)
10. [Entretien et astuces](#10-entretien-et-astuces)
11. [Dépannage](#11-dépannage)
12. [Checklist](#12-checklist)
13. [Références](#13-références)

## 1. Matériel et préparation

- Raspberry Pi 5 avec une alimentation adaptée ; l'alimentation officielle 27 W est un choix simple.
- Refroidissement actif et boîtier ventilé pour fonctionner en continu.
- Carte microSD de qualité pour démarrer ; SSD USB ou NVMe compatible pour un usage régulier.
- Câble Ethernet et accès à l'interface de la box.
- Un ordinateur pour préparer le stockage et administrer le Pi.
- Facultatif pour Internet : un nom de domaine et une connexion autorisant les connexions entrantes.

Le débit montant de la connexion, le stockage et l'application déterminent la capacité réelle. Aucun nombre de visiteurs n'est garanti.

**Conventions :** remplacer `UTILISATEUR` par le compte créé dans Imager, `IP_DU_PI` par son adresse locale et `site.example.com` par son propre domaine. Les commandes « sur le Pi » sont exécutées après connexion SSH ; celles « sur l'ordinateur » sont exécutées dans un terminal local.

## 2. Installer le système

1. Télécharger [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Sélectionner Raspberry Pi 5 et Raspberry Pi OS Lite **64 bits**.
3. Choisir le bon support : l'écriture efface son contenu.
4. Dans la personnalisation, définir le nom d'hôte `webpi`, un utilisateur personnel, le fuseau horaire et, si nécessaire, le Wi-Fi.
5. Activer SSH, de préférence avec une clé publique. Ne jamais copier la clé privée.
6. Écrire l'image, éjecter le support, le connecter au Pi, brancher Ethernet puis l'alimentation.

Avec un SSD, vérifier sa compatibilité et l'ordre de démarrage dans la documentation officielle. On peut commencer sur microSD avant de migrer.

## 3. Connexion et mises à jour

Sur l'ordinateur :

```bash
ssh UTILISATEUR@webpi.local
```

Si le nom ne répond pas, relever l'adresse du Pi dans la liste des appareils de la box :

```bash
ssh UTILISATEUR@IP_DU_PI
```

À la première connexion, vérifier l'empreinte SSH via un accès de confiance au Pi, puis accepter si elle correspond.

Sur le Pi :

```bash
sudo apt update
sudo apt full-upgrade
sudo reboot
```

La connexion se ferme pendant le redémarrage. Se reconnecter puis afficher les informations utiles :

```bash
cat /etc/os-release
uname -m
hostname -I
```

Pour ce parcours, l'architecture attendue est `aarch64`. Dans la box, réserver l'adresse du Pi par DHCP pour éviter qu'elle change. Les intitulés dépendent de la box.

## 4. Installer le serveur web

Sur le Pi :

```bash
sudo apt install nginx curl
sudo systemctl enable --now nginx
systemctl is-active nginx
curl -I http://localhost
```

Résultats attendus : service `active` et réponse HTTP. Depuis un autre appareil du même réseau, ouvrir `http://IP_DU_PI`. Une page Nginx doit apparaître.

## 5. Créer le site

Ces étapes supposent une installation neuve. Si des sites existent déjà, sauvegarder et adapter la configuration avant de changer le site par défaut.

Créer un répertoire appartenant à l'utilisateur connecté. Nginx pourra lire les fichiers sans pouvoir les modifier :

```bash
sudo install -d -o "$USER" -g www-data -m 2755 /var/www/mon-site
nano /var/www/mon-site/index.html
```

Coller cette page, puis enregistrer avec Ctrl+O, Entrée et quitter avec Ctrl+X :

```html
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Mon Raspberry Pi 5</title>
</head>
<body>
  <h1>Mon serveur web fonctionne !</h1>
  <p>Ce site est hébergé sur mon Raspberry Pi 5.</p>
</body>
</html>
```

Créer la configuration :

```bash
sudo nano /etc/nginx/sites-available/mon-site
```

Coller :

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name webpi.local;

    root /var/www/mon-site;
    index index.html;
    server_tokens off;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ /\. {
        deny all;
    }
}
```

Le bloc de fichiers cachés évite de servir des fichiers comme `.env` ou `.git`. Garder les secrets hors du répertoire web dans tous les cas.

Sur une installation neuve, désactiver uniquement le lien du site par défaut, puis activer le nouveau :

```bash
sudo unlink /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/mon-site /etc/nginx/sites-enabled/mon-site
sudo nginx -t
```

Si le lien `default` était déjà absent, passer à la suite. Si `mon-site` existe déjà dans `sites-enabled`, vérifier sa cible plutôt que répéter `ln`.

**Uniquement si le test réussit :**

```bash
sudo systemctl reload nginx
curl -I http://localhost
curl http://localhost
```

Ouvrir `http://IP_DU_PI` : la nouvelle page doit apparaître. Une adresse inexistante doit renvoyer 404.

## 6. Pare-feu et SSH

Sur le Pi, autoriser SSH **avant** d'activer le pare-feu. Ces règles supposent le port SSH standard 22 ; adapter si celui-ci a été changé.

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Garder la session actuelle ouverte et tester une seconde connexion SSH depuis l'ordinateur. Le pare-feu local ne configure pas la box.

Pour ajouter une clé si elle n'a pas été fournie dans Imager, sur un ordinateur disposant de ces commandes :

```bash
ssh-keygen -t ed25519
ssh-copy-id UTILISATEUR@IP_DU_PI
ssh UTILISATEUR@IP_DU_PI
```

Si une clé existe déjà, la réutiliser plutôt que l'écraser. Sous Windows sans `ssh-copy-id`, ajouter la clé publique via Imager ou dans `~/.ssh/authorized_keys` sur le Pi ; permissions : 700 pour `~/.ssh`, 600 pour le fichier.

Après validation de la clé dans une seconde session, le durcissement SSH est facultatif :

```bash
sudo nano /etc/ssh/sshd_config.d/00-webpi.conf
```

Contenu :

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Vérifier la configuration effective puis recharger :

```bash
sudo sshd -t
sudo sshd -T
sudo systemctl reload ssh
```

Vérifier dans la sortie que `passwordauthentication` vaut `no`. Des règles existantes ou des blocs Match peuvent modifier le résultat. Tester encore une nouvelle connexion avant de fermer la session initiale. Ne pas rediriger le port SSH de la box pour ce parcours.

## 7. Publication et HTTPS

Cette partie est facultative. Pour un site local, rester à l'étape précédente.

### Préparer le domaine et la box

1. Faire pointer l'enregistrement DNS A du domaine vers l'IPv4 publique de la connexion.
2. Si cette adresse change, prévoir une mise à jour du DNS.
3. Rediriger les ports TCP 80 et 443 de la box vers les mêmes ports du Pi.
4. Vérifier le pare-feu de la box.
5. Tester le domaine depuis un téléphone en données mobiles.

En présence de CGNAT ou de double NAT, une simple redirection peut ne pas suffire. Demander une IPv4 publique au fournisseur d'accès ou choisir un accès par VPN/tunnel selon le besoin. Un certificat obtenu par validation DNS ne rend pas à lui seul le serveur accessible.

Ne créer un enregistrement AAAA que si l'IPv6 du Pi est joignable et les pare-feux adaptés. Un AAAA incorrect peut casser l'accès et la validation HTTPS. En IPv6, vérifier les règles entrantes même sans redirection NAT.

### Configurer le domaine

Dans `/etc/nginx/sites-available/mon-site`, remplacer :

```nginx
server_name webpi.local;
```

par son domaine réel, par exemple :

```nginx
server_name site.example.com;
```

Puis :

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Continuer seulement si `http://VOTRE_DOMAINE` fonctionne depuis Internet.

### Activer HTTPS

Installation via les paquets de Raspberry Pi OS :

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d site.example.com
```

Remplacer le domaine avant exécution. Suivre les questions de Certbot et activer la redirection vers HTTPS si elle est proposée. Ne demander que les noms DNS déjà configurés et accessibles.

Vérifier :

```bash
sudo nginx -t
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers --all
```

Contrôler qu'une tâche de renouvellement Certbot est planifiée. Garder le port 80 accessible pour les renouvellements utilisant HTTP-01. Consulter `https://VOTRE_DOMAINE` depuis l'extérieur et vérifier le certificat.

## 8. Déployer ses fichiers

Depuis l'ordinateur, pour une mise à jour simple :

```bash
scp ./index.html UTILISATEUR@IP_DU_PI:/var/www/mon-site/index.html
```

Pour un dossier, si rsync est disponible des deux côtés :

```bash
rsync -av --dry-run ./site/ UTILISATEUR@IP_DU_PI:/var/www/mon-site/
rsync -av ./site/ UTILISATEUR@IP_DU_PI:/var/www/mon-site/
```

La première commande prévisualise le transfert. Le slash final de `site/` signifie « son contenu ». Ces commandes ne suppriment pas les anciens fichiers.

N'envoyer que les fichiers destinés au public, jamais le dépôt complet avec son dossier Git, ses secrets ou ses sauvegardes.

Si nécessaire, sur le Pi, rétablir les droits de lecture pour ce site statique :

```bash
find /var/www/mon-site -type d -exec chmod 2755 {} +
find /var/www/mon-site -type f -exec chmod 644 {} +
```

Modifier du HTML ne nécessite pas de recharger Nginx. Pour une application Python, Node.js ou PHP, il faudra une configuration supplémentaire : processus applicatif, dépendances, service et éventuellement proxy inverse. Ce guide livre un site statique.

## 9. Sauvegarde et restauration

Sur le Pi, sauvegarder le contenu et la configuration avant les changements importants :

```bash
mkdir -p "$HOME/sauvegardes-web"
chmod 700 "$HOME/sauvegardes-web"
sudo tar -czf "$HOME/sauvegardes-web/web-$(date +%F-%H%M%S).tar.gz" /var/www/mon-site /etc/nginx
```

L'avertissement tar indiquant la suppression du slash initial est normal. Relever le nom créé :

```bash
sudo ls -lh "$HOME/sauvegardes-web"
```

Pour récupérer une archive précise, remplacer NOM_ARCHIVE :

```bash
sudo chown "$USER":"$(id -gn)" "$HOME/sauvegardes-web/NOM_ARCHIVE.tar.gz"
chmod 600 "$HOME/sauvegardes-web/NOM_ARCHIVE.tar.gz"
```

Puis, depuis l'ordinateur :

```bash
scp UTILISATEUR@IP_DU_PI:sauvegardes-web/NOM_ARCHIVE.tar.gz .
```

Conserver une copie hors du Pi. Cette archive ne contient pas les clés TLS, les paramètres de la box ni les éventuelles bases de données. Sauvegarder séparément `/etc/letsencrypt` dans un stockage privé protégé si HTTPS est activé. Ne jamais publier ces clés dans GitHub.

Pour tester la sauvegarde, lister puis extraire dans un dossier temporaire, sans écraser immédiatement le système :

```bash
tar -tzf NOM_ARCHIVE.tar.gz
mkdir restauration-test
tar -xzf NOM_ARCHIVE.tar.gz -C restauration-test
```

Comparer les fichiers. Pour une restauration réelle, remettre les fichiers nécessaires dans leurs emplacements avec sudo, vérifier propriétaires et permissions, lancer `sudo nginx -t`, puis recharger seulement après succès. Une application avec base de données nécessite aussi un export cohérent et un test de restauration de ses données.

## 10. Entretien et astuces

### Routine

- Chaque semaine : mises à jour, vérification du site et des journaux.
- Après une modification : test Nginx, contrôle HTTP et sauvegarde.
- Chaque mois : vérifier le renouvellement TLS et restaurer une sauvegarde en test.
- Après un redémarrage : vérifier que Nginx est reparti.

Commandes pratiques sur le Pi :

```bash
sudo apt update
sudo apt upgrade
df -h
free -h
uptime
systemctl --failed
vcgencmd measure_temp
vcgencmd get_throttled
```

Redémarrer après une mise à jour du noyau quand une interruption est acceptable. Les indicateurs de limitation peuvent inclure des événements passés ; consulter leur définition officielle avant de conclure à un problème actuel.

### Bonnes habitudes

- Ethernet pour une connexion stable.
- SSD et sauvegardes externes pour un usage régulier.
- Arrêt propre avec `sudo shutdown -h now` avant de couper l'alimentation.
- Pas de `chmod 777` pour corriger un problème de droits.
- Pas de clés privées, mots de passe ou fichiers `.env` dans GitHub.
- Surveiller la place occupée par journaux et sauvegardes.
- Conserver une copie de la configuration fonctionnelle avant modification.
- Le site cesse d'être accessible si le Pi, la box ou la connexion tombe : envisager un onduleur si nécessaire.

## 11. Dépannage

| Symptôme | Vérification | Action |
| --- | --- | --- |
| `webpi.local` ne répond pas | Liste des clients de la box | Utiliser l'IP locale et vérifier le réseau |
| SSH refusé | SSH activé dans Imager, port et pare-feu | Vérifier depuis un écran local si nécessaire |
| Délai dépassé | Alimentation, Ethernet, bonne IP | Vérifier d'abord la connexion locale |
| Ancienne page Nginx | Liens dans sites-enabled et cache navigateur | Vérifier le site actif et rafraîchir la page |
| 403 | index.html et droits des dossiers parents | Rendre les fichiers lisibles et dossiers traversables |
| 404 | Chemin root et nom du fichier | Vérifier orthographe et majuscules |
| Port 80 occupé | `sudo ss -ltnp` | Identifier le service avant de modifier ses ports |
| Nginx ne recharge pas | `sudo nginx -t` | Corriger le fichier et la ligne indiqués |
| Local OK, Internet KO | NAT, CGNAT, DNS, pare-feux | Tester en données mobiles et vérifier chaque couche |
| Certificat refusé | DNS A/AAAA et accès public port 80 | Corriger la joignabilité avant de relancer |
| 502 après ajout d'une application | Service applicatif et port local | Vérifier le backend et le proxy |
| Pi lent ou instable | Température, alimentation, espace disque | Corriger le facteur identifié |

Pour examiner Nginx :

```bash
sudo journalctl -u nginx -n 50 --no-pager
sudo tail -n 50 /var/log/nginx/error.log
sudo tail -n 50 /var/log/nginx/access.log
sudo ls -l /etc/nginx/sites-enabled
```

Les journaux peuvent contenir des adresses IP et des URL privées : les relire avant de les partager.

## 12. Checklist

- [ ] Raspberry Pi OS Lite 64 bits installé.
- [ ] Compte personnel et accès SSH fonctionnels.
- [ ] Adresse locale réservée dans la box.
- [ ] Système à jour.
- [ ] Nginx actif au démarrage.
- [ ] Page visible depuis un autre appareil du réseau.
- [ ] Configuration validée par nginx -t.
- [ ] Pare-feu actif et seconde connexion SSH testée.
- [ ] Sauvegarde copiée hors du Pi et restauration testée.
- [ ] Si publication : domaine, ports et accès externe vérifiés.
- [ ] Si HTTPS : certificat valide et renouvellement testé.
- [ ] Aucun secret publié dans le dépôt.

## 13. Références

- [Installation et préparation — Raspberry Pi](https://www.raspberrypi.com/documentation/computers/getting-started.html)
- [Accès distant et SSH — Raspberry Pi](https://www.raspberrypi.com/documentation/computers/remote-access.html)
- [Configuration — Raspberry Pi](https://www.raspberrypi.com/documentation/computers/configuration.html)
- [Guide Nginx](https://nginx.org/en/docs/beginners_guide.html)
- [Instructions Certbot : sélectionner Nginx et le système adapté](https://certbot.eff.org/instructions)

Les menus et versions peuvent évoluer. Ce parcours est prévu pour une installation neuve avec les paquets de Raspberry Pi OS ; il ne constitue pas une validation sur le matériel de l'utilisateur.
