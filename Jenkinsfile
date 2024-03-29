// Uses Declarative syntax to run commands inside a container.
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jdk11
    image: alledodev/jenkins-nodo-java-bootcamp:latest
    command:
    - sleep
    args:
    - infinity
  - name: nodejs
    image: alledodev/jenkins-nodo-nodejs-bootcamp:latest
    command:
    - sleep
    args:
    - infinity
  - name: imgkaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
'''
/*    volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
  volumes:
  - name: kaniko-secret
    secret:
      secretName: kaniko-secret
      optional: false
'''*/
            defaultContainer 'jdk11'
        }
    }
    parameters {
    choice(
      description: 'Ubicación, para saber en que Minikube se debe desplegar',
      name: 'ubicacion',
      choices: ['ofi', 'casa']
    )
    }
    environment {
		        APP_VERSION = ""

            }
    stages {
        //1
        stage('Prepare environment') {
            steps {
                echo '''01# Stage - Prepare environment
'''
                echo "Running ${env.BUILD_ID} proyecto ${env.JOB_NAME} rama ${env.BRANCH_NAME}"
                sh 'echo "Versión Java instalada en el agente: $(java -version)"'
                sh 'echo "Versión Maven instalada en el agente: $(mvn --version)"'
                script {
                def pom = readMavenPom(file: 'pom.xml')
                APP_VERSION = pom.version
                }
            }
        }
        //2
        stage('Code promotion') {
            when { branch "main" }
            steps {
            echo '''02# Stage - Code promotion
En esta etapa se debe comprobar que la versión indicada en el fichero pom.xml no contiene el sufijo -SNAPSHOT.
De ser así, se debe modificar el fichero pom.xml, eliminando el sufijo.
Una vez hecho esto, se debe hacer commit del cambio y push a la rama main.
De esta forma, todos los artefactos generados en la rama main, no tendrán el sufijo SNAPSHOT.
'''
                script{
                    pom = readMavenPom(file: 'pom.xml')
                    def pom_version = pom.version
                    pom.version = pom_version.replace('-SNAPSHOT','')
                    APP_VERSION = pom.version
                    writeMavenPom file: pomFile, model: pom
                    sh 'git add pom.xml'
                    sh "git commit -m \"Update pom.xml file version:${pom.version}\""
                    sh 'git push origin master'
                }
            }
        }
        //3
        stage('Compile') {
            steps {
                echo '''03# Stage - Compile
'''
                sh 'mvn clean compile -DskipTests'
            }
        }
        //4
        stage('Unit Tests') {
            steps {
            echo '''04# Stage - Unit Tests
(develop y main): Lanzamiento de test unitarios.
'''
                sh "mvn test"
                junit "target/surefire-reports/*.xml"
            }
        }
        //5
        stage('JaCoCo Tests') {
            steps {
            echo '''05# Stage - JaCoCo Tests
(develop y main): Lanzamiento de las pruebas con JaCoCo'
'''
                jacoco()
                step( [ $class: 'JacocoPublisher' ] )
            }
        }
        //6
        stage('Quality Tests 1 de 2') {
            steps {
            echo '''06# Stage - Quality Tests
            (develop y main): Comprobación de la calidad del código con Sonarqube.
'''
                withSonarQubeEnv(credentialsId: "sonarqube-credentials", installationName: "sonarqube-server"){
                    sh "mvn verify sonar:sonar -DskipTests"
                }
            }
        }
        stage('Quality Tests 2 de 2') {
            steps {
            echo '''06# Stage - Quality Tests
            (develop y main): Comprobación de la calidad del código con Sonarqube.
'''
                timeout(time: 10, unit: "MINUTES") {
                    script {
                        def qg = waitForQualityGate(webhookSecretId: 'sonarqube-credentials')
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }    
            }
        }
        //7
        stage('Package') {
            steps {
            echo '''07# Stage - Package
(develop y main): Generación del artefacto .jar (SNAPSHOT)
'''
                sh 'mvn package -DskipTests'
            }
        }
        //8
        stage('Build & Push') {
            steps {
            echo '''08# Stage - Build & Push
(develop y main): Construcción de la imagen con Kaniko y subida de la misma a repositorio personal en Docker Hub.
Para el etiquetado de la imagen se utilizará la versión del pom.xml
'''
                container('imgkaniko') {
                   
                    script {
                        def APP_IMAGE_NAME = "app-pf-backend"
                        def APP_IMAGE_TAG = APP_VERSION //Aqui hay que obtenerlo de POM.txt
                        withCredentials([usernamePassword(credentialsId: 'idCredencialesDockerHub', passwordVariable: 'idCredencialesDockerHub_PASS', usernameVariable: 'idCredencialesDockerHub_USER')]) {
                            AUTH = sh(script: """echo -n "${idCredencialesDockerHub_USER}:${idCredencialesDockerHub_PASS}" | base64""", returnStdout: true).trim()
                            command = """echo '{"auths": {"https://index.docker.io/v1/": {"auth": "${AUTH}"}}}' >> /kaniko/.docker/config.json"""
                            sh("""
                                set +x
                                ${command}
                                set -x
                                """)
                            sh "/kaniko/executor --dockerfile Dockerfile --context ./ --destination ${idCredencialesDockerHub_USER}/${APP_IMAGE_NAME}:${APP_IMAGE_TAG}"
                            sh "/kaniko/executor --dockerfile Dockerfile --context ./ --destination ${idCredencialesDockerHub_USER}/${APP_IMAGE_NAME}:latest --cleanup"
                        }
                    }
                } 
            }
        }
        //9
        stage('Run test environment') {
            steps {
            echo '''09# Stage - Run test environment
(develop y main): Iniciar un pod o contenedor con la imagen que acabamos de generar.
'''
                script {
                    if(fileExists('configuracion')){
                        sh 'rm -r configuracion'
                    }
                }
                sshagent (credentials: ['credencialGITHUB']) {
                    script {
                        if (params.ubicacion == 'casa') {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl apply -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n default --kubeconfig=configuracion/kubernetes-deployment/minikube/casa/config
                    '''
                        } else {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl apply -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n default --kubeconfig=configuracion/kubernetes-deployment/minikube/config
                    '''
                        }
                    }
                } 
            }
        }
        //10
        stage('API Test o Performance TestPackage') {
            steps {
            echo '''10# Stage - API Test o Performance TestPackage
(develop y main): Lanzar los test de JMeter o las pruebas de API con Newman.
'''
                container('nodejs') {
                    sh '''error=$(newman run ./src/main/resources/bootcamp.postman_collection.json --timeout 1500 --reporters cli,junit --reporter-junit-export "newman/report.xml")
if [ $? -eq 0 ]; then
   echo "Newman Test OK"
else
   echo "Problemas al pasar los Test con Newman. Error: $error"
fi
'''                    
                    junit "newman/report.xml"
                } 
            }
            post { 
                failure { 
                    echo "Problemas al pasar los Test con Newman."
                }
            }
        }
        //11
        stage('Nexus') {
            environment {
		        NEXUS_VERSION = "nexus3"
                NEXUS_PROTOCOL = "http"
                NEXUS_URL = "10.101.33.36:8081"
                NEXUS_REPOSITORY = "bootcamp/"
                NEXUS_CREDENTIAL_ID = "nexusidentity"
            }
            steps {
            echo '''11# Stage - Nexus
(develop y main): Si se ha llegado a esta etapa sin problemas:
Se deberá depositar el artefacto generado (.jar) en Nexus.(develop y main)
Generación del artefacto .jar (SNAPSHOT)
'''
            script {
                // Read POM xml file using 'readMavenPom' step , this step 'readMavenPom' is included in: https://plugins.jenkins.io/pipeline-utility-steps
                pom = readMavenPom file: "pom.xml"
                // Find built artifact under target folder
                filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                // Print some info from the artifact found
                echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                // Extract the path from the File found
                artifactPath = filesByGlob[0].path
                // Assign to a boolean response verifying If the artifact name exists
                artifactExists = fileExists artifactPath
                if(artifactExists) {
                    echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}"
                    versionPom = "${pom.version}"
                    nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL,
                        groupId: pom.groupId,
                        version: pom.version,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        artifacts: [
                            // Artifact generated such as .jar, .ear and .war files.
                            [artifactId: pom.artifactId,
                            classifier: "",
                            file: artifactPath,
                            type: pom.packaging],
                            // Lets upload the pom.xml file for additional information for Transitive dependencies
                            [artifactId: pom.artifactId,
                            classifier: "",
                            file: "pom.xml",
                            type: "pom"]
                        ]
                    )
                } else {
                        error "*** File: ${artifactPath}, could not be found"
                }
            }}
        }
        //12
        stage('Deploy') {
            when { branch "main" }
            steps {
            echo '''12# Stage - Deploy  (Stage opcional)
(main): En esta stage se debe desplegar en un pod.
La imagen generada en la etapa 8.
Para ello se deberá generar un Chart de Helm que contenga un ConfigMap y un Pod con dicha imagen
'''              
                script {
                    if(fileExists('configuracion')){
                        sh 'rm -r configuracion'
                    }
                }
                sshagent (credentials: ['credencialGITHUB']) {
                    script {
                        if (params.ubicacion == 'casa') {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl apply -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n producion --kubeconfig=configuracion/kubernetes-deployment/minikube/casa/config
                    '''
                        } else {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl apply -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n produccion --kubeconfig=configuracion/kubernetes-deployment/minikube/config
                    '''
                        }
                    }
                }             
                
            }
        }
    }
    //13
    post { 
        always {            
            echo '''No es una stage como tal sino un bloque post:
Que elimine siempre los recursos creados en la Stage 8.
'''
                            script {
                    if(fileExists('configuracion')){
                        sh 'rm -r configuracion'
                    }
                }
                sshagent (credentials: ['credencialGITHUB']) {
                    script {
                        if (params.ubicacion == 'casa') {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl delete -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n default --kubeconfig=configuracion/kubernetes-deployment/minikube/casa/config
                    '''
                        } else {
                        sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa github.com >> ~/.ssh/known_hosts
                    git clone git@github.com:antoniollv/deploy-to-k8s-conf.git configuracion --branch main
                    kubectl delete -f configuracion/kubernetes-deployment/spring-boot-app/manifest.yaml -n default --kubeconfig=configuracion/kubernetes-deployment/minikube/config
                    '''
                        }
                    }
                }
        }
    }
}
