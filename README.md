# Despliegue local protegido

Este repositorio contiene el Compose de integración. Construye los tres servicios desde los clones de los repos previos y construye `loadtest` desde `trabajo7-ddos`, como imagen separada. Las carpetas `source-repos` y `trabajo7-ddos` deben estar junto a este directorio, como en la estructura de trabajo local.

## Requisitos

- Docker Desktop iniciado y `docker compose` disponible.
- Clones locales de `trabajo7-ldap`, `trabajo7-backend`, `trabajo7-frontend` y `trabajo7-rotator` en `../source-repos/`.
- Repo `trabajo7-ddos` en `../trabajo7-ddos/`.

Desde PowerShell, crea una carpeta de trabajo en el Escritorio y ejecuta estos comandos en orden. Así no se clona por accidente dentro de `C:\Windows\System32`:

```powershell
Set-Location "$HOME\Desktop"
New-Item -ItemType Directory -Force entrega-fail2ban
Set-Location entrega-fail2ban

git clone https://github.com/Gallo-On/trabajo7-fail2ban-deploy.git
New-Item -ItemType Directory -Force source-repos
git clone https://github.com/Gallo-On/trabajo7-ldap.git source-repos/trabajo7-ldap
git clone https://github.com/Gallo-On/trabajo7-backend.git source-repos/trabajo7-backend
git clone https://github.com/Gallo-On/trabajo7-frontend.git source-repos/trabajo7-frontend
git clone https://github.com/Gallo-On/trabajo7-rotator.git source-repos/trabajo7-rotator
git clone https://github.com/Gallo-On/trabajo7-ddos.git

Copy-Item .\trabajo7-fail2ban-deploy\.env.example .\trabajo7-fail2ban-deploy\.env
Set-Location .\trabajo7-fail2ban-deploy
```

Si `git clone` del repo DDoS vuelve a fallar por TLS, reinténtalo desde la carpeta `entrega-fail2ban` con `git -c http.version=HTTP/1.1 clone https://github.com/Gallo-On/trabajo7-ddos.git` antes de levantar Compose.

Antes de arrancar, copia `.env.example` a `.env`. Los valores de ejemplo son únicamente para el laboratorio local; no reutilizarlos en producción ni para servicios expuestos a Internet.

Los puertos predeterminados publicados son frontend `8080`, backend `5000` y LDAPS `636`. Se pueden cambiar con `FRONTEND_HOST_PORT`, `BACKEND_HOST_PORT` y `LDAPS_HOST_PORT`. LDAP sin TLS `389` solo es accesible dentro de la red Compose, para que el backend pueda autenticar; el host publica TLS/636.

El laboratorio fija `LDAP_TLS_VERIFY_CLIENT=never`: LDAPS cifra el canal con el certificado de servidor autofirmado, y las pruebas de login usan bind LDAP, no certificados de cliente. No usar esta configuración de certificados de prueba como confianza de producción.

## Arranque y verificación

Desde esta carpeta:

```sh
docker compose up -d --build
docker compose ps
```

Abrir el dashboard en `http://localhost:8080`. Revisar los jails:

```sh
docker compose exec -T frontend fail2ban-client status http-flood
docker compose exec -T backend fail2ban-client status http-flood
docker compose exec -T openldap fail2ban-client status ldap-auth
docker compose exec -T openldap fail2ban-client status ldaps-connection
```

## Pruebas controladas

El runner se construye desde un repositorio y una imagen propios, fuera de los tres servicios protegidos. Las pruebas se limitan a la red local de Compose.

Frontend HTTP:

```sh
docker compose run --rm loadtest --protocol http --host frontend --port 80 --path / --duration 20 --concurrency 5 --rps 10
```

Backend HTTP:

```sh
docker compose run --rm loadtest --protocol http --host backend --port 5000 --path /api/health --duration 20 --concurrency 5 --rps 10
```

Autenticación por LDAPS/TLS. Usa un DN y una contraseña ficticios para generar fallos que el jail de `636` debe registrar:

```sh
docker compose run --rm loadtest --protocol ldaps --host openldap --port 636 --bind-dn "uid=fail2ban-test,ou=users,dc=example,dc=com" --password "invalid-fail2ban-test-password" --duration 20 --concurrency 2 --rps 5
```

Después de cada corrida, revisar los estados y logs:

```sh
docker compose exec -T frontend fail2ban-client status http-flood
docker compose exec -T backend fail2ban-client status http-flood
docker compose exec -T openldap fail2ban-client status ldap-auth
docker compose exec -T openldap fail2ban-client status ldaps-connection
docker compose logs --tail=100 openldap backend frontend
```

Un bloqueo del generador es el resultado esperado cuando supera el umbral; no equivale a que el servicio haya caído. Confirmar salud desde otro origen no baneado y registrar los datos en la tabla del README del repositorio DDoS. No borrar volúmenes LDAP para reiniciar una prueba; desbanear con `fail2ban-client set <jail> unbanip <ip>` cuando corresponda.

## Detener

```sh
docker compose down
```

Los volúmenes nombrados se conservan. `docker compose down -v` elimina los datos y la configuración LDAP persistidos.

## Nota sobre Debian Buster

La imagen `osixia/openldap:1.5.0` está basada en Debian Buster, que llegó al fin de soporte y cuyos repositorios normales ya no están disponibles. El Dockerfile del repo LDAP reescribe las fuentes APT a `archive.debian.org` únicamente para poder construir este laboratorio. Buster y esa imagen base son obsoletos; esta configuración es para práctica local, no para producción.