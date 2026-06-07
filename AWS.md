# Guia de despliegue en AWS

Este documento describe los pasos necesarios para preparar AWS y poder desplegar el backend de LTI en una instancia EC2 usando PostgreSQL en Amazon RDS.

## 1. Crear la base de datos PostgreSQL en RDS

1. Entra en la consola de AWS.
2. Ve a **RDS**.
3. Crea una nueva base de datos.
4. Selecciona:
   - Motor: **PostgreSQL**
   - Plantilla: **Free tier** si es para pruebas
   - Identificador: `lti-postgres`
   - Usuario administrador: el usuario que prefieras
   - Password: una contrasena segura
5. Crea una base de datos inicial, por ejemplo:
   - Nombre: `lti`
6. En conectividad:
   - Selecciona la misma VPC donde estara la instancia EC2.
   - No habilites acceso publico salvo que sea estrictamente necesario.
7. Guarda el endpoint de RDS. Lo necesitaras para `DATABASE_URL`.

Ejemplo de `DATABASE_URL`:

```env
DATABASE_URL=postgresql://USER:PASSWORD@RDS_ENDPOINT:5432/lti
```

## 2. Crear la instancia EC2

1. Ve a **EC2**.
2. Lanza una nueva instancia.
3. Recomendaciones:
   - AMI: Ubuntu Server LTS o Amazon Linux
   - Tipo: `t2.micro` o `t3.micro` para pruebas
   - Key pair: crea o selecciona una clave SSH
4. Selecciona la misma VPC que usa RDS.
5. Asocia un Security Group para el backend.

## 3. Configurar Security Groups

### Security Group de EC2

Permite:

- SSH `22`: solo desde tu IP.
- Backend `3010`: desde internet si vas a exponer directamente el backend.
- HTTP `80`: si usas Nginx.
- HTTPS `443`: si usas Nginx con certificado SSL.

Para un entorno real, es mejor exponer solo `80` y `443`, y dejar el backend escuchando internamente en `localhost:3010`.

### Security Group de RDS

Permite:

- PostgreSQL `5432`: solo desde el Security Group de la EC2.

No abras PostgreSQL a `0.0.0.0/0`.

## 4. Conectarse a EC2 por SSH

Desde tu maquina local:

```bash
ssh -i tu-clave.pem ubuntu@EC2_PUBLIC_IP
```

En Amazon Linux el usuario suele ser:

```bash
ec2-user
```

En Ubuntu suele ser:

```bash
ubuntu
```

## 5. Instalar dependencias en EC2

En la instancia EC2:

```bash
sudo apt update
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

Comprueba las versiones:

```bash
node --version
npm --version
pm2 --version
```

## 6. Preparar el backend en EC2

Crea una carpeta para la aplicacion:

```bash
mkdir -p ~/apps
cd ~/apps
git clone REPOSITORY_URL lti
cd lti/backend
```

Instala dependencias:

```bash
npm ci
```

Crea el archivo `.env` del backend:

```bash
nano .env
```

Contenido minimo:

```env
DATABASE_URL=postgresql://USER:PASSWORD@RDS_ENDPOINT:5432/lti
PORT=3010
```

No subas este archivo al repositorio.

## 7. Ejecutar Prisma

Desde `backend`:

```bash
npm run prisma:generate
npx prisma migrate deploy
```

Si necesitas cargar datos iniciales:

```bash
npx ts-node prisma/seed.ts
```

Ejecuta el seed solo si realmente quieres insertar datos de ejemplo.

## 8. Compilar y arrancar el backend

Desde `backend`:

```bash
npm run build
pm2 start dist/index.js --name lti-backend
pm2 save
```

Comprueba que esta corriendo:

```bash
pm2 status
pm2 logs lti-backend
```

## 9. Configurar PM2 para reinicio automatico

Ejecuta:

```bash
pm2 startup
```

PM2 mostrara un comando con `sudo`. Copialo y ejecutalo.

Despues:

```bash
pm2 save
```

## 10. Opcional: configurar Nginx

Instala Nginx:

```bash
sudo apt install -y nginx
```

Crea una configuracion:

```bash
sudo nano /etc/nginx/sites-available/lti-backend
```

Ejemplo:

```nginx
server {
    listen 80;
    server_name TU_DOMINIO_O_IP;

    location / {
        proxy_pass http://localhost:3010;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Activa la configuracion:

```bash
sudo ln -s /etc/nginx/sites-available/lti-backend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## 11. Configurar secretos en GitHub

En GitHub:

1. Ve a **Settings**.
2. Entra en **Secrets and variables**.
3. Entra en **Actions**.
4. Crea estos secrets:

```text
EC2_HOST
EC2_USER
EC2_SSH_KEY
EC2_BACKEND_PATH
```

Valores esperados:

- `EC2_HOST`: IP publica o DNS de la instancia EC2.
- `EC2_USER`: `ubuntu` o `ec2-user`.
- `EC2_SSH_KEY`: clave privada SSH que permite conectar a EC2.
- `EC2_BACKEND_PATH`: ruta del backend en EC2, por ejemplo `/home/ubuntu/apps/lti/backend`.

Si el pipeline va a crear el `.env` o ejecutar migraciones, tambien puedes necesitar:

```text
DATABASE_URL
```

## 12. Preparar despliegue desde GitHub Actions

El despliegue deberia ejecutarse despues de:

1. Lint.
2. Tests.
3. Build.

Recomendacion:

- Ejecutar despliegue solo en `push` a `main`.
- No desplegar automaticamente desde cualquier pull request.

Flujo esperado en EC2:

```bash
cd EC2_BACKEND_PATH
git pull
npm ci
npm run prisma:generate
npx prisma migrate deploy
npm run build
pm2 restart lti-backend
```

## 13. Validar el despliegue

Comprueba el estado del backend:

```bash
pm2 status
pm2 logs lti-backend
```

Comprueba la API:

```bash
curl http://localhost:3010
```

Si usas Nginx:

```bash
curl http://EC2_PUBLIC_IP
```

## 14. Buenas practicas

- No guardes credenciales en el repositorio.
- No expongas PostgreSQL a internet.
- Usa Security Groups restrictivos.
- Usa RDS para persistencia y backups.
- Ejecuta migraciones con `prisma migrate deploy`, no con `prisma migrate dev`.
- Despliega solo cuando el pipeline pase correctamente.
- Considera activar HTTPS si expones el backend publicamente.
