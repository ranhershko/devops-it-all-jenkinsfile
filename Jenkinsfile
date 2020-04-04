podTemplate(label: 'jenkins-slave', inheritFrom: 'default', containers: [
    containerTemplate( name: 'docker', image: 'docker:18.02', ttyEnabled: true, command: 'cat')
],
volumes: [
    hostPathVolume( hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
  ]
){
  node('jenkins-slave') {
    properties([
      pipelineTriggers([
        [$class: 'GenericTrigger',
          genericVariables: [
            [key: 'repo_url', value: "\$.repository.git_url", expressionType: 'JSONPath', defaultValue: 'empty'],
            [key: 'repo_files_added', value: "\$.commits.added", expressionType: 'JSONPath', defaultValue: 'empty'],
            [key: 'repo_files_removed', value: "\$.commits.removed", expressionType: 'JSONPath', defaultValue: 'empty'],
            [key: 'repo_files_modified', value: "\$.commits.modified", expressionType: 'JSONPath', defaultValue: 'empty']
          ],
          token: 'asdfghjkl123456789',
          causeString: 'Triggered on $repo_url'
        ]
      ])
    ])
    stage('test run for devops-it-all-apps') {
      if (${repo_url} == 'empty') { 
        sh "echo a test run for jenkins devops-it-all-apps pipeline job"
      }
    }
    stage('Checkout') {
      if (${repo_url} != 'empty') {
        git credentialsId: 'GithubSshKey', url: "$repo_url"
      }
    }
    stage('publish containers') {
      if (${repo_url} != 'empty') {
        container ('docker') {
          script {
            def files_changed = ("$repo_files_added" + ' ' + "$repo_files_modified").flatten().join(' ')
            def files_removed = "$repo_files_removed".join(' ')
            def dir_changed = sh(script: "dirname $files_changed | cut -d\'/\' -f1-2|uniq", returnStdout: true).trim()
            def dir_removed = sh(script: "dirname $files_removed | cut -d\'/\' -f1-2|uniq", returnStdout: true).trim()

            // While app dir exists
            if (! (files_removed.equals(dir_removed))) {
              dir_changed = ("$dir_changed" + ' ' + "$dir_removed")
              dir_changed = sh(script: "dirname $dir_changed | cut -d\'/\' -f1-2|uniq", returnStdout: true).trim()
            }

            // For any Dockerfile app dir that exists
            dir_changed.each { dir_name ->
              def project_dir_name = System.properties['user.dir'].split('/')[-1]
              def repository = "ranhershko/${project_dir_name}-${dir_name}.split(\'/\')"
              dir(dir_name) {
                if (fileExists('Dockerfile')) {
                  withDockerRegistry([ credentialsId: "DockerHubPass", url: "" ]) {
                    sh """
                      docker build -t ${repository}:${BUILD_NUMBER} .
                      docker tag ${repository}:latest ${repository}:${BUILD_NUMBER}
                      docker push ${repository}:${BUILD_NUMBER}
                      docker push ${repository}:latest
                    """
                  } 
                } 
              } 
            }
            def deployment_files = sh(script: "ls |grep '^[0-9].*.yml\$'|sort", returnStdout: true).trim()
            deployment_files.each { deployment_file ->
              kubernetesDeploy(configs: "${deployment_file}", kubeconfigId: "mykubeconfig")
            }
          }
        }
      }
    }
    stage('verify dockers images') {
      if (${repo_url} != 'empty') {
        sh 'docker images'
      }
    }
  }
}
}
