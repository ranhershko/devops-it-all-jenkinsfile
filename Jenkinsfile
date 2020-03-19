pipeline {
  podTemplate(
    label 'jenkins-slave',
    inheritFrom: 'default',
    containers: [
      containerTemplate(
        name: 'docker', 
        image: 'docker:18.02',
        ttyEnabled: true,
        command: 'cat'
      )
    ],
    volumes: [
      hostPathVolume(
        hostPath: '/var/run/docker.sock',
        mountPath: '/var/run/docker.sock'
      )
    ]
  )
  triggers {
    GenericTrigger(
      genericVariables: [
        [key: 'repo_url', value: '$.repository.git_url'],
        [key: 'repo_files_added', value: '$.commits.added'],
        [key: 'repo_files_removed', value: '$.commits.removed'],
        [key: 'repo_files_modified', value: '$.commits.modified']
      ]
    )
  }
  node('jenkins-slave') {
    stages {
      stage('Checkout') {
        steps {
          git credentialsId: 'ran.hershko', url: '$repo_url'
        }
      }
      stage("Docker") {
        container ('docker') {
          steps {
            script {
              def changed = ('$repo_files_added' + ' ' + '$repo_files_modified').flatten().join(' ') 
              def removed = '$repo_files_removed'.join(' ') 
              def dir_changed = sh(script: 'dirname $changed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()
              def dir_removed = sh(script: 'dirname $removed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()
                        
              // While app dir exists
              if not removed.equals(dir_removed) {
                def dir_changed = ('$dir_changed' + ' ' + '$dir_removed')
                def dir_changed = sh(script: 'dirname $changed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()
              }
              // For any Dockerfile app dir that exists
              dir_changed.each { dir_name ->
                def repository = '${dir_name}.split(\'/\')'
                dir(dir_name) {
                  if (fileExists('Dockerfile')) {
                    sh "docker build -t ${repository}:${BUILD_NUMBER} ."
                    sh "docker tag ${repository}:latest ${repository}:${BUILD_NUMBER}"
                    sh "docker push ${repository}::${BUILD_NUMBER}"
                    sh "docker push ${repository}:latest"
                  }
                }
              }
            } 
          }
        }
      }
      stage("verify dockers") {
        steps {
          sh "docker images"
        }
      }
    }     
  }
}
