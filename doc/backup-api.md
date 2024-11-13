# GitHub Action: Production Database Backup API

Este flujo de trabajo automatiza la creación y almacenamiento de copias de seguridad de la base de datos desde una API y
realiza la subida del archivo comprimido a GitHub. La acción se ejecuta diariamente a medianoche o de forma manual a
través de la interfaz de GitHub.

## Descripción del Flujo de Trabajo

### Disparadores

- **schedule**: Ejecuta el flujo de trabajo diariamente a la medianoche (`0 0 * * *`).
- **workflow_dispatch**: Permite ejecutar el flujo de trabajo de forma manual.

### Variables

- `CURRENT_DATE`: Almacena la fecha y hora actuales en el formato `YYYYMMDDHHMM`, lo cual se utiliza para nombrar el
  archivo de respaldo.

## Estructura de los Jobs

### Job: backup

Este job se encarga de ejecutar el proceso de respaldo de la base de datos y subirlo al repositorio.

#### Pasos

1. **Checkout Repository**  
   Descarga el código del repositorio.

   ```yaml
      - name: Checkout Repository
        uses: actions/checkout@v3
    ```
2. **Set current date**
   Obtiene la fecha y hora actual y la almacena en una variable de entorno (CURRENT_DATE), útil para nombrar el archivo
   de respaldo.
   ```yaml
      - name: Set current date
        run: echo "CURRENT_DATE=$(date +%Y%m%d%H%M)" >> $GITHUB_ENV
   ```
3. **Create backups directory**
   Crea el directorio backups donde se almacenará el archivo de respaldo.
    ```yaml
      - name: Create backups directory
        run: mkdir -p backups
   ```
4. **Download Backup from API**   
   Descarga el archivo de respaldo desde una API usando el token secreto BACKUP_API_TOKEN. La descarga se intenta hasta
   5 veces en caso de fallos.
   **API** <br>
   La api que consulto es de un servicio que desarrolle, la cual me permite descargar el backup de la base de datos (servicio del vercel).
   Lo realize ya que la accion **tj-actions/pg-dump@v3** no me permitia descargar el backup de la base de datos por diferencia de version de postgres.<br>
   **Uso [coolify](https://coolify.io/) para alojar mis proyectos**<br>
   [Docker API BACKUP](https://hub.docker.com/r/trxsalo/backup-postgresql-api)

      ```yaml
      - name: Download Backup from API
        env:
          BACKUP_API_TOKEN: ${{ secrets.BACKUP_API_TOKEN }}
        run: |
          attempt_counter=0
          max_attempts=5
          until $(sudo curl -X GET -sl -f "https://${{secrets.DOMAIN_API}}/backup?token=${{ env.BACKUP_API_TOKEN }}" -o "
          backups/backup-${{ env.CURRENT_DATE }}.sql"); do
          if [ ${attempt_counter} -eq ${max_attempts} ]; then
          echo "Failed to download backup after ${max_attempts} attempts."
          exit 1
          fi
          attempt_counter=$(($attempt_counter+1))
          echo "Attempt $attempt_counter failed! Trying again in 10 seconds..."
          sleep 10
          done
     ```
5. **Upload backup**
     Sube el archivo de respaldo como un artefacto de GitHub.
     (Pueden omitir este paso si desean)
      ```yaml
         - name: Upload backup
           uses: actions/upload-artifact@v4
           with:
              name: db-backup
              path: 'backups/backup-${{ env.CURRENT_DATE }}.sql'
     ```
6. **Commit and Push Backup to Repository**
    Configura Git para realizar commits automáticos y sube el respaldo comprimido a la rama backup.
    comprime el backup, y lo sube a la rama backup en la carpeta backup.
   (ojo debe tener su Actions permiso de escritura en el repositorio)
     ```yaml
        - name: Commit and Push Backup to Repository
          env:
               GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          run: |
                 git config --local user.name "github-actions[bot]"
                 git config --local user.email "github-actions[bot]@users.noreply.github.com"
                 tar -czf "backups/backup-${{ env.CURRENT_DATE }}.tar.gz" -C backups "backup-${{ env.CURRENT_DATE }}.sql"
                 git add "backups/backup-${{ env.CURRENT_DATE }}.tar.gz"
                 git commit -m "Add compressed database backup for ${{ env.CURRENT_DATE }}"
                 if git ls-remote --exit-code --heads origin backup; then
                 echo "Branch 'backup' exists. Pushing to 'backup'."
                 else
                 echo "Branch 'backup' does not exist. Creating 'backup' branch."
                 git branch backup
                 fi
                 git push origin backup
    ```
7. **GitActions**
```yaml
  name: Production Database Backup API
  on:
    schedule:
      - cron: '0 0 * * *'
    workflow_dispatch:
  jobs:
    backup:
      runs-on: ubuntu-latest
      steps:
        #Descarga el repositorio
        - name: Checkout Repository
          uses: actions/checkout@v3
        #Obtener la fecha actual
        - name: Set current date
          run: echo "CURRENT_DATE=$(date +%Y%m%d%H%M)" >> $GITHUB_ENV
        #Crear directorio de backups
        - name: Create backups directory
          run: mkdir -p backups
        #Descargar backup desde la API
        - name: Download Backup from API
          env:
            BACKUP_API_TOKEN: ${{ secrets.BACKUP_API_TOKEN }}
          run: |
            attempt_counter=0
            max_attempts=5
            until $(sudo curl -X GET -sl -f "https://${{secrets.DOMAIN_API}}/backup?token=${{ env.BACKUP_API_TOKEN }}" -o "backups/backup-${{ env.CURRENT_DATE }}.sql"); do
              if [ ${attempt_counter} -eq ${max_attempts} ]; then
                echo "Failed to download backup after ${max_attempts} attempts."
                exit 1
              fi
              attempt_counter=$(($attempt_counter+1))
              echo "Attempt $attempt_counter failed! Trying again in 10 seconds..."
              sleep 10
            done
        #Subir backup a GitHub artifacts
        - name: Upload backup
          uses: actions/upload-artifact@v4
          with:
            name: db-backup
            path: 'backups/backup-${{ env.CURRENT_DATE }}.sql'
        #Commit y push del backup a GitHub
        - name: Commit and Push Backup to Repository
          env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          run: |
            git config --local user.name "github-actions[bot]"
            git config --local user.email "github-actions[bot]@users.noreply.github.com"
            #cp "backups/backup-${{ env.CURRENT_DATE }}.sql" .
            #git add "backups/backup-${{ env.CURRENT_DATE }}.sql"
            ## Comprimir el archivo de respaldo
            tar -czf "backups/backup-${{ env.CURRENT_DATE }}.tar.gz" -C backups "backup-${{ env.CURRENT_DATE }}.sql"
            ## Agregar el archivo comprimido al repositorio
            git add "backups/backup-${{ env.CURRENT_DATE }}.tar.gz"
            git commit -m "Add compressed database backup for ${{ env.CURRENT_DATE }}"
            #git push origin main
            ## Crear la rama "backup" si no existe y hacer push
            if git ls-remote --exit-code --heads origin backup; then
              echo "Branch 'backup' exists. Pushing to 'backup'."
            else
              echo "Branch 'backup' does not exist. Creating 'backup' branch."
              git branch backup
            fi
            # Hacer push a la rama "backup"
            git push origin backup
        #Mensaje de finalización
        - name:
          run: echo "Backup completed successfully"

```