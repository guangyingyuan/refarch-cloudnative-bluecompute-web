podTemplate(label: 'mypod',
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
        secretVolume(secretName: 'registry-account', mountPath: '/var/run/secrets/registry-account'),
        configMapVolume(configMapName: 'registry-config', mountPath: '/var/run/configs/registry-config')
    ],
    containers: [
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat')
  ]) {

    node('mypod') {
        checkout scm
        container('docker') {
            stage('Build Docker Image') {
                sh """
                #!/bin/bash
                NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                REGISTRY=`cat /var/run/configs/registry-config/registry`

                docker build -t \${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER} .
                """
            }
            stage('Push Docker Image to Registry') {
                sh """
                #!/bin/bash
                NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                REGISTRY=`cat /var/run/configs/registry-config/registry`

                set +x
                DOCKER_USER=`cat /var/run/secrets/registry-account/username`
                DOCKER_PASSWORD=`cat /var/run/secrets/registry-account/password`
                docker login -u=\${DOCKER_USER} -p=\${DOCKER_PASSWORD} \${REGISTRY}
                set -x

                docker push \${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER}
                """
            }
        }
        container('kubectl') {
            stage('testing'){
                sh """
                #!/bin/bash
                curl -k -s -XGET -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhdF9oYXNoIjoiNWhybWl4bjJsdmR2ZW12MDI0b2IiLCJyZWFsbU5hbWUiOiJjdXN0b21SZWFsbSIsInVuaXF1ZVNlY3VyaXR5TmFtZSI6ImFkbWluIiwiaXNzIjoiaHR0cHM6Ly9pY3Atc2UtZGV2LWNsdXN0ZXIuaWNwOjk0NDMvb2lkYy9lbmRwb2ludC9PUCIsImF1ZCI6IjhiZGViNjM2ODAyOTBmOGQyMzM1M2UwNWYwMzhiNTY3IiwiZXhwIjoxNTI1OTYwNTE4LCJpYXQiOjE1MjU5NjA1MTgsInN1YiI6ImFkbWluIiwidGVhbVJvbGVNYXBwaW5ncyI6W119.PBD0Qmf1yqX2RPPDqkp4XfCUnm5UVvBt8QHjkNDzcTDv_CZbBVhGS1GUcJE5DMxy1qnCkw_uzSGWYNAZ4wkKdhX-Mgf02-M8Sqvpd7Ol0VTJvRswCMqdl2ZyUwZa_Vi_6U2Duq145phEZLbkSOMSZ5kgPSgt1180wqtw_XDJsUTB4E65qj8aj24Evzg2wopswg82d8djFiRVKIZ3HoQtQLMBI3BCI94t6YdW_hcDnC1TzMRlFpC9idQdzw478yJyKkpcf-jRx9e3n3FGkudRg2c9Wj6R1n6rmh8u9ZbW0bEnMAgyYWT454Xt8x2fd2X7asiAK2mj_1VqBolLMa8Y4Q' 'https://172.16.40.4:8443/va/api/get-report?namespace=default/blue-orders-mysql-794b456cc8-svwhz/a3948d937af86210889b9416c9af69fc1bc52cd3dd481b8ebb6e094b99797154&access_group=default&source_type=container&report_type=vulnerability'
                """
            }
            stage('Deploy new Docker Image') {
                sh """
                #!/bin/bash
                set +e
                NAMESPACE=`cat /var/run/configs/registry-config/namespace`
                REGISTRY=`cat /var/run/configs/registry-config/registry`
                DEPLOYMENT=`kubectl get deployments | grep bluecompute | grep web | awk '{print \$1}'`

                kubectl get deployments \${DEPLOYMENT}

                if [ \${?} -ne "0" ]; then
                    # No deployment to update
                    echo 'No deployment to update'
                    exit 1
                fi

                # Update Deployment
                kubectl set image deployment/\${DEPLOYMENT} web=\${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER}
                kubectl rollout status deployment/\${DEPLOYMENT}
                """
            }
        }
    }
}
