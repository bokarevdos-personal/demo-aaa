pipeline {
  agent any

  environment {
    // текущий URL из Jenkins credential (как у тебя сейчас)
    OCP_API_URL = credentials('ocp-api-url')

    // RHACS
    ROX_CENTRAL_ADDRESS = credentials('rhacs-central-address')
    ROX_API_TOKEN       = credentials('rhacs-api-token')
  }

  options {
    skipDefaultCheckout(true)
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('OpenShift connectivity diagnostics (443/6443)') {
      steps {
        sh '''
          set -euo pipefail

          echo "Configured OCP_API_URL=$OCP_API_URL"

          BASE_HOST=$(echo "$OCP_API_URL" | sed -E 's#https?://##' | cut -d/ -f1 | cut -d: -f1)
          echo "BASE_HOST=$BASE_HOST"
          getent hosts "$BASE_HOST" || true

          echo "=== Test 443 ==="
          curl -vk --connect-timeout 10 --max-time 20 "https://$BASE_HOST:443/version" || true
          echo "--- TLS1.2 forced (443) ---"
          curl -vk --tlsv1.2 --connect-timeout 10 --max-time 20 "https://$BASE_HOST:443/version" || true

          echo "=== Test 6443 ==="
          curl -vk --connect-timeout 10 --max-time 20 "https://$BASE_HOST:6443/version" || true
          echo "--- TLS1.2 forced (6443) ---"
          curl -vk --tlsv1.2 --connect-timeout 10 --max-time 20 "https://$BASE_HOST:6443/version" || true

          echo "=== Pick working endpoint ==="
          if curl -sk --connect-timeout 5 --max-time 10 "https://$BASE_HOST:6443/version" >/dev/null 2>&1; then
            echo "Chosen endpoint: https://$BASE_HOST:6443"
            echo "https://$BASE_HOST:6443" > .ocp_server
          elif curl -sk --connect-timeout 5 --max-time 10 "https://$BASE_HOST:443/version" >/dev/null 2>&1; then
            echo "Chosen endpoint: https://$BASE_HOST:443"
            echo "https://$BASE_HOST:443" > .ocp_server
          else
            echo "ERROR: neither 6443 nor 443 responded to /version from this Jenkins node."
            echo "This is a network/LB/TLS issue between Jenkins and API endpoint."
            exit 2
          fi
        '''
      }
    }

    stage('Login to OpenShift + list pipelines') {
      steps {
        withCredentials([string(credentialsId: 'kubeadmin-secret', variable: 'OCP_PASSWORD')]) {
          sh '''
            set -euo pipefail

            SERVER=$(cat .ocp_server)
            echo "Logging in to OpenShift server: $SERVER"

            oc version || true

            oc login "$SERVER" -u kubeadmin -p "$OCP_PASSWORD" \
              --insecure-skip-tls-verify=true \
              --request-timeout=30s \
              --loglevel=8

            oc whoami
            oc whoami --show-server

            echo "Pipelines (Tekton) across all namespaces (if installed):"
            oc get pipelines.tekton.dev -A || echo "Tekton Pipelines not found (CRD pipelines.tekton.dev is missing)."

            echo "PipelineRuns:"
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

            kubectl apply -f policies/acs-demo-unauth-process-exec-kill-pod.yaml
            kubectl apply -f policies/acs-demo-unauth-terminal-kill-pod-labeled.yaml
            kubectl apply -f policies/acs-demo-unauth-terminal-kill-pod.yaml

            kubectl -n rhacs-operator get networkpolicy -o wide
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
