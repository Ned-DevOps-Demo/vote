pipeline {
  agent {
    node {
      label 'base'
    }
  }

  environment {
    // 镜像仓库地址
    REGISTRY_URL          = '192.168.157.92'
    // harbor 访问凭证
    REGISTRY_CERT         = 'harbor'
    // harbor 项目
    HARBOR_PROJECT        = 'demo'
    // 镜像名称
    HARBOR_IMAGE_NAME     = 'vote'
    // 源码地址
    SOURCE_URL            = 'http://192.168.157.90:9090/devops/vote.git'
    // 源码凭证
    SOURCE_CERT           = 'gitlab'
    // helm地址
    HELM_URL              = 'http://192.168.157.90:9090/devops/production-helm-chart.git'
    // helm repo
    HELM_REPO             = '192.168.157.90:9090/devops/production-helm-chart.git'
    // docker build 的执行上下文
    DOCKER_CONTEXT        = "."
    // dockerfile 的文件名与所在位置
    DOCKERFILE            = "Dockerfile"
  }

  stages {
    stage('Git Clone') {
      steps {
        container('base'){
          git(branch: "${SOURCE_BRANCH}", credentialsId: "${SOURCE_CERT}", url: "${SOURCE_URL}")
        }
      }
    }

    stage('Docker Build & Push') {
      agent none
      steps {
        container('base') {
          withCredentials([usernamePassword(credentialsId:"${REGISTRY_CERT}", passwordVariable:'password', usernameVariable:'username')]) {
            script {
              sh '''
              git config --global --add safe.directory '*'
              '''
              def commit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
              def tag = sh(returnStdout: true, script: 'git describe --tags || true').trim()
              env.GIT_COMMIT = commit
              env.GIT_TAG = tag
              def gitLog = sh(returnStdout: true, script: 'git log -1 --pretty=format:"- **%h** | %ad | %s" --date=short ${GIT_TAG}').trim()

              env.GIT_LOG = gitLog
              echo "git info: ${GIT_LOG}"
              env.IMG = "${REGISTRY_URL}/${HARBOR_PROJECT}/${HARBOR_IMAGE_NAME}:${SOURCE_BRANCH}-${GIT_COMMIT}"

              sh '''
              set -ex
              docker build -f ${DOCKERFILE} ${DOCKER_CONTEXT} -t ${IMG} --tls-verify=false
              docker login ${REGISTRY_URL} -u $username -p $password --tls-verify=false
              docker push $IMG --tls-verify=false
              '''
            }
          }
        }
      }
    }

    stage('Update Prod env') {
      steps {
        container('base') {
          dir("${WORKSPACE}/production-helm-chart") {
            // 克隆 Helm Chart 仓库
            git(branch: 'main', credentialsId: "${SOURCE_CERT}", url: "${HELM_URL}")
            withCredentials([usernamePassword(credentialsId:"${SOURCE_CERT}", passwordVariable:'password', usernameVariable:'username')]) {
              script {
                sh '''
                  ls
                  yq eval '.spec.template.spec.containers[0].image = env(IMG)' -i app1/templates/vote-deployment.yaml
                  git config user.name "CI"
                  git config user.email "CI@DCE5.io"

                  git add app1/templates/vote-deployment.yaml
                  # 检查是否有本地修改
                  git diff --quiet && git diff --staged --quiet || git commit -m "update prod env app1 image tag to ${IMG}"

                  git push "http://${username}:${password}@${HELM_REPO}"
                '''
              }
            }
          }
        }
      }
    }
  }
}