Connecting to Relational Databases on Heroku with Java
======================================================

Applications on Heroku can use a variety of relational database services including the [Postgres database](http://devcenter.heroku.com/articles/heroku-postgres-documentation) offered by Heroku and [MySQL](http://devcenter.heroku.com/articles/amazon_rds) offered by AWS. Databases are provisioned using the add-on system. Some applications will have a small, free postgres database provisioned by default. You can check this by running

    :::term
    $ heroku info
    === sparkling-wine-2003
    Web URL:        http://sparkling-wine-2003.herokuapp.com/
    Git Repo:       git@heroku.com:sparkling-wine-2003.git
    Repo size:      21M
    Slug size:      916k
    Stack:          cedar
    Data size:      (empty)
    Addons:         Shared Database 5MB
    Owner:          jesper@heroku.com

and look for the "Shared Database 5MB" under Addons. It depends on the buildpack whether a database is provisioned automatically. You can provision the shared database manually with

    $ heroku addons:add shared-database:5mb
    
The shared database is meant for testing purposes. For a production application you must use one of the production quality SQL database services:

* [Heroku PostgresQL](http://devcenter.heroku.com/articles/heroku-postgres-documentation)
* [Amazon RDS](http://devcenter.heroku.com/articles/amazon_rds)

Once you have provisioned a relational database to your application. The application reads the database connection information from the `DATABASE_URL` config variable. It is formatted like this:

    [database type]://[username]:[password]@[host]/[database name]

For instance:

    postgres://foo:foo@heroku.com/hellodb

You can see the `DATABASE_URL` provided to an application by running:

    :::term
    $ heroku config
    DATABASE_URL            => postgres://foo:foo@heroku.com/hellodb

It is not recommended to copy this value into a static file since the environment may change the value.  Instead an application should read the `DATABASE_URL` environment variable and setup the database connections based on that information.

Using the `DATABASE_URL` in a Play! Framework App
-------------------------------------------------

Play! Framework supports the `DATABASE_URL` environment variable out-of-the box. The built-in ORM framework will automatically use this variable if it is present, so there is no need for any additional configuration.


Using the `DATABASE_URL` in plain JDBC
------------------------------------

To instantiate a JDBC connection in your code, you can use a method like this:

    :::java
    private static Connection getConnection() throws URISyntaxException, SQLException {
        URI dbUri = new URI(System.getenv("DATABASE_URL"));

        String username = dbUri.getUserInfo().split(":")[0];
        String password = dbUri.getUserInfo().split(":")[1];
        String dbUrl = "jdbc:postgresql://" + dbUri.getHost() + dbUri.getPath();

        return DriverManager.getConnection(dbUrl, username, password);
    }


Using the `DATABASE_URL` in Spring with XML configuration
-------------------------------------------------------

This snippet of Spring XML configuration will setup a `BasicDataSource` from the `DATABASE_URL` and can then be used with Hibernate, JPA, etc:

    :::xml
    <bean class="java.net.URI" id="dbUrl">
        <constructor-arg value="#{systemEnvironment['DATABASE_URL']}"/>
    </bean>

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="url" value="#{ 'jdbc:postgresql://' + @dbUrl.getHost() + @dbUrl.getPath() }"/>
        <property name="username" value="#{ @dbUrl.getUserInfo().split(':')[0] }"/>
        <property name="password" value="#{ @dbUrl.getUserInfo().split(':')[1] }"/>
    </bean>


Using the `DATABASE_URL` in Spring with Java configuration
--------------------------------------------------------

Alternatively you can use Java for configuration of the `BasicDataSource` in Spring:

    :::java
    @Configuration
    public class MainConfig {

        @Bean
        public BasicDataSource dataSource() throws URISyntaxException {
            URI dbUri = new URI(System.getenv("DATABASE_URL"));

            String username = dbUri.getUserInfo().split(":")[0];
            String password = dbUri.getUserInfo().split(":")[1];
            String dbUrl = "jdbc:postgresql://" + dbUri.getHost() + dbUri.getPath() + ":" + dbUri.getPort() + dbUri.getPath();

            BasicDataSource basicDataSource = new BasicDataSource();
            basicDataSource.setUrl(dbUrl);
            basicDataSource.setUsername(username);
            basicDataSource.setPassword(password);

            return basicDataSource;
        }
    }

Connecting to a dedicated database remotely
-------------------------------------------

If you're using a Heroku Postgres dedicated database you can connect to it remotely for maintenance and debugging purposes. However doing so requires that you use an SSL connection. This can be done with JDBC.

your JDBC connection URL will need to have the following parameters:

    jdbc:postgresql://host/database?user=user&password=password&ssl=true&sslfactory=org.postgresql.ssl.NonValidatingFactory

If you leave off ssl=true you will get a connection error. If you leave off sslfactory=org.postgresql.ssl.NonValidatingFactory you may get an error like:

    unable to find valid certification path to requested target

Sample project
--------------

A sample project illustrating these three methods of setting up a database connection on Heroku can be found at:
[https://github.com/heroku/devcenter-java-database](https://github.com/heroku/devcenter-java-database)

To try it out first clone the git repository:

    git clone git://github.com/heroku/devcenter-java-database.git

In the `devcenter-java-database` directory run the Maven build for the project:

    mvn package

If you have a local Postgres database and want to test things locally, first set the `DATABASE_URL` environment variable (using the correct values):

* On Linux/Mac:

        export DATABASE_URL=postgres://foo:foo@localhost/hellodb

* On Windows:

        set DATABASE_URL=postgres://foo:foo@localhost/hellodb

To run the example applications locally, execute the generated start scripts:

* On Linux/Mac:

        sh devcenter-java-database-plain-jdbc/target/bin/main
        sh devcenter-java-database-spring-xml/target/bin/main
        sh devcenter-java-database-spring-java/target/bin/main

* On Windows:

        devcenter-java-database-plain-jdbc/target/bin/main.bat
        devcenter-java-database-spring-xml/target/bin/main.bat
        devcenter-java-database-spring-java/target/bin/main.bat

For each command you should see a message like the following indicating that everything worked:

    Read from DB: 2011-11-23 11:37:03.886016

To run on Heroku, first create a new application:

    :::term
    $ heroku create
    Creating stark-sword-398... done, stack is cedar
    http://stark-sword-398.herokuapp.com/ | git@heroku.com:stark-sword-398.git
    Git remote heroku added

Then deploy the application on Heroku:

    :::term
    $ git push heroku master
    Counting objects: 70, done.
    Delta compression using up to 8 threads.
    Compressing objects: 100% (21/21), done.
    Writing objects: 100% (70/70), 8.71 KiB, done.
    Total 70 (delta 14), reused 70 (delta 14)
    
    -----> Heroku receiving push
    -----> Java app detected
    -----> Installing Maven 3.0.3..... done
    -----> Installing settings.xml..... done
    -----> executing /app/tmp/repo.git/.cache/.maven/bin/mvn -B -Duser.home=/tmp/build_2y7ju7daa9t04 -Dmaven.repo.local=/app/tmp/repo.git/.cache/.m2/repository -s /app/tmp/repo.git/.cache/.m2/settings.xml -DskipTests=true clean install
       [INFO] Scanning for projects...
       [INFO] ------------------------------------------------------------------------
       [INFO] Reactor Build Order:
       [INFO] 
       [INFO] devcenter-java-database-plain-jdbc
       [INFO] devcenter-java-database-spring-xml
       [INFO] devcenter-java-database-spring-java
       [INFO] devcenter-java-database
       [INFO]                                                                         
       [INFO] ------------------------------------------------------------------------
       [INFO] Building devcenter-java-database-plain-jdbc 1.0-SNAPSHOT
       [INFO] ------------------------------------------------------------------------
       Downloading: http://s3pository.heroku.com/jvm/postgresql/postgresql/9.0-801.jdbc4/postgresql-9.0-801.jdbc4.pom
    ...

Now execute any of the examples on Heroku:

    :::term
    $ heroku run "sh devcenter-java-database-plain-jdbc/target/bin/main"
    Running sh devcenter-java-database-plain-jdbc/target/bin/main attached to terminal... up, run.1
    Read from DB: 2011-11-29 20:36:25.001468

Further Learning
----------------

[Learn about the shared and dedicated database options](http://devcenter.heroku.com/articles/database)
