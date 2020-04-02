podTemplate(
  label: 'jenkins-slave',
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
      [key: 'repo_url', value: '\$.repository.git_url'],
      [key: 'repo_files_added', value: '\$.commits.added'],
      [key: 'repo_files_removed', value: '\$.commits.removed'],
      [key: 'repo_files_modified', value: '\$.commits.modified']
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
    stage('Docker') {
      container ('docker') {
        steps {
          script {
            def files_changed = ('$repo_files_added' + ' ' + '$repo_files_modified').flatten().join(' ')
            def files_removed = '$repo_files_removed'.join(' ')
            def dir_changed = sh(script: 'dirname $files_changed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()
            def dir_removed = sh(script: 'dirname $files_removed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()

            // While app dir exists
            if (! (files_removed.equals(dir_removed))) {
              dir_changed = ('$dir_changed' + ' ' + '$dir_removed')
              dir_changed = sh(script: 'dirname $dir_changed | cut -d\'/\' -f1-2|uniq', returnStdout: true).trim()
            }

            // For any Dockerfile app dir that exists
            dir_changed.each { dir_name ->
              def project_dir_name = System.properties['user.dir'].split('/')[-1]
              def repository = 'ranhershko/${project_dir_name}-${dir_name}.split(\'/\')'
              dir(dir_name) {
                if (fileExists('Dockerfile')) {
                  sh 'docker build -t ${repository}:${BUILD_NUMBER} .'
                  sh 'docker tag ${repository}:latest ${repository}:${BUILD_NUMBER}'
                  sh 'docker push ${repository}:${BUILD_NUMBER}'
                  sh 'docker push ${repository}:latest'
                }
              }
            }
            def deployment_files = sh(script: 'ls |grep "^[0-9].*.yml$"|sort', returnStdout: true).trim()
            deployment_files.each { deployment_file ->
              kubernetesDeploy(configs: '${deployment_file}', kubeconfigId: "mykubeconfig")
          }
        }
      }
    }
    stage('verify dockers images') {
      steps {
        sh 'docker images'
      }
    }
  }
}
