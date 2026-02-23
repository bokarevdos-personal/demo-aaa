pipeline {
  agent any

  environment {
    // Адрес Central (лучше хранить как credential, но для примера env)
    ROX_CENTRAL_ADDRESS = credentials('rhacs-central-address') // Secret text
    ROX_API_TOKEN       = credentials('rhacs-api-token')       // Secret text
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
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

          # Вариант 1: если roxctl установлен на агенте
          # roxctl version

          # Вариант 2: запуск roxctl из контейнера (часто проще и воспроизводимо)
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

          # Пример: проверить манифест деплоймента на нарушения policy в Central.
          # (Подставьте ваш файл/директорию с манифестами)
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
