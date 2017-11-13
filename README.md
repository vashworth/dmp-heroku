# Django-Mako-Plus Heroku Integration Tutorial
This a walkthrough on how to host a django-mako-plus (DMP) application on Heroku using Postgres.

DMP is a framework created by Dr. Conan C. Albrecht. To learn more about it check out the [repo](https://github.com/doconix/django-mako-plus) or [docs](http://django-mako-plus.readthedocs.io).

This tutorial is written specifically for Mac OS. Adjustments will need to be made to the Requirements and Step 1 for Windows. Shouldn't be too hard to figure out, though.


Requirements:
* [Install Latest Version of Python](https://www.python.org/downloads/)
* [Install Git](https://git-scm.com/downloads)
* Install Pipenv. Accomplish this by running ``pip3 install pipenv``
* Install Postgres for [Mac](http://postgresapp.com), [Linux](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads), [Windows](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads#windows)
* Install psycopg2. Accomplish this by running ``pip3 install psycopg2``
* [Install PGAdmin](https://www.pgadmin.org/download/)
* [Free GitHub Account](https://github.com/join)
* [Free Heroku Account](https://signup.heroku.com/)

### Step 1 - Setup localhost

#### Setup Database

Open Postgres.app
* Select a server or create a new one
* Select Server Settings, take note of name and port

Open pgAdmin 4.app
* In the Dashboard Quick Links, select Add New Sever
* Under the General tab
  * Name: use the name from the Postgres.app Server Settings
  * Don't change the other fields
* Under the Connection tab
  * Host: localhost
  * Port: use the port from the Postgres.app Server Settings
  * Username: your system user name
  * Don't change the other fields
* Click Save
* Expand Servers tree
* Right click on Databases and select create - database
* Name new database 'dmp_heroku'
* Click Save


#### Create DMP Application

The following steps have been adjusted from the Quick Start section of the DMP docs. http://django-mako-plus.readthedocs.io/#quick-start
```
# install django, mako, and DMP
pip3 install django-mako-plus

# create a new project with a 'homepage' app
python3 -m django startproject --template=http://cdn.rawgit.com/doconix/django-mako-plus/master/project_template.zip dmp_heroku
cd dmp_heroku
python3 manage.py startapp --template=http://cdn.rawgit.com/doconix/django-mako-plus/master/app_template.zip --extension=py,htm,html homepage

# open dmp_heroku/settings.py
# append 'homepage' to the INSTALLED_APPS list
INSTALLED_APPS = [
    ...
    'homepage',
]

# replace default DATABASE with
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'dmp_heroku',
        'USER': '[Insert Username used in PgAdmin4]',
        'PASSWORD': '',
        'HOST': 'localhost',
    },
    #'default': {
    #    'ENGINE': 'django.db.backends.sqlite3',
    #    'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    #}
}
```

Run initial migrations and run the server in terminal

``python3 manage.py migrate``

``python3 manage.py runserver``

Open a browser to http://localhost:8000/

You should see "Welcome to the homepage app! ...."

Awesome! You've made a DMP application. You can turn this off now with CTRL-C in the terminal.


### Step 2 - Heroku Integration

#### Edit dmp_heroku/settings.py
You need different settings for development vs production so I create copies to keep them separate. You DO NOT want to use the development settings in GitHub as this usually includes passwords.
* Duplicate your settings.py file and name it settings-dev.py
* Duplicate your settings.py file again and name it settings-prod.py

Open settings-prod.py
```
#add dj_database_url at top with other imports
import dj_database_url

#change your SECRET_KEY to the following:
SECRET_KEY = os.environ.get('SECRET_KEY')

#change DEBUG to false
DEBUG = False

#add '*' to ALLOWED_HOSTS
ALLOWED_HOSTS = ['*']

#append WhiteNoiseMiddleware to the MIDDLEWARE list after SecurityMiddleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
]

#replace DATABASES with the following:
DATABASES = {
}
DATABASES['default'] = dj_database_url.parse(os.environ.get('DATABASE_URL'), conn_max_age=0)

#comment out BASE_DIR in STATICFILES_DIRS
STATIC_URL = '/static/'
STATICFILES_DIRS = (
    # SECURITY WARNING: this next line must be commented out at deployment
    #BASE_DIR,
)

#change STATIC_ROOT to the following:
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

#append the following after STATIC_ROOT
WHITENOISE_MAX_AGE = 604800
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

```


#### Create a Procfile
Inside your dmp_heroku directory, create a new file named Procfile (no extension)

```
web: python manage.py dmp_collectstatic --overwrite; python -m whitenoise.compress staticfiles/homepage/styles; python -m whitenoise.compress staticfiles/homepage/scripts; python manage.py migrate; gunicorn dmp_heroku.wsgi --log-file -
```

The Procfile tells Heroku what commands to run upon deployment. To learn more, check out their documentation on the [Procfile](https://devcenter.heroku.com/articles/procfile).

Django does not support using static files in production. To fix this, we are using an application called [whitenoise](http://whitenoise.evans.io/en/stable/). Also, since DMP handles static files differently than normal django, we have to overwrite the dmp_collectstatic command and compress styles & scripts for each app inside our project.

#### Create a Pipfile
Inside the terminal, run the following:
```
pipenv --three
```
This will create a new file called Pipfile.

Open this file in your editor. Add the following after packages.

```
[packages]
dj-database-url = "*"
django = "*"
mako = "*"
django-mako-plus = "*"
gunicorn = "*"
psycopg2 = "*"
whitenoise = "*"
```
Make sure your python version is correct. This file tells Heroku what dependencies need to be installed.

Now, inside the terminal, run the following:

```
pipenv install
```
This will create a Pipfile.lock

### Step 3 - Create Git Repository

This section assumes you have created a GitHub account and [created and added an ssh key](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).

**IMPORTANT:** Before pushing to GitHub, **Copy settings-prod.py into settings.py**. When you want to run on your localhost, copy settings-dev.py into settings.py

* [Create a new repository on GitHub](https://help.github.com/articles/creating-a-new-repository/). To avoid errors, do not initialize the new repository with README, license, or gitignore files. You can add these files after your project has been pushed to GitHub.
* Initialize your project by running ``git init`` in the terminal
* Create a .gitignore
```
## Python ##
db.sqlite3
__pycache__
*.cached_templates/

## Heroku ##
dmp_heroku/settings-dev.py
dmp_heroku/settings-prod.py
static/
staticfiles/
```
* Run the following commands in the terminal
```
git add .
git commit -am "initial commit"
git remote add origin [remote repository URL]
git push -u origin master
```
### Step 4 - Deploy on Heroku

Generate a new SECRET_KEY
* Open a browser to https://www.miniwebtool.com/django-secret-key-generator/ or use your own method
* Copy this key for later

Create Heroku Application
* Open a browser to https://dashboard.heroku.com/apps
* In the top right, click the New button and select 'Create new app'
* Name the application anything you'd like, I named my 'dmp-heroku' so you won't be able to use it because it's taken
* Navigate to the Resources tab
* In the Add-ons search field, type 'postgres' and select Heroku Postgres
* Select Hobby Dev - Free and click Provision
* Navigate to the Settings tab
* Under Config Variables, click Reveal Config Vars
* Add a new key
  * KEY: SECRET_KEY
  * VALUE: [copied generated key]
  * Make sure to click Add
* Navigate to the Deploy tab
* Select GitHub under Deployment Method
* Click Connect to GitHub
* Login
* Under Connect to GitHub, search for the name of your GitHub repository
* Click connect
* Under Manual Deploy, click Deploy Branch
* Once build and deploy is done running, click the View button

### Congrats! You've officially launched a DMP application in Heroku!

To learn more about Heroku, check out their [documentation](https://devcenter.heroku.com). Happy coding!
