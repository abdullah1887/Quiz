<div align="center">
<h1>Django Deployment Guide</h1>
</div>

# Context

- [Context](#context)
  - [Preparation](#preparation)
  - [Most Important Part](#most-important-part)
  - [Environment variable](#environment-variable)
  - [re\_path vs static vs whitenoise](#re_path-vs-static-vs-whitenoise)
    - [re\_path](#re_path)
    - [static](#static)
    - [whitenoise](#whitenoise)
  - [Deployment](#deployment)
    - [Cpanel Deployment](#cpanel-deployment)
    - [Render Deployment](#render-deployment)
    - [PythonAnywhere Deployment](#pythonanywhere-deployment)
      - [Removing Current files](#removing-current-files)
      - [Uploading Project Files](#uploading-project-files)
      - [Configure Project Path, WSGI \& Virtual Environment](#configure-project-path-wsgi--virtual-environment)
    - [Host locally view globally](#host-locally-view-globally)
  - [Extra](#extra)
    - [Django uploaded content (File/Image) Deletion](#django-uploaded-content-fileimage-deletion)
    - [Django uploaded content (File/Image) Rename](#django-uploaded-content-fileimage-rename)
    - [Sign-in/Sign-up Access Control using Custom Decorator](#sign-insign-up-access-control-using-custom-decorator)

> [!IMPORTANT]
>
> - In this project `main` branch is `development` branch
> - Use `deploy` branch for deployment

## Preparation

- Install [whitenoise](https://pypi.org/project/whitenoise/) & [gunicorn](https://pypi.org/project/gunicorn/)

  ```sh
  pip install whitenoise gunicorn
  ```

- We need to setup our project for deployment
  - [requirements.txt](./imageProject/requirements.txt) must be present, which contains all the required packages we use in our project
  - Generate [requirements.txt](./imageProject/requirements.txt)

    ```sh
    pip freeze > requirements.txt
    ```

- In `settings.py` file modify as below

  ```py
  DEBUG = False
  ALLOWED_HOSTS = ["*"] # change this to domain website

  MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware', # This must be added here after SecurityMiddleware & SessionMiddleware
    ...
  ]

  # static
  STATIC_URL = '/static/'
  STATIC_ROOT = BASE_DIR / 'staticfiles/'
  STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

  # media
  MEDIA_URL = '/media/'
  MEDIA_ROOT = BASE_DIR / 'media/'
  ```

- Modify project `urls.py` ; this is for media

  ```py
  from django.contrib import admin
  from django.conf import settings
  from django.conf.urls.static import static
  from django.views.static import serve
  from django.urls import path, include, re_path

  urlpatterns = [
      path('admin/', admin.site.urls),
      path('',include('imageApp.urls')),

      # for media
      re_path(r'^media/(?P<path>.*)$', serve, {'document_root': settings.MEDIA_ROOT})
  ]
  ```

- Now we have to run `collectstatic` command

  ```sh
  py manage.py collectstatic
  ```

  - We will use this `collectstatic` command in server, or we could just run it and zip the whole project and upload to the server (it's mainly depended on how static contents are being setup)`
  - As some free server does not allow running command so in `deploy` branch `staticfiles` are generated

## Most Important Part

> [!IMPORTANT]
>
> - **After above preparation step run it locally (MUST DO THIS)**
> - **Check frontend if everything working or not**
> - **Check admin page**
> - **Check route that does not exits and see if it's showing normal error or detailed error**
> - **After checking all those things you can move to the server for deployment**

---
[⬆️ Go to Context](#context)

## Environment variable

- For this we can use [python-dotenv](https://pypi.org/project/python-dotenv/) or [python-decouple](https://pypi.org/project/python-decouple/)

  - [python-dotenv](https://pypi.org/project/python-dotenv/)
    - Install it `pip install python-dotenv`
    - Create a `.env`

      ```env
      SECRET_KEY='write the key here'
      DEBUG=False
      ```

    - Now in [settings.py](./imageProject/imageProject/settings.py)

      ```py
      import os
      from dotenv import load_dotenv
      load_dotenv()

      SECRET_KEY = os.getenv('SECRET_KEY')
      DEBUG = os.getenv('DEBUG')
      ```

  - [python-decouple](https://pypi.org/project/python-decouple/)
    - Install it `pip install python-decouple`
    - Create a `.env`

      ```env
      SECRET_KEY='write the key here'
      DEBUG=False
      ```

    - Now in [settings.py](./imageProject/imageProject/settings.py)

      ```py
      from decouple import config

      SECRET_KEY = config('SECRET_KEY')
      DEBUG = config('DEBUG',cast=bool)
      ```

> [!NOTE]
>
> - Some free server does not allow environment variable setup so it is not configure in this project but for production it is very important to setup environment variable

---
[⬆️ Go to Context](#context)

## re_path vs static vs whitenoise

### re_path

- We can use `re_path` in project `urls.py`

  ```py
  from django.contrib import admin
  from django.conf import settings
  from django.conf.urls.static import static
  from django.views.static import serve
  from django.urls import path, include, re_path

  urlpatterns = [
      path('admin/', admin.site.urls),
      path('',include('imageApp.urls')),

      # for media and static files
      re_path(r'^static/(?P<path>.*)$', serve, {'document_root': settings.STATIC_ROOT}),
      re_path(r'^media/(?P<path>.*)$', serve, {'document_root': settings.MEDIA_ROOT})
  ]
  ```

  - But problem is using `re_path` is actually getting `static` and `media` from the app directory directly not from the `staticfiles`
  - And when using this `re_path` for both `static` and `media` we don't need `whitenoise` for static setup
  - But it is recommended to use `whitenoise` static setup in production

---
[⬆️ Go to Context](#context)

### static

- We can use `static` in project `urls.py`

  ```py
  from django.contrib import admin
  from django.conf import settings
  from django.conf.urls.static import static
  from django.views.static import serve
  from django.urls import path, include, re_path

  urlpatterns = [
      path('admin/', admin.site.urls),
      path('',include('imageApp.urls'))
  ]
  urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
  urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
  ```

  - But problem is using `static` won't work in production it will not serve the static or media it only works in local

---
[⬆️ Go to Context](#context)

### whitenoise

- We need to install it [whitenoise](https://pypi.org/project/whitenoise/)
- Then setup it in `settings.py`

  ```py
  MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware', # This must be added here after SecurityMiddleware & SessionMiddleware
    ...
  ]

  STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
  ```

- Now static file setup is done but for `media` we need to add the `re_path`

  ```py
  from django.contrib import admin
  from django.conf import settings
  from django.conf.urls.static import static
  from django.views.static import serve
  from django.urls import path, include, re_path

  urlpatterns = [
      path('admin/', admin.site.urls),
      path('',include('imageApp.urls')),

      # for media
      re_path(r'^media/(?P<path>.*)$', serve, {'document_root': settings.MEDIA_ROOT})
  ]
  ```

> [!IMPORTANT]
>
> - Our main goal is to serve `media` and `static` from the project itself not from external sources this is why we need to do these setup otherwise we could just add those in the remote server production `/static/` and `/media/` directly
> - Now what to use, use `re_path` if you don't want to setup whitenoise
> - But `whitenoise` setup is recommended

---
[⬆️ Go to Context](#context)

## Deployment

### Cpanel Deployment

- In `Cpanel` create a python application
- Select python version >= 3.10 (for django version 5)
- Add `Application root`
- Select domain in `Application URL` and set the url
- Now click on `Create`
- Go to the `Application root` path
  - Edit `passenger_wsgi.py` add the below line and save it
    - `from imageProject.wsgi import application` # here imageProject is the project name
  - Upload Project zip file and it must contain [requirements.txt](./imageProject/requirements.txt)
- Now again go to the python application and add `requirements.txt` in `Configuration files` (Save and run pip install)
- Now in `Execute python script` run this: `manage.py collectstatic`

---
[⬆️ Go to Context](#context)

### Render Deployment

- Go to [render](https://render.com/) and select web service from New tab
- Select GitHub repo of the project
- Now in project setting page write Project Name (unique)
- Select Region, Branch
- Set Build command `pip install -r requirements.txt`
- Set start command `gunicorn imageProject.wsgi:application`
  > here `imageProject` is the project name
- Choose Instance Type `Free` and start deploy.
- For static files `py manage.py collectstatic` which we can't run in free instance so we run it locally and then deploy which is in `deploy` branch

---
[⬆️ Go to Context](#context)

### PythonAnywhere Deployment

#### Removing Current files

- Create an account and login to [PythonAnywhere](https://www.pythonanywhere.com/)
- Go to `Consoles` tab and open `bash`
- Now cleanup current files and directory (only for `PythonAnywhere` read **warning!**)

  ```sh
  rm -rf ~/*
  rm -rf ~/.??*
  ```

> [!WARNING]
>
> - This `rm` command is use to delete files,folders permanently so make sure it is done carefully
> - We are deleting everything cause we will setup it from the very beginning with our uploaded file on `PythonAnywhere`
> - DO NOT RUN THOSE COMMAND IN OTHER CONSOLE/SERVER

#### Uploading Project Files

- Now upload the `deploy` branch as zip file
  - make sure `django` version is `5.2.x` in `requirements.txt` cause in [PythonAnywhere](https://www.pythonanywhere.com/) latest python version currently is `3.10.x` which does not support `django` version `6.0.x`. Refs: [docs](https://docs.djangoproject.com/en/6.0/faq/install/)
  - We can edit the `requirements.txt` inside console using `nano` or directly change it before zipping it and change the version to `5.2.x`

    ```sh
    Django==5.2.14
    ```

> [!IMPORTANT]
>
> - If you get python version 3.13.x you don't need to edit the django version in `requirements.txt`
> - Always check the ofccial docs to setup django in any specific version of python.
> - Official Docs: [What Python version can I use with Django?](https://docs.djangoproject.com/en/6.0/faq/install/#what-python-version-can-i-use-with-django)

- Now upload and unzip using the bash console
  - Make sure it will be unzipped in a single directory
  - For example if we unzip `unzip imageProject.zip` output will be inside `imageProject` directory
  - To ensure this we can open the zip file and see if the contents are in single folder or not
  - Otherwise we have to mention directory name

    ```sh
    unzip imageProject.zip -d imageProject
    ```

> [!IMPORTANT]
>
> - Check the python version if it is `3.10.x` we can create the `.venv` and install required packages from `requirements.txt`
> - If python version is not `3.10.x` continue below

#### Configure Project Path, WSGI & Virtual Environment

- Now go to `Web` tab
- Select `Manual configuration (including virtualenvs)`
- Select `Python 3.10.x`
- Add source code path `/home/aatansen/imageProject`
- Now we have to configure `WSGI configuration file:/var/www/aatansen_pythonanywhere_com_wsgi.py`
  - We already have this in our project called `wsgi.py` so we can use it in the `aatansen_pythonanywhere_com_wsgi.py` where first we need to get the path where our project is then `import` the application

    ```py
    import sys
    sys.path.append('/home/aatansen/imageProject') # path of the project

    from imageProject.wsgi import application
    ```

- Now Create virtual environment
  - Open new bash in `Consoles` tab
  - Make sure python version is `3.10.x`

    ```sh
    python --version
    ```

  - Now create virtual environment named `.venv`

    ```sh
    python -m venv .venv
    ```

  - Activate it

    ```sh
    source .venv/bin/activate
    ```

  - Install `requirements.txt`

    ```sh
    pip install -r requirements.txt
    ```

- Add its path in `Virtualenv: /home/aatansen/.venv`
- Reload the site and it is done
- [https://aatansen.pythonanywhere.com/](https://aatansen.pythonanywhere.com/)

---
[⬆️ Go to Context](#context)

### Host locally view globally

- Using [Serveo](https://serveo.net/)
  - First run the project `http://127.0.0.1:8000/`
  - `ssh -R 80:localhost:8000 serveo.net`
  - To use unique subdomain, need to generate a key
    - `ssh-keygen -t ed25519`
    - `ssh -R <unique subdomain>:80:<your local host>:<your local port> serveo.net`
    - Example: `ssh -R youruniquesubdomain:80:localhost:8000 serveo.net`
    - Add that domain in `ALLOWED_HOSTS` and include `CSRF_TRUSTED_ORIGINS = ['https://youruniquesubdomain.serveo.net']` in `settings.py`
  - Now a login prompt to `serveo` will be given; use google/github to login
  - Rerun `ssh -R youruniquesubdomain:80:localhost:8000 serveo.net`, now it will work

---
[⬆️ Go to Context](#context)

- Using [pinggy](https://pinggy.io/)
  - First run the project `http://127.0.0.1:8000/`
  - Go to [pinggy dashboard](https://dashboard.pinggy.io/)
  - Copy the ssh of `HTTP(S) Tunnel` and make sure to change the port to `8000`
  - After running the command give a password
  - Now a domain will be provided by [pinggy](https://pinggy.io/) to access

---
[⬆️ Go to Context](#context)

- Using [runlocal](https://runlocal.eu/)
  - First run the project `http://127.0.0.1:8000/`
  - Now run `npx runlocal 8000`
    > It will install the `runlocal` and provide the domain to access

---
[⬆️ Go to Context](#context)

> [!NOTE]
>
> - For custom domain `CSRF_TRUSTED_ORIGINS` is important to setup
> - There are more like this [ngrok](https://ngrok.com/), [localhost.run](https://localhost.run/),[localtunnel](https://theboroer.github.io/localtunnel-www/),[cloudflare-tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/),[localxpose](https://localxpose.io/),[tunnelmole](https://tunnelmole.com/) etc

## Extra

### Django uploaded content (File/Image) Deletion

- Install `django-cleanup`
  - `pip install django-cleanup`
  - Documentation: [Django Cleanup](https://github.com/un1t/django-cleanup)
- Modify in `settings.py`'s `INSTALLED_APPS`

  ```py
  INSTALLED_APPS = [
      ...
      'django_cleanup.apps.CleanupConfig',
  ]
  ```

> Make sure to add it at the bottom

- Now first import the `cleanup` in `models.py`
  - `from django_cleanup import cleanup`
  - Add `@cleanup.select` before class defied

    ```py
    from django.db import models
    from django_cleanup import cleanup

    # Create your models here.
    @cleanup.select
    class UserModel(models.Model):
        name=models.CharField(max_length=100)
        profile_image=models.ImageField(upload_to='profile_image')
        def __str__(self):
            return self.name
    ```

> Note: To ignore a model we can add `@cleanup_ignore`

---
[⬆️ Go to Context](#context)

### Django uploaded content (File/Image) Rename

- Add this function in `models.py`

  ```py
  def user_directory_path(instance, filename):
      # file will be uploaded to MEDIA_ROOT/user_<id>/<filename>
      return f"user_{instance.name}_{filename}"
  ```

  - Here file will be saved as `user_{instance.name}_{file_name}`; e.g: `user_tansen_1.jpg`
- Now include this function in `upload_to`

  ```py
  class UserModel(models.Model):
      name=models.CharField(max_length=100)
      profile_image=models.ImageField(upload_to=user_directory_path)
      def __str__(self):
          return self.name
  ```

---
[⬆️ Go to Context](#context)

### Sign-in/Sign-up Access Control using Custom Decorator

- Create a custom function

  ```py
  from django.contrib import messages

  def logout_required(view_func):
      def wrapper(request, *args, **kwargs):
          if request.user.is_authenticated:
              messages.warning(request, "You are already logged in.")
              return redirect('dashboard')
          return view_func(request, *args, **kwargs)
      return wrapper
  ```

  - Here message is added to show the user that he is already logged in
  - Now add this `@logout_required`before `signin` and `singup` function

---
[⬆️ Go to Context](#context)
