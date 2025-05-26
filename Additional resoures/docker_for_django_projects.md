# Docker for Django Projects

Docker helps containerize your Django application, including all its dependencies, to ensure that 
it works seamlessly across different environments. Using Docker for Django along with Celery allows you to run 
background tasks efficiently in containers. Below is a basic guide for setting up Docker with Django, PostgreSQL, Redis (for Celery), and Celery itself.

---

### Docker Setup for Django and Celery

Here’s a step-by-step setup using Docker:

#### 1. Project Structure
Here’s the folder structure for your project:

```
my_django_project/
│
├── django_app/
│   ├── Dockerfile
│   ├── manage.py
│   ├── requirements.txt
│   ├── settings.py
│   └── other_django_files/
│
├── celery_app/
│   └── tasks.py
│
├── docker-compose.yml
└── entrypoint.sh
```

---

#### 2. `Dockerfile` (for Django)

```dockerfile
# Pull base image
FROM python:3.9-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set working directory
WORKDIR /usr/src/app

# Install dependencies
COPY requirements.txt /usr/src/app/
RUN pip install --no-cache-dir -r requirements.txt

# Copy the project files into the container
COPY . /usr/src/app/

# Run the Django server
CMD ["sh", "/usr/src/app/entrypoint.sh"]
```

---

#### 3. `docker-compose.yml`

```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - my_network

  redis:
    image: redis:6
    networks:
      - my_network

  web:
    build: ./django_app
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./django_app:/usr/src/app
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    networks:
      - my_network

  celery:
    build: ./django_app
    command: celery -A django_app worker --loglevel=info
    volumes:
      - ./django_app:/usr/src/app
    depends_on:
      - db
      - redis
    networks:
      - my_network

  celery-beat:
    build: ./django_app
    command: celery -A django_app beat --loglevel=info
    volumes:
      - ./django_app:/usr/src/app
    depends_on:
      - db
      - redis
    networks:
      - my_network

volumes:
  postgres_data:

networks:
  my_network:
```

---

#### 4. Entrypoint Script (`entrypoint.sh`)

This script ensures that the database is ready before running Django commands:

```bash
#!/bin/sh

if [ "$DATABASE" = "postgres" ]
then
    echo "Waiting for postgres..."

    while ! nc -z db 5432; do
      sleep 0.1
    done

    echo "PostgreSQL started"
fi

# Run migrations and start server
python manage.py migrate
python manage.py collectstatic --no-input --clear

exec "$@"
```

Make sure to give execute permission to the script:

```bash
chmod +x entrypoint.sh
```

---

#### 5. Requirements (`requirements.txt`)

Include the necessary packages for Django, Celery, PostgreSQL, and Redis:

```
Django>=3.2,<4.0
psycopg2-binary
celery
redis
```

---

### Running the Project

1. Build and run the containers:

```bash
docker-compose up --build
```

2. Access your Django app at `http://localhost:8000`.

3. Celery workers will automatically start to handle tasks defined in the `tasks.py` file.

---

### Key Benefits of Dockerizing Django with Celery:
- **Consistency**: Your Django and Celery environments are consistent across development and production.
- **Scalability**: Containers allow for easy scaling of both the web application and Celery workers.
- **Isolation**: Each service (Django, Celery, PostgreSQL, Redis) runs in its own container, making them modular and easy to manage.

---
