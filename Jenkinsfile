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
        DB_TIME_ZONE          = 'GMT-5'
        DB_USER_ADMIN         = 'postgres'
        PAIS                  = 'PE'
        
        // VARIABLES DE JIRA Y WEBHOOK
        JIRA_API_URL = 'https://bancoripley1.atlassian.net/rest/api/3/issue/'
        TEAMS_WEBHOOK = 'https://accenture.webhook.office.com/webhookb2/8fb63984-6f5f-4c2a-a6d3-b4fce2feb8ee@e0793d39-0939-496d-b129-198edd916feb/IncomingWebhook/334818fae3a84ae484512967d1d3f4f1/b08cc148-e951-496b-9f46-3f7e35f79570/V27mobtZgWmAzxIvjHCY5CMAFKPZptkEnQbT5z7X84QNQ1'
        PROYECT_JIRA = "AJI"
        TITULO_JIRA = "Creación de Instancia base de datos PostgreSQL"
        ID_ISSUETYPE_JIRA = "14898"
    }

    parameters {
        string(name: 'TICKET_JIRA', defaultValue: 'AJI-83', description: 'Ticket de Jira')

        // GCP
        string(name: 'PROJECT_ID', defaultValue: '', description: 'ID del proyecto')
        string(name: 'REGION',      defaultValue: '', description: 'Región')
        string(name: 'ZONE',        defaultValue: '', description: 'Zona')
        choice(name: 'ENVIRONMENT', choices: ['Desarrollo', 'Pre-productivo (PP)', 'Produccion'], description: 'Ambiente') // Etiqueta clave: '3-Produccion'

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

                    env.DB_BACKUP_ENABLED = params.DB_BACKUP_ENABLED
                    echo "Valor inicial de DB_BACKUP_ENABLED (seleccionado por usuario): ${env.DB_BACKUP_ENABLED}"

                    if (params.ENVIRONMENT == 'Produccion') {
                        if (env.DB_BACKUP_ENABLED == 'false') {
                            echo "ADVERTENCIA: Ambiente de Producción detectado. Se forzará DB_BACKUP_ENABLED a 'true'."
                            env.DB_BACKUP_ENABLED = 'true'
                        } else {
                            echo "Validación OK: Ambiente de Producción. DB_BACKUP_ENABLED ya está en 'true'."
                        }
                    } else if (params.ENVIRONMENT == 'Desarrollo' || params.ENVIRONMENT == 'Pre-productivo (PP)') {
                        echo "Ambiente de Desarrollo/Pre-productivo. Se respeta el valor de DB_BACKUP_ENABLED: ${env.DB_BACKUP_ENABLED}"
                    } else {
                        error("ERROR CRÍTICO: Ambiente de despliegue no válido: ${params.ENVIRONMENT}")
                    }
                }
            }
        }

        stage('Imprimir variables por sección') {
            steps {
                script {
                    // --- Sección: Ocultas (environment)
                    def ocultas = [
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
                        DB_BACKUP_ENABLED            : params.DB_BACKUP_ENABLED,
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
        } // Fin del Stage 'Imprimir variables por sección'
        // NOTA: Aquí estaba la llave extra que cerraba el bloque 'stages'. ¡Ha sido eliminada!
        
        stage('Validar y Transicionar Ticket Jira') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'JIRA_TOKEN', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_API_TOKEN')]) {
                        echo "Consultando estado actual del ticket ${TICKET_JIRA}..."
        
                        def estado = sh(
                            script: """bash -c ' curl -s -u "$JIRA_USER:$JIRA_API_TOKEN" \
                            -X GET "${JIRA_API_URL}${TICKET_JIRA}" \
                            -H "Accept: application/json" \
                            | jq -r ".fields.status.name // \\"Desconocido\\"" '""",
                            returnStdout: true
                        ).trim()
        
                        echo "Estado actual: ${estado}"
        
                        // IDs de transición Jira
                        def transiciones = [
                            "Tareas por hacer": "31" // directo a Done
                        ]
        
                        // Guardar estado para siguiente stage
                        env.ESTADO_TICKET = estado
        
                        if (estado == "Tareas por hacer") {
                            def transitionId = transiciones[estado]
                            def mensajeJira = "El ticket ${TICKET_JIRA} fue cerrado automáticamente por ejecución del pipeline."
        
                            echo "Transicionando ticket ${TICKET_JIRA} a 'Done'..."
        
                            // Realizar transición
                            def payloadTrans = groovy.json.JsonOutput.toJson([transition: [id: transitionId]])
                            writeFile file: 'transicion.json', text: payloadTrans
                            sh """curl -s -u "$JIRA_USER:$JIRA_API_TOKEN" \
                            -X POST -H "Content-Type: application/json" \
                            --data @transicion.json \
                            "${JIRA_API_URL}${TICKET_JIRA}/transitions" """
        
                            // Agregar comentario en Jira
                            def comentario = groovy.json.JsonOutput.toJson([
                                body: [
                                    type: "doc",
                                    version: 1,
                                    content: [[
                                        type: "paragraph",
                                        content: [[type: "text", text: mensajeJira]]
                                    ]]
                                ]
                            ])
                            writeFile file: 'comentario.json', text: comentario
                            sh """curl -s -u "$JIRA_USER:$JIRA_API_TOKEN" \
                            -X POST -H "Content-Type: application/json" \
                            --data @comentario.json \
                            "${JIRA_API_URL}${TICKET_JIRA}/comment" """
        
                            echo "Ticket ${TICKET_JIRA} transicionado exitosamente a Done."
        
                        } else if (estado in ["Done", "Finalizado"]) {
                            error("El ticket ${TICKET_JIRA} ya está marcado como '${estado}'. No se puede volver a transicionar.")
                        } else {
                            def mensajeError = "Ticket ${TICKET_JIRA} no puede ejecutarse. Estado actual: '${estado}'. Solo se permite si está en 'Tareas por hacer'."
                            echo mensajeError
        
                            // Comentario Jira
                            def comentarioError = groovy.json.JsonOutput.toJson([
                                body: [
                                    type: "doc",
                                    version: 1,
                                    content: [[
                                        type: "paragraph",
                                        content: [[type: "text", text: mensajeError]]
                                    ]]
                                ]
                            ])
                            writeFile file: 'comentario_error.json', text: comentarioError
                            sh """curl -s -u "$JIRA_USER:$JIRA_API_TOKEN" \
                            -X POST -H "Content-Type: application/json" \
                            --data @comentario_error.json \
                            "${JIRA_API_URL}${TICKET_JIRA}/comment" """
        
                            error("Error: El estado del ticket ${TICKET_JIRA} no permitido.")
                        }
                    }
                }
            }
        }
        
        stage('Notificar a Teams') {
    steps {
        script {
            // Verificar si ya estaba en Done
            if (env.ESTADO_TICKET in ["Done", "Finalizado"]) {
                def mensajeError = "El ticket ${TICKET_JIRA} ya estaba '${env.ESTADO_TICKET}'. No se realizó nueva transición ni despliegue."
                echo mensajeError

                def payloadError = groovy.json.JsonOutput.toJson([ text: mensajeError ])
                writeFile file: 'teams_error.json', text: payloadError
                sh "curl -s -X POST -H 'Content-Type: application/json' --data @teams_error.json ${TEAMS_WEBHOOK}"

                error("Pipeline detenido: ticket ya finalizado.")
            }

            // Mensaje con saltos de línea reales (JsonOutput se encargará de escapar)
            def mensajeTeams = """Ticket ${TICKET_JIRA} cambió automáticamente de 'Tareas por hacer' a 'Finalizado (Done)'.

Pipeline completado correctamente.

Instancia de base de datos creada con los siguientes detalles:"""

            def facts = [
                [ name: "Pais", value: env.PAIS ?: "" ],
                [ name: "Ambiente", value: params.ENVIRONMENT ?: "" ],
                [ name: "Base de Datos", value: "${params.DB_ENGINE} (${params.DB_VERSION})".toString() ],
                [ name: "Proyecto GCP", value: params.PROJECT_ID ?: "" ],
                [ name: "Nombre de la instancia", value: params.DB_INSTANCE_NAME ?: "" ],
                [ name: "Región/Zona", value: "${params.REGION}/${params.ZONE}" ],
                [ name: "Tipo de Máquina", value: params.MACHINE_TYPE ?: "" ],
                [ name: "Almacenamiento", value: "${params.DB_STORAGE_SIZE} GB (${params.DB_STORAGE_TYPE})" ],
                [ name: "Backup", value: params.DB_BACKUP_ENABLED ?: "" ]
            ]

            def card = [
                '@type': 'MessageCard',
                '@context': 'http://schema.org/extensions',
                text: mensajeTeams,
                summary: "Nueva Instancia ${env.DB_ENGINE}",
                themeColor: "0076D7",
                sections: [[
                    activitySubtitle: "Ticket Jira: ${params.TICKET_JIRA}",
                    facts: facts,
                    markdown: true
                ]],
                potentialAction: [[
                    '@type': "OpenUri",
                    name: "Ver Build",
                    targets: [[ os: "default", uri: env.BUILD_URL ?: "" ]]
                ]]
            ]

            def payload = groovy.json.JsonOutput.toJson(card)
            writeFile file: 'teams_final.json', text: payload
            sh "curl -s -X POST -H 'Content-Type: application/json' --data @teams_final.json ${TEAMS_WEBHOOK}"

            echo "Notificación enviada a Teams con éxito."
        }
    }
}

        stage("descripción Jira"){
            steps{
                script{

                    def notificationText = """
                    Pais : ${env.PAIS}
                    Instancia: ${params.DB_INSTANCE_NAME ?: 'N/A'}
                    Ambiente: ${params.ENVIRONMENT}
                    Proyecto GCP: ${params.PROJECT_ID}
                    Región/Zona: ${params.REGION} / ${params.ZONE}

                    Base de datos:
            
                    - Nombre: ${params.DB_ENGINE}
                    - Versión: ${params.DB_VERSION}
                    - Usuario: ${params.DB_USERNAME}
                    - Máx conexiones: ${params.DB_MAX_CONNECTIONS}

                    Recursos:
   
                    - Máquina: ${params.MACHINE_TYPE}
                    - Almacenamiento: ${params.DB_STORAGE_SIZE} GB (${params.DB_STORAGE_TYPE})
                    - Auto-resize: ${params.DB_STORAGE_AUTO_RESIZE}

                    Red y acceso:
             
                    - VPC/Subnet: ${params.DB_VPC_NETWORK} / ${params.DB_SUBNET}
                    - IP Privada: ${params.DB_PRIVATE_IP_ENABLED}
                    - Acceso Público: ${params.DB_PUBLIC_ACCESS_ENABLED}

                    Seguridad:
                    - Encriptación: ${params.DB_ENCRYPTION_ENABLED}
 
                    - Protección eliminación: ${params.DB_DELETION_PROTECTION}
                    - IAM Role: ${params.DB_IAM_ROLE}

                    Backup / Mantenimiento:
                    - Retención (días): ${params.DB_BACKUP_RETENTION_DAYS}
          
                    - Backup start: ${params.DB_BACKUP_START_TIME}
                    - Ventana mantenimiento: ${params.DB_MAINTENANCE_WINDOW_DAY} ${params.DB_MAINTENANCE_WINDOW_HOUR}

                    Alta disponibilidad / Monitoreo:
                    - BACKUP Habilitado: ${params.DB_BACKUP_ENABLED}
                 
                    - Monitoring: ${params.DB_MONITORING_ENABLED}
                    """

                    env.mensaje = notificationText
                }
            }
        }

        stage('Crear ticket en Jira') {
            when {
                expression { params.ENVIRONMENT == 'Produccion' }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'JIRA_TOKEN', usernameVariable: 'JIRA_USER', passwordVariable: 'JIRA_API_TOKEN')]) {
  
                        def auth = "${JIRA_USER}:${JIRA_API_TOKEN}".bytes.encodeBase64().toString()

                        // Construir el payload como mapa

                        def payloadMap = [
                   
                            fields: [
                                project: [ key: env.PROYECT_JIRA ],
                                summary: env.TITULO_JIRA,
                     
                                description: [
                                    type: "doc",
                                    version: 1,
              
                                    content: [
                                        [
                                     
                                            type: "paragraph",
                                            content: 
                                            [
   
                                                [ 
                                                    type: "text", 
                                                    text: env.mensaje ?: "Descripción no disponible"
                                                ],                                    
          
                                                [
                                                    type: "text",
         
                                                    text: "Ver Build",
                                                    marks: [
  
                                                        [ type: "link", attrs: [ href: "${env.BUILD_URL}" ] ]
                                      
                                                    ]
                                                ]

                                      
                                            ]
                                        ]
                                    ],

                  
                                ],
                                issuetype: [ id: env.ID_ISSUETYPE_JIRA ]
                            ]
                      
                        ]
                        // Convertir a JSON válido
                        def payloadJson = groovy.json.JsonOutput.toJson(payloadMap)
                        echo "${payloadJson}"
                  
                        def response = sh(
                            script: """
                            curl -s -X POST "${JIRA_API_URL}" \\
                             
                            -H "Authorization: Basic ${auth}" \\
                                -H "Content-Type: application/json" \\
                                -d '${payloadJson}'
                         
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Comentario enviado: ${response}"
                 
                    }
                }
            }
        }
    } 

    post {
        success { echo 'Pipeline ejecutado correctamente.' }
        failure { echo 'Error al ejecutar el pipeline.' }
    }
}
