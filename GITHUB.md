# Guia de configuracion en GitHub

Este documento describe los pasos que debes configurar en GitHub para ejecutar el pipeline de CI y preparar el despliegue del backend en EC2.

## 1. Revisar la rama principal

1. Entra en el repositorio de GitHub.
2. Ve a **Settings**.
3. Entra en **Branches**.
4. Confirma que la rama principal es `main`.

El workflow actual se ejecuta cuando se abre una pull request hacia `main`.

## 2. Activar GitHub Actions

1. Ve a la pestana **Actions**.
2. Si GitHub muestra un aviso para habilitar workflows, pulsa **Enable workflows**.
3. Comprueba que existe el workflow `CI`.

El archivo del pipeline esta en:

```text
.github/workflows/ci.yml
```

## 3. Configurar proteccion de rama

Recomendado para evitar merges con errores:

1. Ve a **Settings**.
2. Entra en **Branches**.
3. Pulsa **Add branch protection rule**.
4. En **Branch name pattern**, escribe:

```text
main
```

5. Activa:
   - **Require a pull request before merging**
   - **Require status checks to pass before merging**
   - **Require branches to be up to date before merging**

6. En status checks, selecciona los jobs del pipeline:
   - `Backend tests and build`
   - `Frontend tests and build`

## 4. Configurar secrets para EC2

Para que GitHub Actions pueda conectarse a EC2, configura los secrets del repositorio.

1. Ve a **Settings**.
2. Entra en **Secrets and variables**.
3. Entra en **Actions**.
4. Pulsa **New repository secret**.

Crea estos secrets:

```text
EC2_HOST
EC2_USER
EC2_SSH_KEY
EC2_BACKEND_PATH
```

Valores esperados:

- `EC2_HOST`: IP publica o DNS publico de la instancia EC2.
- `EC2_USER`: usuario SSH de la instancia. Normalmente `ubuntu` o `ec2-user`.
- `EC2_SSH_KEY`: clave privada SSH para acceder a la EC2.
- `EC2_BACKEND_PATH`: ruta absoluta del backend en EC2.

Ejemplo de `EC2_BACKEND_PATH`:

```text
/home/ubuntu/apps/lti/backend
```

## 5. Configurar secret de base de datos

Si el pipeline va a crear el archivo `.env` o ejecutar migraciones, crea tambien:

```text
DATABASE_URL
```

Ejemplo:

```text
postgresql://USER:PASSWORD@RDS_ENDPOINT:5432/lti
```

Recomendacion:

- Si la EC2 ya tiene su propio `.env`, no es obligatorio guardar `DATABASE_URL` en GitHub.
- Si GitHub Actions va a ejecutar `npx prisma migrate deploy`, entonces necesita acceso a `DATABASE_URL`.

## 6. Configurar la clave SSH para GitHub Actions

La EC2 debe aceptar la clave publica correspondiente al secret `EC2_SSH_KEY`.

En EC2, revisa este archivo:

```bash
~/.ssh/authorized_keys
```

Debe contener la clave publica.

En GitHub, `EC2_SSH_KEY` debe contener la clave privada completa, incluyendo:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

No subas esta clave al repositorio.

## 7. Permitir acceso SSH desde GitHub Actions

En el Security Group de EC2, el puerto `22` debe permitir conexiones SSH.

Opciones:

- Permitir temporalmente SSH desde `0.0.0.0/0`.
- Restringirlo a los rangos de IP de GitHub Actions.
- Usar un runner self-hosted dentro de AWS.

Para un proyecto de practica, la opcion mas simple es abrir SSH temporalmente, pero no es la mas segura.

## 8. Revisar permisos del repositorio

1. Ve a **Settings**.
2. Entra en **Actions**.
3. Entra en **General**.
4. En **Workflow permissions**, selecciona:
   - **Read repository contents permission**

Si mas adelante el workflow necesita crear releases, comentar en PRs o subir artefactos, puede requerir permisos adicionales.

## 9. Crear pull request para validar CI

1. Crea una rama nueva.
2. Sube tus cambios.
3. Abre una pull request hacia `main`.
4. Comprueba que se ejecuta el workflow `CI`.
5. Verifica que pasan:
   - Lint backend
   - Tests backend
   - Build backend
   - Lint frontend
   - Tests frontend
   - Build frontend

## 10. Preparar despliegue automatico

Recomendacion:

- Mantener la CI en pull requests.
- Ejecutar despliegue solo cuando haya `push` a `main`.

Flujo recomendado:

```text
Pull Request -> CI
Merge a main -> CI + Deploy backend to EC2
```

Esto evita desplegar codigo que todavia esta en revision.

## 11. Validar el despliegue desde GitHub

Cuando se anada la fase de deploy:

1. Haz merge de una PR a `main`.
2. Ve a **Actions**.
3. Abre la ejecucion del workflow.
4. Revisa el job de deploy.
5. Comprueba que no hay errores SSH.
6. Comprueba que PM2 reinicio el backend.

Despues, valida la API:

```bash
curl http://EC2_PUBLIC_IP:3010
```

O si usas Nginx:

```bash
curl http://EC2_PUBLIC_IP
```

## 12. Checklist final

- [ ] GitHub Actions esta habilitado.
- [ ] La rama principal es `main`.
- [ ] La rama `main` esta protegida.
- [ ] El workflow `CI` se ejecuta en pull requests.
- [ ] Los status checks son obligatorios antes del merge.
- [ ] Los secrets de EC2 estan configurados.
- [ ] La clave publica SSH esta en EC2.
- [ ] El Security Group permite SSH segun la estrategia elegida.
- [ ] `DATABASE_URL` esta configurada en EC2 o como secret.
- [ ] El despliegue se ejecutara solo despues de integrar en `main`.
