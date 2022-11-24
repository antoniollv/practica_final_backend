// Uses Declarative syntax to run commands inside a container.
pipeline {
    agent {
        kubernetes {
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
            steps {
                echo "Running ${env.BUILD_ID} proyecto ${env.JOB_NAME} rama ${env.BRANCH_NAME}"
                sh 'echo "Versión Java instalada en el agente: $(java -version)"'
                sh 'echo "Versión Maven instalada en el agente: $(mvn --version)"'

            }
        }
        //2
        stage('Code promotion') {
            when { branch "main" }
            steps {
            echo '''En esta etapa se debe comprobar que la versión indicada en el fichero pom.xml no contiene el sufijo -SNAPSHOT.
De ser así, se debe modificar el fichero pom.xml, eliminando el sufijo.
Una vez hecho esto, se debe hacer commit del cambio y push a la rama main.
De esta forma, todos los artefactos generados en la rama main, no tendrán el sufijo SNAPSHOT.
'''
            }
        }
        //3
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        //4
        stage('Unit Tests') {
            steps {
            echo '''(develop y main): Lanzamiento de test unitarios.
'''
            }
        }
        //5
        stage('JaCoCo Tests') {
            steps {
            echo '''(develop y main): Lanzamiento de las pruebas con JaCoCo'
'''
            }
        }
        //6
        stage('Quality Tests') {
            steps {
            echo '''(develop y main): Comprobación de la calidad del código con Sonarqube.
'''
            }
        }
        //7
        stage('Package') {
            steps {
            echo '''(develop y main): Generación del artefacto .jar (SNAPSHOT)
'''
            }
        }
        //8
        stage('Build & Push') {
            steps {
            echo '''(develop y main): Construcción de la imagen con Kaniko y subida de la misma a vuestro repositorio personal en Docker Hub.
Ppara el etiquetado de la imagen se utilizará la versión del pom.xml
'''
            }
        }
        //9
        stage('Run test environment') {
            steps {
            echo '''(develop y main): Iniciar un pod o contenedor con la imagen que acabamos de generar.
'''
            }
        }
        //10
        stage('API Test o Performance TestPackage') {
            steps {
            echo '''(develop y main): Lanzar los test de JMeter o las pruebas de API con Newman.
'''
            }
        }
        //11
        stage('Nexus') {
            steps {
            echo '''(develop y main): Si se ha llegado a esta etapa sin problemas:
Se deberá depositar el artefacto generado (.jar) en Nexus.(develop y main)
Generación del artefacto .jar (SNAPSHOT)
'''
            }
        }
        //12
        stage('Deploy') {
            when { branch "main" }
            steps {
            echo '''(main): En esta stage se debe desplegar en un pod.
La imagen generada en la etapa 8.
Para ello se deberá generar un Chart de Helm como los vistos en clase que contenga un ConfigMap y un Pod con dicha imagen (stage opcional)
'''
            }
        }
    }
    //13
    post { 
        always { 
            echo '''No es una stage como tal sino un bloque post:
Que elimine siempre los recursos creados en la stage 8.
'''
        }
    }
}
