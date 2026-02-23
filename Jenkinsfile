pipeline {
  agent any

  environment {
    // RHACS
    ROX_CENTRAL_ADDRESS = credentials('rhacs-central-address') // Secret text
    ROX_API_TOKEN       = credentials('rhacs-api-token')       // Secret text

    // OpenShift API URL (добавь credential типа Secret text с этим id)
    OCP_API_URL         = credentials('ocp-api-url')
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Login to OpenShift + list pipelines') {
      steps {
        withCredentials([string(credentialsId: 'kubeadmin-secret', variable: 'OCP_PASSWORD')]) {
          sh '''
            set -euo pipefail

            echo "Logging in to OpenShift..."
            oc version || true

            # Логин (kubeadmin + пароль из Jenkins secret)
            oc login "$OCP_API_URL" -u kubeadmin -p "$OCP_PASSWORD" --insecure-skip-tls-verify=true

            echo "Who am I?"
            oc whoami
            oc whoami --show-server

            echo "Namespaces:"
            oc get ns | head -n 50 || true

            echo "Pipelines (Tekton) across all namespaces (if installed):"
            # В OpenShift Pipelines (Tekton) ресурс называется pipelines.tekton.dev
            # Если Tekton не установлен — команда упадет, поэтому делаем мягко.
            oc get pipelines.tekton.dev -A || echo "Tekton Pipelines not found (CRD pipelines.tekton.dev is missing)."

            echo "PipelineRuns (optional):"
            oc get pipelineruns.tekton.dev -A || true
          '''
        }
      }
    }

    stage('K8s Policy Sync (kubectl apply)') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-acs-cluster', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            set -euo pipefail
            export KUBECONFIG="$KUBECONFIG_FILE"

            kubectl version --client=true

            # Namespace для демо (если уже есть — ок)
            kubectl apply -f policies/k8s/00-namespace.yaml

            # Синк политик
            kubectl apply -f policies/k8s/

            # Быстрая проверка: вывести примененные NetworkPolicy
            kubectl -n demo-policy get networkpolicy -o wide
          '''
        }
      }
    }

    stage('ACS Policy Gate (roxctl)') {
      steps {
        sh '''
          set -euo pipefail

          docker run --rm \
            -e ROX_API_TOKEN="$ROX_API_TOKEN" \
            -e ROX_CENTRAL_ADDRESS="$ROX_CENTRAL_ADDRESS" \
            registry.redhat.io/advanced-cluster-security/rhacs-roxctl-rhel8:4.8.8 \
            roxctl version
        '''
      }
    }

    stage('ACS Check Example (deployment check)') {
      steps {
        sh '''
          set -euo pipefail

          docker run --rm \
            -e ROX_API_TOKEN="$ROX_API_TOKEN" \
            -e ROX_CENTRAL_ADDRESS="$ROX_CENTRAL_ADDRESS" \
            -v "$PWD:/work" -w /work \
            registry.redhat.io/advanced-cluster-security/rhacs-roxctl-rhel8:4.8.8 \
            roxctl deployment check -f /work/manifests/deployment.yaml -o table
        '''
      }
    }
  }

  post {
    always {
      echo 'Done.'
    }
  }
}
