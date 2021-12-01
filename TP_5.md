# I. Setup DB

Côté base de données, on va utiliser MariaDB.

![Easy database](pics/easy_database.jpg)

## Sommaire

- [I. Setup DB](#i-setup-db)
  - [Sommaire](#sommaire)
  - [1. Install MariaDB](#1-install-mariadb)
  - [2. Conf MariaDB](#2-conf-mariadb)
  - [3. Test](#3-test)

## 1. Install MariaDB

> Pour rappel, le gestionnaire de paquets sous les OS de la famille RedHat, c'est pas `apt`, c'est `dnf`.

🌞 **Installer MariaDB sur la machine `db.tp5.linux`**

```bash
[antho@db ~]$ sudo dnf install mariadb-server
```

🌞 **Le service MariaDB**

- lancez-le avec une commande `systemctl`
```bash
[antho@db ~]$ sudo systemctl start mariadb
```
- exécutez la commande `sudo systemctl enable mariadb` pour faire en sorte que MariaDB se lance au démarrage de la machine
```bash
[antho@db ~]$ sudo systemctl enable mariadb
```
- vérifiez qu'il est bien actif avec une commande `systemctl`
```bash
[antho@db ~]$ systemctl status mariadb
```
- déterminer sur quel port la base de données écoute avec une commande `ss`
  - je veux que l'information soit claire : le numéro de port avec le processus qu'il y a derrière
```bash
[antho@db ~]$ ss -tunlp
Netid  State   Recv-Q  Send-Q   Local Address:Port   Peer Address:Port Process              
tcp    LISTEN  0       80                   *:3306              *:*   
```

- isolez les processus liés au service MariaDB (commande `ps`)
```bash
[antho@db ~]$ ps -ef | grep mariadb
antho       4660    1438  0 17:18 pts/0    00:00:00 grep --color=auto mariadb
```
🌞 **Firewall**

- pour autoriser les connexions qui viendront de la machine `web.tp5.linux`, il faut conf le firewall
  - ouvrez le port utilisé par MySQL à l'aide d'une commande `firewall-cmd`
```bash
[antho@db ~]$ sudo firewall-cmd --remove-port=3306/tcp --permanent
```
> Rappel : il y a [le mémo Réseau Rocky](../../cours/memos/rocky_network.md) pour ça.

## 2. Conf MariaDB

Première étape : le `mysql_secure_installation`. C'est un binaire qui sert à effectuer des configurations très récurrentes, on fait ça sur toutes les bases de données à l'install.  
C'est une question de sécu.

🌞 **Configuration élémentaire de la base**

- exécutez la commande `mysql_secure_installation`
  - plusieurs questions successives vont vous être posées
  - expliquez avec des mots, de façon concise, ce que signifie chacune des questions
  - expliquez pourquoi vous répondez telle ou telle réponse (avec la sécurité en tête)

> Il existe des tonnes de guides sur internet pour expliquer ce que fait cette commande et comment répondre aux questions, afin d'augmenter le niveau de sécurité de la base.
```bash
[antho@db ~]$ mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

You already have a root password set, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] oui
Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```
---

🌞 **Préparation de la base en vue de l'utilisation par NextCloud**

- pour ça, il faut vous connecter à la base
- il existe un utilisateur `root` dans la base de données, qui a tous les droits sur la base
- si vous avez correctement répondu aux questions de `mysql_secure_installation`, vous ne pouvez utiliser le user `root` de la base de données qu'en vous connectant localement à la base
- donc, sur la VM `db.tp5.linux` toujours :

```bash
# Connexion à la base de données
# L'option -p indique que vous allez saisir un mot de passe
# Vous l'avez défini dans le mysql_secure_installation
$ sudo mysql -u root -p
```

Puis, dans l'invite de commande SQL :

```sql
# Création d'un utilisateur dans la base, avec un mot de passe
# L'adresse IP correspond à l'adresse IP depuis laquelle viendra les connexions. Cela permet de restreindre les IPs autorisées à se connecter.
# Dans notre cas, c'est l'IP de web.tp5.linux
# "meow" c'est le mot de passe :D
CREATE USER 'nextcloud'@'10.5.1.11' IDENTIFIED BY 'meow';

# Création de la base de donnée qui sera utilisée par NextCloud
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

# On donne tous les droits à l'utilisateur nextcloud sur toutes les tables de la base qu'on vient de créer
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'10.5.1.11';

# Actualisation des privilèges
FLUSH PRIVILEGES;
```
```bash
[antho@db ~]$ sudo mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 29
Server version: 10.3.28-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> CREATE USER 'nextcloud'@10.5.1.11 IDENTIFIED BY 'meow'
    -> CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci
    -> GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'10.5.1.11'
    -> FLUSH PRIVILEGES
    -> 
```
## 3. Test

Bon, là il faut tester que la base sera utilisable par NextCloud.

Concrètement il va faire quoi NextCloud vis-à-vis de la base MariaDB ?

- se connecter sur le port où écoute MariaDB
- la connexion viendra de `web.tp5.linux`
- il se connectera en utilisant l'utilisateur `nextcloud`
- il écrira/lira des données dans la base `nextcloud`

Il faudrait donc qu'on teste ça, à la main, depuis la machine `web.tp5.linux`.

Bah c'est parti ! Il nous faut juste un client pour nous connecter à la base depuis la ligne du commande : il existe une commande `mysql` pour ça.

🌞 **Installez sur la machine `web.tp5.linux` la commande `mysql`**

- vous utiliserez la commande `dnf provides` pour trouver dans quel paquet se trouve cette commande
```bash
[antho@db ~]$ sudo dnf install @mysql
```
🌞 **Tester la connexion**

- utilisez la commande `mysql` depuis `web.tp5.linux` pour vous connecter à la base qui tourne sur `db.tp5.linux`
- vous devrez préciser une option pour chacun des points suivants :
  - l'adresse IP de la machine où vous voulez vous connectez `db.tp5.linux` : `10.5.1.12`
  - le port auquel vous vous connectez
  - l'utilisateur de la base de données sur lequel vous connecter : `nextcloud`
  - l'option `-p` pour indiquer que vous préciserez un password
    - vous ne devez PAS le préciser sur la ligne de commande
    - sinon il y aurait un mot de passe en clair dans votre historique, c'est moche
  - la base de données à laquelle vous vous connectez : `nextcloud`
- une fois connecté à la base en tant que l'utilisateur `nextcloud` :
  - effectuez un bête `SHOW TABLES;`
  - simplement pour vous assurer que vous avez les droits de lecture
  - et constater que la base est actuellement vide

> Je veux donc dans le compte-rendu la commande `mysql` qui permet de se co depuis `web.tp5.linux` au service de base de données qui tourne sur `db.tp5.linux`, ainsi que le `SHOW TABLES`.

---

C'est bon ? Ca tourne ? [**Go installer NextCloud maintenant !**](./web.md)

![To the cloud](./pics/to_the_cloud.jpeg)
```bash
[antho@web ~]$  mysql -h 10.5.1.12 -P 3306 -u nextcloud -p nextcloud
Enter password: 
ERROR 2013 (HY000): Lost connection to MySQL server at 'reading initial communication packet', system error: 0
[antho@web ~]$ 



# II. Setup Web

Comme annoncé dans l'intro, on va se servir d'Apache dans le rôle de serveur Web dans ce TP5. Histoire de varier les plaisirs è_é

![Linux is a tipi](./pics/linux_is_a_tipi.jpg)

## Sommaire

- [II. Setup Web](#ii-setup-web)
  - [Sommaire](#sommaire)
  - [1. Install Apache](#1-install-apache)
    - [A. Apache](#a-apache)
    - [B. PHP](#b-php)
  - [2. Conf Apache](#2-conf-apache)
  - [3. Install NextCloud](#3-install-nextcloud)
  - [4. Test](#4-test)

## 1. Install Apache

### A. Apache

🌞 **Installer Apache sur la machine `web.tp5.linux`**

- le paquet qui contient Apache s'appelle `httpd`
- le service aussi s'appelle `httpd`
```bash
[antho@web ~]$ sudo dnf install httpd
```
---

🌞 **Analyse du service Apache**

- lancez le service `httpd` et activez le au démarrage
- isolez les processus liés au service `httpd`
- déterminez sur quel port écoute Apache par défaut
- déterminez sous quel utilisateur sont lancés les processus Apache
```bash
[antho@web ~]$ sudo systemctl start httpd
```
```bash
[antho@web ~]$ ss -tunlp
Netid  State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port Process 
tcp    LISTEN  0       128            0.0.0.0:22           0.0.0.0:*            
tcp    LISTEN  0       70                   *:33060              *:*            
tcp    LISTEN  0       128                  *:3306               *:*            
tcp    LISTEN  0       128                  *:80                 *:*            
tcp    LISTEN  0       128               [::]:22              [::]:*            
[antho@web ~]$ sudo firewall-cmd --zone=public --add-port=80/tcp
[sudo] Mot de passe de antho : 
success
[antho@web ~]$ ps aux | egrep '(apache|httpd)'
root        5498  0.0  1.4 282908 11840 ?        Ss   nov.26   0:00 /usr/sbin/httpd -DFOREGROUND
apache      5499  0.0  1.0 296788  8568 ?        S    nov.26   0:00 /usr/sbin/httpd -DFOREGROUND
apache      5500  0.0  1.2 1354572 10244 ?       Sl   nov.26   0:00 /usr/sbin/httpd -DFOREGROUND
apache      5501  0.0  1.4 1354572 12288 ?       Sl   nov.26   0:00 /usr/sbin/httpd -DFOREGROUND
apache      5502  0.0  1.4 1485700 12236 ?       Sl   nov.26   0:00 /usr/sbin/httpd -DFOREGROUND
antho       5754  0.0  0.1 221928  1164 pts/0    S+   00:09   0:00 grep -E --color=auto (apache|httpd)
```
---

🌞 **Un premier test**

- ouvrez le port d'Apache dans le firewall
- testez, depuis votre PC, que vous pouvez accéder à la page d'accueil par défaut d'Apache
  - avec une commande `curl`
  - avec votre navigateur Web
```bash
[antho@web ~]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 80/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
````
```bash
anthonygaillard@MBP-de-ANTHONY ~ % curl 10.5.1.11
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>
    <style type="text/css">
      /*<![CDATA[*/
      
      html {
        height: 100%;
        width: 100%;
      }  
        body {
  background: rgb(20,72,50);
  background: -moz-linear-gradient(180deg, rgba(20,72,50,1) 30%, rgba(0,0,0,1) 90%)  ;
  background: -webkit-linear-gradient(180deg, rgba(20,72,50,1) 30%, rgba(0,0,0,1) 90%) ;
  background: linear-gradient(180deg, rgba(20,72,50,1) 30%, rgba(0,0,0,1) 90%);
  background-repeat: no-repeat;
  background-attachment: fixed;
  filter: progid:DXImageTransform.Microsoft.gradient(startColorstr="#3c6eb4",endColorstr="#3c95b4",GradientType=1); 
        color: white;
        font-size: 0.9em;
        font-weight: 400;
        font-family: 'Montserrat', sans-serif;
        margin: 0;
        padding: 10em 6em 10em 6em;
        box-sizing: border-box;      
        
      }

   
  h1 {
    text-align: center;
    margin: 0;
    padding: 0.6em 2em 0.4em;
    color: #fff;
    font-weight: bold;
    font-family: 'Montserrat', sans-serif;
    font-size: 2em;
  }
  h1 strong {
    font-weight: bolder;
    font-family: 'Montserrat', sans-serif;
  }
  h2 {
    font-size: 1.5em;
    font-weight:bold;
  }
  
  .title {
    border: 1px solid black;
    font-weight: bold;
    position: relative;
    float: right;
    width: 150px;
    text-align: center;
    padding: 10px 0 10px 0;
    margin-top: 0;
  }
  
  .description {
    padding: 45px 10px 5px 10px;
    clear: right;
    padding: 15px;
  }
  
  .section {
    padding-left: 3%;
   margin-bottom: 10px;
  }
  
  img {
  
    padding: 2px;
    margin: 2px;
  }
  a:hover img {
    padding: 2px;
    margin: 2px;
  }

  :link {
    color: rgb(199, 252, 77);
    text-shadow:
  }
  :visited {
    color: rgb(122, 206, 255);
  }
  a:hover {
    color: rgb(16, 44, 122);
  }
  .row {
    width: 100%;
    padding: 0 10px 0 10px;
  }
  
  footer {
    padding-top: 6em;
    margin-bottom: 6em;
    text-align: center;
    font-size: xx-small;
    overflow:hidden;
    clear: both;
  }
 
  .summary {
    font-size: 140%;
    text-align: center;
  }

  #rocky-poweredby img {
    margin-left: -10px;
  }

  #logos img {
    vertical-align: top;
  }
  
  /* Desktop  View Options */
 
  @media (min-width: 768px)  {
  
    body {
      padding: 10em 20% !important;
    }
    
    .col-md-1, .col-md-2, .col-md-3, .col-md-4, .col-md-5, .col-md-6,
    .col-md-7, .col-md-8, .col-md-9, .col-md-10, .col-md-11, .col-md-12 {
      float: left;
    }
  
    .col-md-1 {
      width: 8.33%;
    }
    .col-md-2 {
      width: 16.66%;
    }
    .col-md-3 {
      width: 25%;
    }
    .col-md-4 {
      width: 33%;
    }
    .col-md-5 {
      width: 41.66%;
    }
    .col-md-6 {
      border-left:3px ;
      width: 50%;
      

    }
    .col-md-7 {
      width: 58.33%;
    }
    .col-md-8 {
      width: 66.66%;
    }
    .col-md-9 {
      width: 74.99%;
    }
    .col-md-10 {
      width: 83.33%;
    }
    .col-md-11 {
      width: 91.66%;
    }
    .col-md-12 {
      width: 100%;
    }
  }
  
  /* Mobile View Options */
  @media (max-width: 767px) {
    .col-sm-1, .col-sm-2, .col-sm-3, .col-sm-4, .col-sm-5, .col-sm-6,
    .col-sm-7, .col-sm-8, .col-sm-9, .col-sm-10, .col-sm-11, .col-sm-12 {
      float: left;
    }
  
    .col-sm-1 {
      width: 8.33%;
    }
    .col-sm-2 {
      width: 16.66%;
    }
    .col-sm-3 {
      width: 25%;
    }
    .col-sm-4 {
      width: 33%;
    }
    .col-sm-5 {
      width: 41.66%;
    }
    .col-sm-6 {
      width: 50%;
    }
    .col-sm-7 {
      width: 58.33%;
    }
    .col-sm-8 {
      width: 66.66%;
    }
    .col-sm-9 {
      width: 74.99%;
    }
    .col-sm-10 {
      width: 83.33%;
    }
    .col-sm-11 {
      width: 91.66%;
    }
    .col-sm-12 {
      width: 100%;
    }
    h1 {
      padding: 0 !important;
    }
  }
        
  
  </style>
  </head>
  <body>
    <h1>HTTP Server <strong>Test Page</strong></h1>

    <div class='row'>
    
      <div class='col-sm-12 col-md-6 col-md-6 '></div>
          <p class="summary">This page is used to test the proper operation of
            an HTTP server after it has been installed on a Rocky Linux system.
            If you can read this page, it means that the software it working
            correctly.</p>
      </div>
      
      <div class='col-sm-12 col-md-6 col-md-6 col-md-offset-12'>
     
       
        <div class='section'>
          <h2>Just visiting?</h2>

          <p>This website you are visiting is either experiencing problems or
          could be going through maintenance.</p>

          <p>If you would like the let the administrators of this website know
          that you've seen this page instead of the page you've expected, you
          should send them an email. In general, mail sent to the name
          "webmaster" and directed to the website's domain should reach the
          appropriate person.</p>

          <p>The most common email address to send to is:
          <strong>"webmaster@example.com"</strong></p>
    
          <h2>Note:</h2>
          <p>The Rocky Linux distribution is a stable and reproduceable platform
          based on the sources of Red Hat Enterprise Linux (RHEL). With this in
          mind, please understand that:

        <ul>
          <li>Neither the <strong>Rocky Linux Project</strong> nor the
          <strong>Rocky Enterprise Software Foundation</strong> have anything to
          do with this website or its content.</li>
          <li>The Rocky Linux Project nor the <strong>RESF</strong> have
          "hacked" this webserver: This test page is included with the
          distribution.</li>
        </ul>
        <p>For more information about Rocky Linux, please visit the
          <a href="https://rockylinux.org/"><strong>Rocky Linux
          website</strong></a>.
        </p>
        </div>
      </div>
      <div class='col-sm-12 col-md-6 col-md-6 col-md-offset-12'>
        <div class='section'>
         
          <h2>I am the admin, what do I do?</h2>

        <p>You may now add content to the webroot directory for your
        software.</p>

        <p><strong>For systems using the
        <a href="https://httpd.apache.org/">Apache Webserver</strong></a>:
        You can add content to the directory <code>/var/www/html/</code>.
        Until you do so, people visiting your website will see this page. If
        you would like this page to not be shown, follow the instructions in:
        <code>/etc/httpd/conf.d/welcome.conf</code>.</p>

        <p><strong>For systems using
        <a href="https://nginx.org">Nginx</strong></a>:
        You can add your content in a location of your
        choice and edit the <code>root</code> configuration directive
        in <code>/etc/nginx/nginx.conf</code>.</p>
        
        <div id="logos">
          <a href="https://rockylinux.org/" id="rocky-poweredby"><img src= "icons/poweredby.png" alt="[ Powered by Rocky Linux ]" /></a> <!-- Rocky -->
          <img src="poweredby.png" /> <!-- webserver -->
        </div>       
      </div>
      </div>
      
      <footer class="col-sm-12">
      <a href="https://apache.org">Apache&trade;</a> is a registered trademark of <a href="https://apache.org">the Apache Software Foundation</a> in the United States and/or other countries.<br />
      <a href="https://nginx.org">NGINX&trade;</a> is a registered trademark of <a href="https://">F5 Networks, Inc.</a>.
      </footer>
      
  </body>
</html>
anthonygaillard@MBP-de-ANTHONY ~ % 
```
### B. PHP

NextCloud a besoin d'une version bien spécifique de PHP.  
Suivez **scrupuleusement** les instructions qui suivent pour l'installer.

🌞 **Installer PHP**

```bash
# ajout des dépôts EPEL
$ sudo dnf install epel-release
$ sudo dnf update
# ajout des dépôts REMI
$ sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm
$ dnf module enable php:remi-7.4

# install de PHP et de toutes les libs PHP requises par NextCloud
$ sudo dnf install zip unzip libxml2 openssl php74-php php74-php-ctype php74-php-curl php74-php-gd php74-php-iconv php74-php-json php74-php-libxml php74-php-mbstring php74-php-openssl php74-php-posix php74-php-session php74-php-xml php74-php-zip php74-php-zlib php74-php-pdo php74-php-mysqlnd php74-php-intl php74-php-bcmath php74-php-gmp
```
```bash
[antho@web ~]$ sudo dnf install epel-release
[antho@web ~]$ sudo dnf update
[antho@web ~]$  sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm
[antho@web ~]$ sudo dnf module enable php:remi-7.4
[antho@web ~]$ sudo dnf install zip unzip libxml2 openssl php74-php php74-php-ctype php74-php-curl php74-php-gd php74-php-iconv php74-php-json php74-php-libxml php74-php-mbstring php74-php-openssl php74-php-posix php74-php-session php74-php-xml php74-php-zip php74-php-zlib php74-php-pdo php74-php-mysqlnd php74-php-intl php74-php-bcmath php74-php-gmp
```


## 2. Conf Apache

➜ Le fichier de conf utilisé par Apache est `/etc/httpd/conf/httpd.conf`.  
Il y en a plein d'autres : ils sont inclus par le premier.

➜ Dans Apache, il existe la notion de *VirtualHost*. On définit des *VirtualHost* dans les fichiers de conf d'Apache.  
On crée un *VirtualHost* pour chaque application web qu'héberge Apache.

> "Application Web" c'est le terme de hipster pour désigner un site web. Disons qu'aujourd'hui les sites peuvent faire tellement de trucs qu'on appelle plutôt ça une "application" à part entière. Une application web donc.

➜ Dans le dossier `/etc/httpd/` se trouve un dossier `conf.d`.  
Des dossiers qui se terminent par `.d`, vous en rencontrerez plein, ce sont des dossiers de *drop-in*.  
Plutôt que d'écrire 40000 lignes dans un seul fichier de conf, on l'éclate en plusieurs fichiers la conf.  
C'est + lisible et + facilement maintenable.

Les dossiers de *drop-in* servent à accueillir ces fichiers de conf additionels.  
Le fichier de conf principal a une ligne qui inclut tous les fichiers de conf contenus dans le dossier de *drop-in*.

---

🌞 **Analyser la conf Apache**

- mettez en évidence, dans le fichier de conf principal d'Apache, la ligne qui inclut tout ce qu'il y a dans le dossier de *drop-in*
```bash
[antho@web ~]$ cat /etc/httpd/conf/httpd.conf
# Load config files in the "/etc/httpd/conf.d" directory, if any.
IncludeOptional conf.d/*.conf
```

🌞 **Créer un VirtualHost qui accueillera NextCloud**

- créez un nouveau fichier dans le dossier de *drop-in*
  - attention, il devra être correctement nommé (l'extension) pour être inclus par le fichier de conf principal
- ce fichier devra avoir le contenu suivant :

```bash
[antho@web ~]$ sudo nano myfile.conf 

<VirtualHost *:80>
  # on précise ici le dossier qui contiendra le site : la racine Web
  DocumentRoot /var/www/nextcloud/html/  

  # ici le nom qui sera utilisé pour accéder à l'application
  ServerName  web.tp5.linux  

  <Directory /var/www/nextcloud/html/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
```

> N'oubliez pas de redémarrer le service à chaque changement de la configuration, pour que les changements prennent effet.

🌞 **Configurer la racine web**

- la racine Web, on l'a configurée dans Apache pour être le dossier `/var/www/nextcloud/html/`
- creéz ce dossier
- faites appartenir le dossier et son contenu à l'utilisateur qui lance Apache (commande `chown`, voir le [mémo commandes](../../cours/memos/commandes.md))

> Jusqu'à la fin du TP, tout le contenu de ce dossier doit appartenir à l'utilisateur qui lance Apache. C'est strictement nécessaire pour qu'Apache puisse lire le contenu, et le servir aux clients.
```bash
[antho@web ~]$  sudo nano /etc/opt/remi/php74/php.ini 
[antho@web ~]$ cd /var/www/ 
[antho@web www]$ sudo mkdir nextcloud
[sudo] Mot de passe de antho : 
[antho@web www]$ cd nextcloud/
[antho@web nextcloud]$ sudo mkdir html
[antho@web nextcloud]$ ls 
html
[antho@web nextcloud]$ sudo chown -R apache /var/www/nextcloud/html/ 
```

🌞 **Configurer PHP**

- dans l'install de NextCloud, PHP a besoin de conaître votre timezone (fuseau horaire)
- pour récupérer la timezone actuelle de la machine, utilisez la commande `timedatectl` (sans argument)
- modifiez le fichier `/etc/opt/remi/php74/php.ini` :
  - changez la ligne `;date.timezone =`
  - par `date.timezone = "<VOTRE_TIMEZONE>"`
  - par exemple `date.timezone = "Europe/Paris"`
```bash
[antho@web ~]$ sudo nano /etc/opt/remi/php74/php.ini 
    [Date]
    ; Defines the default timezone used by the date functions
    ; http://php.net/date.timezone
    ;date.timezone = "Europe/Paris" 
```

## 3. Install NextCloud

On dit "installer NextCloud" mais en fait c'est juste récupérer les fichiers PHP, HTML, JS, etc... qui constituent NextCloud, et les mettre dans le dossier de la racine web.

🌞 **Récupérer Nextcloud**

```bash
[antho@web nextcloud]$ cd
[antho@web ~]$  curl -SLO https://download.nextcloud.com/server/releases/nextcloud-21.0.1.zip 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (60) SSL certificate problem: certificate is not yet valid
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
[antho@web ~]$ ls
myfile.conf
[antho@web ~]$ curl -SLO https://download.nextcloud.com/server/releases/nextcloud-21.0.1.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
curl: (60) SSL certificate problem: certificate is not yet valid
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
[antho@web ~]$ ls
myfile.conf

```

