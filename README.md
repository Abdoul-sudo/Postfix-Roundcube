# CONFIGURATION D'UN SERVEUR MAIL AVEC POSTFIX ET ROUNDCUBE
## Table des matières
1. [POSTFIX](#postfix)   
    - [Installation](#installP)
    - [Base de donnée](#bddP)
    - [Fichiers de liaison postfix-bdd](#lienP)
    - [Configuration](#configP)
2. [COURIER](#courier)
    - [Installation](#installC)
    - [Configuration](#configC)
3. [ROUNDCUBE](#roundcube)
    - [Installation](#installR)
    - [Configuration](#configR)
4. [TEST](#test)

## <a name="postfix"> POSTFIX</a>
---------------------------

## <a name="installP"> Installation de postfix</a>
`apt-get install postfix postfix-mysql`    
On choisit __pas de configuration__ lors de l'installation  

## <a name="bddP"> Base de donnée postfix</a>
On va la créer sur phpmyadmin en tant que root   
>Si jamais vous n'avez pas mis de mot de passe à root lors de la configuration , vous ne pourrez pas vous connecter à phpmyadmin en tant que root, vous devrez alors lui créer un mot de passe avec la commande:   
`mysql_secure_installation`   
* On crée la base de donnée postfix   
![postfix_bdd1](/assets/postfixBdd1.png)

* Dans la base de donnée postfix, on va dans préférence et on ajoute un nouvel utilisateur:   
![postfix_Bdd2](/assets/postfixBdd2.png)   
![postfix_Bdd3](/assets/postfixBdd3.png)

* On crée ces tables dans la base de donnée postfix:   
```
CREATE TABLE `domaines` (
  `domaine` varchar(255) NOT NULL default '',
  `etat` tinyint(1) NOT NULL default '1',
  PRIMARY KEY  (`domaine`)
) ENGINE=MyISAM;

CREATE TABLE `comptes` (
  `email` varchar(255) NOT NULL default '',
  `password` varchar(255) NOT NULL default '',
  `quota` int(10) NOT NULL default '0',
  `etat` tinyint(1) NOT NULL default '1',
  `imap` tinyint(1) NOT NULL default '1',
  `pop3` tinyint(1) NOT NULL default '1',
  PRIMARY KEY  (`email`)
) ENGINE=MyISAM;

CREATE TABLE `alias` (
  `source` varchar(255) NOT NULL default '',
  `destination` text NOT NULL,
  `etat` tinyint(1) NOT NULL default '1',
  PRIMARY KEY  (`source`)
) ENGINE=MyISAM;
```

## <a name="lienP"> Lier Postfix à la Base de donnée</a>
>Pour les fichiers ci-dessous:   
`password =` votre mdp de l'utilisateur postfix créé précédement. 
* `nano /etc/postfix/mysql-virtual_domaines.cf`   
On ajoute dans le fichier vide:     
```
hosts = 127.0.0.1
user = postfix
password = postfix
dbname = postfix
select_field = 'virtual'
table = domaines
where_field = domaine
additional_conditions = AND etat=1
```
* `nano /etc/postfix/mysql-virtual_comptes.cf`   
```
hosts = 127.0.0.1
user = postfix
password = postfix
dbname = postfix
table = comptes
select_field = CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/')
where_field = email
additional_conditions = AND etat=1
```

* `nano /etc/postfix/mysql-virtual_aliases.cf`
```
hosts = 127.0.0.1
user = postfix
password = postfix
dbname = postfix
table = alias
select_field = destination
where_field = source
additional_conditions = AND etat=1
```

* `nano /etc/postfix/mysql-virtual_aliases_comptes.cf`
```
hosts = 127.0.0.1
user = postfix
password = postfix
dbname = postfix
table = comptes
select_field = email
where_field = email
additional_conditions = AND etat=1
```

* `nano /etc/postfix/mysql-virtual_quotas.cf`   
```
hosts = 127.0.0.1
user = postfix
password = postfix
dbname = postfix
table = comptes
select_field = quota
where_field = email
```
## <a name="configP"> Configuration de Postfix</a>
* On crée un utilisateur:   
![adduser](/assets/adduser.png)   

* `nano /etc/postfix/main.cf`  
On copie dans le fichier (videz le s'il n'est pas vide):
```
# Bannière afficher lorsqu'on se connecte en SMTP sur le port 25
smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)

# Service qui envoie des notifications "nouveau message"
biff = no

# Desactive la commande SMTP VRFY. Arrête certaine technique pour avoir des adresses email
disable_vrfy_command = yes

# Impose au client SMTP de démarrer la session SMTP par une commande Helo (ou ehlo)
smtpd_helo_required = yes

# Avec le courier local ça ajoute .NDD aux adresses incomplètes (seulement le nom d'hote)
append_dot_mydomain = no

# Le nom de la machine du système de messagerie
# Par défaut c'est host.domain.tld mais on peut mettre un reverse dns
myhostname = 1.1.168.192.in-addr.arpa

# Le domaine utilisé par defaut pour poster les message local
myorigin = 1.1.168.192.in-addr.arpa

# Liste des domaines pour lequel le serveur doit accepter le courrier
mydestination = 1.1.168.192.in-addr.arpa, localhost.localdomain, localhost

# Pour effectuer des livraisons de courrier avec un relay (ici non)
relayhost =

# Liste des réseaux locaux autorisés
mynetworks = 127.0.0.0/8, 192.168.1.1

# Taille des boîtes au lettre (0 = illimité)
mailbox_size_limit = 0

# Séparateur entre le nom d'utilisateur et les extensions d'adresses
recipient_delimiter = +

# Interfaces réseaux à écouter (ici toutes)
inet_interfaces = all

# Gestion des boites mails virtuelle
# Contient les fichiers qui permettent de relier postfix  mysql
virtual_alias_maps = mysql:/etc/postfix/mysql-virtual_aliases.cf,mysql:/etc/postfix/mysql-virtual_aliases_comptes.cf
virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual_domaines.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual_comptes.cf

# Le dossier ou seront contenu les mails (=home de l'user vmail)
virtual_mailbox_base = /var/spool/vmail/

# L'id du groupe et de l'utilisateur vmail créé précédement
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

# Créer un dossier par comte email
virtual_create_maildirsize = yes

# A activer si on souhaite ajouter des quotas
virtual_mailbox_extended = yes

# Impose les limites au niveau des mails, dans notre cas aucune
virtual_mailbox_limit_maps = mysql:/etc/postfix/mysql-virtual_quotas.cf

# Ajouter une limite sur la taille des messages pour les boites virtuelles
virtual_mailbox_limit_override = yes
virtual_maildir_limit_message = "La boite mail de votre destinataire est pleine, merci de reessayez plus tard."
virtual_overquota_bounce = yes

# adresses d'expedition
smtpd_sender_restrictions =
        permit_mynetworks,
        warn_if_reject reject_unverified_sender

# adresses de destination
smtpd_recipient_restrictions =
        permit_mynetworks,
        reject_unauth_destination,
        reject_non_fqdn_recipient

# client
smtpd_client_restrictions =
        permit_mynetworks
```
> Modifiez ces lignes:   
`myhostname = VOTRE_REVERSE_DNS`        
`myorigin = VOTRE_REVERSE_DNS`       
`mydestination = VOTRE_REVERSE_DNS, localhost.localdomain, localhost`   
`mynetworks = 127.0.0.0/8, VOTRE_IP_DNS`    

  

* On donne des droits aux utilisateurs et on attribue __mysql-virtual\_*.cf__ au groupe __postfix__:   
`chmod -u=rw,g=r mysql-virtual_*.cf`   
`chgrp postfix mysql-virtual_*.cf`  
  
  `systemctl restart postfix`

* On regarde s'il y a des erreurs:   
`/etc/init.d/postfix check`   
`tail /var/log/syslog`

* On revient dans phpmyadmin
  * Dans la table domaines, on va ajouter notre nom de domaine DNS directement via l'onglet __insérer__:   
  ![bdd_domaine](/assets/bdd_domaine.png)   
  * Dans la table comptes, on va cette fois dans l'onglet __SQL__:   
  ``INSERT INTO `comptes` (`email`, `password`, `quota`, `etat`, `imap`,`pop3` ) VALUES ('contact@abdoul.mg', ENCRYPT('test'), '0', '1', '1', '1');``   
  ![bdd_compte](/assets/bdd_compte.png)   
    > Vous remplacez __@abdoul.mg__ par votre nom de domaine et __test__ par un mot de passe de votre choix.

* Maintenant, on va créer le dossier où les mails vont arriver: 
  * Pour le créer, on va envoyé un mail à l'adresse créée précédement:   
  `telnet 127.0.0.1 25`   
  `ehlo VOTRE_DOMAINE`   
  `mail from:<test@test.com>`   
  `rcpt to:<contact@VOTRE_DOMAINE>`  
  `data`   
  `ECRIRE_UN_MESSAGE`   
  `.`   
  `quit`   
    ![telnet](/assets/telnet.png) 
  * On regarde si le dossier a bien été créé:   
   ![m_delivered1](/assets/m_delivered1.png)   
   ![m_delivered2](/assets/m_delivered2.png) 


## <a name="courier"> COURIER</a>
---
## <a name="installC"> Installation de Courier</a>
`apt-get install courier-base courier-authdaemon courier-authlib-mysql courier-imap courier-pop`   
On appuie sur __Non__ lors de l'installation   
![installCourier](/assets/installCourier.png)   

## <a name="configC"> Configuration de Courier</a>
`cd /etc/courier`   
* `nano authdaemonrc`   
Remplacer __authmodulelist="authpam"__ par __authmodulelist="authmysql"__

* `nano authmysqlrc`   
  * Remplacer:
![authmysqlrcF](/assets/authmysqlrcF.png)   
par:
![authmysqlrcT](/assets/authmysqlrcT.png)   

  * Remplacer aussi:   
   __MYSQL_DATABASE  `mysql`__ par __MYSQL_DATABASE  `postfix`__   
   __MYSQL_USER_TABLE `passwd`__ par __MYSQL_USER_TABLE  `comptes`__   
   __MYSQL_CRYPT_PWFIELD `crypt`__ par __MYSQL_CRYPT_PWFIELD `password`__   
   __MYSQL_UID_FIELD `uid`__ par __MYSQL_UID_FIELD `5000`__   
   __MYSQL_GID_FIELD `gid`__ par __MYSQL_GID_FIELD `5000`__   
   __MYSQL_LOGIN_FIELD `id`__ par __MYSQL_LOGIN_FIELD `email`__   
   __MYSQL_HOME_FIELD home__ par __MYSQL_HOME_FIELD "var/spool vmail"__   
   On commente:   
   __MYSQL_NAME_FIELD name__   
   Ajouter en bas du fichier:   
__MYSQL_MAILDIR_FIELD    CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/')__ 
* `/etc/init.d/courier-authdaemon restart`   
  `/etc/init.d/courier-imap restart`   
  `/etc/init.d/courier-pop restart`

## <a name="roundcube"> ROUNDCUBE</a>
---
## <a name="installR"> Installation de Roundcube</a>
* `cd`   
* `wget https://github.com/roundcube/roundcubemail/releases/download/1.4.11/roundcubemail-1.4.11-complete.tar.gz`   
* `tar -zxvf roundcubemail-1.4.11-complete.tar.gz`   
* `mv roundcubemail-1.4.11/ roundcube` 

## <a name="configR"> Configuration de roundcube</a>
* On ajoute un user roundcube   
![adduserRoundcube](/assets/adduserRoundcube.png)   
`mv roundcube /var/www/html/`     
`cd /var/www/html`     
On va changer l'user et le groupe pour www   
`chown -R roundcube:roundcube roundcube/`    

* On va dans phpmyadmin et on crée la base de donnée de roundcube
![bdd_roundcube1](/assets/bdd_roundcube1.png)  
* On va dans privilège (de la base de donnée roundcube) et on ajoute un nouvel utilisateur
![bdd_roundcube2](/assets/bdd_roundcube2.png)
* `cd /etc/apache2/sites-available/`   
`nano roundcube.conf`   
On copie ceci dans le fichier vide:
```
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        ServerName webmail.abdoul.mg
        ServerAlias webmail.abdoul.mg
        DocumentRoot /var/www/html/roundcube
        <Directory /var/www/html/roundcube>
                Options FollowSymLinks
              #  AllowOverride All

                Options FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
  >Vous modifiez:    
  ServerAdmin webmaster@localhost   
  ServerName webmail.abdoul.mg   
  ServerAlias webmail.abdoul.mg 

  `a2ensite roundcube.conf` (pour activer le site)   
  `systemctl reload apache2`  

  __PS:__
  --- 
   >`a2dissite roundcube.conf` (pour désactiver le site)

  

* On crée les tables de la bdd roundcube
  * `cd /var/www/html/roundcube/SQL`      
  * `mysql -u root -p roundcube < /var/www/html/roundcube/SQL/mysql.initial.sql`     
  En revenant dans phpmyadmin, on peut voir que des tables ont été créées
  ![bdd_roundcube3](/assets/bdd_roundcube3.png)

* On va utiliser l'installateur automatique de roundcube
  * On tape l'url:   
  __http://abdoul.mg/roundcube/installer/__
  ![url_rInstaller](/assets/url_rInstaller.png)   
  * On appuie sur next, et ici on met les informations de la base de donnée roundcube:   
  ![url_rInstaller2](/assets/url_rInstaller2.png)   
  On doit également changer le port smtp en __25__ (au lieu de 587 qui est celui par défaut)   
  ![url_rInstaller2_2](/assets/url_rInstaller2_2.png)   
  * On continue, et on arrive ici   
  ![url_rInstaller3](/assets/url_rInstaller3.png)   
        
    On copie les lignes de code, on va dans   
    `cd /var/www/html/roundcube/config`   
    `nano config.inc.php`   
    Et on les colles dans le fichier vide,   
    il faut aussi ajouter cette ligne à la fin du fichier:   
    `$config['smtp_user'] = '';`


    On attribue le fichier à l'user roundcube    
    `chown roundcube:roundcube config.inc.php`   
    On revient sur roundcube/installer et on continue
  * On voit que les dossiers temporaires et logs sont __NOT OK__
  ![url_rInstaller4](/assets/url_rInstaller4.png)   
  `cd /var/www/html/roundcube`   
  `chown www-data:www-data temp/`   
  `chown www-data:www-data logs/`
  > PS: www-data c'est apache

  * On actualise et on voit que tout est __OK__   
  ![url_rInstaller5](/assets/url_rInstaller5.png)   

  * En bas, on teste l'envoi d'email et l'imap:   
  ![url_rInstaller6](/assets/url_rInstaller6.png)  
  ![url_rInstaller7](/assets/url_rInstaller7.png)   
 


  > En cas d'erreurs:   
  `nano /var/www/html/roundcube/config/defaults.inc.php`   
  et  mettre en __true__: `$config['smtp_debug'] = false;`  

  > `tail /var/log/syslog`   
  `tail /var/log/mail.log`


## <a name="test"> TEST</a>
---
* On va dans __http://abdoul.mg/roundcube/__   
![test1](/assets/test1.png)      
![test2](/assets/test2.png)   
  > L'utilisateur kirito@abdoul.mg est un utilisateur que j'ai créé (de la meme manière que contact@abdoul.mg) [ajout à la bdd -> test sur telnet]

* On vérifie si le mail est bien arrivé   
![test3](/assets/test3.png)    
![test4](/assets/test4.png)