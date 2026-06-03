pipeline {
    agent {
        docker {
            image 'devops-agent:latest'
            args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        APELLIDO = "moreno"

        ACR_NAME = "acrglobalcicd"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"

        RESOURCE_GROUP = "rg-cicd-terraform-app-baraujo03"
        AKS_NAME = "aks-dev-eastus"
    }

    stages {

        stage('[CI] Instalar dependencias de app') {
            steps {
                sh '''
                  echo ">>> Instalando dependencias..."
                  npm install
                '''
            }
        }

        stage('[CI] Ejecutar pruebas unitarias') {
            steps {
                sh '''
                  echo ">>> Ejecutando pruebas unitarias..."
                  npm run test:unit -- --passWithNoTests
                '''
            }
        }

        stage('[CI] Ejecutar pruebas de integración') {
            steps {
                sh '''
                  echo ">>> Ejecutando pruebas de integración..."
                  npm run test:integration -- --passWithNoTests
                '''
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId',       variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                      echo ">>> Azure login..."
                      az login --service-principal \
                        --username="$AZ_CLIENT_ID" \
                        --password="$AZ_CLIENT_SECRET" \
                        --tenant="$AZ_TENANT_ID"

                      az account set --subscription "$AZ_SUBSCRIPTION_ID"
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                  echo ">>> Obteniendo credenciales de AKS..."
                  az aks get-credentials \
                    --resource-group "$RESOURCE_GROUP" \
                    --name "$AKS_NAME" \
                    --overwrite-existing
                '''
            }
        }

        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.FULL_IMAGE_NAME = "${env.ACR_LOGIN_SERVER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    echo "IMAGE_TAG generado: ${env.IMAGE_TAG}"
                    echo "Imagen completa: ${env.FULL_IMAGE_NAME}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                  echo ">>> Login al ACR..."
                  az acr login --name "$ACR_NAME"

                  echo ">>> Build de imagen..."
                  docker build -t "$FULL_IMAGE_NAME" .

                  echo ">>> Push al ACR..."
                  docker push "$FULL_IMAGE_NAME"
                '''
            }
        }

        stage('[CD-DEV] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "dev"
                    env.API_PROVIDER_URL = "https://dev.api.com"
                }

                sh '''
                  echo ">>> Renderizando manifiesto DEV..."
                  envsubst < k8s.yml > k8s-dev.yml
                  cat k8s-dev.yml

                  echo ">>> Desplegando en DEV..."
                  az aks command invoke \
                    --resource-group "$RESOURCE_GROUP" \
                    --name "$AKS_NAME" \
                    --command "kubectl apply -f k8s-dev.yml" \
                    --file k8s-dev.yml
                '''
            }
        }

        stage('[CD-DEV] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer DEV..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-dev"

                  kubectl get svc "$SERVICE_NAME" || true

                  LB_IP=""
                  MAX_RETRIES=10
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)

                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer DEV."
                    exit 1
                  else
                    echo ">>> IP DEV: http://$LB_IP"
                  fi
                '''
            }
        }

        stage('Aprobación QA') {
            steps {
                input message: '¿Deseas desplegar a QA?', ok: 'Aprobar QA'
            }
        }

        stage('[CD-QA] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "qa"
                    env.API_PROVIDER_URL = "https://qa.api.com"
                }

                sh '''
                  echo ">>> Renderizando manifiesto QA..."
                  envsubst < k8s.yml > k8s-qa.yml
                  cat k8s-qa.yml

                  echo ">>> Desplegando en QA..."
                  az aks command invoke \
                    --resource-group "$RESOURCE_GROUP" \
                    --name "$AKS_NAME" \
                    --command "kubectl apply -f k8s-qa.yml" \
                    --file k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer QA..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-qa"

                  kubectl get svc "$SERVICE_NAME" || true

                  LB_IP=""
                  MAX_RETRIES=10
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)

                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer QA."
                    exit 1
                  else
                    echo ">>> IP QA: http://$LB_IP"
                  fi
                '''
            }
        }

        stage('Aprobación PRD') {
            steps {
                input message: '¿Deseas desplegar a PRD?', ok: 'Aprobar PRD'
            }
        }

        stage('[CD-PRD] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "prd"
                    env.API_PROVIDER_URL = "https://api.com"
                }

                sh '''
                  echo ">>> Renderizando manifiesto PRD..."
                  envsubst < k8s.yml > k8s-prd.yml
                  cat k8s-prd.yml

                  echo ">>> Desplegando en PRD..."
                  az aks command invoke \
                    --resource-group "$RESOURCE_GROUP" \
                    --name "$AKS_NAME" \
                    --command "kubectl apply -f k8s-prd.yml" \
                    --file k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer PRD..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-prd"

                  kubectl get svc "$SERVICE_NAME" || true

                  LB_IP=""
                  MAX_RETRIES=10
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)

                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer PRD."
                    exit 1
                  else
                    echo ">>> IP PRD: http://$LB_IP"
                  fi
                '''
            }
        }
    }

    post {
        success {
            echo ">>> Pipeline CI/CD finalizado correctamente."
        }
        failure {
            echo ">>> Pipeline falló. Revisar logs de Jenkins."
        }
    }
}