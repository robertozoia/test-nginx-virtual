# Nginx Virtual Host Setup with SSL

This setup runs two domains on a single Nginx server using Docker:

- `app.example.com`: Python Flask application
- `example.com`: WordPress installation

The idea is that Wordpress is the frontpage to your site/app, and a link somewhere in Wordpress redirects the user to the actual app.


## Prerequisites

- Docker Engine 24.0.0 or later
- Domains pointed to your server's IP
- Ports 80 and 443 open

## Initial Setup for SSL Certificates

1. First, create the required directories:
```bash
mkdir -p nginx/conf certbot/conf certbot/www flask-app wordpress
```

2. Create initial Nginx configurations for HTTP only (required for SSL certificate generation)

`nginx/conf/app.example.com.conf`:
```nginx
server {
    listen 80;
    server_name app.example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://flask-app:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`nginx/conf/example.com.conf`:
```nginx
server {
    listen 80;
    server_name example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        root /var/www/html;
        index index.php;
        
        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass wordpress:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }
    }
}
```

3. Create initial `docker-compose.yml` for certificate generation:
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
      - wordpress:/var/www/html
    depends_on:
      - flask-app
      - wordpress
    networks:
      - web_network

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    command: certonly --webroot -w /var/www/certbot 
      --email your@email.com 
      -d app.example.com 
      -d example.com 
      --agree-tos
      --force-renewal
      --non-interactive

  flask-app:
    build: ./flask-app
    expose:
      - 8000
    networks:
      - web_network

  wordpress:
    image: wordpress:php8.1-fpm
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=${WORDPRESS_DB_USER}
      - WORDPRESS_DB_PASSWORD=${WORDPRESS_DB_PASSWORD}
      - WORDPRESS_DB_NAME=${WORDPRESS_DB_NAME}
    volumes:
      - wordpress:/var/www/html
    depends_on:
      - db
    networks:
      - web_network

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${WORDPRESS_DB_NAME}
      - MYSQL_USER=${WORDPRESS_DB_USER}
      - MYSQL_PASSWORD=${WORDPRESS_DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - web_network

volumes:
  wordpress:
  db_data:

networks:
  web_network:
    driver: bridge
```

4. Create `.env` file:
```env
WORDPRESS_DB_USER=wordpress
WORDPRESS_DB_PASSWORD=wordpress_password
WORDPRESS_DB_NAME=wordpress
MYSQL_ROOT_PASSWORD=somewordpress
```

5. Create Flask application files:

`flask-app/app.py`:
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Flask app on app.example.com!'

if __name__ == '__main__':
    app.run()
```

`flask-app/wsgi.py`:
```python
from app import app

if __name__ == "__main__":
    app.run()
```

`flask-app/Dockerfile`:
```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY . .

RUN pip install flask gunicorn

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "wsgi:app"]
```

## Getting SSL Certificates

1. Start services:
```bash
docker compose up -d
```

2. Generate certificates:
```bash
docker compose run --rm certbot
```

3. After certificates are generated, update Nginx configurations with SSL:

`nginx/conf/app.example.com.conf`:
```nginx
server {
    listen 80;
    server_name app.example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://flask-app:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`nginx/conf/example.com.conf`:
```nginx
server {
    listen 80;
    server_name example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/html;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

4. Restart Nginx:
```bash
docker compose restart nginx
```

## Accessing the Applications

- Flask application: https://app.example.com
- WordPress: https://example.com

## Notes

- Replace `your@email.com` with your actual email in the certbot configuration
- Use strong passwords in production
- SSL certificates will auto-renew every 90 days
- For production, consider adding:
  - Regular database backups
  - Enhanced security configurations
  - Monitoring and logging
  - Rate limiting

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.



