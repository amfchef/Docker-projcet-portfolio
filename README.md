## Assignment 2 Project

### Documentaion
I decided to create a portfolio of myself, using Django, Nginx and Docker
I created a Dockerfile.dev which I'm using in developing mode.
I also created a docker-compose and a second Dockerfile for Nginx, this is meant for production.

First of all I created a new project directory (docker_project).
- `sudo apt install python3-venv`
- `python3 -m venv docker_project/env`
- `source docker_project/env/bin/activate (activate env)`
- `deactivate`
- `pip install django`
- `django-admin startproject app .`
- `gitignore.io -> django`
- Test the local server and write `Hello world` as input
- `python manage.py runserver`
- Go to localhost:8000 and see if `Hello world` shows as output
- `python manage.py startapp pages` This will add a start up directoty for your django project
- Add config content to your app/settings.py
```
  INSTALLED_APPS = [
    'pages.apps.PagesConfig',
    ]
```
- Add config content to pages/urls.py
```
from django.urls import path
from . import views

urlpatterns = [
  path('', views.index, name="index"),
  path('contact', views.contact, name="contact"),
  path('resume', views.resume, name="resume"),
]
```
- Add config content to pages/views.py
```
from django.shortcuts import render, redirect

def index(request):
  return render(request, 'pages/index.html')

def contact(request):
  return render(request, 'pages/contact.html')

def resume(request):
  return render(request, 'pages/resume.html')

```
- Add config content to pages/views.py
```
from django.contrib import admin
from django.urls import path, include


urlpatterns = [
    path('', include('pages.urls')),
    path('admin/', admin.site.urls),
]
```
- Add config content to app/settings.py
```
import os

TEMPLATES = [
    {
        ...,
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        ...
       ]
    }
DEBUG=1 or os.getenv('DEBUG')

ALLOWED_HOSTS = ['*']
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
STATIC_URL = '/static/'
STATICFILES_DIRS = [
os.path.join(BASE_DIR, 'app/static')
]
```
- `python manage.py collectstatic`
- Create a base.html file and add some content, to be able to add all nececary informaion to all pages
- Create a resume.html file to add a separate file which will be a resume page
- Create folder "partials" and add file _navbar.html and _footer.html
- Add a nice structure for the navbar and footer, this will be showed on all pages

### Dockerize the development
- python freeze > requirements.txt make sure to be in the main path
- Add a Dockerfile.dev
```
FROM python:3.8-slim-buster

WORKDIR /app

COPY requirements.txt requirements.txt

RUN pip3 install -r requirements.txt

COPY . .

CMD [ "python3", "manage.py", "runserver", "0.0.0.0:8000" ]
```
CMD
- `docker build -f Dockerfile.dev -t amfchef/resume_dev .`
- `docker push amfchef/resume_dev`
- `docker run -p 8000:8000 $(pwd):/app amfchef/resume_dev`
- open your localhost:8000 and see if your website shows
- Try to change some text in your main html file, and see if it changes automatically
- This is your development

### Dockerize your production
We will be using: 
- Gunicorn as a WSGI server (web server gateway). Gunicorn communicates with the python application. To send back responses to the webserver,
that will forward it to client.
- Nginx as a webserver. Nginx will manage the requests from the users,who open the web browser.
The request will either look for static files, or dynamic files. And we tell Nginx where to look for the files.

Set up Nginx
- Create a folder named Nginx
- create a file default.conf and add configurations:
```
  upstream django {
  server django:8000;
}

server {
  listen 80;

  location / {
    proxy_pass http://django;
  }
  location /static {
    alias /static/;
  }
}
```
- In the same folder, create a Dockerfile in Nginx
This will copy the Nginx config to docker
```
FROM nginx:1.19-alpine
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

- Create a Dockerfile in out django folder
```
FROM python:3.8.5-alpine

RUN pip install --upgrade pip

COPY ./requirements.txt .
RUN pip install -r requirements.txt

COPY ./app /app

WORKDIR /app

COPY ./entrypoint.sh /
ENTRYPOINT ["sh", "/entrypoint.sh"]
```
- Add a requirements.txt file and add:
```
Django==3.2.6
gunicorn==20.0.4
```
- Create a second Dockerfile
```
FROM python:3.8.5-alpine

RUN pip install --upgrade pip

COPY ./requirements.txt .
RUN pip install -r requirements.txt

COPY ./app /app

WORKDIR /app

COPY ./entrypoint.sh /
ENTRYPOINT ["sh", "/entrypoint.sh"]
``` 

- Create an entrypoint.sh file
```
#!/bin/sh

python manage.py migrate --no-input
python manage.py collectstatic --no-input

gunicorn app.wsgi:application --bind 0.0.0.0:8000
```
- Create a docker-compose.yml file, this is to bind two different dockerfile together
```
version: "3.7"

services: 
  django:
    volumes:
      - static:/static
    env_file:
      - .env
    build:
      context: ./django
    ports:
      - "8000:8000"
  nginx:
    build: ./nginx
    volumes:
      - static:/static
    ports:
      - "80:80"
    depends_on: 
      - django

volumes:
  static:
```
- Create a .env file
```
SECRET_KEY = 'mySecretKey
DEBUG = True
```
- We dont want this file in gitignore, so we add a .gitignore file
```
.env
```

- `docker-compose up --build`
This command will set up new implemented code to our production
Thus I dont have a server with a domain to access it, therefore I can only access it from my localhost.
