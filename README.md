# IBM WebSphere Application Server Liberty Cartridge

Provides the Liberty profile server on OpenShift. This cartridge makes use of the [IBM WebSphere Application Server Liberty Buildpack][] for Cloud Foundry to provide a more consistent experience between clouds.

It is provided unsupported, for development use only.

## Accepting the Liberty and JVM Licenses

To deploy applications using the IBM WebSphere Application Server Liberty Cartridge, you are required to accept the development IBM Liberty license by following the instructions below:

1. Read the current IBM [Liberty-License][].
2. Extract the `D/N: <License code>` from the Liberty-License.
3. Set the IBM_LIBERTY_LICENSE environment variable to the extracted license code when you create your application (this must be done using the rhc command-line client because the web UI does not provide a way). 

To use OpenJDK instead of the IBM JRE set the JVM=openjdk environment variable. This is recommended because the IBM JRE download and install is currently slower and sometimes is timed out by OpenShift. If you do wish to use IBM JRE, then you are also required to accept its license by following the instructions below:

1. Read the current IBM [JVM-License][].
2. Extract the `D/N: <License code>` from the JVM-License.
3. Set the IBM_JVM_LICENSE environment variables to the extracted license code when you create your application (this must be done using the rhc command-line client because the web UI does not provide a way).


## Layout and Deployment Options

There are multiple options for deploying applications on Liberty in OpenShift. In the basic workflows below, each operation will require associated git add/commit/push operations to take effect.

