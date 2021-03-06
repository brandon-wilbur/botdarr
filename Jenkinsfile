def dockerFileContents = """
FROM centos:7
RUN yum update clean all
RUN yum -y install java-1.8.0-openjdk-devel.x86_64
RUN yum -y install maven
RUN yum -y install git
RUN adduser jenkins
ENV JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
""";

def getChangelistDescription() {
  def description = "";
  def changeLogSets = currentBuild.changeSets;
  for (int i = 0; i < changeLogSets.size(); i++) {
    def entries = changeLogSets[i].items;
    for (int j = 0; j < entries.length; j++) {
      def entry = entries[j]
      description += "${entry.commitId} by ${entry.author} on ${new Date(entry.timestamp)}: ${entry.msg}\n";
    }
  }
  return description;
}

def getNextVersion() {
  def latestVersion = readFile "${env.WORKSPACE}/src/main/resources/version.txt"
  print "version=" + latestVersion;
  def (major, minor, patch) = latestVersion.tokenize('.').collect { it.toInteger() };
  print "next version: major=" + major + ",minor=" + minor + ",patch=" + (patch + 1);
  return "${major}.${minor}.${patch + 1}";
}

// whether or not to deploy to github & dockerhub
def deploy;
// the version being built/released
def tag;
// the docker image
def image;

def email;
withCredentials([string(credentialsId: 'botdarr-email', variable: 'botdarr_email')]) {
  email = "${botdarr_email}";
}

def username;
withCredentials([string(credentialsId: 'botdarr-username', variable: 'botdarr_username')]) {
  username = "${botdarr_username}";
}

pipeline {
  agent any
  stages {
    stage("Checkout") {
      steps {
        script {
          checkout([$class: 'GitSCM', branches: [[name: '**']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'github', url: "https://github.com/${username}/botdar.git"]]])
          env.GIT_COMMIT_MSG = sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
          deploy = !env.GIT_COMMIT_MSG.startsWith("Update version to") && env.BRANCH_NAME == "development";
        }
      }
    }

    stage('Prepare docker') {
      steps {
        script {
          fileOperations([fileCreateOperation(fileContent: "${dockerFileContents}", fileName: './Dockerfile')]);
          tag = getNextVersion();
          image = docker.build("botdarr-image", "-f ./Dockerfile .");
        }
      }
    }
    stage('Build') {
      steps {
        script {                
          image.inside('-u root') {
            sh './mvnw --no-transfer-progress compile'
          }
        }
      }
    }
    stage('Test') {
      steps {
        script {                
          image.inside('-u root') {
            sh './mvnw --no-transfer-progress test'
          }
        }
      }
    }
    stage('Package') {
      steps {
        script {                
          image.inside('-u root') {
            sh "echo ${tag} > ./src/main/resources/version.txt"
            sh './mvnw --no-transfer-progress package -DskipTests'
            archiveArtifacts 'target/botdarr-release.jar'
          }
        }
      }
    }
    
    stage('Update version') {
      when {
        expression {
          return deploy
        }
      }
      steps {
        sshagent(['jenkins-ssh-key-github']) {
          sh "git remote set-url origin git@github.com:${username}/botdarr.git"
          sh "git config --global user.name ${username}"
          sh "git config --global user.email ${email}"
          sh "ssh -oStrictHostKeyChecking=no ${username}@github.com || true"
          sh "git add src/main/resources/version.txt"
          sh "git commit -m \"Update version to ${tag}\""
          sh "git push origin HEAD:${env.BRANCH_NAME}"
        }
      }
    }
    
    stage('Create/Upload Release') {
      when {
        expression {
          return deploy
        }
      }
      steps {
        script {
          withCredentials([string(credentialsId: 'git-token', variable: 'token')]) {
            def description = getChangelistDescription();
            print "description=" + description;
            print "branch name=" + env.BRANCH_NAME;
            print "tag=" + tag;
            sh 'chmod 700 upload-release.sh'
            sh "./upload-release.sh ${token} ${tag} \"${description}\""
          }
        }
      }
    }
    
    stage('Upload to dockerhub') {
      when {
        expression {
          return deploy
        }
      }
      steps {
        script {
          def dockerFileImage = """
          FROM centos:7
          RUN yum update; yum clean all; yum -y install java-1.8.0-openjdk-devel.x86_64;
          ENV JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
          ENV PATH=$PATH:$JAVA_HOME/bin

          RUN mkdir -p /home/botdarr
          ADD target/botdarr-release.jar /home/botdarr

          WORKDIR /home/botdarr
          RUN java -version

          ENTRYPOINT ["java", "-jar", "botdarr-release.jar"]
          """;
          fileOperations([fileCreateOperation(fileContent: "${dockerFileImage}", fileName: './DockerfileUpload')]);
          def releaseTag = "latest";
          def imageWithReleaseTag = docker.build("${username}/botdarr:${releaseTag}", "-f ./DockerfileUpload .");
          withDockerRegistry(credentialsId: 'docker-credentials') {
            imageWithReleaseTag.push();
          }
        }
      }
    }
  }
  post {
    always {
      deleteDir()
    }
  }
}