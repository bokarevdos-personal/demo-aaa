pipeline {
  agent any

  environment {
    // OpenShift API (credential Secret text)
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
          curl -vk --connect-timeout 10 --max-time 15 "https://$BASE_HOST:443/version" || true

          echo "=== Test 6443 ==="
          curl -vk --connect-timeout 10 --max-time 15 "https://$BASE_HOST:6443/version" || true

          echo "=== Pick working endpoint ==="
          if curl -sk --connect-timeout 5 --max-time 10 "https://$BASE_HOST:6443/version" >/dev/null 2>&1; then
            echo "https://$BASE_HOST:6443" > .ocp_server
          elif curl -sk --connect-timeout 5 --max-time 10 "https://$BASE_HOST:443/version" >/dev/null 2>&1; then
            echo "https://$BASE_HOST:443" > .ocp_server
          else
            echo "ERROR: neither 6443 nor 443 responded to /version from this Jenkins node."
            exit 2
          fi

          echo "Chosen endpoint: $(cat .ocp_server)"
        '''
      }
    }

    stage('Login to OpenShift') {
      steps {
        withCredentials([string(credentialsId: 'kubeadmin-secret', variable: 'OCP_PASSWORD')]) {
          sh '''
            set -euo pipefail

            SERVER="$(cat .ocp_server)"
            echo "Logging in to OpenShift server: $SERVER"

            oc version || true

            oc login "$SERVER" -u kubeadmin -p "$OCP_PASSWORD" \
              --insecure-skip-tls-verify=true \
              --request-timeout=30s

            oc whoami
            oc whoami --show-server
          '''
        }
      }
    }

    stage('Cluster info + Pipelines check') {
      steps {
        sh '''
          set -euo pipefail

          echo "Namespaces (top 50):"
          oc get ns | head -n 50 || true

          echo "Tekton check:"
          if oc api-resources | awk '{print $1}' | grep -qx "pipelines"; then
            echo "Tekton CRDs present. Listing Pipelines:"
            oc get pipelines -A || true
            oc get pipelineruns -A || true
          else
            echo "Tekton Pipelines NOT installed (no pipelines resource)."
          fi
        '''
      }
    }

    stage('Policy sync to cluster (RHACS SecurityPolicy CRD)') {
      steps {
        sh '''
          set -euo pipefail

          echo "Apply SecurityPolicy manifests to cluster..."
          oc apply -f policies/

          echo "Detect SecurityPolicy resource name..."
          # На разных версиях может отличаться, но у тебя точно есть securitypolicies
          RES="$(oc api-resources --api-group=config.stackrox.io -o name | grep -E '^securitypolic' | head -n1 || true)"

          if [ -z "$RES" ]; then
            echo "ERROR: SecurityPolicy resource not found in api-resources for api-group=config.stackrox.io"
            oc api-resources | grep -i stackrox || true
            exit 3
          fi

          echo "Using resource: $RES"

          echo "List SecurityPolicies in rhacs-operator:"
          oc get "$RES" -n rhacs-operator -o wide || true

          echo "Show applied objects (names):"
          # Печатаем имена файлов и имена CR
          for f in policies/*.yaml; do
            echo "---- $f ----"
            # безопасно: вытащим metadata.name
            NAME=$(awk '/^metadata:/ {m=1} m && /^  name:/ {print $2; exit}' "$f" || true)
            if [ -n "$NAME" ]; then
              oc get "$RES" -n rhacs-operator "$NAME" -o yaml | head -n 60 || true
            fi
          done
        '''
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

    stage('ACS Check deployments in repo (optional)') {
      steps {
        sh '''
          set -euo pipefail

          if [ ! -d "deployments" ]; then
            echo "No deployments/ directory, skipping."
            exit 0
          fi

          shopt -s nullglob
          files=(deployments/*.yaml deployments/*.yml)
          if [ ${#files[@]} -eq 0 ]; then
            echo "No deployment yaml files in deployments/, skipping."
            exit 0
          fi

          echo "Running roxctl deployment check on repo deployments..."
          for f in "${files[@]}"; do
            echo "== Checking $f =="
            docker run --rm \
              -e ROX_API_TOKEN="$ROX_API_TOKEN" \
              -e ROX_CENTRAL_ADDRESS="$ROX_CENTRAL_ADDRESS" \
              -v "$PWD:/work" -w /work \
              registry.redhat.io/advanced-cluster-security/rhacs-roxctl-rhel8:4.8.8 \
              roxctl deployment check -f "/work/$f" -o table
          done
        '''
      }
    }
  }

  post {
    always { echo 'Done.' }
  }
}
