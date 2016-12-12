# Cerberus Management Service

[![][travis img]][travis]
[![][license img]][license]

The Cerberus Management Service (CMS) is a core component of the Cerberus [REST API](http://engineering.nike.com/cerberus/docs/architecture/rest-api) 
that facilities user and AWS IAM role authentication and the management of Safe Deposit Boxes (SDBs), an abstraction on top of Hashicorp's Vault.

To learn more about Cerberus, please visit the [Cerberus website](http://engineering.nike.com/cerberus/).

## Getting Started

**MySQL** is required to run the application locally.

To get MySQL setup on OS X:

    $ brew install mysql
    $ mysql.server restart
    $ mysql_secure_installation

You'll need to create a database and user for it.  Run the following SQL against your mysql database:

    CREATE DATABASE IF NOT EXISTS cms;
    
    CREATE USER 'cms'@'localhost' IDENTIFIED BY '<YOUR DB PASSWORD HERE>';
    
    GRANT ALL ON cms.* TO 'cms'@'localhost';

**Vault** is required to run the application locally.

To get Vault setup on OS X:

    $ brew install vault
    $ vault server -dev

This will output a root token, you can now setup vault using the supplied script:

    $ /path/to/project/vault-setup.sh

That will setup the default policy and generate a token for CMS and output:

    export VAULT_ADDR="http://localhost:8200"
    export VAULT_TOKEN="<token>"

## Configuration 

### Configurable Properties

There are a few parameters that need to be configured for CMS to run properly, they are defined in this table. 

property                    | required | notes
--------------------------- | -------- | ----------
JDBC.url                    | Yes | The JDBC url for the mysql db
JDBC.username               | Yes | The JDBC user name for the mysql db
JDBC.password               | Yes | The JDBC JDBC.password for the mysql db
root.user.arn               | Yes | The arn for the root AWS user, needed to make the KMS keys deletable.
admin.role.arn              | Yes | The arn for an AWS user, needed to make the KMS keys deletable.
cms.role.arn                | Yes | The arn for the Instance profile for CMS instances, so they can admin KMS keys that they create.
cms.admin.group             | Yes | Group that user can be identified by to get admin privileges, currently this just enables users to access `/v1/stats` see API.md
cms.auth.connector          | Yes | The user authentication connector implementation to use for user auth.
cms.user.token.ttl.override | No  | By default user tokens are created with a TTL of 1h, you can override that with this param
cms.iam.token.ttl.override  | No  | By default IAM tokens are created with a TTL of 1h, you can override that with this param

For local dev see `Running CMS Locally`.

For deployed environments they are configured via the CLI, which will generate a props file and stuff it into S3 encrypted with KMS.

    cerberus --debug \
    -e demo \
    -r us-west-2 \
    create-cms-config \
    --admin-group cerberus-admins \
    -P cms.auth.connector=com.nike.cerberus.auth.connector.onelogin.OneLoginAuthConnector \
    -P auth.connector.onelogin.api_region=us \
    -P auth.connector.onelogin.client_id=$ONE_LOGIN_CLIENT_ID \
    -P auth.connector.onelogin.client_secret=$ONE_LOGIN_CLIENT_SECRET \
    -P auth.connector.onelogin.subdomain=nike
    
See [Creating an environment](http://engineering.nike.com/cerberus/docs/administration-guide/creating-an-environment) for more information.
    
CMS will download the props file at startup time and load the props into Guice.

### User Authentication

#### Auth Connector Interface

The User authentication contract is defined by the [AuthConnector](https://github.com/Nike-Inc/cerberus-management-service/blob/master/src/main/java/com/nike/cerberus/auth/connector/AuthConnector.java) interface.  

The only included implementation of this interface targets 
OneLogin.  We expect to implement more connectors in the near future.

##### OneLogin Auth Connector

property                              | required | notes
------------------------------------- | -------- | ----------
auth.connector.onelogin.api_region    | Yes      | `us` or `eu`
auth.connector.onelogin.client_id     | Yes      | The OneLogin API client id
auth.connector.onelogin.client_secret | Yes      | The OneLogin API client secret
auth.connector.onelogin.subdomain     | Yes      | Your orgs OneLogin subdomain [xxxxx].onelogin.com
    
**Assumption: The current implementation looks up group membership for a user via the member_of field on the getUserById API response.**
    
## Running CMS Locally

First, a few properties must be configured in `cms-core-code/src/main/resources/cms-local-overrides.conf`

You'll need a few pieces of information before you can run the application:
 
- The DB password you setup earlier
- The group that identifies which users are administrators 
- The root user ARN for your AWS account
- The AWS IAM role ARN that represents administrators and CMS instances
- The authentication connector class that is used to authenticate users and get their group membership

```
    # Database connection details.
    JDBC.url="jdbc:mysql://localhost:3306/cms?useUnicode=true&characterEncoding=utf8&useLegacyDatetimeCode=false&serverTimezone=UTC&useSSL=false"
    JDBC.username="cms"
    JDBC.password="<YOUR DB PASSWORD HERE>"
    
    # Group that user can be identified by to get admin privileges.
    cms.admin.group="<YOUR ADMIN GROUP>"
    
    # AWS ARNs used when setting up KMS keys for IAM role authentication.
    root.user.arn="arn:aws:iam::<YOUR AWS ACCOUNT ID>:root"
    admin.role.arn="arn:aws:iam::<YOUR AWS ACCOUNT ID>:role/<YOUR IAM ROLE FOR ADMINS>"
    cms.role.arn="arn:aws:iam::<YOUR AWS ACCOUNT ID>:role/<YOUR IAM ROLE FOR CMS>"
    
    # Auth Connector
    cms.auth.connector=<YOUR AUTH CONNECTOR CLASS>
```

### From the IDE
 
Simply run `com.nike.cerberus.Main`.  The following VM arguments should be set:

    -D@appId=cms -D@environment=local -Dvault.addr=http://localhost:8200 -Dvault.token=<token>

Note that if the VAULT_ADDR and VAULT_TOKEN environment variables are set, you don't need vault.addr or vault.token VM arguments.

### From the CLI

    ./gradlew clean build
    
    ./debugShadowJar.sh -Dvault.addr=http://localhost:8200 -Dvault.token=<token>
    
Note that if the VAULT_ADDR and VAULT_TOKEN environment variables are set, you don't need vault.addr or vault.token VM arguments.

## Setting up your IDE

Import the build.gradle file.

## API documentation

See [API.md](API.md)

## License

Cerberus Management Service is released under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)

[travis]:https://travis-ci.org/Nike-Inc/cerberus-management-service
[travis img]:https://api.travis-ci.org/Nike-Inc/cerberus-management-service.svg?branch=master

[license]:LICENSE.txt
[license img]:https://img.shields.io/badge/License-Apache%202-blue.svg

