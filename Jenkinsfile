pipeline {
  agent any
  stages {
    stage("verify tooling") {
      steps {
        sh 'ssh naveenn@192.168.1.114 docker version'
	sh 'ssh naveenn@192.168.1.114 docker-compose version'
	sh 'ssh naveenn@192.168.1.114 curl --version' 
      }
    }
    stage('Prune Docker data') {
      steps {
        sh 'ssh naveenn@192.168.1.114 docker system prune -a --volumes -f'
      }
    }
    stage('Start container') {
      steps {
sh '''echo "#bin/bash \\n export job_name=$JOB_NAME" > /tmp/properties.sh
scp /tmp/properties.sh naveenn@192.168.1.114:/tmp/.
ssh naveenn@192.168.1.114 <<\'ENDSSH\'
chmod 755 /tmp/properties.sh
source /tmp/properties.sh
echo $job_name
cd /home/naveenn/jenkins_home/workspace/$job_name/
pwd
docker network create example-net
docker-compose up -d
ENDSSH'''
      }
    }
    stage('Run tests against the container') {
      steps {
        sh 'ssh naveenn@192.168.1.114 curl http://localhost:80'
      }
    }
  }
}