For methods 1 and 2, the server.xml will be automatically generated when the application changes or an additional database cartridge is added/removed and the application is restarted (see [Database Auto-configuration](#database-auto-configuration)). The context-root is `/`. For methods 3 and 4 you are providing your own server.xml. The cartridge will only attempt to modify it enough so that the server will be able to start in the OpenShift environment. For more details see [Buildpack-enabled Options for Server.xml][].

### Method 1

You can upload your content in a Maven src structure as in the sample project and on git push have the application built and deployed. For this to work you'll need your pom.xml at the root of your repository and a maven-war-plugin like in this sample to move the output from the build to the apps/ directory.

### Method 2

You can git push a pre-built WAR or EAR.

1. Add new zipped content and deploy it: 

  a. `cp target/example.war ./`

  b. Delete other app files in your git repository because they will be overriden by the WAR or EAR file when deployed

2. Undeploy currently deployed content: `git rm old.war`

### Method 3

You can git push a Liberty server package.

1. Add new zipped content and deploy it:

  a. Run: `wlp/bin/server package --include=usr`

  b. Run: `cp wlp/usr/servers/<server name>/<server name>.zip ./`

  c. Delete other app files in your git repository because they will be overriden by the server package when deployed

### Method 4

You can git push a Liberty server directory.

1. Create a new git repository in the server directory, then deploy:

  a. `cd wlp/usr/servers/<server name>`
  
  b. `git init`
  
  c. `rhc app-show`
  
  d. `git remote add openshift <git remote from previous command>`
  
  e. `git pull openshift master`
  
  f. Delete any extra files that are not part of the server directory you want to deploy
  
  g. `git add -A .`
  
  h. `git commit -m "descriptive message"`
  
  i. `git push openshift master`
  
  
## Database Auto-configuration

The Liberty Buildpack performs auto-configuration for a subset of services and this cartridge utilizes that. In OpenShift, that means auto-configuration for the database cartridges for MongoDB, PostgreSQL, and MySQL. The buildpack will automatically download appropriate client drivers and update the server.xml configuration file with the right information for a given service.

### Service Names

OpenShift does not provide a built-in way to associate a custom name with added catridges. However, the Liberty Buildpack uses custom service names during auto-configuration to match a service to the right configuration elements in the server configuration or when generating service variables. 

In OpenShift the default names will come from the URL environment variable exposed by the database cartridge, so these would be `mongodb`, `postgresql`, and `mysql` (from `OPENSHIFT_MONGODB_DB_URL`, `OPENSHIFT_POSTGRESQL_DB_URL`, and `OPENSHIFT_MYSQL_DB_URL`). 

With the Liberty Buildpack you can associate a custom name with a service by setting the `SERVICE_NAME_MAP` environment variable. The `SERVICE_NAME_MAP` environment variable has the following syntax:
```
SERVICE_NAME_MAP=variableName=customName[,variableNameN=customNameN]*
```

The `variableName` is the URL environment variable exposed by the given service and the `customName` is the user-defined name for the service.

### Opting Out

Opt-out of service auto-configuration by setting the `services_autoconfig_excludes` environment variable. The `services_autoconfig_excludes` environment variable has the following syntax:
```
services_autoconfig_excludes=variableName=excludeType[ variableName=excludeType]*
```

The `variableName` is the URL environment variable exposed by the given service and the `excludeType` is set to:
* `all` - indicates opting out of all automatic configuration for the service.
* `conifg` - indicates opting out of configuration updates only.


## rhc Examples

See the [User Guide][] for more details.

Example of creating a scalable app with a downloadable cartridge at OpenShift Online:

```bash
rhc create-app <app name> http://cartreflect-claytondev.rhcloud.com/reflect?github=opiethehokie/openshift-liberty-cartridge -s -e IBM_LIBERTY_LICENSE=<liberty license code> -e IBM_JVM_LICENSE=<jre license code>
rhc create-app <app name> http://cartreflect-claytondev.rhcloud.com/reflect?github=opiethehokie/openshift-liberty-cartridge -s -e IBM_LIBERTY_LICENSE=<liberty license code> -e JVM=openjdk --from-code https://github.com/WASdev/sample.acmeair.git
```

Example of adding a database cartridge, and triggering an auto-generated server.xml update using a custom JNDI name of `psdatasource`:

```bash
rhc cartridge-add postgresql-9.2
rhc set-env SERVICE_NAME_MAP="OPENSHIFT_POSTGRESQL_DB_URL=psdatasource"
rhc app-restart
```

Example of opting-out of all auto-configuration for the previously created PostgreSQL database:

```bash
rhc set-env services_autoconfig_excludes="OPENSHIFT_POSTGRESQL_DB_URL=all"
```

Examples of tailing logs and server config:

```bash
rhc tail --opts "-n 50"
rhc tail -f liberty/logs/ffdc/*
rhc tail -f liberty/droplet/.liberty/usr/servers/defaultServer/server.xml
```


## Markers

Adding marker files to .openshift/markers will have the following effects:

| Marker               | Effect
| -------------------- | --------------------------------------------------
| enable_jpda          | Enable remote debug of code running inside the Liberty server.
| skip_maven_build     | Maven build step will be skipped.
| force_clean_build    | Will start the build process by removing all non-essential Maven dependencies. Any current dependencies specified in your pom.xml file will then be re-downloaded.
| hot_deploy           | Will prevent a Liberty container restart during build/deployment.
| disable_auto_scaling | Disables the auto-scaling provided by OpenShift.


## Environment Variables

| Variable  | Description    
| ----------| ----------------------------------------
| JVM_ARGS  | A list of JVM command line options
| JAVA_OPTS | Appended to JVM_ARGS


## Developing an Application in Eclipse

See [Getting started with PaaS Eclipse integration][]. If you are using OpenShift Online this cartridge will not be available when you are creating an app in Eclipse. Create the app with rhc then use the existing application in Eclipse. It may be necessary to increase the Git remote connection timeout at Window -> Preferences -> Team -> Git.

The OpenShift Eclipse plugin, JBoss Tools, can co-exist with with WebSphere Development Tools (WDT). The port-forwarding provided by OpenShift also provides a way to run the app locally on Liberty while still using your databases in the cloud, and to remotely debug an app running in the cloud.

The [openshift-jrebel-cartridge][] and the Jenkins cartridge have also been tested with this cartirdge.


## Remote JMX Connections

For the simplest configuration that will work, add the following to your server.xml when deploying a server directory or server package (replacing `user` and `pass` with your own values):

```xml
<features>
    <feature>restConnector-1.0</feature>
</features>

<quickStartSecurity userName="user" userPassword="pass"/>

<keyStore id="defaultKeyStore" password="Liberty"/>

<webContainer httpsIndicatorHeader="X-Forwarded-Proto" />
```

This loads the server's JMX REST connector, sets the server's keystore, creates a single administrator user role, and sets the header that indicates a HTTPS connection (because SSL is terminated before the application). See [Configuring secure JMX connection to the Liberty profile][] for more detailed documentation. 

Then run the following commands (replacing the values in <>):

```bash
rhc scp <app name> download <local dir> liberty/servers/defaultServer/resources/security/key.jks
rhc port-forward
jconsole -J-Djava.class.path="%JAVA_HOME%/lib/jconsole.jar;%JAVA_HOME%/lib/tools.jar;%WLP_HOME%/clients/restConnector.jar" -J-Dcom.ibm.ws.jmx.connector.client.disableURLHostnameVerification=true -J-Djavax.net.ssl.trustStore=<local path>/key.jks -J-Djavax.net.ssl.trustStorePassword=Liberty -J-Djavax.net.ssl.trustStoreType=jks
```

The URL JConsole is looking for will then be `service:jmx:rest://localhost:port/IBMJMXConnectorREST` (get port value from rhc port-forward output, the default is 9443) and the user and password are the values you added to your server.xml.

If the application is scaled you can list the gears with `rhc app show <app name> --gears`. Then for the secondary gears you'll have to use regular scp (not rhc scp) to get key.jks from the listed SSH URL and use `rhc port-forward -g <gear id>`.


## Troubleshooting

Relative to your gear's home directory, the following locations may be useful:
- `liberty/droplet/.liberty` – Liberty profile and your application files
-	`liberty/droplet/.liberty/usr/servers/defaultServer` – The user/custom configuration directory used to store shared and server-specific configuration, i.e. WLP_USER_DIR
-	`liberty/droplet/.java` – JRE used to run your Liberty server, i.e. JAVA_HOME
-	`liberty/logs` – The directory containing output files for defined servers, i.e. WLP_OUTPUT_DIR

### Buildpack

If the problem is in the Liberty Buildpack, try enabling and viewing its log:

```bash
rhc set-env JBP_LOG_LEVEL=(DEBUG|INFO|WARN|ERROR|FATAL)
rhc tail -f liberty/droplet/.buildpack-diagnostics/buildpack.log
```

### Dumps

See [How to generate javacores, heapdumps and system cores for the WebSphere Application Server V8.5 Liberty profile][].


## Limitations

1. The [Buildpack Restrictions][] 1-3 still apply to OpenShift, 4-6 are not a limitation in OpenShift.
2. This cartridge has not been tested with all the features of the Liberty Buildpack.
3. The JDK used to build an application deployed from source is already provided on OpenShift gears, but is not what runs the application. The Liberty Buildpack downloads the IBM JRE or OpenJDK JRE and that is used to run the application.


[Liberty-License]: http://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wlp/8.5.5.2/lafiles/runtime/en.html
[JVM-License]: http://www14.software.ibm.com/cgi-bin/weblap/lap.pl?la_formnum=&li_formnum=L-AWON-8GALN9&title=IBM%C2%AE+SDK%2C+Java-+Technology+Edition%2C+Version+7.0&l=en
[Getting started with PaaS Eclipse integration]: https://www.openshift.com/blogs/getting-started-with-eclipse-paas-integration
[action hooks documentation]: http://openshift.github.io/documentation/oo_user_guide.html#action-hooks
[User Guide]: http://openshift.github.io/documentation/oo_user_guide.html
[Configuring secure JMX connection to the Liberty profile]: http://www-01.ibm.com/support/knowledgecenter/SSAW57_8.5.5/com.ibm.websphere.wlp.nd.doc/ae/twlp_admin_restconnector.html?cp=SSAW57_8.5.5%2F1-3-11-0-3-3-9-1&lang=en
[openshift-jrebel-cartridge]: https://github.com/openshift-cartridges/openshift-jrebel-cartridge
[IBM WebSphere Application Server Liberty Buildpack]: https://github.com/cloudfoundry/ibm-websphere-liberty-buildpack
[Buildpack-enabled Options for Server.xml]: https://github.com/cloudfoundry/ibm-websphere-liberty-buildpack/blob/master/docs/server-xml-options.md
[Buildpack Restrictions]: https://github.com/cloudfoundry/ibm-websphere-liberty-buildpack/blob/master/docs/restrictions.md
[How to generate javacores, heapdumps and system cores for the WebSphere Application Server V8.5 Liberty profile]: http://www-01.ibm.com/support/docview.wss?uid=swg21597830
