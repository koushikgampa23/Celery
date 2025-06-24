# Celery
Contains celery, redis
## To Run Celery Requirements
    1. docker-compose.yml file that contains redis service, celery service, django service
    2. Dockerfile for django application
    3. celery.py file need to present next to settings.py file
    4. In the __init__.py file import celery app
    5. tasks.py file need to placed in app example: apis/tasks.py next to view.py file
    6. Add CELERY_BROKER_URL = os.environ.get("CELERY_BROKER", "redis://redis:6379/0")
### Project structure Example
    Repos/   # This Repos folder can contain many projects 
        dcelery/
            - dcelery
                - __init__.py
                - settings.py
                - urls
                - celery.py
            - requirements.txt/poetry.toml/manage.py
            - Dockerfile
            - README.md (poetry won't work if not present)
            - apis/
                - views.py
                - tasks.py
        docker-compse.yml
#### 1.docker-compose.yml (Mind the spacing in yml file)
    This docker compose file contains 2 workers included
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
            command: poetry run celery -A dcelery worker -l INFO -Q queue1 # -Q is for queue #celery --app=dcelery worker -l INFO # -l stands for logs
            volumes:
                - ./dcelery:/app
            depends_on:
                - redis
                - dcelery
        # Optional iam creating worker2
        celery2:
            build:
                context: ./dcelery
                dockerfile: Dockerfile
            command: poetry run celery -A dcelery worker -l INFO -Q queue2 # -Q is for queue # -l stands for logs
            volumes:
                - ./dcelery:/app
            depends_on:
                - redis
                - dcelery
    volumes:
        shared_location:

#### 2.Dockerfile
    FROM python:slim
    WORKDIR /app
    COPY poetry.lock /app/
    COPY pyproject.toml /app/
    COPY . .
    # RUN apt-get update && apt-get install -y gcc libpq-dev
    RUN pip install poetry && poetry install
    EXPOSE 8001
#### 3.celery.py
    import os
    from celery import Celery

    # we get this from manage.py file
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "dcelery.settings")
    app = Celery("dcelery")
    app.config_from_object("django.conf:settings", namespace="CELERY")  # all the settings that starts with celery are taken

    # @app.task  # Registring tasks to celery
    # def add_numbers():
    #     return
    
    # Code for Routers
    app.conf.task_routes = {
        "api.tasks.task1": {"queue": "queue1"},
        "api.tasks.task2": {"queue": "queue2"},
    }

    app.autodiscover_tasks() # To discover tasks across the project
#### 4.__init__.py
    from .celery import app as celery_app

    __all__ = celery_app

#### 5.tasks.py
    from celery import shared_task
    @shared_task
    def shared_task_demo1():
        return
    @shared_task
    def shared_task_demo2():
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
#### Enter into dcelery container and run shared task(play around with container)
    docker container ls
    copy the id of dcelery
    docker container exec -it <id> bash
    ls
    poetry run python manage.py shell
        from dcelery.celery import app
         __all__ = ("app",)
        from apis.tasks import shared_task_demo
        shared_task_demo.delay() # Connection refuse error if i dont import celery app
        Output: <AsyncResult: 267c6d43-a108-4e7d-b857-173b1a1ceab1>

#### Run this application
    docker compose up --build
    if everything goes well we can see this output
![alt text](docker_compose_output.png)

### Celery Task Prioritization
    Why do we need Task Prioritization?
        Ensure Critical Tasks are executed first
        Optimize resource utilization
        Meet SLAs and deadlines
        Handle high-priority or time-sensitive requests efficiently
    Setting Task Prioritization
        0 and 9(0 being lowest, 9 being the highest)
        Default specified in the task decorator or configuration
    Configuring worker queues:
        Define multiple queues representing different priorities
        Associate each queue with specific priority level
        Celery worker consumes tasks from queues based on priority
### Task Grouping
    Task Grouping allows us to execute multiple tasks in parallel
    It helps us manage dependencies among tasks and ensure that certain tasks are executed only after other.
    So result aggregation, process tracking, error handling, task composition
    So grouping tasks provides us a way to build complex workflow by combining multiple smaller tasks
#### Task Grouping Handson
    from celery import group
    task_group = group(
            shared_task_demo1.s(), #s() stands for signature
            shared_task_demo2.s(),
            shared_task_demo3.s(),
            shared_task_demo4.s(),
        )
    task_group.apply_async()
    This four tasks will be executed parallelly
    demo1 and demo3 will be executed first since there time.sleep is 10 seconds
    later demo2, demo4 will be executed since it is 20 seconds
### Task Chaining
    Gives us the ability to create a sequence of tasks
    This potentially provides us the ability to use output from one task for the input for the next task in the chain.
    So we can dependencies and create order of execution
### Task Grouping vs Task Chaining
    Task chaining is a sequential exection of tasks, where as output of one task is the input of other.
    Task grouping involves executing mulitple tasks parallel without any specific dependencies or order between them 
#### Task Chaining Handson
    from celery import chain
    task_group = chain(
            shared_task_demo1.s(),
            shared_task_demo3.s(),
            shared_task_demo2.si(),
            shared_task_demo4.si(),
        )
    Change shared_task_demo3
    @shared_task
    def shared_task_demo3(previous_value):
        time.sleep(10)
        return previous_value + 10
    task_group.apply_async()
    si() ignores previous result
    s() previous value is needed in chaining 






