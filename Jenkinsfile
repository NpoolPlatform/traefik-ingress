pipeline {
  agent any
  environment {
    GOPROXY = 'https://goproxy.cn,direct'
  }
  tools {
    go 'go'
  }
  stages {
    stage('Clone traefik ingress') {
      steps {
        git(url: scm.userRemoteConfigs[0].url, branch: '$BRANCH_NAME', changelog: true, credentialsId: 'KK-github-key', poll: true)
      }
    }

    stage('Build traefik') {
      when {
        expression { BUILD_TARGET == 'true' }
      }
      steps {
        sh 'rm .traefik -rf'
        sh 'git clone https://github.com/NpoolPlatform/traefik.git .traefik; cd .traefik; git checkout entropy-v2.5.3'
        sh 'cp Makefile.service .traefik/Makefile'
        sh 'cp build.Dockerfile.service .traefik/build.Dockerfile'
        sh 'cd .traefik; mkdir -p v2; cp * v2 -rf | true; rm -rf v2/v2; make generate-crd'
        sh 'cd .traefik; make traefik-binary'
        sh 'mkdir -p .traefik-release'
        sh 'cp .traefik/dist/traefik .traefik-release'
        sh 'cp entrypoint.sh .traefik-release'
        sh 'cp .traefik/script/ca-certificates.crt .traefik-release'
        sh 'cp Dockerfile.service .traefik-release/Dockerfile'
        sh(returnStdout: true, script: '''
          set +e
          docker images | grep entropypool | grep traefik-service
          rc=$?
          set -e
          if [ 0 -eq $rc ]; then
            docker rmi uhub.service.ucloud.cn/entropypool/traefik-service:v2.5.3.6 | true
          fi
        '''.stripIndent())
        sh 'cd .traefik-release; docker build -t uhub.service.ucloud.cn/entropypool/traefik-service:v2.5.3.6 .'

        nodejs('nodejs') {
          sh 'cd .traefik/webui; npm install'
          sh 'cd .traefik/webui; NODE_ENV=production APP_ENV=production PLATFORM_URL=http://traefik-webui.internal-devops.$TARGET_ENV.npool.top/traefik/dashboard APP_API=http://traefik-api.internal-devops.$TARGET_ENV.npool.top/traefik/api APP_PUBLIC_PATH=traefik/dashboard npm run build-quasar'
          sh 'mkdir -p .webui/static; cp .traefik/webui/dist/spa/* .webui/static -rf'
          sh 'cp Dockerfile.webui .webui/Dockerfile'
          sh 'cp nginx.conf.template .webui/nginx.conf.template'
          sh(returnStdout: true, script: '''
            set +e
            docker images | grep entropypool | grep traefik-webui-$TARGET_ENV
            rc=$?
            set -e
            if [ 0 -eq $rc ]; then
              docker rmi uhub.service.ucloud.cn/entropypool/traefik-webui-$TARGET_ENV:v2.5.3.6 | true
              docker rmi entropypool/traefik-webui-$TARGET_ENV:v2.5.3.6 | true
            fi
          '''.stripIndent())
          sh 'cd .webui; docker build -t uhub.service.ucloud.cn/entropypool/traefik-webui-$TARGET_ENV:v2.5.3.6 .'
        }
      }
    }

    stage('Push docker image') {
      when {
        expression { RELEASE_TARGET == 'true' }
      }
      steps {
        sh(returnStdout: true, script: '''
          set +e
          while true; do
            docker push uhub.service.ucloud.cn/entropypool/traefik-service:v2.5.3.6
            if [ $? -eq 0 ]; then
              break
            fi
          done

          while true; do
            docker push uhub.service.ucloud.cn/entropypool/traefik-webui-$TARGET_ENV:v2.5.3.6
            if [ $? -eq 0 ]; then
              break
            fi
          done
          set -e
        '''.stripIndent())
      }
    }

    stage('Deploy traefik') {
      when {
        expression { DEPLOY_TARGET == 'true' }
      }

      steps {
        sh 'sed -i "s/internal-devops.development.npool.top/$TARGET_ENV.npool.top/g" k8s/08-traefik-dashboard-ingress.yaml'
        sh 'sed -i "s/traefik-webui-development:v2.5.3.6/traefik-webui-$TARGET_ENV:v2.5.3.6/g" k8s/04-deployments.yaml'
        sh 'cd /etc/kubeasz; ./ezctl checkout $TARGET_ENV'
        sh 'kubectl apply -f k8s/01-ingress.yaml'
        sh 'kubectl apply -f k8s/02-services.yaml'
        sh 'kubectl apply -f k8s/03-service-account.yaml'
        sh 'kubectl apply -f k8s/04-deployments.yaml'
        sh 'kubectl apply -f k8s/05-services-vpn.yaml'
        sh 'kubectl apply -f k8s/06-deployments-vpn.yaml'
        sh 'kubectl apply -f k8s/07-middlewares.yaml'
        sh 'kubectl apply -f k8s/08-traefik-dashboard-ingress.yaml'
      }
    }

  }

  post('Report') {
    fixed {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh fixed')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    success {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh successful')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/success_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    failure {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh failure')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
    aborted {
      script {
        sh(script: 'bash $JENKINS_HOME/wechat-templates/send_wxmsg.sh aborted')
     }
      script {
        // env.ForEmailPlugin = env.WORKSPACE
        emailext attachmentsPattern: 'TestResults\\*.trx',
        body: '${FILE,path="$JENKINS_HOME/email-templates/fail_email_tmp.html"}',
        mimeType: 'text/html',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: '$DEFAULT_RECIPIENTS'
      }
     }
  }
}
