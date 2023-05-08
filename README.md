version: "3"

services:
  cli:
    build: ./cli
    image: evgeniy-khyst/letsencrypt-docker-compose-cli
    user: ${CURRENT_USER}
    group_add:
      - ${DOCKER_GROUP}
    environment:
      - COMPOSE_PROJECT_NAME
      - CURRENT_USER
      - DOCKER_GROUP
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/workdir
    profiles:
      - config

  nginx:
    build: ./nginx
    image: evgeniy-khyst/nginx
    environment:
      - DRY_RUN
    volumes:
      - ./config.json:/letsencrypt-docker-compose/config.json:ro
      - ./nginx-conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx-conf/conf.d:/etc/nginx/conf.d:ro
      - ./html:/var/www/html:ro
      - nginx_conf_ssl:/etc/nginx/ssl
      - letsencrypt_certs:/etc/letsencrypt
      - certbot_acme_challenge:/var/www/certbot
    ports:
      - "80:80"
      - "443:443"
    healthcheck:
      test: ["CMD", "nc", "-z", "nginx", "80"]
      interval: 30s
      timeout: 30s
      start_period: 30s
      retries: 10
    restart: unless-stopped

  certbot:
    build: ./certbot
    image: evgeniy-khyst/certbot
    environment:
      - DRY_RUN
    volumes:
      - ./config.json:/letsencrypt-docker-compose/config.json:ro
      - letsencrypt_certs:/etc/letsencrypt
      - certbot_acme_challenge:/var/www/certbot
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'test -n "$$(ls -A /etc/letsencrypt/live/)" || test "$$DRY_RUN" == "true" || exit 1',
        ]
      interval: 30s
      timeout: 30s
      start_period: 30s
      retries: 5
    depends_on:
      nginx:
        condition: service_healthy
    restart: unless-stopped

  cron:
    build: ./cron
    image: evgeniy-khyst/cron
    environment:
      - COMPOSE_PROJECT_NAME
      - DRY_RUN
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./:/workdir:ro
    depends_on:
      certbot:
        condition: service_healthy
    restart: unless-stopped

  # Replace with your backend image or remove
  example-backend:
    build: ./examples/nodejs-backend
    image: evgeniy-khyst/expressjs-helloworld
    restart: unless-stopped

volumes:
  nginx_conf_ssl:
  letsencrypt_certs:
  certbot_acme_challenge:


  example-backend:
    
    image: jboss/keycloak:16.1.1
   
    restart: unless-stopped
    depends_on:
      - keycloak-postgres
    environment:
      DB_ADDR: keycloak-postgres
      DB_USER: keycloak
      DB_PASSWORD: keycloak
      DB_VENDOR: postgres
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: admin
      KEYCLOAK_IMPORT: "/tmp/trial_realm.json"

      JAVA_OPTS: "-Djboss.http.port=10000 -Djgroups.bind_addr=127.0.0.1 -Dkeycloak.profile.feature.scripts=enabled -Dkeycloak.profile.feature.upload_scripts=enabled"
    ports:
      - "10000:10000"
    volumes:
      - ./keycloak/export:/tmp
      - ./keycloak/themes/aos:/opt/jboss/keycloak/themes/aos
      - ./keycloak/themes/trial:/opt/jboss/keycloak/themes/trial
      - ./keycloak/themes/custom:/opt/jboss/keycloak/themes/custom
      - ./keycloak/themes/basic:/opt/jboss/keycloak/themes/basic

  keycloak-postgres:
    container_name: keycloak-postgres
    image: postgres:13
    
    restart: unless-stopped
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak

