# Despliegue en AWS EC2 y exportación del APK *release*

Guía paso a paso para poner **Civic Security** en internet (comunicación entre varios
dispositivos con datos móviles o cualquier Wi-Fi, por **HTTPS/WSS**) usando una sola
instancia **AWS EC2** con **Docker Compose**, y luego exportar e instalar el **APK
`release` firmado**.

> Complementa la lista de verificación de seguridad de [despliegue.md](despliegue.md).

---

## Arquitectura que vas a desplegar

```
Teléfonos Android (4G/5G o cualquier Wi-Fi)
       │ HTTPS (/v1/*) + WSS (/socket/websocket)
       ▼ :80 y :443
┌─────────────────────── AWS EC2 (Ubuntu 24.04 + Elastic IP) ───────────────────────┐
│                                                                                   │
│  [Caddy]  TLS automático (Let's Encrypt) + WebSockets + X-Real-IP                 │
│     │  (red interna Docker; puerto 4000 cerrado a internet)                       │
│     ▼                                                                             │
│  [Phoenix Release]  API REST + Phoenix Channels (mapa:arequipa)                   │
│     │               Fotos en volumen privado (`0700`, usuario `civic` sin root)   │
│     │  (red interna Docker; puerto 5432 cerrado a internet)                       │
│     ▼                                                                             │
│  [PostgreSQL 17 + PostGIS 3.7]  rol limitado `civic_app` para la app              │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## Fase 1 — Crear y asegurar la instancia EC2 en AWS

1. **Crear instancia EC2** (región cercana, p. ej. `us-east-1` o `sa-east-1`):
   - **SO:** Ubuntu Server 24.04 LTS (**64 bits x86**).
   - **Tipo recomendado:** **`t3.small`** (x86_64, 2 vCPU, 2 GB RAM, ~USD 15/mes) o **`t3.medium`** (x86_64, 2 vCPU, 4 GB RAM, ~USD 30/mes).
     > Usa arquitectura **x86 (`t3.*`)** y no ARM (`t4g.*`): la imagen oficial `postgis/postgis:17-3.7` publica manifiesto oficial para `linux/amd64`. Evita además `t3.micro` (1 GB RAM) porque Argon2id usa 32 MiB por login/registro concurrente y compilar la imagen Docker con 1 GB puede quedarse sin memoria.
   - **Almacenamiento:** 25 GB `gp3` (suficiente para Docker, base de datos, fotos y respaldos).
   - **Key pair:** crea y descarga tu llave `.pem` (guárdala con `chmod 400`, nunca en el repo).
2. **Security Group (firewall de AWS):**
   - Reglas de entrada (*Inbound*):
     - `HTTP` — TCP `80` — Origen `0.0.0.0/0` (Let's Encrypt y redirección a HTTPS).
     - `HTTPS` — TCP `443` — Origen `0.0.0.0/0` (API y WebSockets).
     - `SSH` — TCP `22` — Origen **Solo tu IP** (*My IP*).
   - **No abras** el puerto `4000` ni el `5432`.
3. **Asignar una Elastic IP:**
   - En la consola de EC2 → *Elastic IPs* → *Allocate Elastic IP* → *Associate* a tu instancia.
   - Así la IP pública **no cambia** si reinicias la EC2. Anota esa IP (ej. `3.14.15.92`).
4. **Conectarte por SSH e instalar Docker:**

```bash
ssh -i ~/ruta/a/tu-llave.pem ubuntu@TU_ELASTIC_IP

sudo apt-get update
sudo apt-get install -y ca-certificates curl git ufw
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker "$USER"
```

Cierra y vuelve a abrir la sesión SSH para que el grupo `docker` surta efecto, y activa el firewall del sistema:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw enable
docker --version && docker compose version
```

---

## Fase 2 — Configurar un subdominio gratuito para HTTPS

La app `release` **exige `https://`** y rechaza `http://` o direcciones sin certificado válido. Elige una opción:

- **Opción A (recomendada): DuckDNS**
  1. Entra a https://www.duckdns.org e inicia sesión.
  2. Crea un subdominio (p. ej. `civic-arequipa`) y pega tu **Elastic IP**.
  3. Tu dominio será `civic-arequipa.duckdns.org`.
- **Opción B (cero registro): `sslip.io`**
  - Si tu Elastic IP es `3.14.15.92`, usa directamente `3-14-15-92.sslip.io` (resuelve automáticamente a `3.14.15.92` y funciona con Let's Encrypt).

Comprueba desde tu computadora que ya resuelve a tu Elastic IP:

```bash
dig +short civic-arequipa.duckdns.org
```

---

## Fase 3 — Desplegar el backend en EC2

Dentro de la EC2:

### 3.1 Clonar el repositorio

```bash
git clone https://github.com/aaabriceno/Proyecto-MVP-Civic-Security.git
cd Proyecto-MVP-Civic-Security
```

### 3.2 Crear `infra/.env.prod` con secretos únicos

```bash
cp infra/.env.prod.example infra/.env.prod
chmod 600 infra/.env.prod
```

Genera los secretos (ejecuta cada línea y copia su salida a `infra/.env.prod`):

```bash
echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)"
echo "POSTGRES_APP_PASSWORD=$(openssl rand -hex 24)"
echo "SECRET_KEY_BASE=$(openssl rand -base64 48)"
echo "CIVIC_KEY_VERIF_V1=$(openssl rand -base64 32)"
echo "CIVIC_KEY_REP_V1=$(openssl rand -base64 32)"
echo "CIVIC_KEY_VOTE_V1=$(openssl rand -base64 32)"
```

Edita `infra/.env.prod` (`nano infra/.env.prod`), coloca tu dominio en `PHX_HOST=` (sin `https://`, p. ej. `civic-arequipa.duckdns.org`) y pega las 6 claves generadas.

> **Importante:** respalda `infra/.env.prod` en un gestor de contraseñas (fuera de la EC2 y nunca en Git). Si pierdes `CIVIC_KEY_VERIF_V1`, los DNI ya verificados dejarán de reconocerse.

### 3.3 Construir, migrar la base de datos y arrancar

```bash
# 1. Levantar primero PostgreSQL + PostGIS (crea automáticamente el rol limitado `civic_app`)
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml up -d db

# 2. Compilar la imagen release de Phoenix (tarda ~3-5 min la primera vez)
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml build backend

# 3. Correr migraciones usando el usuario administrador `postgres` (no `civic_app`)
set -a && source infra/.env.prod && set +a
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml run --rm --no-deps \
  -e DATABASE_URL="ecto://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}" \
  backend /app/bin/civic eval "Civic.Release.migrate"

# 4. Levantar backend (con rol limitado `civic_app`) y Caddy (emite el certificado TLS)
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml up -d
```

### 3.4 Verificar TLS, API y WebSocket

Desde tu computadora (reemplaza el dominio por el tuyo):

```bash
# 1. HTTP redirige a HTTPS
curl -sI http://civic-arequipa.duckdns.org/health | head -5

# 2. HTTPS responde 200 {"status":"ok"} y trae HSTS
curl -i https://civic-arequipa.duckdns.org/health

# 3. Revisar logs en la EC2 si algo no responde
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml logs -f caddy backend
```

> **Tip para demostración con pocos teléfonos:** si necesitas bajar el umbral de votos de 3 a 2, descomenta `CIVIC_VOTE_THRESHOLD=2` en `infra/.env.prod` y reinicia solo `backend` (`docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml up -d backend`).

---

## Fase 4 — Firmar y exportar el APK *release*

Esto se hace en la computadora donde tienes **Flutter SDK** y **Android SDK**.

### 4.1 Generar el keystore de firma (una sola vez)

Guarda el keystore **fuera del repositorio** (p. ej. en `~/.keys/`) y hazle copia de seguridad:

```bash
mkdir -p ~/.keys
chmod 700 ~/.keys

keytool -genkey -v \
  -keystore ~/.keys/civic-security-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias civic_release
```

### 4.2 Crear `app/android/key.properties` (ignorado por Git)

Crea el archivo `app/android/key.properties` con tus datos (ajusta la ruta absoluta al `.jks`):

```properties
storePassword=TU_CONTRASEÑA_DEL_KEYSTORE
keyPassword=TU_CONTRASEÑA_DE_LA_LLAVE
keyAlias=civic_release
storeFile=/home/TU_USUARIO/.keys/civic-security-release.jks
```

### 4.3 Compilar el APK *release* apuntando a tu servidor HTTPS

Desde `app/`:

- **Opción A — Un solo APK universal** (más simple para repartir por WhatsApp/Drive a cualquier Android):

```bash
cd app
flutter pub get
flutter build apk --release \
  --dart-define=API_URL=https://civic-arequipa.duckdns.org \
  --obfuscate \
  --split-debug-info=debug-info
```

El archivo queda en `app/build/app/outputs/flutter-apk/app-release.apk`.

- **Opción B — APKs livianos por arquitectura (`--split-per-abi`)** (~20 MB en vez de ~55 MB):

```bash
cd app
flutter build apk --release --split-per-abi \
  --dart-define=API_URL=https://civic-arequipa.duckdns.org \
  --obfuscate \
  --split-debug-info=debug-info
```

Para la mayoría de teléfonos Android actuales comparte `app-arm64-v8a-release.apk` (y `app-armeabi-v7a-release.apk` solo para equipos antiguos de 32 bits).

### 4.4 Verificar que el APK quedó firmado correctamente

```bash
# Con apksigner (en $ANDROID_HOME/build-tools/<version>/apksigner)
apksigner verify --print-certs app/build/app/outputs/flutter-apk/app-release.apk

# Alternativa con keytool (viene con Java)
keytool -printcert -jarfile app/build/app/outputs/flutter-apk/app-release.apk
```

Debe mostrar tu certificado (`CN=...`), **no** `CN=Android Debug`.

---

## Fase 5 — Prueba multi-dispositivo real y operación

### Prueba de punta a punta con datos móviles

1. Instala el APK *release* en **2 o más teléfonos Android** (por cable con `adb install -r` o enviando el archivo por Drive/WhatsApp y habilitando *Instalar apps desconocidas*).
2. Desconecta el Wi-Fi en al menos uno de los teléfonos para probar con **datos móviles (4G/5G)**.
3. Verifica que el chip del mapa pase a **"En vivo"** (confirma conexión `wss://` sobre TLS).
4. En el teléfono A: crea cuenta → verifica DNI → crea un reporte con foto.
5. En el teléfono B: confirma que el pin y la foto aparecen **al instante**, abre el detalle y vota **Lo confirmo** (con cuenta verificada con otro DNI y a < 1 km). Verifica que el contador y el estado se actualizan en vivo en el teléfono A.

### Actualizar el servidor cuando subas cambios a `main`

En la EC2:

```bash
cd ~/Proyecto-MVP-Civic-Security
git pull
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml build backend
set -a && source infra/.env.prod && set +a
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml run --rm --no-deps \
  -e DATABASE_URL="ecto://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}" \
  backend /app/bin/civic eval "Civic.Release.migrate"
docker compose --env-file infra/.env.prod -f infra/docker-compose.prod.yml up -d
```

### Respaldo rápido de la base de datos

```bash
set -a && source infra/.env.prod && set +a
docker exec civic_db_prod pg_dump -U "$POSTGRES_USER" -Fc "$POSTGRES_DB" \
  > "backup-civic-$(date +%F).dump"
```

Puedes además activar **AWS EBS Snapshots** diarios sobre el volumen de la EC2 para respaldar base de datos y fotos juntos con tus créditos de AWS.
