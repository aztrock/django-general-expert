# Docker & Deployment Guide

Guide to containerizing and deploying Django applications.

## Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

# Set work directory
WORKDIR /app

# Install dependencies
COPY requirements/production.txt .
RUN pip install --no-cache-dir -r production.txt

# Copy project
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

# Expose port
EXPOSE 8000

# Run server
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

## Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=0
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/media

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  celery:
    build: .
    command: celery -A config worker -l info
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - .:/app

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - static_volume:/app/staticfiles:ro
      - media_volume:/app/media:ro
    depends_on:
      - web

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

## Nginx Configuration

```nginx
# nginx.conf
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name example.com;

    client_max_body_size 10M;

    # Static files
    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Media files
    location /media/ {
        alias /app/media/;
        expires 7d;
    }

    # Proxy to Django
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}
```

## Environment Variables

```bash
# .env
# Django
SECRET_KEY=your-secret-key-here
DEBUG=0
ALLOWED_HOSTS=example.com

# Database
DB_ENGINE=django.db.backends.postgresql
DB_NAME=mydb
DB_USER=user
DB_PASSWORD=pass
DB_HOST=db
DB_PORT=5432

# Redis
REDIS_URL=redis://redis:6379/0

# Email
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
```

## Production Checklist

### 1. Security Settings

```python
# settings/production.py

DEBUG = False

SECRET_KEY = os.environ.get('SECRET_KEY')

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# Security settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Static files
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

### 2. Database

```bash
# Run migrations
python manage.py migrate --noinput

# Create superuser
python manage.py createsuperuser
```

### 3. Collectstatic

```bash
python manage.py collectstatic --noinput
```

## Deployment Scripts

```bash
#!/bin/bash
# deploy.sh

# Pull latest code
git pull origin main

# Build Docker image
docker-compose build

# Run migrations
docker-compose run --rm web python manage.py migrate --noinput

# Collect static
docker-compose run --rm web python manage.py collectstatic --noinput

# Restart services
docker-compose restart web celery

# Clean up
docker system prune -f
```

## CI/CD Example (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker
        run: docker-compose build
      
      - name: Run tests
        run: docker-compose run --rm web pytest
      
      - name: Deploy to production
        run: |
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/key
          chmod 600 ~/.ssh/key
          ssh -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} "cd /app && docker-compose up -d"
```

## Summary

1. **Use Docker** for consistent environments
2. **Use docker-compose** for local development
3. **Use gunicorn** in production
4. **Use nginx** for static files
5. **Use environment variables** for secrets
6. **Use SSL/HTTPS** in production
7. **Set DEBUG=False** in production
8. **Collect static files** before deploy
9. **Run migrations** automatically
10. **Use CI/CD** for automated deployment
