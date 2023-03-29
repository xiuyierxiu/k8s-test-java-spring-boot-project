pipeline {
  agent {
    kubernetes {
      cloud 'study-kubernetes'
      slaveConnectTimeout 1200
	  workspaceVolume hostPathWorkspaceVolume(hostPath: "/opt/workspace", readOnly: false) 
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
      image: 'registry.cn-beijing.aliyuncs.com/citools/jnlp:alpine'
      name: jnlp
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "localtime"
          readOnly: false  
    - command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-beijing.aliyuncs.com/citools/maven:3.5.3"
      imagePullPolicy: "IfNotPresent"
      name: "build"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "localtime"
          readOnly: false
        - mountPath: "/root/.m2/"
          name: "cachedir"
          readOnly: false
    - command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
      image: "registry.cn-beijing.aliyuncs.com/citools/docker:19.03.9-git"
      imagePullPolicy: "IfNotPresent"
      name: "docker"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "localtime"
          readOnly: false
        - mountPath: "/var/run/docker.sock"
          name: "dockersock"
          readOnly: false
      image: "registry.cn-beijing.aliyuncs.com/citools/kubectl:self-1.17"
      imagePullPolicy: "IfNotPresent"
      name: "kubectl"
      tty: true
      volumeMounts:
        - mountPath: "/etc/localtime"
          name: "localtime"
          readOnly: false
    - command:
        - "cat"
      env:
        - name: "LANGUAGE"
          value: "en_US:en"
        - name: "LC_ALL"
          value: "en_US.UTF-8"
        - name: "LANG"
          value: "en_US.UTF-8"
  restartPolicy: "Never"
  nodeSelector:
    build: "true"
  securityContext: {}
  volumes:
    - hostPath:
        path: "/var/run/docker.sock"
      name: "dockersock"
    - hostPath:
        path: "/usr/share/zoneinfo/Asia/Shanghai"
      name: "localtime"
    - name: "cachedir"
      hostPath:
        path: "/opt/m2"
'''	
	}
}

  stages {
    stage('pulling Code') {
      parallel {
        stage('pulling Code by jenkins') {
          when {
            expression {
              env.gitlabBranch == null
            }
          }
          steps {
            git(changelog: true, url: 'git@github.com:xiuyierxiu/k8s-test-java-spring-boot-project.git', branch: "${BRANCH}", credentialsId: 'git-key')
			script {
				COMMIT_ID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
				TAG = BUILD_TAG + '-' + COMMIT_ID
				println "Current branch is ${BRANCH}, Commit ID is ${COMMIT_ID}, Image TAG is ${TAG}"
			}
          }
        }

        stage('pulling Code by trigger') {
          when {
            expression {
              env.gitlabBranch != null
            }
          }
          steps {
            git(url: 'git@github.com:xiuyierxiu/k8s-test-java-spring-boot-project.git', branch: env.gitlabBranch, changelog: true, poll: true, credentialsId: 'gitlab-key')
			script {
				COMMIT_ID = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
				TAG = BUILD_TAG + '-' + COMMIT_ID
				println "Current branch is ${BRANCH}, Commit ID is ${COMMIT_ID}, Image TAG is ${TAG}"
			}
          }
        }

      }
    }

    stage('Building') {
      parallel {
        stage('Building') {
          steps {
            container(name: 'build') {
            sh """
				curl repo.maven.apache.org 
				mvn clean install -DskipTests 
				ls target/*
			"""
            }

          }
        }

        stage('Scan Code') {
          steps {
            sh 'echo "Scan Code"'
          }
        }

      }
    }

    stage('Docker build for creating image') {
	   environment {
	   	HARBOR_USER	= credentials('HARBOR_USERNAME_TX')
	   }
	   
		steps {
			container(name: 'docker') { 
				sh """
				echo ${HARBOR_USER_USR} ${HARBOR_USER_PSW} ${TAG}
				docker build -t ${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} . 
				docker login -u	${HARBOR_USER_USR}	-p	${HARBOR_USER_PSW} ${HARBOR_ADDRESS}
				docker push ${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} 
				"""
			}
		}

      }

    stage('Deploying to K8s') {
      environment {
      	MY_KUBECONFIG = credentials('study-k8s-kubeconfig')
      }
		steps {
			container(name: 'kubectl'){ 
				sh """
				/usr/local/bin/kubectl --kubeconfig	$MY_KUBECONFIG	set	image deploy -l app=${IMAGE_NAME} ${IMAGE_NAME}=${HARBOR_ADDRESS}/${REGISTRY_DIR}/${IMAGE_NAME}:${TAG} -n $NAMESPACE
				"""
			}
		}
    }

	environment {
		CommitID = ''
		HARBOR_ADDRESS = "43.139.169.26"
		REGISTRY_DIR = "kubernetes" 
		IMAGE_NAME = "k8s-test-java-spring-boot-project" 
		NAMESPACE = "kubernetes"
		TAG = ""
	}

	parameters {
		gitParameter(branch: '', branchFilter: 'origin/(.*)', defaultValue: '', description: 'Branch for build and deploy', name: 'BRANCH', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'PT_BRANCH')
	}
}
}
