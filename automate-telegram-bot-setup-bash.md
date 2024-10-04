You can automate the entire setup process using a Bash script. This script will:

1. Create the necessary files and directories.
2. Set up the Dockerized Django project with DRF.
3. Set up PostgreSQL for Django.
4. Run Docker Compose to build the containers.
5. Set up Ngrok.
6. Register the webhook with the Telegram API.

### 1. **Create the Bash Script**

Let's create a Bash script called `setup.sh` that will automate the process.

```bash
#!/bin/bash

# Variables
PROJECT_NAME="telegram_bot"
DJANGO_APP="bot"
POSTGRES_DB="botdb"
POSTGRES_USER="botuser"
POSTGRES_PASSWORD="botpassword"
DJANGO_PORT=8000
NGROK_PORT=4040
NGROK_URL=""
TELEGRAM_TOKEN="your_telegram_bot_token_here"  # Replace this with your Telegram Bot Token
WEBHOOK_URL=""

# Create project directory
echo "Creating project directory..."
mkdir -p $PROJECT_NAME
cd $PROJECT_NAME

# Create Dockerfile
echo "Creating Dockerfile..."
cat <<EOF > Dockerfile
# Dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED 1

WORKDIR /app

COPY requirements.txt /app/
RUN pip install -r requirements.txt

COPY . /app/

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
EOF

# Create docker-compose.yml
echo "Creating docker-compose.yml..."
cat <<EOF > docker-compose.yml
version: '3'

services:
  db:
    image: postgres:13
    environment:
      POSTGRES_DB: $POSTGRES_DB
      POSTGRES_USER: $POSTGRES_USER
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
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
      - "$DJANGO_PORT:8000"
    depends_on:
      - db
    environment:
      - POSTGRES_DB=$POSTGRES_DB
      - POSTGRES_USER=$POSTGRES_USER
      - POSTGRES_PASSWORD=$POSTGRES_PASSWORD

volumes:
  postgres_data:
EOF

# Create requirements.txt
echo "Creating requirements.txt..."
cat <<EOF > requirements.txt
django
djangorestframework
psycopg2
python-telegram-bot
EOF

# Build the Docker containers
echo "Building Docker containers..."
docker-compose up -d --build

# Create the Django project and app
echo "Creating Django project and app..."
docker-compose exec web django-admin startproject mybot .
docker-compose exec web python manage.py startapp $DJANGO_APP

# Configure Django settings for PostgreSQL
echo "Configuring Django settings..."
sed -i "/DATABASES =/a\\
DATABASES = {\\
    'default': {\\
        'ENGINE': 'django.db.backends.postgresql',\\
        'NAME': '$POSTGRES_DB',\\
        'USER': '$POSTGRES_USER',\\
        'PASSWORD': '$POSTGRES_PASSWORD',\\
        'HOST': 'db',\\
        'PORT': '5432',\\
    }\\
}" mybot/settings.py

# Add DRF to installed apps
sed -i "/INSTALLED_APPS =/a\\
    'rest_framework',\\
    '$DJANGO_APP'," mybot/settings.py

# Create basic webhook API in views.py
echo "Creating basic webhook API..."
cat <<EOF > $DJANGO_APP/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['POST'])
def telegram_webhook(request):
    data = request.data
    print(data)  # Log incoming data for now
    return Response({"status": "ok"})
EOF

# Create URLs for the webhook
echo "Configuring URLs..."
cat <<EOF > $DJANGO_APP/urls.py
from django.urls import path
from .views import telegram_webhook

urlpatterns = [
    path('webhook/', telegram_webhook, name='telegram_webhook'),
]
EOF

# Update main Django URLs
sed -i "/urlpatterns =/a\\
    path('$DJANGO_APP/', include('$DJANGO_APP.urls'))," mybot/urls.py

# Run Django migrations
echo "Running migrations..."
docker-compose exec web python manage.py migrate

# Start ngrok for port 8000
echo "Starting ngrok..."
NGROK_URL=\$(curl --silent --show-error http://127.0.0.1:$NGROK_PORT/api/tunnels | grep -oP 'https://[0-9a-zA-Z]+\.ngrok\.io')

if [[ -z "\$NGROK_URL" ]]; then
  docker run -d -p $NGROK_PORT:4040 -p $DJANGO_PORT:8000 --network="host" wernight/ngrok ngrok http 8000
  sleep 2
  NGROK_URL=\$(curl --silent --show-error http://127.0.0.1:$NGROK_PORT/api/tunnels | grep -oP 'https://[0-9a-zA-Z]+\.ngrok\.io')
fi

if [[ -z "\$NGROK_URL" ]]; then
  echo "Ngrok failed to start."
  exit 1
else
  echo "Ngrok started at \$NGROK_URL"
fi

# Set the Telegram bot webhook
echo "Setting Telegram webhook..."
WEBHOOK_URL="\$NGROK_URL/$DJANGO_APP/webhook/"
curl -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/setWebhook" -d "url=\$WEBHOOK_URL"

echo "Webhook set to \$WEBHOOK_URL"

# Done
echo "Setup complete! Your bot is running."
```

### 2. **How the Script Works**

1. **Sets up project variables** like project name, database credentials, Django app name, and the Telegram bot token.
2. **Creates the project structure**: Dockerfile, Docker Compose file, and requirements.txt.
3. **Runs Docker Compose** to start the Django project and PostgreSQL.
4. **Creates Django project and app**, configures the database, and sets up the basic webhook view.
5. **Starts Ngrok** and fetches the public URL.
6. **Sets the Telegram webhook** using the public Ngrok URL.

### 3. **How to Use the Script**

1. **Make the script executable**:
   ```bash
   chmod +x setup.sh
   ```

2. **Run the script**:
   ```bash
   ./setup.sh
   ```

The script will build the Docker containers, start the project, configure Ngrok, and set the Telegram webhook. Once the script finishes, your bot will be running, and Telegram will send updates to your Ngrok URL.

### 4. **Replace with Your Telegram Token**

Make sure to replace `your_telegram_bot_token_here` with the token you get from [BotFather](https://core.telegram.org/bots#botfather).
