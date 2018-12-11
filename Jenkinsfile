@Library('dynatrace@master') _

def tagMatchRules = [
  [
    meTypes: [
      [meType: 'SERVICE']
    ],
    tags : [
      [context: 'CONTEXTLESS', key: 'app', value: 'payment'],
      [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
    ]
  ]
]

pipeline {
  agent {
    label 'golang2'
  }
  environment {
    APP_NAME = "payment"
    ARTEFACT_ID = "sockshop/" + "${env.APP_NAME}"
    VERSION = readFile('version').trim()
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/library/${env.ARTEFACT_ID}"
    TAG_DEV = "${env.TAG}-${env.VERSION}-${env.BUILD_NUMBER}"
    TAG_STAGING = "${env.TAG}-${env.VERSION}"
  }
  stages {
    stage('Go build') {
      steps {
        checkout scm
        container('gobuilder') {
          sh '''
            export GOPATH=$PWD

            mkdir -p src/github.com/dynatrace-sockshop/payment/
            cp -R ./api src/github.com/dynatrace-sockshop/payment/
            cp -R ./main.go src/github.com/dynatrace-sockshop/payment/
            cp -R ./glide.* src/github.com/dynatrace-sockshop/payment/
            cd src/github.com/dynatrace-sockshop/payment
            
            glide install
            go build -a -ldflags -linkmode=external -installsuffix cgo -o $GOPATH/payment main.go
          '''
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker build -t ${env.TAG_DEV} ."
        }
      }
    }
    stage('Docker push to registry'){
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('docker') {
          sh "docker push ${env.TAG_DEV}"
        }
      }
    }
    stage('Deploy to dev namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('kubectl') {
          sh "sed -i 's#image: .*#image: ${env.TAG_DEV}#' manifest/payment.yml"
          sh "kubectl -n dev apply -f manifest/payment.yml"
        }
      }
    }
    stage('DT Deploy Event') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
        container("curl") {
          script {
            def status = pushDynatraceDeploymentEvent (
              tagRule : tagMatchRules
            )
          }
        }
      }
    }
    stage('Run health check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        echo "Waiting for the service to start..."
        sleep 60

        container('jmeter') {
          script {
            def status = executeJMeter ( 
              scriptName: 'jmeter/basiccheck.jmx',
              resultsDir: "HealthCheck_${env.APP_NAME}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "HealthCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Health check in dev failed."
            }
          }
        }
      }
    }
    stage('Run functional check in dev') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        container('jmeter') {
          script {
            def status = executeJMeter (  
              scriptName: "jmeter/${env.APP_NAME}_load.jmx",
              resultsDir: "FuncCheck_${env.APP_NAME}",
              serverUrl: "${env.APP_NAME}.dev", 
              serverPort: 80,
              checkPath: '/health',
              vuCount: 1,
              loopCount: 1,
              LTN: "FuncCheck_${BUILD_NUMBER}",
              funcValidation: true,
              avgRtValidation: 0
            )
            if (status != 0) {
              currentBuild.result = 'FAILED'
              error "Functional check in dev failed."
            }
          }
        }
      }
    }
    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        container('docker'){
          sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
          sh "docker push ${env.TAG_STAGING}"
        }
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}")
          ]
      }
    }
  }
}
