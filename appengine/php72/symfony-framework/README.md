## Run Symfony on App Engine Standard for PHP 7.2

This tutorial will walk you through how to create and deploy a Symfony project
to App Engine Standard for PHP 7.2. You will learn how to:

1. Create a [Symfony][symfony] project
1. Configure it to run in the App Engine environment
1. Deploy it to App Engine
1. Set up a [Cloud SQL][cloud-sql] database
1. Configure Doctrine to communicate with Cloud SQL

> **Note**: This repository is just a tutorial and is not a Symfony project in
  and of itself. The steps will require you to set up a new Symfony project in a
  separate directory.

## Prerequisites

1. [Create a project][create-project] in the Google Cloud Platform Console
   and make note of your project ID.
1. [Enable billing][enable-billing] for your project.
1. Install the [Google Cloud SDK](https://cloud.google.com/sdk/).

## Install

This tutorial uses the [Symfony Demo][symfony-demo] application. Run the
following command to install it:

```sh
PROJECT_DIR='symfony-on-appengine'
composer create-project symfony/symfony-demo:^1.2 $PROJECT_DIR
```

## Run

1. Run the app with the following command:

        php bin/console server:run

1. Visit [http://localhost:8000](http://localhost:8000) to see the Symfony
   Welcome page.

## Deploy

1. Remove the `scripts` section from `composer.json` in the root of your
   project. You can do this manually, or by running the following line of code
   below in the root of your Symfony project:

   ```sh
   php -r "file_put_contents('composer.json', json_encode(array_diff_key(json_decode(file_get_contents('composer.json'), true), ['scripts' => 1]), JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES));"
   ```

   > **Note**: The composer scripts run on the [Cloud Build][cloud-build] server.
    This is a temporary fix to prevent errors prior to deployment.

1. Copy the [`app.yaml`](app.yaml) file from this repository into the root of
   your project and replace `YOUR_APP_SECRET` with a new secret or the generated
   secret in `.env`:

    ```yaml
    runtime: php72

    env_variables:
        APP_ENV: prod
        APP_SECRET: YOUR_APP_SECRET

    # URL handlers
    # ...
    ```

    > **NOTE** Read more about the [env][symfony-env] and [secret][symfony-secret]
    parameters in Symfony's documentation.

1. [Override the cache and log directories][symfony-override-cache] so that
   they use `/tmp` in production. This is done by modifying the functions
   `getCacheDir` and `getLogDir` to the following in `src/Kernel.php`:


    ```php
    class Kernel extends BaseKernel
    {
        //...

        public function getCacheDir()
        {
            if ($this->environment === 'prod') {
                return sys_get_temp_dir();
            }
            return $this->getProjectDir() . '/var/cache/' . $this->environment;
        }

        public function getLogDir()
        {
            if ($this->environment === 'prod') {
                return sys_get_temp_dir();
            }
            return $this->getProjectDir() . '/var/log';
        }

        // ...
    }
    ```

    > **NOTE**: This is required because App Engine's file system is **read-only**.

1. Deploy your application to App Engine:

        gcloud app deploy

1. Visit `http://YOUR_PROJECT_ID.appspot.com` to see the Symfony demo landing
   page.

The homepage will load when you view your application, but browsing to any of
the other demo pages will result in a **500** error. This is because you haven't
set up a database yet. Let's do that now!

## Connect to Cloud SQL with Doctrine

Next, connect your Symfony demo application with a [Cloud SQL][cloud-sql]
database. This tutorial uses the database name `symfonydb` and the username
`root`, but you can use whatever you like.

### Setup

1. Follow the instructions to set up a
   [Google Cloud SQL Second Generation instance for MySQL][cloud-sql-create].

1. Create a database for your Symfony application. Replace `INSTANCE_NAME`
   with the name of your instance:

       gcloud sql databases create symfonydb --instance=INSTANCE_NAME

1. Enable the [Cloud SQL APIs][cloud-sql-apis] in your project.

1. Follow the instructions to
   [install and run the Cloud SQL proxy client on your local machine][cloud-sql-install].
   The Cloud SQL proxy is used to connect to your Cloud SQL instance when
   running locally. This is so you can run database migrations locally to set up
   your production database.

    * Use the [Cloud SDK][cloud-sdk] from the command line to run the following
      command. Copy the `connectionName` value for the next step. Replace
      `INSTANCE_NAME` with the name of your instance:

          gcloud sql instances describe INSTANCE_NAME

    * Start the Cloud SQL proxy and replace `INSTANCE_CONNECTION_NAME` with
      the connection name you retrieved in the previous step:

          cloud_sql_proxy -instances=INSTANCE_CONNECTION_NAME=tcp:3306 &

    **Note:** Include the `-credential_file` option when using the proxy, or
    authenticate with `gcloud`, to ensure proper authentication.

### Configure

1.  Modify your Doctrine configuration in `config/packages/doctrine.yml` and
    change the parameters under `doctrine.dbal` to be the following:

    ```yaml
    # Doctrine Configuration
    doctrine:
        dbal:
            driver: pdo_mysql
            url: '%env(resolve:DATABASE_URL)%'

        # ORM configuration
        # ...
    ```

1.  Use the Symfony CLI to connect to your instance and create a database for
    the application. Be sure to replace `DB_PASSWORD` with the root password you
    configured:

        # create the database using doctrine
        DATABASE_URL="mysql://root:DB_PASSWORD@127.0.0.1:3306/symfonydb" \
            bin/console doctrine:schema:create

1.  Modify your `app.yaml` file with the following contents. Be sure to replace
    `DB_PASSWORD` and `INSTANCE_CONNECTION_NAME` with the values you created for
    your Cloud SQL instance:

    ```yaml
    runtime: php72

    env_variables:
      APP_ENV: prod
      APP_SECRET: YOUR_APP_SECRET

      # Add the DATABASE_URL environment variable
      DATABASE_URL: mysql://root:DB_PASSWORD@localhost?unix_socket=/cloudsql/INSTANCE_CONNECTION_NAME;dbname=symfonydb

    # URL handlers
    # ...
    ```

### Run

1.  Now you can run locally and verify the connection works as expected.

        DB_HOST="127.0.0.1" DB_DATABASE=symfony DB_USERNAME=root DB_PASSWORD=YOUR_DB_PASSWORD \
            php bin/console server:run

1.  Reward all your hard work by running the following command and deploying
    your application to App Engine:

        gcloud app deploy

## Set up Stackdriver Logging and Error Reporting

Install the Google Cloud libraries for Stackdriver integration:

```sh
# Set the environment variable below to the local path to your symfony project
SYMFONY_PROJECT_PATH="/path/to/my-symfony-project"
cd $SYMFONY_PROJECT_PATH
composer require google/cloud-logging google/cloud-error-reporting
```

### Copy over App Engine files

For your Symfony application to integrate with Stackdriver Logging and Error Handling,
you will need to copy over the `monolog.yaml` config file and the `ExceptionSubscriber.php`
Exception Subscriber:

```sh
# clone the Google Cloud Platform PHP samples repo somewhere
cd /path/to/php-samples
git clone https://github.com/GoogleCloudPlatform/php-docs-samples

# enter the directory for the symfony framework sample
cd appengine/php72/symfony-framework/

# copy monolog.yaml into your Symfony project
cp config/packages/prod/monolog.yaml \
    $SYMFONY_PROJECT_PATH/config/packages/prod/

# copy ExceptionSubscriber.php into your Symfony project
cp src/EventSubscriber/ExceptionSubscriber.php \
    $SYMFONY_PROJECT_PATH/src/EventSubscriber
```

The two files needed are as follows:

  1. [`config/packages/prod/monolog.yaml`](app/config/packages/prod/monolog.yaml) - Adds Stackdriver Logging to your Monolog configuration.
  1. [`src/EventSubscriber/ExceptionSubscriber.php`](src/EventSubscriber/ExceptionSubscriber.php) - Event subscriber which sends exceptions to Stackdriver Error Reporting.

If you'd like to test the logging and error reporting, you can also copy over `LoggingController.php`, which
exposes the routes `/en/logging/notice` and `/en/logging/exception` for ensuring your logs are being sent to
Stackdriver:

```
# copy LoggingController.php into your Symfony project
cp src/Controller/LoggingController.php \
    $SYMFONY_PROJECT_PATH/src/Controller
```

  1. [`src/Controller/LoggingController.php`](src/Controller/LoggingController.php) - Controller for testing logging and exceptions.

### View application logs and errors

Once you've redeployed your application using `gcloud app deploy`, you'll be able to view
Application logs in the [Stackdriver Logging UI][stackdriver-logging-ui], and errors in
the [Stackdriver Error Reporting UI][stackdriver-errorreporting-ui]! If you copied over the
`LoggingController.php` file, you can test this by pointing your browser to
`https://YOUR_PROJECT_ID.appspot.com/en/logging/notice` and
`https://YOUR_PROJECT_ID.appspot.com/en/logging/exception`

## Send emails

The recommended way to send emails is to use a third-party mail provider such as [Sendgrid][sendgrid], [Mailgun][mailgun] or [Mailjet][mailjet]. 
Hosting your application on GAE, most of these providers will offer you up to 30,000 emails per month and you will be charged only if you send more.
You will have the possibility to track your email delivery and benefit from all the feature of a real email broadcasting system.

### Install

First you need to install the mailer component:

```
composer require symfony/mailer
```

For this example, we will use `Mailgun`.  To use a different mail provider, see the [Symfony mailer documentation][symfony-mailer].

```
composer require symfony/mailgun-mailer
```

This recipe will automatically add the following ENV variable to your .env file:

```
# Will be provided by mailgun once your account will be created
MAILGUN_KEY= xxxxxx
# Should be your Mailgun MX record
MAILGUN_DOMAIN= mg.yourdomain.com
# Region is mandatory if you chose server outside the US otherwise your domain will not be found
MAILER_DSN=mailgun://$MAILGUN_KEY:$MAILGUN_DOMAIN@default?region=eu
```

From that point, you just need to create your account and first domain adding all the DNS Records.
[Mailgun documentation][mailgun-add-domain] will lead you through these steps.

You can now send emails in Controller and Service as usual:
```
// src/Controller/MailerController.php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class ExampleController extends AbstractController
{
    /**
     * @Route("/email")
     */
    public function sendEmail(MailerInterface $mailer)
    {
        $email = (new Email())
            ->from('hello@example.com')
            ->to('you@example.com')
            ->subject('Time for Symfony Mailer!')
            ->text('Sending emails is fun again!');

        $mailer->send($email);
    }
} 
```

## Session Management

To persist sessions across multiple App Engine instances, you'll need to use a database.
Fortunately, Symfony provides a way to handle this using [PDO session storage][symfony-pdo-session-storage].

### Configuration

Modify your Framework configuration in `config/packages/framework.yaml` and change the parameters under session to be the following:

```
    session:
        handler_id: Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler
        cookie_secure: auto
        cookie_samesite: lax
        # Adjust the max lifetime to your need
        gc_maxlifetime: 36000
```

You should then activate the service in `config/services.yaml.

```
    Symfony\Component\HttpFoundation\Session\Storage\Handler\PdoSessionHandler:
        arguments:
            - !service { class: PDO, factory: ['@database_connection', 'getWrappedConnection'] }
            # If you get transaction issues (e.g. after login) uncomment the line below
            # - { lock_mode: 1 }
```

### Database update

Next, add the `sessions` table to your database by connecting to your database and
executing the following query:

```
# MySQL Query
CREATE TABLE `sessions` (
    `sess_id` VARCHAR(128) NOT NULL PRIMARY KEY,
    `sess_data` BLOB NOT NULL,
    `sess_time` INTEGER UNSIGNED NOT NULL,
    `sess_lifetime` INTEGER UNSIGNED NOT NULL
) COLLATE utf8mb4_bin, ENGINE = InnoDB;
```

You are now all set, the session will be persisted in the database and your users will remain authenticated.

## Upload files

Pre-requisites:
In order to write files on App Engine you will need to use a Storage Bucket, so you have to [enable the Storage API][cloud-enable-api-storage].
On top of that, you will need a service account having the Role Storage Admin, once this service account created you can download the private key in JSON format.

### Install all the bundles

First thing you will need is the [Google Cloud Storage for PHP][cloud-storage-php] package which will communicate with the Storage API:
```
    composer require google/cloud-storage
```

You will need the Google_client class from the [Google APIs Client Library for PHP][cloud-api-client].

```
    composer require google/apiclient:"^2.0"
```

You can then install the [KnpGaufretteBundle][knp-gaufrette] which will automate the communication with Google Storage:

```
    composer require knplabs/knp-gaufrette-bundle
```

To make it even easier, we use the famous [VichUploaderBundle][vich-uploader]:

```
    composer require vich/uploader-bundle
```

### Create an Uploadable Entity

You can modify the [`src/Entity/ExampleEntity`](src/Entity/ExampleEntity.php) to your need and be careful that the field updatedAt is changed when the file is set.
 
### Configuration

Installing the Google Cloud API Client Library you should automatically add the following keys to your .env:

```
    # If you want to have Read and write access to your repository
    GOOGLE_CLOUD_STORAGE_SCOPE='https://www.googleapis.com/auth/devstorage.full_control'
    GCS_PRIVATE_KEY_LOCATION='path_to_your_service_account_private_key.json'
    BUCKET_NAME='your-bucket-name'
    GOOGLE_CLOUD_PROJECT='your-project-id'
```

You will need to create a Factory to create a Google Client, check the [`src/Factory/GoogleCloudStorageServiceFactory`](src/Factory/GoogleCloudStorageServiceFactory.php) for reference. 

Then in your config/services.yaml you need to activate those 2 services, Google_Client_Storage will just use the Factory above to create the client:

```
    app.google_cloud_storage_factory:
        class: App\Factory\GoogleCloudStorageServiceFactory
        arguments:
            - '%env(GOOGLE_CLOUD_STORAGE_SCOPE)%'
            - '%kernel.project_dir%/%env(GCS_PRIVATE_KEY_LOCATION)%'

    app.google_cloud_storage.service:
        class: Google_Service_Storage
        factory: ['@app.google_cloud_storage_factory', createService]
``` 

Finally you will have to configure KNPGaufrette and VichUploader, here is the logic:

```
    # in your config/packages/knp_gaufrette.yaml
    knp_gaufrette:
      stream_wrapper: ~
      adapters:
        # Each type of file to be uploaded in your app will need an adapter and a filesystem 
        image_adapter:
          google_cloud_storage:
            service_id: 'app.google_cloud_storage.service'
            bucket_name: '%env(BUCKET_NAME)%'
            options:
              directory: 'your-directory-in-your-bucket'
    
      filesystems:
        image_filesystem:
          adapter:    image_adapter
          alias:      image_filesystem
``` 

```
    # in your config/packages/vich_uploader.yaml
    vich_uploader:
        db_driver: orm
        # This is how you connect VichUploader to KnpGaufrette
        storage: gaufrette
        # Each type of file needs its own mapping
        mappings:
            # See the method @Vich\UploadableField src/Entity/ExampleEntity
            example_mapping:
                uri_prefix: '/your-upload-directory'
                # this is how you specify the correct Gaufrette filesystem
                upload_destination: image_filesystem
                # the namer is not mandatory, this one generates a unique ID instead of using your file name
                namer: Vich\UploaderBundle\Naming\UniqidNamer
```

So every time an ExampleEntity will be persisted, the file will be automatically uploaded in your bucket.  


[cloud-sdk]: https://cloud.google.com/sdk/
[cloud-build]: https://cloud.google.com/cloud-build/
[cloud-sql]: https://cloud.google.com/sql/docs/
[cloud-sql-create]: https://cloud.google.com/sql/docs/mysql/create-instance
[cloud-sql-install]: https://cloud.google.com/sql/docs/mysql/connect-external-app#install
[cloud-sql-apis]: https://console.cloud.google.com/apis/library/sqladmin.googleapis.com/?pro
[cloud-migration]: https://cloud.google.com/appengine/docs/standard/php7/php-differences?hl=en#migrating_from_the_app_engine_php_sdk
[cloud-enable-api-storage]: https://console.cloud.google.com/flows/enableapi?apiid=storage_api
[cloud-api-client]: https://github.com/googleapis/google-api-php-client
[cloud-storage-php]: https://github.com/googleapis/google-cloud-php-storage
[create-project]: https://cloud.google.com/resource-manager/docs/creating-managing-projects
[enable-billing]: https://support.google.com/cloud/answer/6293499?hl=en
[symfony]: http://symfony.com
[symfony-install]: http://symfony.com/doc/current/setup.html
[symfony-welcome]: https://symfony.com/doc/current/_images/welcome.png
[symfony-demo]: https://github.com/symfony/demo
[symfony-secret]: http://symfony.com/doc/current/reference/configuration/framework.html#secret
[symfony-env]: https://symfony.com/doc/current/configuration/environments.html#executing-an-application-in-different-environments
[symfony-override-cache]: https://symfony.com/doc/current/configuration/override_dir_structure.html#override-the-cache-directory
[symfony-mailer]: https://symfony.com/doc/current/mailer.html
[symfony-pdo-session-storage]: https://symfony.com/doc/current/doctrine/pdo_session_storage.html
[stackdriver-logging-ui]: https://console.cloud.google.com/logs
[stackdriver-errorreporting-ui]: https://console.cloud.google.com/errors
[sendgrid]: https://sendgrid.com/
[mailgun]: https://www.mailgun.com/
[mailjet]: https://www.mailjet.com/
[mailgun-add-domain]: https://help.mailgun.com/hc/en-us/articles/203637190-How-Do-I-Add-or-Delete-a-Domain-
[vich-uploader]: https://github.com/dustin10/VichUploaderBundle
[knp-gaufrette]: https://github.com/KnpLabs/KnpGaufretteBundle
