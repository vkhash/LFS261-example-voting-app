pipeline {
  agent none
  environment {
    DISCORD_WEBHOOK_URL = credentials('DISCORD-WEBHOOK-URL')
  }
  stages {
    stage("worker-build") {
      when {
        changeset "**/worker/**"
      }
      agent {
        docker {
          image 'maven:3.9.12-sapmachine-21'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps {
        echo 'Compiling worker app'
        dir('worker') {
          sh 'mvn compile'
        }
      }
    }
    stage("worker-test") {
      when {
        changeset "**/worker/**"
      }
      agent {
        docker {
          image 'maven:3.9.12-sapmachine-21'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      steps {
        echo 'Running Unit Tests on worker app'
        dir('worker') {
          sh 'mvn clean test'
        }
      }
    }
    stage("worker-docker-package") {
      agent any
      when {
        branch 'master'
        changeset "**/worker/**"
      }
      steps {
        echo 'Packaging worker app with Docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'docker-vkhash') {
            def workerImage = docker.build("vkhash/private:worker-v${env.BUILD_ID}", "./worker")
            workerImage.push()
            workerImage.push("worker-latest")
          }
        }
      }
    }
    stage("result-build") {
      when {
        changeset "**/result/**"
      }
      agent {
        docker {
          image 'node:22.22.0-trixie-slim'
        }
      }
      steps {
        echo 'npm install packages for result app'
        dir('result') {
          sh 'npm install'
        }
      }
    }
    stage("result-test") {
      when {
        changeset "**/result/**"
      }
      agent {
        docker {
          image 'node:22.22.0-trixie-slim'
        }
      }
      steps {
        echo 'npm test result app'
        dir('result') {
          sh '''
            npm install
            npm test
          '''
        }
      }
    }
    stage("result-docker-package") {
      agent any
      when {
        branch 'master'
        changeset "**/result/**"
      }
      steps {
        echo 'Packaging result app with Docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'docker-vkhash') {
            def workerImage = docker.build("vkhash/private:result-v${env.BUILD_ID}", "./result")
            workerImage.push()
            workerImage.push("result-latest")
          }
        }
      }
    }
    stage('vote-build') {
      when {
        changeset "**/vote/**"
      }
      agent {
        docker {
          image 'python:3.11-slim'
          args '--user root'
        }
      }
      steps {
        echo 'Compiling vote app.'
        dir('vote') {
          sh "pip install -r requirements.txt"
        }
      }
    }
    stage('vote-test') {
      when {
        changeset "**/vote/**"
      }
      agent {
        docker {
          image 'python:3.11-slim'
          args '--user root'
        }
      }
      steps {
        echo 'Running Unit Tests on vote app'
        dir('vote') { 
          sh "pip install -r requirements.txt"
          sh 'nosetests -v'
        }
      }
    } 
    stage('vote-docker-package') {
      when {
        branch 'master'
        changeset "**/vote/**"
      }
      agent any
      steps {
        echo 'Packaging vote app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'docker-vkhash') {
            def voteImage = docker.build("vkhash/private:vote-v${env.BUILD_ID}", "./vote")
            voteImage.push()
            // voteImage.push("vote-${env.BRANCH_NAME}")
            voteImage.push("vote-latest")
          }
        }
      }
    }
    stage('Sonarqube') {
      agent any
      when {
        branch 'master'
      }
      environment{
        sonarpath = tool 'SonarQubeScanner'
      }
      steps {
        echo 'Running Sonarqube Analysis..'
        withSonarQubeEnv('sonar-qube-vkhash-instavote') {
          sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
        }
      }
    }
    stage("Quality Gate") {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
          // true = set pipeline to UNSTABLE, false = don't
          waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('deploy to dev') {
      agent any
      when {
        branch 'master'
      }
      steps {
        echo 'Deploy instavote app with docker compose'
        sh 'docker compose up -d'
      }
    }
  }
  post {
    always {
      echo 'Building multibranch pipeline for instavote app is completed'
      discordSend webhookURL: DISCORD_WEBHOOK_URL,
                  title: "${env.JOB_BASE_NAME} #${env.BUILD_NUMBER}",
                  result: currentBuild.currentResult,
                  description: "**App:** instavote\n**Build:** ${env.BUILD_NUMBER}\n",
                  enableArtifactsList: false,
                  successful: true,
                  showChangeset: true
    }
  }
}