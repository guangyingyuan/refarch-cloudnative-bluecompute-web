podTemplate(label: 'mypod',
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
        secretVolume(secretName: 'registry-account', mountPath: '/var/run/secrets/registry-account'),
        secretVolume(secretName: 'secret-token', mountPath: '/var/run/secrets/secret-token'),
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
            stage('va'){
                sh """
                #!/bin/bash
                TOKEN=`cat /var/run/secrets/secret-token/token`
                set +x
                CONTAINER=`default/jenkins-75d99f5657-pl6hs/d3ae07e93b34bdb8cd8a94dd6329c99e79ac0b8a0593a2537b2d0bd962a7a778`
                set -x
                echo 'set token and container'
                VULNERABILITIES=`curl -k -s -XGET -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhdF9oYXNoIjoidGV6ano2dDU0eHVsMDYwZGZvZnQiLCJyZWFsbU5hbWUiOiJjdXN0b21SZWFsbSIsInVuaXF1ZVNlY3VyaXR5TmFtZSI6ImFkbWluIiwiaXNzIjoiaHR0cHM6Ly9pY3Atc2UtZGV2LWNsdXN0ZXIuaWNwOjk0NDMvb2lkYy9lbmRwb2ludC9PUCIsImF1ZCI6IjhiZGViNjM2ODAyOTBmOGQyMzM1M2UwNWYwMzhiNTY3IiwiZXhwIjoxNTI2NTc0NDQ0LCJpYXQiOjE1MjY1NzQ0NDQsInN1YiI6ImFkbWluIiwidGVhbVJvbGVNYXBwaW5ncyI6W119.Qb6GvkHZGemU-86md9uOa37WDOBljvYUQe_b2brwaymZeX6s5z0Ti8j6DuQni7cM3BXwpDlOau0vxGmYTVL4NwkTPv2v5X1xJB21ecTqNd_0LNJMJRY3YjBy1jbG0gLiuarr0-Z13ajvfivD8rDl0pnjyoBMPGFjirZSntU9Zbv8koU8qmB458TqNWmXh2f5CEo73s7K_OGfZLbVUFZa3ESM5AtD9sEd5LL_QGL3GCbZJ36rTJMNQMRTenr8nHikF6PR39ciG792wccDEHKZs1bl7Bj6Rg5B114mfHQw1GoNNzW4ZqYk6OObfPNEtJwwaSSP8DpX1QCIDprCuUBWXw' 'https://172.16.40.4:8443/va/api/get-report?namespace=default/jenkins-75d99f5657-pl6hs/d3ae07e93b34bdb8cd8a94dd6329c99e79ac0b8a0593a2537b2d0bd962a7a778&access_group=default&source_type=container&report_type=vulnerability' | jq '.result.vulnerability.body.vulnerable_packages'`
                if [ \${VULNERABILITIES} -ne 0 ]; then
                    echo "There are \${VULNERABILITIES} vulnerabilities detected in \${CONTAINTER}"
                    exit 1
                else
                    echo "No vulnerabilities detected in \${CONTAINTER}"
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
