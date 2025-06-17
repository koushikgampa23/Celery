# Celery
Contains celery, redis
## Celery Implementation
### Django Project Setup
    1. Create Virtual Env
        virtualenv venv
    2. Activate env
        venv\Scripts\activate
    3. Install django
        pip install django
    4. Create Project
        django-admin startproject dcelery
    5. Navigate to dcelery application
        cd dcelery
        python manage.py runserver
### Run this application using Dockerfile
    1. Write docker file
        FROM python:slim
        WORKDIR /app
        COPY . .
        RUN pip install -r requirements.txt
        CMD ["python", "manage.py", "runserver"]
    2. To build image and run
        docker build -t dcelery .
