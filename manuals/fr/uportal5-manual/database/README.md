# Configuration de la Base de Données

uPortal est configuré pour utiliser une base de données HSQL par défaut.

**Cette configuration de base de données ne convient pas aux déploiements de production mais est mieux adaptée à des fins de test.**

uPortal prend en charge un certain nombre de bases de données de production populaires et vous pouvez configurer la base de données en suivant les exemples publiés sous Configuration de la base de données de production.

## Étape 1 : Configurer la version du pilote de base de données

Après avoir déterminé les coordonnées Maven du pilote, ouvrez le fichier `gradle.properties` et ajoutez les coordonnées de version du pilote en tant que valeur de propriété.

Par exemple, la version du pilote MS SQL Server est configurée dans `mssqlJdbcVersion` ci-dessous:

```groovy
jasyptVersion=1.9.2
mssqlJdbcVersion=6.2.1.jre8
personDirectoryVersion=1.8.5
```

## Étape 2 : Ajouter la dépendance du pilote de base de données

Ouvrir le fichier `overlays/build.gradle` et ajouter les coordonnées du pilote sous
les coordonnées hsqldb autour de la ligne 46. Assurez-vous d'utiliser la propriété version définie
à la première étape.

À titre d'exemple, un pilote pour SQL Server est ajouté :

```groovy
    dependencies {
        /*
         * Add additional JDBC driver jars to the 'jdbc' configuration below;
         * do not remove the hsqldb driver jar that is already listed.
         */
        jdbc "org.hsqldb:hsqldb:${hsqldbVersion}"
        jdbc "com.microsoft.sqlserver:mssql-jdbc:${mssqlJdbcVersion}"

        ...
    }
```

## Étape 3 : Saisir quelques détails génériques

Bien que les informations d'identification et l'URL de la base de données ne doivent pas être enregistrées dans votre référentiel, la classe de pilote (<i>driver class</i>), le dialecte (<i>dialect</i>) et la requête de validation (<i>validation query</i>)
peuvent généralement être conservés sans problèmes de sécurité.

Dans `etc/portal/global.properties`, enregistrez les détails de la base de données qui sont cohérents entre les environnements:

```groovy
environment.build.hibernate.connection.driver_class=com.microsoft.sqlserver.jdbc.SQLServerDriver
environment.build.hibernate.connection.url=jdbc:sqlserver://localhost:1433;
environment.build.hibernate.connection.username=sa
environment.build.hibernate.connection.password=
environment.build.hibernate.dialect=org.hibernate.dialect.SQLServerDialect
environment.build.hibernate.connection.validationQuery=select 1
```

## Étape 4 : Copier `global.properties` dans l'emplacement de l'environnement local et ajouter les informations d'identification et l'URL

Dans uPortal 5, les déployeurs sont fortement encouragés à configurer un répertoire `portal.home` local pour conserver une configuration
spécifique à leur environnement mais qui ne devrait pas être saisie dans un repo. En particulier, la base de données et les autres
credentials de service ne doivent pas être saisis. Si `portal.home` n'est pas configuré, la valeur par défaut est le répertoire `portal/` dans Tomcat.

Pendant le tâches `./gradlew portalInit` ou `./gradlew tomcatInstall`, les fichiers du répertoire `etc/portal/` du repo sont
copiés dans `portal.home`. L'une de ces deux tâches est une condition préalable à cette étape.

Dans `global.properties` du répertoire `portal.home`, éditer les détails de connexion:

```groovy
environment.build.hibernate.connection.driver_class=com.microsoft.sqlserver.jdbc.SQLServerDriver
environment.build.hibernate.connection.url=[actual URL for this server]
environment.build.hibernate.connection.username=[actual user for this db]
environment.build.hibernate.connection.password=[actual password for this db]
environment.build.hibernate.dialect=org.hibernate.dialect.SQLServerDialect
environment.build.hibernate.connection.validationQuery=select 1
```

## Étape 5: Configuration spécifique portlet / uPortal (optionnel)

La configuration utilisée par défaut pour déployer toutes les applications vient du fichier `global.properties` dans le répertoire `portal.home`.
Mais il est tout à fait possible de définir une configuration par application/portlet, le fichier `global.properties` sera toujours utilisé mais il peut être surchargé par un fichier spécifique s'il est trouvé.

Pour la base de données uPortal il sera nécessaire de recopier les mêmes propriétés de base de données du fichier `global.properties` dans le fichier `uPortal.properties`.
Pour chaque portlet il faudra aussi redéfinir les mêmes propriétés en les ajoutant dans un fichier `specific-portlet.properties` du répertoire `portal.home`, où `specific-portlet.properties` est le nom du fichier défini dans les source de configurations de contexte spring du portlet.
Par exemple, pour `NewsReaderPortlet` le fichier sera `news-reader.properties`, le nom du fichier à définir se trouvera [dans le projet NewsReaderPortlet ici.](https://github.com/Jasig/NewsReaderPortlet/blob/master/src/main/resources/context/databaseContext.xml)

Remarque: Aussi ces fichiers peuvent être utilisés pour définir d'autres propriétés !

## Configuration de la base de données de production uPortal

Sélectionner la base de données ci-dessous pour des notes et des exemples de configuration.

- [DB2](db2.md)
- [Hypersonic](hypersonic.md)
- [Microsoft SQL Server](ms-sqlserver.md)
- [MySQL](mysql.md)
- [MariaDB](mariadb.md)
- [Oracle RDBMS](oracle.md)
- [PostgreSQL](postgresql.md)
- [Sybase](sybase.md)
