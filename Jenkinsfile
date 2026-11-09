pipeline {
  agent { label 'podman' }

  triggers {
    GenericTrigger(
      genericVariables: [
        [key: 'FILENAME',    value: '$.filename'],
        [key: 'CONTENT_B64', value: '$.content_b64'],
        [key: 'AWX_JOB_ID',  value: '$.awx_job_id']
      ],
      token: 'awx-java-build',
      causeString: 'Triggerato da AWX job $AWX_JOB_ID',
      printContributedVariables: false,
      printPostContent: false
    )
  }

  stages {
    stage('Verifica input') {
      steps {
        script {
          if (!env.CONTENT_B64 || !env.FILENAME) {
            error('Nessun file ricevuto')
          }
        }
        echo "Ricevuto ${env.FILENAME} dal job AWX ${env.AWX_JOB_ID}"
      }
    }

    stage('Crea artefatto') {
      steps {
        sh '''
          mkdir -p src
          echo "$CONTENT_B64" | base64 -d > "src/$FILENAME"
        '''
        archiveArtifacts artifacts: "src/${env.FILENAME}", fingerprint: true
      }
    }

    stage('Validazione') {
      steps {
        sh '''
          docker run --rm -v "$PWD/src/$FILENAME":/etc/nginx/nginx.conf:ro nginx:stable nginx -t
        '''
      }
    }
  }
}