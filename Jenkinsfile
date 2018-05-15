podTemplate(label: 'mypod',
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
        secretVolume(secretName: 'registry-account', mountPath: '/var/run/secrets/registry-account'),
        configMapVolume(configMapName: 'registry-config', mountPath: '/var/run/configs/registry-config')
    ],
    containers: [
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'curl' , image: 'everpeace/curl-jq', ttyEnabled: true, command: 'cat')
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
        container('curl') {
            stage('testing'){
                sh """
                #!/bin/bash
                curl -k -s -XGET -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhdF9oYXNoIjoiMXRqc3Jwc3FhcDM2bjNzMG9tZjYiLCJyZWFsbU5hbWUiOiJjdXN0b21SZWFsbSIsInVuaXF1ZVNlY3VyaXR5TmFtZSI6ImFkbWluIiwiaXNzIjoiaHR0cHM6Ly9pY3Atc2UtZGV2LWNsdXN0ZXIuaWNwOjk0NDMvb2lkYy9lbmRwb2ludC9PUCIsImF1ZCI6IjhiZGViNjM2ODAyOTBmOGQyMzM1M2UwNWYwMzhiNTY3IiwiZXhwIjoxNTI2NDgyMjQ5LCJpYXQiOjE1MjY0ODIyNDksInN1YiI6ImFkbWluIiwidGVhbVJvbGVNYXBwaW5ncyI6W119.R0JBh0Z037oTjOLjla5egju8eRepPLCoXa5Mq3yUU9LB11953pj4EkepBpXdIS1fE2WtX994tiMpOIUE9WtbsQ7ay-N44CX57jKa5f_NNFeADEV5Ojtn5_gjCRbEv8AFfeT8ZJRFnGqwmZXxpVGjEGBUFnavAx-RERyJuKGl01PA6_2ZxXSmyWHzMh9HngwISpci8vAyIQS6rXepn9OXubLwIdvLztiPhNGHo-iTgk9BPmo5pChUIgXt5Y7s2omRemm5uhD6YpXdC0AyDkT4Tv-DQ5f_xXyp8GBKL5gFiS2qiDP7Hil7xp_EkZPQHCOCHL6odhBaFM0dhYup9aRv0A' 'https://172.16.40.4:8443/va/api/get-report?namespace=default/blue-web-c6c46447c-qm4tv/d65ec48faefe49115c8e86a59b438a525b9e8fe32905f9dce4c3b541d4d42180&access_group=default&source_type=container&report_type=vulnerability' | jq '.result.vulnerability.body.vulnerable_packages'
                if [ 1 -ne 0 ]; then
                    echo "There are 0 vulnerabilities detected in a container"
                else
                    echo "No vulnerabilities detected in a container"
                fi
                """
            }
        }
        container('kubectl') {
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
