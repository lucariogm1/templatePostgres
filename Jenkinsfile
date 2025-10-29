pipeline {
    agent any

    environment {
        DB_ENGINE             = 'PostgreSQL'
        DB_PASSWORD_ADMIN     = 'password'
        DB_PLATFORM_PASS      = 'password'
        DB_PLATFORM_USER      = 'equipo_plataforma'
        DB_RESOURCE_LABELS    = 'test'
        DB_SERVICE_PROVIDER   = 'GCP - Cloud SQL'
        DB_TAGS               = 'test'
        DB_TIME_ZONE          = 'GMT-4'
        DB_USER_ADMIN         = 'postgres'
        // DB_BACKUP_ENABLED se inicializa aquí pero será sobreescrito por el parámetro en el primer script stage.
        //DB_BACKUP_ENABLED     = 'true' 
        PAIS                  = 'PE'
        
        // VARIABLES DE JIRA
        JIRA_API_URL = 'https://bancoripley1.atlassian.net/rest/api/3/issue/'

        mensajeAteams = 'Despliegue de DB iniciado.'
    }

    parameters {
        string(name: 'TICKET_JIRA', defaultValue: 'AJI-83', description: 'Ticket de Jira')

        // GCP
        string(name: 'PROJECT_ID', defaultValue: '', description: 'ID del proyecto')
        string(name: 'REGION',      defaultValue: '', description: 'Región')
        string(name: 'ZONE',        defaultValue: '', description: 'Zona')
        choice(name: 'ENVIRONMENT', choices: ['1-Desarrollo', '2-Pre-productivo (PP)', '3-Produccion'], description: 'Ambiente') // Etiqueta clave: '3-Produccion'

        // TYPE / INSTANCIA
        string(name: 'DB_INSTANCE_NAME', defaultValue: '', description: 'Nombre instancia')
        string(name: 'DB_INSTANCE_ID',    defaultValue: '', description: 'ID instancia')
        choice(name: 'DB_AVAILABILITY_TYPE', choices: ['regional', 'single zone'], description: 'Disponibilidad')
        choice(name: 'DB_VERSION',          choices: ['POSTGRES_15', 'POSTGRES_14'], description: 'Versión PostgreSQL')
        choice(name: 'MACHINE_TYPE',          choices: ['db-custom-4-16384', 'standard'], description: 'Máquina')
        string(name: 'DB_MAX_CONNECTIONS',    defaultValue: '', description: 'Máx conexiones')
        string(name: 'DB_STORAGE_SIZE',       defaultValue: '', description: 'Almacenamiento (GB)')
        choice(name: 'DB_STORAGE_AUTO_RESIZE', choices: ['false', 'true'], description: 'Auto-resize')
        choice(name: 'DB_STORAGE_TYPE',       choices: ['SSD', 'HDD'], description: 'Tipo de disco')
        string(name: 'DB_USERNAME', defaultValue: 'sa-app',  description: 'Usuario')
        string(name: 'DB_PASSWORD', defaultValue: 'password', description: 'Password')

        // REDES
        string(name: 'DB_VPC_NETWORK', defaultValue: '', description: 'VPC')
        string(name: 'DB_SUBNET',      defaultValue: '', description: 'Subred')
        choice(name: 'DB_PUBLIC_ACCESS_ENABLED', choices: ['false', 'true'], description: 'Acceso público')
        choice(name: 'DB_PRIVATE_IP_ENABLED',    choices: ['false', 'true'], description: 'IP privada')
        choice(name: 'DB_IP_RANGE_ALLOWED',      choices: ['true', 'false'], description: 'Rangos permitidos')

        // SEGURIDAD / OPERACIÓN
        // Movido de environment a parameters para que el usuario pueda elegir
        choice(name: 'DB_BACKUP_ENABLED', choices: ['true', 'false'], description: 'Habilitar Backup (Obligatorio en Prod)') 
        string(name: 'DB_BACKUP_START_TIME',      defaultValue: '', description: 'Hora inicio backup (HH:MM)')
        string(name: 'DB_BACKUP_RETENTION_DAYS', defaultValue: '', description: 'Retención (días)')
        string(name: 'DB_MAINTENANCE_WINDOW_DAY',  defaultValue: '', description: 'Día mantención')
        string(name: 'DB_MAINTENANCE_WINDOW_HOUR', defaultValue: '', description: 'Hora mantención')
        choice(name: 'DB_MONITORING_ENABLED',   choices: ['true', 'false'], description: 'Monitoring')
        choice(name: 'DB_AUDIT_LOGS_ENABLED',   choices: ['true', 'false'], description: 'Audit logs')
        choice(name: 'CREDENTIAL_FILE',         choices: ['sa-plataforma', 'sa-transacciones'], description: 'Ruta credenciales (JSON)')
        string(name: 'DB_IAM_ROLE',             defaultValue: '', description: 'IAM Role')
        choice(name: 'DB_DELETION_PROTECTION',  choices: ['true', 'false'], description: 'Protección borrado')
        choice(name: 'CHECK_DELETE',            choices: ['true', 'false'], description: 'Check delete')
        choice(name: 'ENABLE_CACHE',            choices: ['false', 'true'], description: 'Habilitar caché (por defecto false)')
        choice(name: 'DB_ENCRYPTION_ENABLED',   choices: ['true', 'false'], description: 'Encripción')

        // REPLICA / FAILOVER
        choice(name: 'DB_FAILOVER_REPLICA_ENABLED', choices: ['false', 'true'], description: 'Failover replica')
        choice(name: 'DB_READ_REPLICA_ENABLED',     choices: ['false', 'true'], description: 'Read replica')
    }

    stages {
        
        stage('Validacion de Variables') {
            steps {
                script {
                    echo "Validando el ambiente: ${params.ENVIRONMENT}"

                    // 1. Inicializar env.DB_BACKUP_ENABLED con el valor seleccionado por el usuario.
                    env.DB_BACKUP_ENABLED = params.DB_BACKUP_ENABLED
                    echo "Valor inicial de DB_BACKUP_ENABLED (seleccionado por usuario): ${env.DB_BACKUP_ENABLED}"

                    // 2. Aplicar regla de Producción
                    // La etiqueta '3-Produccion' indica el ambiente productivo.
                    if (params.ENVIRONMENT == '3-Produccion') {
                        if (env.DB_BACKUP_ENABLED == 'false') {
                            echo "ADVERTENCIA: Ambiente de Producción detectado. Se forzará DB_BACKUP_ENABLED a 'true'."
                            env.DB_BACKUP_ENABLED = 'true'
                        } else {
                            echo "Validación OK: Ambiente de Producción. DB_BACKUP_ENABLED ya está en 'true'."
                        }
                    } else if (params.ENVIRONMENT.startsWith('1-') || params.ENVIRONMENT.startsWith('2-')) {
                        echo "Ambiente de Desarrollo/Pre-productivo. Se respeta el valor de DB_BACKUP_ENABLED: ${env.DB_BACKUP_ENABLED}"
                    } else {
                        // Por si el ENVIRONMENT no coincide con las opciones esperadas, aunque sea choice.
                        error("ERROR CRÍTICO: Ambiente de despliegue no válido: ${params.ENVIRONMENT}")
                    }
                }
            }
        }
        
//          stage('Post-Jira Status') {
//          steps {
//
//              script {
//                  def TICKET_JIRA = params.TICKET_JIRA
//                  def JIRA_API_URL_FULL = "${env.JIRA_API_URL}${TICKET_JIRA}"
//
//
//                  withCredentials([usernamePassword(credentialsId: 'JIRA_TOKEN', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_API_TOKEN')]) {
//                      def auth = java.util.Base64.encoder.encodeToString("${JIRA_USER}:${JIRA_API_TOKEN}".getBytes("UTF-8"))
//                      def response = sh(
//                          script: """
//                              curl -s -X GET "${JIRA_API_URL_FULL}" \
//                              -H "Authorization: Basic ${auth}" \
//                              -H "Accept: application/json"
//                          """,
//                          returnStdout: true
//                      ).trim()
//
//                      def json = new groovy.json.JsonSlurper().parseText(response)
//                      def estado = json.fields.status.name
//                      echo "Estado actual del ticket\n ${JIRA_API_URL_FULL}: ${estado}"
//                  }
//
//
//              }
//
//          }
//      }
//      
//      stage('Post-Coment-jira'){
//          steps{
//              script{
//
//                  withCredentials([usernamePassword(credentialsId: 'JIRA_TOKEN', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_API_TOKEN')]) {
//                      def auth = java.util.Base64.encoder.encodeToString("${JIRA_USER}:${JIRA_API_TOKEN}".getBytes("UTF-8"))
//                      def comentario = "Este ticket fue comentario por Vinka (¿Funcionó?)"
//
//                      def response = sh(
//                          script: """
//                              curl -s -X POST "${JIRA_API_URL}${params.TICKET_JIRA}/comment" \
//                              -H "Authorization: Basic ${auth}" \
//                              -H "Content-Type: application/json" \
//                              -d '{
//                                      "body": {
//                                          "type": "doc",
//                                          "version": 1,
//                                          "content": [
//                                          {
//                                              "type": "paragraph",
//                                               "content": [
//                                               {
//                                                   "type": "text",
//                                                   "text": "${comentario}"
//                                               }
//                                               ]
//                                           }
//                                          ]
//                                      }
//                                      }'
//                          """,
//                          returnStdout: true
//                      ).trim()
//
//                      echo "Comentario enviado: ${response}"
//                  }
//
//              }
//          }
//      }

    stage('Notify Teams') {
        steps {
            script {
                // Notar que la variable teamsWebhookUrl está hardcodeada. En un entorno real se usaría una Credencial ID
                def teamsWebhookUrl = 'https://accenture.webhook.office.com/webhookb2/8fb63984-6f5f-4c2a-a6d3-b4fce2feb8ee@e0793d39-0939-496d-b129-198edd916feb/IncomingWebhook/334818fae3a84ae484512967d1d3f4f1/b08cc148-e951-496b-9f46-3f7e35f79570/V27mobtZgWmAzxIvjHCY5CMAFKPZptkEnQbT5z7X84QNQ1'
                def jsonFile = "teams-notification.json"
                
                // 1. Construcción del mensaje Teams con formato Markdown
                // El valor de env.DB_BACKUP_ENABLED usado aquí será el valor validado/forzado del stage anterior.
                def notificationText = """
**Pipeline ejecutado**

**Detalles de la Instancia:**
* **Instancia DB:** ${params.DB_INSTANCE_NAME}
* **Ambiente:** ${params.ENVIRONMENT}
* **Tipo de Servicio:** ${env.DB_SERVICE_PROVIDER}

**Configuración de Backup:**
* **Backup Habilitado:** ${env.DB_BACKUP_ENABLED}

**Enlace de la Build:** [Ver Build en Jenkins](${env.BUILD_URL})
"""

                // 2. Definición del JSON payload inyectando la variable 'notificationText'
                // Reemplazamos saltos de línea y comillas dobles para que el JSON sea válido en el archivo.
                def escapedText = notificationText.replaceAll("\\n", "\\\\n").replaceAll('"', '\\"')
                
                def message = """
{
    "@type": "MessageCard",
    "@context": "http://schema.org/extensions",
    "summary": "Notificación de Jenkins - ${params.DB_INSTANCE_NAME}",
    "themeColor": "0076D7",
    "title": "Pipeline de Despliegue de Base de Datos",
    "text": "${escapedText}"
}
"""
                
                // 3. Escribir el payload en un archivo temporal
                writeFile file: jsonFile, text: message

                // 4. Usar el comando 'curl' para enviar el archivo
                try {
                    sh """
                        curl -s -X POST -H 'Content-Type: application/json' --data @${jsonFile} ${teamsWebhookUrl}
                    """
                    echo 'Notificación enviada a Teams exitosamente usando curl.'
                } catch (Exception e) {
                    echo "ADVERTENCIA: Falló el envío de notificación a Teams (Error: ${e.getMessage()}). Se continúa con el pipeline."
                } finally {
                    // 5. Limpiar el archivo temporal
                    sh "rm ${jsonFile}"
                }
            }
        }
    }

    stage('Imprimir variables por sección') {
      steps {
        script {
          // --- Sección: Ocultas (environment)
          // El valor de DB_BACKUP_ENABLED en esta sección reflejará el cambio si se hizo en Producción.
          def ocultas = [
            DB_BACKUP_ENABLED     : env.DB_BACKUP_ENABLED,
            DB_ENGINE             : env.DB_ENGINE,
            DB_PASSWORD_ADMIN     : env.DB_PASSWORD_ADMIN,
            DB_PLATFORM_PASS      : env.DB_PLATFORM_PASS,
            DB_PLATFORM_USER      : env.DB_PLATFORM_USER,
            DB_RESOURCE_LABELS    : env.DB_RESOURCE_LABELS,
            DB_SERVICE_PROVIDER   : env.DB_SERVICE_PROVIDER,
            DB_TAGS               : env.DB_TAGS,
            DB_TIME_ZONE          : env.DB_TIME_ZONE,
            DB_USER_ADMIN         : env.DB_USER_ADMIN,
            PAIS                  : env.PAIS
          ]

          // --- Sección: GCP
          def gcp = [
            ENVIRONMENT : params.ENVIRONMENT,
            PROJECT_ID  : params.PROJECT_ID,
            REGION      : params.REGION,
            ZONE        : params.ZONE
          ]

          // --- Sección: Type / Instancia
          def typeInst = [
            DB_AVAILABILITY_TYPE : params.DB_AVAILABILITY_TYPE,
            DB_INSTANCE_ID       : params.DB_INSTANCE_ID,
            DB_INSTANCE_NAME     : params.DB_INSTANCE_NAME,
            DB_MAX_CONNECTIONS   : params.DB_MAX_CONNECTIONS,
            DB_PASSWORD          : params.DB_PASSWORD,
            DB_STORAGE_AUTO_RESIZE: params.DB_STORAGE_AUTO_RESIZE,
            DB_STORAGE_SIZE      : params.DB_STORAGE_SIZE,
            DB_STORAGE_TYPE      : params.DB_STORAGE_TYPE,
            DB_USERNAME          : params.DB_USERNAME,
            DB_VERSION           : params.DB_VERSION,
            MACHINE_TYPE         : params.MACHINE_TYPE
          ]

          // --- Sección: Redes
          def redes = [
            DB_IP_RANGE_ALLOWED      : params.DB_IP_RANGE_ALLOWED,
            DB_PRIVATE_IP_ENABLED    : params.DB_PRIVATE_IP_ENABLED,
            DB_PUBLIC_ACCESS_ENABLED : params.DB_PUBLIC_ACCESS_ENABLED,
            DB_SUBNET                : params.DB_SUBNET,
            DB_VPC_NETWORK           : params.DB_VPC_NETWORK
          ]

          // --- Sección: Seguridad / Operación
          def segOp = [
            DB_BACKUP_ENABLED            : params.DB_BACKUP_ENABLED, // Ahora es un parámetro
            CHECK_DELETE                 : params.CHECK_DELETE,
            CREDENTIAL_FILE              : params.CREDENTIAL_FILE,
            DB_AUDIT_LOGS_ENABLED        : params.DB_AUDIT_LOGS_ENABLED,
            DB_BACKUP_RETENTION_DAYS     : params.DB_BACKUP_RETENTION_DAYS,
            DB_BACKUP_START_TIME         : params.DB_BACKUP_START_TIME,
            DB_DELETION_PROTECTION       : params.DB_DELETION_PROTECTION,
            DB_ENCRYPTION_ENABLED        : params.DB_ENCRYPTION_ENABLED,
            DB_IAM_ROLE                  : params.DB_IAM_ROLE,
            DB_MAINTENANCE_WINDOW_DAY    : params.DB_MAINTENANCE_WINDOW_DAY,
            DB_MAINTENANCE_WINDOW_HOUR   : params.DB_MAINTENANCE_WINDOW_HOUR,
            DB_MONITORING_ENABLED        : params.DB_MONITORING_ENABLED,
            ENABLE_CACHE                 : params.ENABLE_CACHE
          ]

          // --- Sección: Replica / Failover
          def replica = [
            DB_FAILOVER_REPLICA_ENABLED: params.DB_FAILOVER_REPLICA_ENABLED,
            DB_READ_REPLICA_ENABLED    : params.DB_READ_REPLICA_ENABLED
          ]

          // Helper CPS-safe para imprimir ordenado
          def printSection = { String titulo, Map m ->
            echo "================= ${titulo} ================="
            def keys = m.keySet().toList(); keys.sort()
            for (k in keys) {
              def v = m[k]
              echo "${k}: ${v == null ? '' : v}"
            }
          }

          printSection('DEFAULT (Ocultas)', ocultas)
          printSection('GCP', gcp)
          printSection('TYPE / INSTANCIA', typeInst)
          printSection('REDES', redes)
          printSection('SEGURIDAD / OPERACIÓN', segOp)
          printSection('REPLICA / FAILOVER', replica)
        }
      }
    }
  }

    post {
        success { echo 'Pipeline ejecutado correctamente.' }
        failure { echo 'Error al ejecutar el pipeline.' }
    }
}
