pipeline {
  agent { label 'tool' }

  environment {
    PYTHONNOUSERSITE = "1"
    NAMESPACE        = "koganlab7"
    REGISTRY_ID      = "crpist2uge71cahfb48e"
    IMAGE            = "cr.yandex/${REGISTRY_ID}/restoringvalues:latest"
  }

  options {
    timestamps()
    timeout(time: 25, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Fetch artifacts from L2') {
      steps {
        sh 'rm -rf deploy_art && mkdir -p deploy_art'
        copyArtifacts(projectName: 'KoganSK/KoganSK_lab2', selector: lastSuccessful())
        sh '''#!/usr/bin/env bash
          set -e
          echo "=== Artifacts from L2 ==="
          find . -type f \\( -name "*.tgz" -o -name "*.whl" \\) | head -20
          
          # Копируем артефакты
          cp -f app-restoringvalues.tgz deploy_art/ 2>/dev/null || true
          find . -path "./dist/*.whl" -exec cp {} deploy_art/ \\; 2>/dev/null || true
          
          echo "=== Copied artifacts ==="
          ls -la deploy_art/
        '''
      }
    }

    stage('Docker build') {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          echo "=== Building Docker image ==="
          docker build --no-cache -t "$IMAGE" -f Docker/Dockerfile .
        '''
      }
    }

    stage('Docker login + push to YCR') {
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail

          YC=/home/ubuntu/yandex-cloud/bin/yc
          test -x "$YC" || (echo "yc not found at $YC" && exit 1)

          echo "=== Getting IAM token ==="
          TOKEN="$($YC iam create-token)"

          echo "=== Docker login to YCR ==="
          echo "$TOKEN" | docker login --username iam --password-stdin cr.yandex

          echo "=== Pushing image ==="
          docker push "$IMAGE"
        '''
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          echo "=== Creating namespace if not exists ==="
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

          echo "=== Cleaning up previous deployment ==="
          kubectl delete deployment restoringvalues -n "$NAMESPACE" --ignore-not-found=true
          kubectl delete pod -l app=restoringvalues -n "$NAMESPACE" --ignore-not-found=true
          sleep 5

          echo "=== Deploying to Kubernetes ==="
          # Подставляем registry_id в манифест
          sed "s|<REGISTRY_ID>|${REGISTRY_ID}|g" k8s/deployment.yaml | kubectl apply -n "$NAMESPACE" -f -
          
          echo "=== Deploying services ==="
          kubectl apply -n "$NAMESPACE" -f k8s/service-simulator.yaml
          kubectl apply -n "$NAMESPACE" -f k8s/service-reciever.yaml
          kubectl apply -n "$NAMESPACE" -f k8s/service-business.yaml

          echo "=== Waiting for rollout ==="
          timeout 180 kubectl rollout status deployment/restoringvalues -n "$NAMESPACE" || true
          
          echo "=== Pods status ==="
          kubectl get pods -n "$NAMESPACE" -o wide --show-labels
          
          echo "=== Services status ==="
          kubectl get svc -n "$NAMESPACE"
        '''
      }
    }

    stage('Show logs') {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          echo "=== Getting pod logs ==="
          
          # Ждем немного
          sleep 15
          
          POD=$(kubectl get pods -n "$NAMESPACE" -l app=restoringvalues -o jsonpath="{.items[0].metadata.name}" 2>/dev/null || echo "")
          
          if [ -n "$POD" ]; then
            echo "Pod name: $POD"
            echo "=== Pod description ==="
            kubectl describe pod -n "$NAMESPACE" "$POD" || true
            
            echo "=== Pod logs (last 200 lines) ==="
            kubectl logs -n "$NAMESPACE" "$POD" --tail=200 || true
          else
            echo "No pods found with label app=restoringvalues"
            echo "=== All pods in namespace ==="
            kubectl get pods -n "$NAMESPACE"
          fi
        '''
      }
    }
  }
  
  post {
    always {
      sh '''#!/usr/bin/env bash
        echo "=== Final status ==="
        kubectl get all -n "$NAMESPACE"
        
        echo "=== Events ==="
        kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | tail -20
        
        echo "=== Problem pods details ==="
        kubectl get pods -n "$NAMESPACE" -o jsonpath="{range .items[*]}{.metadata.name}{'\\n'}{end}" | while read pod; do
          if [ "$pod" != "" ]; then
            echo "--- Pod: $pod ---"
            kubectl get pod "$pod" -n "$NAMESPACE" -o jsonpath="{.status.phase}{' - '}{.status.message}{'\\n'}"
          fi
        done
      '''
    }
  }
}
