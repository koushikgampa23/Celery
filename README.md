# Celery
Contains celery, redis
## To Run Celery Requirements
    1. docker-compose.yml file that contains redis service, celery service, django service
    2. Dockerfile for django application
    3. celery.py file need to present next to requirements.txt/poetry.toml file
    4. tasks.py file need to placed in app example: apis/tasks.py next to view.py file
    5. Add CELERY_BROKER_URL = os.environ.get("CELERY_BROKER", "redis://redis:6379/0")
### Project structure Example
    Repos/   # This Repos folder can contain many projects 
        dcelery/
            - dcelery
                - settings.py
                - urls
            - requirements.txt/poetry.toml/manage.py
            - celery.py
            - Dockerfile
            - README.md (poetry won't work if not present)
            - apis/
                - views.py
                - tasks.py
        docker-compse.yml
#### docker-compose.yml
    version: "3.8"
    services:
    redis:
        image: redis
    dcelery:
        build:
        context: ./dcelery
        dockerfile: Dockerfile
        ports:
        - 8001:8001
        command:
        ["poetry", "run", "python", "manage.py", "runserver", "0.0.0.0:8001"]
        volumes:
        - ./dcelery:/app
        depends_on:
        - redis
    # Copy django compose code remove ports and modify command to make celery compose
    celery:
        build:
        context: ./dcelery
        dockerfile: Dockerfile
        command: poetry run celery --app=dcelery worker -l INFO # -l stands for logs
        volumes:
        - ./dcelery:/app
        depends_on:
        - redis
        - dcelery
    volumes:
    shared_location:
#### Dockerfile
    FROM python:slim
    WORKDIR /app
    COPY poetry.lock /app/
    COPY pyproject.toml /app/
    # RUN apt-get update && apt-get install -y gcc libpq-dev
    RUN pip install poetry && poetry install
    COPY . .
    EXPOSE 8001
#### celery.py
    import os
    from celery import Celery

    # we get this from manage.py file
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "dcelery.settings")
    app = Celery("dcelery")
    app.config_from_object("django.conf:settings", namespace="CELERY")  # all the settings that starts with celery are taken

    @app.task  # Registring tasks to celery
    def add_numbers():
        return

    app.autodiscover_tasks() # To discover tasks across the project
#### tasks.py
    from celery import shared_task
    @shared_task
    def shared_task_demo():
        return
#### Django Project Setup using poetry
    1. I have [Project] folder
    2. mkdir dcelery
    3. cd dcelery
    4. poetry init
    5. poetry add django
    6. poetry run django-admin startproject dcelery . (or) django-admin startproject dcelery .
    7. poetry run python manage.py runserver (or) python manage.py runserver
    8. poetry add redis celery djangorestframework
    9. poetry run python manage.py startapp apis
    10. Create urls file and add it to main urls using include, Add apis in installed apps(See code in below)
    11. Add celery next to manage.py file
    12. Add tasks in apis folder next to view file
    13. Add in settings.py CELERY_BROKER_URL = os.environ.get("CELERY_BROKER", "redis://redis:6379/0")
    14. Add dockerfile next to manage.py file
    15. Add docker-compose.yml same level as main dcelery(for reference see project structure header)

