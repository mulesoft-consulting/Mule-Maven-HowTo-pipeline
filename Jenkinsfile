"""
NAME: Developer Pipeline
DESCRIPTION:
- Used by developer to build, test and deploy the specified application and branch. Uses linux bash shell.

REQUIRES:
- Standard Suggested Jenkins plugins
- Pipeline Utility Steps

"""

def logSeparator = "++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++"

def abort(String message)
{
    echo "Aborting pipeline..."
    currentBuild.result = 'ABORTED'
    error(message)
}

def log(String message)
{
    echo "++++++++++++++++++++++++++ [ ${message} ]"
}


pipeline {
    agent { label 'master' }

    environment {
        ANYPOINT = credentials('ANYPOINT_PLATFORM_CREDENTIALS')
        MULE_SETTINGS = '/Users/pdunworth/Documents/Customers/Peter.Dunworth/blogs-stuff/np-settings.xml'
        MVN = 'mvn'
        GIT = 'git'
    }

    stages {

        stage('Checkout Source') {
            steps {
                script {
			checkout([
				$class: 'GitSCM',
				branches: scm.branches,
				extensions: scm.extensions + [[$class: 'WipeWorkspace'], [$class: 'CleanCheckout'], [$class: 'LocalBranch']],
				userRemoteConfigs: scm.userRemoteConfigs
			])
                    sh "${GIT} status"
                    echo "Multi-branch set to: ${env.BRANCH_NAME}"
                    echo logSeparator
                }
           }
        }

        stage('Build and Test') {
            steps {
                sh "${MVN} clean install -Du=${ANYPOINT_USR} -Dp='${ANYPOINT_PSW}' --settings ${MULE_SETTINGS}"
                echo logSeparator
            }
        }

        stage('Deploy to CloudHub dev') {
            when {
                environment ignoreCase: true, name: 'BRANCH_NAME', value: 'develop'
            }
            steps {
                script {
                    pom = readMavenPom(file: 'pom.xml')
                    projectVersion = pom.getVersion()
                    projectArtifactId = pom.getArtifactId()
					
                    sh "${MVN} mule:deploy -Dmule.artifact=target/${projectArtifactId}-${projectVersion}-mule-application.jar -Pcloudhub -Ppublic-lb -Denv=dev -Du=${ANYPOINT_USR} -Dp='${ANYPOINT_PSW}' -DskipTests -Dch.workers=1 -Dch.workerType=MICRO --settings ${MULE_SETTINGS}"
                    echo logSeparator
		}
            }
        }

        stage('Publish Artifact') {
            when {
                environment ignoreCase: true, name: 'BRANCH_NAME', value: 'release'
            }
            steps {
            	script {
 	               pom = readMavenPom(file: 'pom.xml')
	               projectVersion = pom.getVersion()
	               projectArtifactId = pom.getArtifactId()
			        
                    try {
//withCredentials([sshUserPrivateKey(credentialsId: 'GIT_SERVICE_ACCOUNT', keyFileVariable: 'GIT_KEY_FILE', passphraseVariable: 'GIT_PASSPHRASE', usernameVariable: 'GIT_USER')]) {
                      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'GIT_SERVICE_ACCOUNT', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
                        sh "${GIT} tag v${projectVersion}"
                        sh "${GIT} config credential.username ${env.GIT_USERNAME}" 
                        sh "${GIT} config credential.helper '!echo password=\$GIT_PASSWORD; echo'" 
                        sh "GIT_ASKPASS=true ${git} push origin --tags"
                      }
                    } finally {
                        sh "${GIT} config --unset credential.username"
                        sh "${GIT} config --unset credential.helper"
                    }
             		sh "${MVN} clean install deploy -Pexchange -Du=${ANYPOINT_USR} -Dp='${ANYPOINT_PSW}' --settings ${MULE_SETTINGS}"
                }
                echo logSeparator
            }
        }

    }
}
