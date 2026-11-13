pipeline {
    agent any

    environment {
        AWX_URL      = 'http://192.168.2.140:8080'
        AWX_CREDS_ID = 'awx-api-token'
    }

    parameters {
        string(name: 'AWX_JOB_ID',   defaultValue: '', description: 'AWX job ID passato dal webhook')
        string(name: 'AWX_STATUS',   defaultValue: '', description: 'Stato del job AWX')
        string(name: 'AWX_JOB_URL',  defaultValue: '', description: 'URL del job AWX')
    }

    stages {
        stage('Validate input') {
            steps {
                script {
                    if (!params.AWX_JOB_ID?.trim()) {
                        error "AWX_JOB_ID non fornito."
                    }
                    echo "Triggerato da #${params.AWX_JOB_ID} (status: ${params.AWX_STATUS})"
                }
            }
        }

        stage('Recupero Artefatto') {
            steps {
                withCredentials([string(credentialsId: AWX_CREDS_ID, variable: 'AWX_TOKEN')]) {
                    sh '''
                        mkdir -p src
                        
                        echo "Download dei metadati del job AWX #${AWX_JOB_ID}"
                        RESPONSE=$(curl -sk -H "Authorization: Bearer ${AWX_TOKEN}" "${AWX_URL}/api/v2/jobs/${AWX_JOB_ID}/")
                        
                        echo "Estrazione dati"
                        FILENAME=$(echo "$RESPONSE" | jq -r '.artifacts.source_filename')
                        SOURCE_B64=$(echo "$RESPONSE" | jq -r '.artifacts.source_b64')
                        EXPECTED_SHA256=$(echo "$RESPONSE" | jq -r '.artifacts.sha256')
                        
                        if [ "$SOURCE_B64" = "null" ] || [ -z "$SOURCE_B64" ]; then
                            echo "Errore: Nessun artefatto 'source_b64' trovato nel job #${AWX_JOB_ID}"
                            echo "Risposta API: $RESPONSE"
                            exit 1
                        fi
                        
                        echo "Decodifica e salvataggio del file: src/${FILENAME}"
                        echo "$SOURCE_B64" | base64 -d > "src/${FILENAME}"
                        
                        echo "Verifica integrità SHA256"
                        ACTUAL_SHA256=$(sha256sum "src/${FILENAME}" | awk '{print $1}')
                        if [ "$EXPECTED_SHA256" != "null" ] && [ "$ACTUAL_SHA256" != "$EXPECTED_SHA256" ]; then
                            echo "SHA256 mismatch! Atteso: $EXPECTED_SHA256, Ottenuto: $ACTUAL_SHA256"
                            exit 1
                        fi
                        
                        echo "Artefatto salvato e verificato: src/${FILENAME}"
                    '''
                }
            }
        }

        stage('Compila Java') {
            steps {
                sh '''
                    echo "Compilazione del sorgente Java"
                    mkdir -p build/classes
                    javac -d build/classes src/*.java
                    echo "Compilazione completata."
                '''
            }
        }
    }

    post {
        always {
            echo "Archiviazione artefatti"
            archiveArtifacts artifacts: 'src/*.java, build/classes/*.class',
                             fingerprint: true,
                             allowEmptyArchive: true
        }
        success {
            echo "Build completata #${params.AWX_JOB_ID}"
        }
        failure {
            echo "Build fallita #${params.AWX_JOB_ID}"
        }
    }
}