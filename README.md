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
### Task Rate Limit
    We have done setup for multiple queues and each queue is assigned a worker 
    Queue1 can have potentally bigger cpu mainly used for calculations
    Queue4 which had a lower priority
    Where will this task rate limit fits
    This is gone prevent overwhelming of the system
    We can enforce a rate restriction on task execution
    The queue1 that has higher priority can have higher rate limiting
    The queue4 that has lower priority can have lower rate limiting
    Pending Implementation
## Configuring Task Prioritization using Rabbit MQ
    Rabbit MQ Stores tasks similar to Redis
    Celery will always selects tasks based on highest priority to process first
### Install RabbitMQ
    pip install pika or poetry add pika, add in settings.py file installed apps
### Defination of configuration and difference
    worker_concurrency: 
        Number of worker that can run concurrently. and each worker executes one task at a time
        Higher concurrency means more tasks can be processed in parallel, but it can also lead to higher memory usage and contention for resources.
    worker_prefetch_multiplier: 
        Number of tasks that each worker can prefetch at a time.
    Difference between worker_concurrency and worker_prefetch_multiplier:
        Consider if you have 6 tasks?
            set worker_concurrency to 2 and worker_prefetch_multiplier to 3, then we get 2 workers, each worker will prefetch 3 tasks, so total of 6 tasks are performed in parallel. the worker 1 can handle 3 tasks in fifo.
### File Modifications
    1. Settings.py
        Replace redis broker url with rabbitmq broker url
        CELERY_BROKER_URL = os.environ.get(
            "CELERY_BROKER", "amqp://guest:guest@rabbitmq:5672//"
        )
    2. In the celery.py file add this
        # RabbitMQ message priority
        app.conf.task_queues = [
            Queue(
                "tasks",
                Exchange("tasks"),
                routing_key="tasks",
                queue_arguments={"x-max-priority": 10},
            ),
        ]

        app.conf.task_default_queue = "tasks"
        app.conf.task_default_exchange = "tasks"
        app.conf.task_default_exchange_type = "direct"
        app.conf.task_default_routing_key = "tasks"

        app.conf.task_acks_late = (
            True  # to ensure that the task is acknowledged only after it has been executed
        )
        app.conf.task_default_priority = 5
        app.conf.worker_prefetch_multiplier = (
            1  # to ensure that only one task is prefetched at a time
        )
        app.conf.worker_concurrency = 2  # number of worker processes to run
    3. In the shared_tasks
        @app.task(queue="tasks")
        def t1():
            time.sleep(20)
            return
        @app.task(queue="tasks")
        def t2():
            time.sleep(20)
            return
        @app.task(queue="tasks")
        def t3():
            time.sleep(20)
            return
    4. in the dummy api of test add this 
        t2.apply_async(priority=5)
        t1.apply_async(priority=1)
        t3.apply_async(priority=10)
    5. Docker compose file run celery like this
        poetry run celery -A dcelery worker --loglevel=info -Q tasks
### Different ways to run Celery
    poetry run celery -A dcelery -l Info (--queues are optional)

    poetry run celery -A dcelery -l Info -Q tasks
    poetry run celery --app=dcelery --loglevel=info --queues=tasks (both commands are equal)

    poetry run celery -A dcelery -l Info -Q celery,celery1,celery2,celery3 #Use only if broker_transport_options is enabled and created celery priorites
### Passing Arguments to celery task
    @app.task(queue="tasks")
    def t1(a: int, b: int, message="None"):
        result = a + b
        if message:
            result = f"{a} + {b} = {a + b}, message: {message}"
        return result
    t1.apply_async(priority=6, args=[1, 2], kwargs={"message": "addition performed"})












