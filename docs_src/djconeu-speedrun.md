# Speedrun

This is a breakdowm of the speedrun script used in my DjangoCon Europe 2019 talk on Serverless Django.

The speedrun showed a minimal setup for running a Django application as an AWS Lambda function using
Zappa for deployment.

It starts from nothing locally and ends up with a live application without any user interaction or
using the AWS Console beyond the prerequistites.

[View an Asciinema Cast of a speedrun](https://asciinema.org/a/237320)
(a 1 minute cast reflecting 3 minutes of real-time).

## Prerequisites

- Python 3 local environment
- AWS Account with:
    - IAM User with Programatic access and sufficient permissions
      (_Administrative Access_ is the equivalent of root)
- IAM User's credentials added to `~/.aws/credentials` locally
- Optional for a Custom Domain:
    - AWS Route 53 hosted zone for the domain or delegated subdomain
    - AWS Certificate Manager SSL certificate for the domain/subdomain
      (generated in the `us-east-1` region regardless of what region is used for Zappa)

## Demonstration stack

- AWS Lambda function to run a Django application
- AWS API Gateway to delivery requests to the Lambda function
- Wagtail Bakery as a sample Django application to demonstrate a front and backend CMS
- S3 buckets to provided persistence for:
    - static assets and media
    - SQLite database for Django
    - Zappa application deployment
- Django Storages to transfer static assets and media to S3
- Zappa and Zappa-Django-Utils to build and deploy the Lambda function and manage the application

## Script breakdown

This cuts everything back to the miminum shell commands and assumes every step runs smoothly.

This code is available in this
[source script](https://github.com/nealtodd/serverless-django/blob/master/samples/speedrun.sh).

In this example `${SITE}` is the name of the project being created and also used to name
related things like S3 buckets.

The IAM User name in this example is `iam-zappa`.

If you happen to read this before my talk an example is at this "Here's One I Made Earlier" site:
[speedrun-hoime.bygge.net](https://speedrun-hoime.bygge.net).

The deployment is made to the AWS London region (`eu-west-2`).

Name it what you want (although the S3 bucket names that will be created must be globally unique):

```bash
export SITE=mysite
```

Create a project directory:

```bash
mkdir ${SITE} && cd ${SITE}
```

Zappa needs a virtualenv to operate in:

```bash
python3 -m venv ./venv
source ./venv/bin/activate
pip install --upgrade pip
```

Install our support packages:

```bash
pip install zappa zappa-django-utils django-storages awscli
```

`awscli` doesn't need to be installed in the vritualenv if you already have it available.

Install our sample Django application, in this case Wagtail's [Bakery Demo](https://github.com/wagtail/bakerydemo):

```bash
git clone https://github.com/wagtail/bakerydemo.git
cd bakerydemo
pip install -r requirements/base.txt
```

(you could instead use your own Django application or simply a fresh Django install but you'll need to tweak
the Django configuration as appropriate).

We're going to use the Bakey Demo's development settings with some tweaking:

```bash
cat << EOF >> bakerydemo/settings/dev.py
DATABASES = {
    'default': {
        'ENGINE': 'zappa_django_utils.db.backends.s3sqlite',
        'NAME': '${SITE}-sqlite.db',
        'BUCKET': '${SITE}-db'
    }
}

ALLOWED_HOSTS = ['.eu-west-2.amazonaws.com', '.bygge.net']

INSTALLED_APPS += ('storages',)
AWS_STORAGE_BUCKET_NAME = '${SITE}-static'
AWS_QUERYSTRING_AUTH = False
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

DEBUG = os.getenv('DEBUG', 'off') == 'on'
EOF
```

!!! Notes
    - This uses a SQLite database hosted in a S3 bucket via the `zappa_django_utils.db.backends.s3sqlite` backend
    - `ALLOWED_HOSTS` covers the domain that AWS creates for the Lamdba function and also the custom domain
    that will be configured with Zappa.
    - Django Storages is added and configured to send unsigned static assets and media to a public S3 bucket
    that will be created.

We're creating a Zappa settings file directly with the configuration we want rather than use `zappa init`
to create one:

```bash
cat << EOF > zappa_settings.json
{
    "dev": {
        "django_settings": "bakerydemo.settings.dev",
        "profile_name": "iam-zappa",
        "project_name": "${SITE}",
        "runtime": "python3.6",
        "s3_bucket": "${SITE}-zappa",
        "aws_region": "eu-west-2",
        "timeout_seconds": 60,
        "domain": "${SITE}.bygge.net",
        "certificate_arn": "${CERT_ARN}",
        "aws_environment_variables": {
            "DEBUG": "off"
        }
    }
}
EOF
```

!!! Notes
    - This configures Zappa minimal settings in order to deploy. See the
    [Zappa project](https://github.com/Miserlou/Zappa) for full details of the settings.
    - `timeout_seconds` is increased from the default of 30s to allow comfortable time for the
    initial sync of the Bakery Demo's static assets to the S3 bucket
    - The `domain` and `certificate_arn` settings are for the custom domain. These can be left out
    if not using a custom domain. The `${CERT_ARN}` comes from AWS Certificate Manager and is the ARN
    (Amazon Resource Name) of the SSL certificate generated for the domain in the prerequisites.
    It won't hurt to leave them in even with an undefined `${CERT_ARN}` if the `certify` step
    (further down) is skipped.
    - `aws_environment_variables` is an example of setting an environment variable and having it
    accessible in the AWS Lambda console. Here, it allows Django's `DEBUG` mode to be toggled.

In order to store the SQLite database and the static assets & media we create two S3 buckets. Zappa
will create its own bucket to store the packaged application on deployment (based on the `s3_bucket`
in Zappa settings. We make the static bucket publicly accessible and apply a permissive CORs policy:

```bash
aws s3api create-bucket --bucket ${SITE}-db --profile iam-zappa \
    --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2
aws s3api create-bucket --bucket ${SITE}-static --profile iam-zappa \
    --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2
aws s3api put-bucket-cors --bucket ${SITE}-static --profile iam-zappa --cors-configuration \
    '{"CORSRules": [{"AllowedOrigins": ["*"], "AllowedMethods": ["GET"]}]}'
aws s3api put-bucket-policy --bucket ${SITE}-static --profile iam-zappa --policy \
    '{"Statement": [{"Effect": "Allow","Principal": "*","Action": "s3:GetObject","Resource": "arn:aws:s3:::'${SITE}'-static/*"}]}'
```

The Wagtail Bakery Demo comes with sample content, including images so we sync them to the static bucket
(where media is also stored). This is specific to the Wagtail Bakery Demo:

```bash
aws s3 sync bakerydemo/media/original_images/ s3://${SITE}-static/original_images/ --profile iam-zappa
```

Okay, that's the configuration out of the way, time to use Zappa to deploy the application as a Lambda function:

```bash
zappa deploy dev
```

The `deploy` command is used to create the Lambda function and its dependencies. Once they are created it is not used again
for the environment (in this case, `dev`) - instead the `update` command is used for subsequent deployments. Zappa will create
a package for the virtual environment and local codebase and upload it to its S3 bucket before transferring it into the Lambda
function.

At the end of the deployment Zappa makes a request to the Lambda function to check it. In this case it'll get a server error
(`500` response). This isn't because it failed but because the Wagtail Bakery Demo isn't yet in a runnable state - the database
hasn't been migrated so no tables yet exist in the SQLite database. We'll fix that sortly.

As we are using a custom domain we can get Zappa to certify it so that the site runs under HTTPS on the domain. Zappa takes
care of creating an API Gateway endpoint for the custom domain using the configured certificate and a CNAME record in Route 53 to
the Lambda function's AWS domain:

```bash
zappa certify dev -y
```

Now we collect the static assets for the Wagtail Bakery Demo as we would do with a Django project. Because Lambda functions do not
have shell access we use Zappa's `manage` command to run the normal Django `collectstatic` management command (with Django Storages
handling the upload to the S3 bucket):

```bash
zappa manage dev "collectstatic --noinput"
```

Now lets use further Django management commands to migrate the database and load sample content into the Wagtail Bakery Demo:

```
zappa manage dev migrate
zappa manage dev load_initial_data
```

We can get the status of the deployment which will include the AWS Gateway URL and custom domain URL for the site:

```bash
zappa status dev
```

Visiting the custom domain URL (or the AWS Gateway URL if a custom domain wasn't used) shows us our serverless
Wagtail Bakery Demo site.

(Wagtail Admin is on `/admin/` (log in with `admin` and `changeme`) and Django Admin is on `/django-admin/`)

!!! Note
    Page request might feel slighlty sluggish but this isn't an effect of them being serverless but rather because
    we're running off a SQLite database in an S3 bucket and Zappa syncs it with a local copy within the Lambda function
    on every page request. This is fine for experimentation but proper performance we can use an RDS database instead,
    but that's another topic.

!!! Note
    Subsequent deployments of the site are made with `zappa update dev`

Steps to remove the site from AWS are available in a
[teardown script](https://github.com/nealtodd/serverless-django/blob/master/samples/speedrun_teardown.sh).
