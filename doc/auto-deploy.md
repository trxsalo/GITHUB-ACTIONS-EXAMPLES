# Documentación de Configuración para GitHub Actions con Despliegue Automático en Vercel

Esta guía describe cómo configurar las variables de entorno necesarias para habilitar el despliegue automático en Vercel a través de GitHub Actions.

## Requisitos Previos

1. **Acceso a GitHub**: Necesitas tener permisos de administrador en el repositorio de GitHub donde se configurará el flujo de trabajo.
2. **Cuenta en Vercel**: La cuenta debe tener acceso al proyecto en Vercel donde se realizará el despliegue.

## Variables Necesarias

### 1. VERCEL_TOKEN
Este token es necesario para autorizar a GitHub Actions a interactuar con la API de Vercel para activar el despliegue.

#### Pasos para obtener el `VERCEL_TOKEN`:
1. Inicia sesión en [Vercel](https://vercel.com/).
2. Ve a **Settings** (Configuración) en el perfil de usuario o del equipo.
3. Selecciona **Tokens** o **Access Tokens** en el menú.
4. Genera un nuevo token, dale un nombre descriptivo (ej., `GitHub Actions Deployment`), y copia el token generado.
5. En GitHub, ve al repositorio donde configurarás el flujo de trabajo.
6. Dirígete a **Settings > Secrets and variables > Actions > New repository secret**.
7. Crea un nuevo secreto con el nombre `VERCEL_TOKEN` y pega el token generado en Vercel.

### 2. GITHUB_TOKEN (Generado Automáticamente)
GitHub proporciona un token (`GITHUB_TOKEN`) automáticamente a cada flujo de trabajo. Este token permite a GitHub Actions realizar acciones en el repositorio, como hacer commits y cambios. **Asegúrate de que tiene permisos de lectura y escritura**.

#### Pasos para verificar permisos de `GITHUB_TOKEN`:
1. En el repositorio de GitHub, ve a **Settings > Actions > General**.
2. Busca la sección **Workflow permissions**.
3. Asegúrate de que la opción **Read and Write permissions** esté seleccionada.

> **Nota**: Si `GITHUB_TOKEN` no tiene los permisos necesarios o deseas mayor control, considera usar un Personal Access Token (PAT) con permisos adecuados. <br>
> **Nota** Si tiene aun problemas puede considerar crear un token [Personal access tokens (classic)](https://github.com/settings/tokens)


## Ejemplo de Archivo `YAML` para GitHub Actions

Este es un ejemplo de flujo de trabajo para GitHub Actions que realiza un despliegue automático en Vercel usando las variables configuradas:

```yaml
name: Auto-deploy to Vercel
# Ejecutar en eventos de push, pull request y manualmente
on:
  # Ejecutar en cada push a la rama main (si se realiza un merge a la rama, igual se ejecuta)
  push:
    branches:
      - main #Define la rama donde se realizara los despliegues
  workflow_dispatch:
# Definir trabajos
jobs:
  # Trabajo para re-firmar y desplegar
  re-sign-and-deploy:
    # Ejecutar en la última versión de Ubuntu
    runs-on: ubuntu-latest
    steps:
      # Obtener el código del repositorio
      - name: Check out repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 1
      # Configurar Git
      - name: Configure Git
        run: |
          git config user.name "${{ github.actor }}"
          git config user.email "${{ github.actor }}@users.noreply.github.com"
      # Re-firmar el último commit
      - name: Re-sign commit as repository owner
        env:
          GITHUB_TOKEN: ${{ secrets.ME_GITHUB_TOKEN }}
        run: |
          git commit --allow-empty -m "Trigger deployment"
          git push
      # Desplegar en Vercel
      - name: Trigger Vercel deployment
        env:
          VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}
        run: |
          curl -X POST https://api.vercel.com/v1/integrations/deploy/prj_${{secrets.ID_PROYECTO}}?teamId=${{secrets.ID_DEL_EQUIPO}} -H "Authorization: Bearer $VERCEL_TOKEN"

```
