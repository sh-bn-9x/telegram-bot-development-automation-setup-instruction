# Quick Start: Setting Up a Telegram Bot API Development Environment Using Docker and DRF


To quickly set up a development environment for a Telegram Bot API using **Docker**, **Django REST Framework (DRF)**, and **Ngrok** for tunneling, follow these steps:

### 1. **Project Setup**

Start by creating a Django project using Docker, with Django REST Framework (DRF) to serve the Telegram bot API, and Ngrok to expose the local development server to the internet.

#### Prerequisites:
- Docker installed
- Ngrok installed (or use the official Docker container)
- A Telegram Bot token (You can get this by talking to [BotFather](https://core.telegram.org/bots#botfather))

---

### 2. **Step-by-Step Setup**

#### 2.1. Create the Dockerized Django + DRF Project

1. **Create a project directory**:
   ```bash
   mkdir telegram_bot
   cd telegram_bot
   ```

2. **Create `Dockerfile`** in the root of the project:

   ```Dockerfile
   # Dockerfile
   FROM python:3.11-slim

   ENV PYTHONUNBUFFERED 1

   WORKDIR /app

   COPY requirements.txt /app/
   RUN pip install -r requirements.txt

   COPY . /app/

   EXPOSE 8000

   CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
   ```

3. **Create `docker-compose.yml`** for Docker Compose:

   ```yaml
   version: '3'

   services:
     db:
       image: postgres:13
       environment:
         POSTGRES_DB: botdb
         POSTGRES_USER: botuser
         POSTGRES_PASSWORD: botpassword
       volumes:
         - postgres_data:/var/lib/postgresql/data
       ports:
         - "5432:5432"

     web:
       build: .
       command: python manage.py runserver 0.0.0.0:8000
       volumes:
         - .:/app
       ports:
         - "8000:8000"
       depends_on:
         - db
       environment:
         - POSTGRES_DB=botdb
         - POSTGRES_USER=botuser
         - POSTGRES_PASSWORD=botpassword

   volumes:
     postgres_data:
   ```

4. **Create `requirements.txt`** to include necessary dependencies:

   ```txt
   django
   djangorestframework
   psycopg2
   python-telegram-bot
   ```

5. **Start the Docker containers**:

   Run the following command to build and start the containers:

   ```bash
   docker-compose up --build
   ```

---

#### 2.2. Set up the Django Project with DRF

1. **Create a Django project and an app**:

   Open a terminal inside the `web` container:

   ```bash
   docker-compose exec web bash
   ```

   Inside the container, create a Django project:

   ```bash
   django-admin startproject mybot .
   python manage.py startapp bot
   ```

2. **Configure PostgreSQL**:

   Update `mybot/settings.py` to configure PostgreSQL as the database:

   ```python
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.postgresql',
           'NAME': 'botdb',
           'USER': 'botuser',
           'PASSWORD': 'botpassword',
           'HOST': 'db',
           'PORT': '5432',
       }
   }
   ```

3. **Add DRF to `INSTALLED_APPS`**:

   In `mybot/settings.py`, add the following:

   ```python
   INSTALLED_APPS = [
       'rest_framework',
       'bot',
       # other apps
   ]
   ```

4. **Create a Simple API in DRF**:

   In the `bot/views.py`, create a simple endpoint to receive the Telegram bot webhook:

   ```python
   from rest_framework.decorators import api_view
   from rest_framework.response import Response

   @api_view(['POST'])
   def telegram_webhook(request):
       data = request.data
       print(data)  # For now, just log the incoming data
       return Response({"status": "ok"})
   ```

   Define the URL route for this API in `bot/urls.py`:

   ```python
   from django.urls import path
   from .views import telegram_webhook

   urlpatterns = [
       path('webhook/', telegram_webhook, name='telegram_webhook'),
   ]
   ```

   Include this URL in the main `urls.py` file of the project:

   ```python
   from django.contrib import admin
   from django.urls import path, include

   urlpatterns = [
       path('admin/', admin.site.urls),
       path('bot/', include('bot.urls')),
   ]
   ```

---

#### 2.3. Setup Ngrok for Tunneling

Ngrok will help you expose the local server running on Docker to a public URL that Telegram's API can access.

1. **Run Ngrok**:

   Run Ngrok to tunnel your localhost. If Ngrok is installed locally, use:

   ```bash
   ngrok http 8000
   ```

   Alternatively, use a Dockerized version of Ngrok:

   ```bash
   docker run -it --rm -p 4040:4040 -p 8000:8000 --network="host" wernight/ngrok ngrok http 8000
   ```

   Copy the public forwarding URL provided by Ngrok (e.g., `https://abc123.ngrok.io`).

2. **Set Webhook in Telegram**:

   You can now set your Telegram webhook URL using the bot token you obtained from the BotFather. Replace `<NGROK_URL>` and `<TOKEN>` with your values.

   ```bash
   curl -X POST https://api.telegram.org/bot<TOKEN>/setWebhook \
       -d url=<NGROK_URL>/bot/webhook/
   ```

   For example:

   ```bash
   curl -X POST https://api.telegram.org/bot123456789:ABCdefGhijKlmnoPqr/setWebhook \
       -d url=https://abc123.ngrok.io/bot/webhook/
   ```

3. **Test the Webhook**:

   When you send a message to your Telegram bot, you should see the data being logged in the Docker container logs where the Django app is running.

---

### 3. **Final Project Structure**

After the setup, your project structure should look something like this:

```
telegram_bot/
├── bot/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── migrations/
│   ├── models.py
│   ├── tests.py
│   ├── urls.py
│   ├── views.py
├── mybot/
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
```

---

### 4. **Next Steps**

- **Extend Bot Functionality**: Use `python-telegram-bot` to add more complex functionality to your bot.
- **Environment Variables**: For security, store sensitive data like the Telegram bot token in environment variables using Docker's `env_file` option or a secrets management tool.
- **Production Deployment**: You can use services like Heroku, AWS, or DigitalOcean to deploy the app in production.

This setup should help you quickly build and test Telegram bot APIs using Django, Docker, and Ngrok!
