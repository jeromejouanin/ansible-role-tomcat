# ansible-role-tomcat
An Ansible Role that install and manage an Apache Tomcat server on Debian/RedHat VM.

Page index :

[[_TOC_]]

## Description

Ce role offre les fonctionnalités suivantes :
- compatibilité Debian / RedHat
- création d'un keystore au format PKCS12 avec un certif + AC
- installation/désinstallation de tomcat à partir du téléchargement de apache-tomcat sur le repo officiel
- validé pour tomcat versions 8 et 9
- installation/désinstallation d'instances multiples sur une même VM
- protocole HTTPS entre apache et tomcat (utilisant le keystore)
- protocole AJP désactivé
- option tomcat_vault pour gérer les secrets (keystore java tomcat + applicatif bdd) dans un coffre fort vault
- variabilisation d'éléments de configuration :
  - ports
  - version
  - java_Xm_conf
  - tomcat_ssl_protocols
  - tomcat_maxThreads
  - tomcat_minSpareThreads
  - tomcat_connectionTimeout
- installation jprofiler
- mode réinstallation
- mode local (pré requis : accès depuis la machine qui lance le playbook à des URL locales
pour les livrables nécessaires : apache-tomcat, jprofiler éventuellement)
- ...

## Pre requisites and dependencies
This ansible role install tomcat and jprofiler components from their standalone archives.
Tomcat 8.5 is designed to run on Java 7 and later.
Java adoptium (or ex adoptopenjdk) is a pre requisite. It probably works with other openjdk variants (but not tested).
Vault functionality needs a self compiled jar from https://github.com/web-servers/tomcat-vault in files dir,
see the vault section below ..

## TODO
- playbook call complexes examples
- comments traduction in english
- keystore management to clarify
- update and tests : java, tomcat, jprofiler
- test of vault functionality for java >9
- deploy app war integration
- ...

## Basic install plan
- destruction d'une instance  (when: reinstall_software)
- tomcat_install_catalina_base.yml
- vault configuration when: tomcat_vault (false par défaut)
- keystore_management.yml (attention pré requis que les certificats soient présents sur la VM, role certificate_management)
- install jprofiler when: jprofiler_install (true par défaut)
- tomcat_instance_create.yml

## Mandatory parameters
```yaml
  client:
  environnement:
  server_type:
  apache_servername:
```

## Parameters list
TODO

## Configuration et prérequis de l'option mode local (pour VM sans accès à internet)
Pré-requis :
Le mode local requiert de disposer des livrables - nécessaires au bon fonctionnement d'une application webWeb ou WS,
c-à-d:  apache-tomcat, jprofiler éventuellement et apache -
sur un serveur du réseau local accessible au serveur sur lequel le playbook ansible est exécuté,
ex: un serveur d'archive nexus ou artifactory local.

Après quoi on peut configurer les URL locales dans les variables suivantes, exemple de configuration :
```yaml
  # vars mode local / no internet
  install_mode: local
  java_home: /usr/lib/jvm/temurin-8-jdk
  tomcat_minor_version: 8.5.88
  tomcat_dl_url: "https://.../apache-tomcat-8.5.88.tar.gz"
  jprofiler_dl_url: "https://.../jprofiler_linux_12_0_4.tar.gz"
```

## Informations supplémentaires
Par défaut, on installe la version Tomcat 8.5.88.

Les répertoires permettant l'externalisation des fichiers de configuration de l'applicatif sont créés.

Ils auront le format suivant : 

/etc/<server-type>_<tomcat_instance_num>.
server-type peut prendre les valeurs suivantes: web ou ws

Pour que ces fichiers de propriétés soient pris en compte au démarrage de l'instance Tomcat,
un fichier setenv.sh sera créé dans le répertoire /opt/tomcat/<version>/<server-type>_<tomcat_instance_num>/bin
à l'aide d'un template jinja.

Ainsi selon le type de serveur applicatif (web ou ws),
le contenu du fichier setenv.sh changera.

Les répertoires pour l'installation de tomcat auront le format suivant :
/opt/tomcat/<version>/<server-type>_<tomcat_instance_num> avec version : 8.5 ou 9.0

Les noms des services tomcat auront le formalisme suivant :
tomcat_<server-type>_<tomcat_instance_num>

Exemple d'A/R: 

Pour un tomcat web, 
```code 
sudo systemctl restart tomcat_web_1 
```
Pour un tomcat webservice,
```code 
sudo systemctl restart tomcat_ws_1
```

## Configuration et prérequis de l'option tomcat_vault pour gérer les secrets dans un coffre fort vault

Une fonctionnalité de chiffrement/déchiffrement des secrets a été intégrée dans ce role.
Il s'agit de https://github.com/web-servers/tomcat-vault,
projet mis à disposition par une personne qui travaille chez RedHat jfclere@gmail.com,
fork de la fonctionnalité vault dispo dans jboss/wildfly (JWS5 version 5.7 pour l'instant qui utilise tomcat9 9.0.62).

Le présent role s'attend donc à disposer d'un jar compilé manuellement dans le répertoire files.

**Pour activer ce mode, mettre la variable tomcat_vault: true dans l'inventaire.**
Cette option requiert une réinstallation.

Un nouveau keystore sera créé ($catalina_base/vault.keystore), que le server tomcat va utiliser afin de pouvoir
déchiffrer le coffre-fort ($catalina_base/vault-enc-dir/VAULT.dat).

Lors de l'installation d'une instance tomcat on crée un keystore et un coffre fort en y inscrivant les secrets suivants :
- VAULT::JAVA::keystore_pwd::$value
- VAULT::DATABASE::oracle_password::$value

Ces valeurs peuvent être utilisées dans des fichiers xml de scope tomcat SYSTEM_PROPERTIES avec une syntaxe du type :
  certificateKeystorePassword="${VAULT::JAVA::keystore_pwd::}

Le serveur tomcat est en mesure de fournir ces valeurs mais 
ni les fichiers var=value (application.properties) ni les fichiers xml embarqués dans le war
ne sont compatibles avec ce mécanisme.
L'astuce est qu'ils sont passés aux vars des webapps par des context-param dans le web.xml. Exemple :
```code
<context-param>
  <param-name>oracle.password</param-name>
  <param-value>${VAULT::DATABASE::oracle_password::}</param-value>
</context-param>
```
Le code ansible s'occupe de ça lorsqu'on active ce mode tomcat_vault.

Sur la VM cible, il n'y a plus de secret en clair et le système est autonome,
ne dépendant pas de mécanismes extérieurs pour fonctionner.

Le secret du keystore vault (permettant de récupérer la clé privée) se trouve dans le fichier vars/keystore.yml
du playbook, qui est lui même chiffré avec vault et accessible par un master secret
d'administrateur sur les VM controleurs ansible, utilisé pour le déploiement.

Pour le moment le secret du keystore vault est commun aux installations,
mais rien n'empêche d'en créer des différents, il suffit par exemple de déclarer la variable dans les inventaires.
Attention dans ce cas là à bien déplacer la var vault_keystore_pwd du playbook vers les inventaires concernés
car les vars de playbooks dans vars/ prévalent sur celles de l'inventaire.

Pour consulter le keystore, voici la commande :
```code
keytool -keystore vault.keystore -storetype jceks -list     # (saisir Entrée 2x)
[...]
vault_tomcat_web_2, 16 mars 2023, SecretKeyEntry,
[...]
```

Encore plus d'information ici :
- https://github.com/web-servers/tomcat-vault (il est convenu avec le RSSI de s'abonner à ce projet pour info..)
