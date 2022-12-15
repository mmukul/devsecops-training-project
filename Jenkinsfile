pipeline {
  agent any 
  
  stages {
    stage ('Initialize Environment') {
      steps {
        sh '''
            echo "PATH = ${PATH}"
            echo "MAVEN_HOME = ${MAVEN_HOME}"
            ''' 
      }
    }
    
    /*...........................Pre-commit hooks............................*/
    stage ('Pre-commit hooks') {
      steps {
         sh 'rm -rf .git/hooks/pre-commit || true'
         sh 'curl https://raw.githubusercontent.com/mmukul/pre-commit-hooks/main/install.sh > install-precommit.sh'
         sh 'chmod +x install-precommit.sh'
         sh './install-precommit.sh pre-commit'
      }
    }

    /*...........................Git Secrets................................*/
    stage ('Scan Git Secrets') {
      steps {
        sh 'mkdir reports || true'
        sh 'rm reports/trufflehog || true'
        sh 'docker run gesellix/trufflehog --json https://github.com/mmukul/webapp > reports/trufflehog'
        sh 'cat reports/trufflehog'
      }
    }
    
    /*....................Software Composition Analysis....................*/
    stage ('Software Composition Analysis - SCA') {
      steps {
         sh 'rm owasp* || true'
         sh 'wget "https://raw.githubusercontent.com/mmukul/webapp/master/owasp-dependency-check.sh" '
         sh 'chmod +x owasp-dependency-check.sh'
         sh 'bash owasp-dependency-check.sh'
         sh 'mv dependency-check-report.* reports/'
        }
      }
    
    /*...........................SAST................................*/
    stage ('SonarQube - SAST') {
      steps {
          /*sh 'docker run -d -p 9000:9000 -p 9002:9002 owasp/sonarqube || true'*/
          sh 'mvn sonar:sonar -Dsonar.projectKey=devsecops -Dsonar.host.url=http://localhost:9000 -Dsonar.login=d5dc048c2ed785e40716f6ae1e17e47a1df847a2'
        }
      }
    
    stage ('Build App') {
      steps {
      sh 'mvn clean package'
       }
    }

    /*...........................DAST................................*/
    stage ('OWASP ZAP - DAST') {
      steps {
        /* sh 'docker run --name webgoat -p 8080:8080 -p 9090:9090 -d webgoat/goatandwolf
              docker run --name nginx -d -p 8081:80 nginx' */
        sh '''
          IPADD=$(ip -f inet -o addr show ens33 | awk '{print $4}' | cut -d '/' -f 1)
          docker run --user $(id -u):$(id -g) -v $(pwd):/zap/wrk/:rw --rm -t owasp/zap2docker-stable zap-baseline.py -t http://${IPADD}:8081 -r reports/zap-baseline-scan.html || true
          '''
          }
        }
    
    stage ('Vulnerability Scan - App Image') {
      steps {
        /* curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin */
        sh 'grype docker.io/nginx'
       }
    }
  }
    post {
        always {
            dependencyCheckPublisher failedNewCritical: 10, failedNewHigh: 8, failedNewLow: 2, failedNewMedium: 5, pattern: '/var/jenkins/workspace/devsecops_demo/reports/dependency-check-report.xml', unstableNewCritical: 8, unstableNewHigh: 6, unstableNewLow: 1, unstableNewMedium: 3
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '/var/jenkins/workspace/devsecops_demo/reports', reportFiles: 'zap-baseline-scan.html', reportName: 'OWASP ZAP Report', reportTitles: 'OWASP ZAP Report', useWrapperFileDirectly: true])
        }
     }
  }