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
        stage('Prepare environment') {
            steps {
                echo "Running ${env.BUILD_ID} proyecto ${env.JOB_NAME} rama ${env.BRANCH_NAME}"
                sh 'echo "Versión Java instalada en el agente: $(java -version)'
                sh 'echo "Versión Maven instalada en el agente: $(mvn --version)'

            }
        }
        stage('Code promotion') {
            when { branch "master" }
            steps {
            echo '''En esta etapa se debe comprobar que la versión indicada en el fichero pom.xml no contiene el sufijo -SNAPSHOT.
De ser así, se debe modificar el fichero pom.xml, eliminando el sufijo.
Una vez hecho esto, se debe hacer commit del cambio y push a la rama master.
De esta forma, todos los artefactos generados en la rama master, no tendrán el sufijo SNAPSHOT.            
'''
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }

    }
}