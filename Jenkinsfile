podTemplate(label: 'jenkins-slave', inheritFrom: 'default', containers: [
    containerTemplate( name: 'docker', image: 'docker:18.02', ttyEnabled: true, command: 'cat')
],
volumes: [
    hostPathVolume( hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
  ]
)
{
  node('jenkins-slave') {
    properties([
      pipelineTriggers([
        [$class: 'GenericTrigger',
          genericVariables: [
            [key: 'repo_ssh_url', value: "\$.repository.ssh_url", expressionType: 'JSONPath'],
      [key: 'repo_full_name', value: "\$.repository.full_name", expressionType: 'JSONPath'],
            [key: 'repo_files_added', value: "\$.head_commit.added", expressionType: 'JSONPath'],
            [key: 'repo_files_removed', value: "\$.head_commit.removed", expressionType: 'JSONPath'],
            [key: 'repo_files_modified', value: "\$.head_commit.modified", expressionType: 'JSONPath']
          ],
          token: '1234567890',
          causeString: "Triggered on $repo_ssh_url"
        ]
      ])
    ])
        stage('Not real run, required manual run before running automatically') {
          if (repo_ssh_url == 'empty') { 
            println repo_ssh_url
            sh "echo $repo_ssh_url a test run for jenkins devops-it-all-apps pipeline job"
          }
        }
        stage('Checkout') {
          if (repo_ssh_url != 'empty') {
            git credentialsId: 'GithubSshKey', url: repo_ssh_url
          }
        }
        stage('publish containers') {
          println "repo_ssh_url is $repo_ssh_url, repo_files_modified is $repo_files_modified, repo_files_added is: $repo_files_added, repo_files_removed is: $repo_files_removed"
          if (repo_ssh_url != 'empty') {
            container ('docker') {
              env.WORKSPACE = pwd()
        script {
                def changed_path_list = Eval.me(repo_files_added) + Eval.me(repo_files_modified)
        if (! changed_path_list.isEmpty() ) {
          if ( changed_path_list != null ) {
            def changed_base_paths = changed_path_list.collect { it.split('/')[0] }
            changed_dirs_base_path = changed_base_paths.each { base_path -> 'sh returnStdout: true, script: "if [ -d \${base_path} ]; then echo \${base_path}; fi"' }
            changed_dirs_base_path = changed_dirs_base_path.unique()
            println "changed_dirs_base_path are $changed_dirs_base_path"
          }
        }

        def removed_path_list = Eval.me(repo_files_removed)
        println "removed_path_list is: $removed_path_list"
        if (! removed_path_list.isEmpty() ) { 
          if ( removed_path_list != null ) {
            def removed_base_paths = removed_path_list.collect { it.split('/')[0] }
            println "removed_base_paths is: $removed_base_paths"
            def removed_dirs_base_paths = removed_base_paths.unique()
          removed_dirs_base_paths = removed_dirs_base_paths.collect { base_path -> 'sh returnStdout: true, script: "if [ -d \${base_path} ]; then echo \${base_path}; fi"' }
            println "removed_dirs_base_paths is: $removed_dirs_base_paths"
          }
        }

                // For any Dockerfile app dir that exists
        def repository = repo_full_name
        if (! changed_dirs_base_path.isEmpty() ) {
          if ( changed_dirs_base_path != null ) {
            changed_dirs_base_path.each { dir_name ->
                      repository = "${repository}-" + dir_name.split('/')[0]
            println "Building image $repository and push to Dockerhub"
                      dir(dir_name) {
                        if (fileExists('Dockerfile')) {
                          withDockerRegistry([ credentialsId: "DockerHubPass", url: "" ]) {
              sh """
                              docker build --network=host -t $repository:1.$BUILD_NUMBER .
                              docker tag $repository:1.$BUILD_NUMBER $repository:latest
                              docker push $repository:1.$BUILD_NUMBER
                              docker push $repository:latest
                              docker images
                            """
              }
                        } 
                      }
                    }
          }
        }
        def deployment_files = sh(script: "ls |grep '^[0-9].*yaml'|sort", returnStdout: true).readLines()
        println "Deployment file list: ${deployment_files}"
                deployment_files.each { deployment_file ->
          kubernetesDeploy(configs: deployment_file, kubeconfigId: "mykubeconfig") 
                }
              }
            }
          }
        }
  }
}
