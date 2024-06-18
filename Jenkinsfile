pipeline { 
    environment {        
        registryPath = "edisaac"
        appName = "springboot"
        dockerImage = ''
        GIT_COMMIT_REV = ''
        imageName = ''
        imagePath = ''    
        registryHost = "registry.hub.docker.com"
        registryCredential = "docker-hub-credential"
        sshCredentials = "git-jenkins-ssh"
        kubernetesGitOpsUrl = "git@github.com:edisaac/kubernetes-configuration.git"
        }
    agent any
    
    stages {
        stage('Build') {           
            when { branch pattern: "develop|release|main", comparator: "REGEXP"}
            steps {
                script {
                    GIT_COMMIT_REV = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()                     
                   
                    imageName =  registryPath + "/" + appName + ":" + "$BRANCH_NAME-" + GIT_COMMIT_REV
                    imagePath =   imageName
                    docker.withRegistry( "https://"+ registryHost, registryCredential ) {
                        dockerImage = docker.build  imageName                      
                    }
                    echo "BUILD"
                }
            }
        }
        stage('Push') {      
            when { branch pattern: "develop|release|main", comparator: "REGEXP"}     
            steps {
                
                script {
                    
                     docker.withRegistry( 'https://' + registryHost, registryCredential ) {
                        dockerImage.push()
                        if ("main".equals(env.BRANCH_NAME)){
                           dockerImage.push("latest")
                        }
                    }
                     echo "PUSH"
                }
            }
        }
        stage('Deploy to DEV') {
            when { branch pattern: "develop", comparator: "REGEXP"}
            steps {
                script {
                    git branch: "SNAPSHOT", credentialsId: sshCredentials,url: kubernetesGitOpsUrl
                    sh "(cd dev/apps/${appName} && kustomize edit set image ${imagePath})"
                    sh "git add -A"
                    sh "git commit -m 'modificado por jenkins job  ${env.BUILD_URL}'"

                    sshagent(credentials:[sshCredentials])
                        {
                            sh "git push --set-upstream origin SNAPSHOT" 
                        }
                }
            }
        }
        stage('Deploy to QA') {
            when { branch pattern: "release", comparator: "REGEXP"}
            steps {
                script {
                    git branch: "RELEASE", credentialsId: sshCredentials,url: kubernetesGitOpsUrl
                    sh "(cd qa/apps/${appName} && kustomize edit set image ${imagePath})"
                    sh "git add -A"
                    sh "git commit -m 'modificado por jenkins job  ${env.BUILD_URL}'"
                    sshagent([sshCredentials])
                        {
                            sh "git push --set-upstream origin RELEASE" 
                        }
                }
            }
        }
        stage('Deploy approval'){
            when {branch 'master'}
            options {timeout(time: 5, unit: 'MINUTES') }
            steps {                 
                input( message: '"Deploy to prod?')
            }
        }
        stage('Deploy to PROD') {
            when {branch 'master'}
            steps {
                script {
                    git branch: "master", credentialsId: sshCredentials,url: kubernetesGitOpsUrl
                    sh "(cd prd/apps/${appName} && kustomize edit set image ${imagePath})"
                    sh "git add -A"
                    sh "git commit -m 'modificado por jenkins job  ${env.BUILD_URL}'"
                    sshagent([sshCredentials])
                        {
                            sh "git push --set-upstream origin master" 
                        }
                }
            }
        }
    }
}