  // Docker parameters
  def dockerCredentials = 'docker_test_credentials'
  def dockerRegistry = ''
  def dockerImageName = 'product-service'
  def dockerImageNamespace = 'huertalv'
  def mavenArgs = [maven: 'maven-3.5.0']

  // Project parameters
  def version = 'unspecified'

  // SCM parameters
  def defaultBranch = 'main'
  def branchName
  pipeline {

      agent none // Force to specify a valid node in each stage.

      options {
        skipDefaultCheckout true
      }

      stages {

        stage('Prepare') {
          agent {
            node { label 'build && java' }
          }
          steps {
            script {
              checkout scm
              stash includes: '**', name: 'source', useDefaultExcludes: false
              branchName = env.BRANCH_NAME ?: sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()

              final matcher = readFile('pom.xml') =~ "<version>(.+)</version>"
              version = matcher ? matcher[1][1] : null
            }
          }
        }

        stage('Versioning') {
          agent {
            node { label 'build && java' }
          }
          when {
            expression { branchName == defaultBranch }
          }
          steps {
            script {
              unstash 'source'
              withMaven(mavenArgs) {
                sh "mvn -DnewVersion=${version} versions:set"
              }
            }
          }
        }

        stage('Compile and Test') {
          agent {
            node { label 'build && java' }
          }
          steps {
            script {
              unstash 'source'
              withMaven(mavenArgs) {
                sh "mvn clean compile compiler:testCompile test"
              }
            }
          }
        }

        stage('Push Docker Artifacts') {
          agent {
            node { label 'docker' }
          }
          when {
            expression { branchName == defaultBranch }
          }
          steps {
            script {
              unstash 'source'
              withCredentials([usernamePassword(credentialsId: dockerCredentials, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                sh "docker login -u ${DOCKER_USER} -p '${DOCKER_PASS}' ${dockerRegistry}"
                final String dockerFullImageName = "${dockerImageNamespace}/${dockerImageName}:${version}"
                sh "docker build -t ${dockerFullImageName} ."
                sh "docker push ${dockerFullImageName}"
                sh "docker rmi ${dockerFullImageName}"
                sh "docker logout ${dockerRegistry}"
              }
            }
          }
        }

      }
  }