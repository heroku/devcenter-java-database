Connecting to Relational Databases on Heroku with Java
======================================================

Applications on Heroku can use any back-end data storage system.  You can either use a data storage system provided as a Heroku add-on or do your own thing.  The data storage systems available as Heroku add-ons range from relational databases to NoSQL databases.  Some of the relational databases that are available on Heroku are the shared Postgres database service from Heroku, the enterprise-grade Postgres database service from Heroku, various MySQL third party add-on providers, and the Oracle RDMS available from Amazon RDS.  Every application you create is automatically provisioned a shared Postgres database.  But you can easily add any other database service via Heroku add-ons.

The relational database add-ons on Heroku provide the provisioned database connection information through an environment variable named `DATABASE_URL`.  Since Heroku is a polyglot platform the format of the connection information is not specific to Java and will need to be parsed for use an a Java application that connects via JDBC.  The format of the DATABASE_URL is:

    [database type]://[username]:[password]@[host]/[database name]

For instance:

    postgres://foo:foo@heroku.com/hellodb

Note: Play Framework has out-of-the-box support for the DATABASE_URL format.

You can see the `DATABASE_URL` provided to an application by running:

    heroku config

It is not recommended to copy this value into a static file since the environment may change the value.  Instead the recommendation is for an application to read the `DATABASE_URL` environment variable and setup the database connections based on that information.


Using the DATABASE_URL in Plain JDBC
------------------------------------

Here is a simple Java method that reads the `DATABASE_URL` environment variable and returns a `Connection`:

    private static Connection getConnection() throws URISyntaxException, SQLException {
        URI dbUri = new URI(System.getenv("DATABASE_URL"));

        String username = dbUri.getUserInfo().split(":")[0];
        String password = dbUri.getUserInfo().split(":")[1];
        String dbUrl = "jdbc:postgresql://" + dbUri.getHost() + dbUri.getPath();

        return DriverManager.getConnection(dbUrl, username, password);
    }


Using the DATABASE_URL in Spring with XML Configuration
-------------------------------------------------------

Here is a snippet of Spring XML configuration that will setup a `BasicDataSource` from the `DATABASE_URL` and can then be used with Hibernate, JPA, etc:

    <bean class="java.net.URI" id="dbUrl">
        <constructor-arg value="#{systemEnvironment['DATABASE_URL']}"/>
    </bean>

    <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
        <property name="url" value="#{ 'jdbc:postgresql://' + @dbUrl.getHost() + @dbUrl.getPath() }"/>
        <property name="username" value="#{ @dbUrl.getUserInfo().split(':')[0] }"/>
        <property name="password" value="#{ @dbUrl.getUserInfo().split(':')[1] }"/>
    </bean>


Using the DATABASE_URL in Spring with Java Configuration
--------------------------------------------------------

Alternatively you can use Java for configuration of the `BasicDataSource` in Spring:

    @Configuration
    public class MainConfig {

        @Bean
        public BasicDataSource dataSource() throws URISyntaxException {
            URI dbUri = new URI(System.getenv("DATABASE_URL"));

            String username = dbUri.getUserInfo().split(":")[0];
            String password = dbUri.getUserInfo().split(":")[1];
            String dbUrl = "jdbc:postgresql://" + dbUri.getHost() + dbUri.getPath();

            BasicDataSource basicDataSource = new BasicDataSource();
            basicDataSource.setUrl(dbUrl);
            basicDataSource.setUsername(username);
            basicDataSource.setPassword(password);

            return basicDataSource;
        }
    }


Sample Project
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

    heroku create -s cedar

Then deploy the application on Heroku:

    git push heroku master

Now execute the examples on Heroku:

    heroku run "sh devcenter-java-database-plain-jdbc/target/bin/main"
    heroku run "sh devcenter-java-database-spring-xml/target/bin/main"
    heroku run "sh devcenter-java-database-spring-java/target/bin/main"

Further Learning
----------------

[Learn about the shared and dedicated database options](http://devcenter.heroku.com/articles/database)