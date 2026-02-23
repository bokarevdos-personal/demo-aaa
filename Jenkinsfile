pipeline {
  agent any

  environment {
    // OpenShift API
    OCP_API_URL = credentials('ocp-api-url')

    // RHACS
    ROX_CENTRAL_ADDRESS = credentials('rhacs-central-address')
    ROX_API_TOKEN       = credentials('rhacs-api-token')
  }

  options {
    skipDefaultCheckout(true)
    timestamps()
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('OpenShift connectivity diagnostics') {
      steps {
        sh '''
          set -euo pipefail

          echo "OCP_API_URL=$OCP_API_URL"

          echo "oc version:"
          oc version || true

          echo "DNS check:"
          HOST=$(echo "$OCP_API_URL" | sed -E 's#https?://##' | cut -d/ -f1 | cut -d: -f1)
          echo "HOST=$HOST"
          getent hosts "$HOST" || true

          echo "TCP check (best-effort):"
          # если nc есть — проверим порт
          if command -v nc >/dev/null 2>&1; then
            PORT=$(echo "$OCP_API_URL" | sed -E 's#https?://##' | cut -d/ -f1 | awk -F: '{print ($2==""?443:$2)}')
            echo "PORT=$PORT"
            nc -vz -w 5 "$HOST" "$PORT" || true
          else
            echo "nc not installed, skipping"
          fi

          echo "HTTP checks:"
          # /version и /readyz обычно доступны без auth и быстро показывают, жив ли API
          curl -vk --connect-timeout 10 --max-time 20 "$OCP_API_URL/version" || true
          curl -vk --connect-timeout 10 --max-time 20 "$OCP_API_URL/readyz"  || true
        '''
      }
    }

    stage('Login to OpenShift + list pipelines') {
      steps {
        withCredentials([string(credentialsId: 'kubeadmin-secret', variable: 'OCP_PASSWORD')]) {
          sh '''
            set -euo pipefail

            echo "Logging in to OpenShift (debug enabled)..."
            # loglevel=8 даст подробности, почему EOF (TLS, proxy, route, etc.)
            oc login "$OCP_API_URL" -u kubeadmin -p "$OCP_PASSWORD" \
              --insecure-skip-tls-verify=true \
              --request-timeout=30s \
              --loglevel=8

            echo "Who am I?"
            oc whoami
            oc whoami --show-server

            echo "Namespaces:"
            oc get ns | head -n 50 || true

            echo "Pipelines (Tekton) across all namespaces (if installed):"
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

            kubectl apply -f policies/k8s/00-namespace.yaml
            kubectl apply -f policies/k8s/

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

          if [ ! -f "manifests/deployment.yaml" ]; then
            echo "manifests/deployment.yaml not found - skipping check (example stage)."
            exit 0
          fi

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
    always { echo 'Done.' }
  }
}
