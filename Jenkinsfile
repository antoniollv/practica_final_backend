// Uses Declarative syntax to run commands inside a container.
pipeline {
    agent {
        kubernetes {
            label 'default'
            // Rather than inline YAML, in a multibranch Pipeline you could use: yamlFile 'jenkins-pod.yaml'
            // Or, to avoid YAML:
            // containerTemplate {
            //     name 'shell'
            //     image 'ubuntu'
            //     command 'sleep'
            //     args 'infinity'
            // }
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: alledodev/jenkins-nodo-java-bootcamp:latest
    command:
    - sleep
    args:
    - infinity
'''
            // Can also wrap individual steps:
            // container('shell') {
            //     sh 'hostname'
            // }
            defaultContainer 'shell'
        }
    }
    stages {
        //1
        stage('Prepare environment') {
            agent { 
                label 'default'
            }
            steps {
                echo '''01# Stage - Prepare environment
'''
                echo "Running ${env.BUILD_ID} proyecto ${env.JOB_NAME} rama ${env.BRANCH_NAME}"
                sh 'echo "Versión Java instalada en el agente: $(java -version)"'
                sh 'echo "Versión Maven instalada en el agente: $(mvn --version)"'

            }
        }
        //2
        stage('Code promotion') {
            agent { 
                label 'default'
            }
            when { branch "main" }
            steps {
            echo '''02# Stage - Code promotion
En esta etapa se debe comprobar que la versión indicada en el fichero pom.xml no contiene el sufijo -SNAPSHOT.
De ser así, se debe modificar el fichero pom.xml, eliminando el sufijo.
Una vez hecho esto, se debe hacer commit del cambio y push a la rama main.
De esta forma, todos los artefactos generados en la rama main, no tendrán el sufijo SNAPSHOT.
'''
            }
        }
        //3
        stage('Compile') {
            agent { 
                label 'default'
            }
            steps {
                echo '''03# Stage - Compile
'''
                sh 'mvn clean compile'
            }
        }
        //4
        stage('Unit Tests') {
            agent { 
                label 'default'
            }
            steps {
            echo '''04# Stage - Unit Tests
(develop y main): Lanzamiento de test unitarios.
'''
            }
        }
        //5
        stage('JaCoCo Tests') {
            agent { 
                label 'default'
            }
            steps {
            echo '''05# Stage - JaCoCo Tests
(develop y main): Lanzamiento de las pruebas con JaCoCo'
'''
            }
        }
        //6
        stage('Quality Tests') {
            agent { 
                label 'default'
            }
            steps {
            echo '''06# Stage - Quality Tests
            (develop y main): Comprobación de la calidad del código con Sonarqube.
'''
            }
        }
        //7
        stage('Package') {
            agent { 
                label 'default'
            }
            steps {
            echo '''07# Stage - Package
(develop y main): Generación del artefacto .jar (SNAPSHOT)
'''
                sh 'mvn package'
            }
        }
        //8
        stage('Build & Push') {
            agent { 
                kubernetes {
                    label 'kaniko'
                    yaml """
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
  - name: builder
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: kaniko-secret
        mountPath: /kaniko/.docker
  volumes:
  - name: kaniko-secret
    secret:
      secretName: kaniko-secret
      optional: false
"""
                    }
                }
            }
            steps {
            echo '''08# Stage - Build & Push
(develop y main): Construcción de la imagen con Kaniko y subida de la misma a vuestro repositorio personal en Docker Hub.
Para el etiquetado de la imagen se utilizará la versión del pom.xml
'''
                sh '/kaniko/executor --dockerfile $(pwd)/Dockerfile --context $(pwd) \
                --destinatión=alledodev/app-pf-backend:1.0'
            }
        }
        //9
        stage('Run test environment') {
            agent { 
                label 'default'
            }
            steps {
            echo '''09# Stage - Run test environment
(develop y main): Iniciar un pod o contenedor con la imagen que acabamos de generar.
'''
            }
        }
        //10
        stage('API Test o Performance TestPackage') {
            agent { 
                label 'default'
            }
            steps {
            
            echo '''10# Stage - API Test o Performance TestPackage
(develop y main): Lanzar los test de JMeter o las pruebas de API con Newman.
'''
            }
        }
        //11
        stage('Nexus') {
            agent { 
                label 'default'
            }
            steps {
            echo '''11# Stage - Nexus
(develop y main): Si se ha llegado a esta etapa sin problemas:
Se deberá depositar el artefacto generado (.jar) en Nexus.(develop y main)
Generación del artefacto .jar (SNAPSHOT)
'''
            }
        }
        //12
        stage('Deploy') {
            agent { 
                label 'default'
            }
            when { branch "main" }
            steps {
            echo '''12# Stage - Deploy
(main): En esta stage se debe desplegar en un pod.
La imagen generada en la etapa 8.
Para ello se deberá generar un Chart de Helm como los vistos en clase que contenga un ConfigMap y un Pod con dicha imagen (stage opcional)
'''
            }
        }
    }
    //13
    post { 
        always { 
            agent { 
                label 'default'
            }
            echo '''No es una stage como tal sino un bloque post:
Que elimine siempre los recursos creados en la stage 8.
'''
        }
    }
}
