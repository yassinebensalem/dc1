version: "3.9"

services:
  keycloak:
    container_name: keycloak
    image: jboss/keycloak:16.1.1
    networks:
      - letsencrypt-docker-compose
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
    networks:
      - letsencrypt-docker-compose
    restart: unless-stopped
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
networks:
  letsencrypt-docker-compose:
    name: letsencrypt-docker-compose_default
    external: true
