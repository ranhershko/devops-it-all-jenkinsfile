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
            [key: 'repo_ssh_url', value: "\$.repository.ssh_url", expressionType: 'JSONPath'],
            [key: 'repo_files_added', value: "\$.head_commit.added", expressionType: 'JSONPath'],
            [key: 'repo_files_removed', value: "\$.head_commit.removed", expressionType: 'JSONPath'],
            [key: 'repo_files_modified', value: "\$.head_commit.modified", expressionType: 'JSONPath']
          ],
          token: 'asdfghjkl123456789',
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
                def changed_files_n_dirs_list = Eval.me(repo_files_added) + Eval.me(repo_files_modified)
                println "changed_files_n_dirs_list are $changed_files_n_dirs_list"
        changed_files_n_dirs_list = changed_files_n_dirs_list.split(" ") 
        def changed_base_dirs = changed_files_n_dirs_list.each ({ file_or_dir -> sh 'export base_file_or_dir=\$(ls $file_or_dir |awk -F\'/\' \'{print $1}\'); if [ "$(file ${base_file_or_dir})" = "$(echo ${base_file_or_dir}: directory)" ] ; then echo ${base_file_or_dir} ; fi'
        //def changed_base_dirs = changed_files_n_dirs_list.findAll({ path -> !(new File("${env.WORKSPACE}" + path.split('/')[0]).isFile()) }).collect { path.split('/')[0] }
          changed_base_dirs = changed_base_dirs.unique()
          println "changed_base_dirs are $changed_base_dirs"

        def removed_files_n_dirs_list = Eval.me(repo_files_removed)
        println "removed_files_n_dirs_list is: $removed_files_n_dirs_list"
        if (! removed_files_n_dirs_list.isEmpty()) {
          def removed_base_dirs = removed_files_n_dirs_list.each ({ file_or_dir -> sh(export base_file_or_dir=\$(ls $file_or_dir |awk -F'/' '{print $1}'); if [ "$(file ${base_file_or_dir})" = "\$(echo ${base_file_or_dir}: directory)" ] ; then echo ${base_file_or_dir} ; fi
          //def removed_base_dirs = removed_files_n_dirs_list.findAll({ path -> !(new File(path.split('/')[1]).isFile())}).unique()
          println "removed_base_dirs are $removed_base_dirs"
          println "removed_files_n_dirs_list are $removed_files_n_dirs_list"
    
          // While app dir exists
          if (! (removed_base_dirs.equals(removed_files_n_dirs_list))) {
          changed_base_dirs = (changed_base_dirs + removed_base_dirs).unique()
                  }
        }

                // For any Dockerfile app dir that exists
        if (! changed_base_dirs.isEmpty()) {
                  changed_base_dirs.each { dir_name ->
                    def project_dir_name = System.properties['user.dir'].split('/')
                    def repository = "ranhershko/" + project_dir_name + "-" + dir_name.split('/')
                    dir(dir_name) {
                      if (fileExists('Dockerfile')) {
                        withDockerRegistry([ credentialsId: "DockerHubPass", url: "" ]) {
                          sh """
                            docker build -t repository:BUILD_NUMBER .
                            docker tag repository:latest repository:BUILD_NUMBER
                            docker push repository:BUILD_NUMBER
                            docker push repository:latest
              docker images
                          """
                        } 
                      } 
                    } 
                  }
                }
                def deployment_files = sh(script: "ls |grep '^[0-9].*.yml\$'|sort", returnStdout: true).trim()
                deployment_files.each { deployment_file ->
                  kubernetesDeploy(configs: deployment_file, kubeconfigId: "mykubeconfig")
                }
              }
            }
          }
        }
  }
}
