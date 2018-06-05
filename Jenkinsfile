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
                
                set +x
                TOKEN=`cat /var/run/secrets/secret-token/token`
                CONTAINER='default/blue-orders-mysql-794b456cc8-2w45w/3a64e5f74bffdb98e0f305b80597df66b70317ffccb580c6c77c3175f0b77c8'
                VULNERABILITIES=`curl -k -s -XGET -H "Authorization: Bearer \${TOKEN}" "https://172.16.40.4:8443/va/api/get-report?namespace=\${CONTAINER}&access_group=default&source_type=container&report_type=vulnerability" | jq '.result.vulnerability.body.vulnerable_packages'`

                echo \${VULNERABILITIES}
                if [ \${VULNERABILITIES} -ne 0 ]; then
                    echo "There are \${VULNERABILITIES} vulnerabilities detected in \${CONTAINTER}"
                    exit 1
                else
                    echo "No vulnerabilities detected in CONTAINTER"
                fi
                set -x
                echo \${VULNERABILITIES}
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
