pipeline {
  agent any

  environment {
    PROJECT = 'apache-roller'
    VERSION = '1.0'
    BRANCH = 'master'
    POLARIS_ACCESS_TOKEN = credentials('mj-polaris-token')
    BLACKDUCK_ACCESS_TOKEN = credentials('mj-blackduck-token')
    IO_TOKEN = credentials('mj-io-token')
    CODEDX_TOKEN = credentials('mj-codedx-token')
    GITHUB_TOKEN = credentials('mj-github-token')
    SERVER_START = "mvn jetty:run"
    SERVER_STRING = "Started Jetty Server"
    SERVER_WORKINGDIR = "app"
  }

  stages {
    stage('Build') {
      agent { label 'ubuntu' }
      steps {
        sh 'mvn -e clean package -DskipTests'
      }
    }

    stage('Set Up Environment') {
      agent { label 'ubuntu' }
      steps {
        sh '''
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/getProjectID.sh > /tmp/getProjectID.sh
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/serverStart.sh > /tmp/serverStart.sh
          curl -s -L https://raw.githubusercontent.com/jones6951/io-scripts/main/isNumeric.sh > /tmp/isNumeric.sh
          curl -s -L https://raw.githubusercontent.com/synopsys-sig/io-artifacts/main/prescription.sh > /tmp/prescription.sh

          chmod +x /tmp/getProjectID.sh
          chmod +x /tmp/serverStart.sh
          chmod +x /tmp/isNumeric.sh
          chmod +x /tmp/prescription.sh
        '''
      }
    }

    stage('IO - Get Prescription') {
      agent { label 'ubuntu' }
      steps {
        echo "Getting Prescription"
        sh '''
          projectID=$(/tmp/getProjectID.sh --url=${CODEDX_SERVER_URL} --apikey=${CODEDX_TOKEN} --project=${PROJECT})
          /tmp/prescription.sh \
            --stage="IO" \
            --persona="devsecops" \
            --io.url="${IO_SERVER_URL}/api/ioiq/" \
            --io.token="${IO_TOKEN}" \
            --manifest.type="json" \
            --project.name="${PROJECT}" \
            --asset.id="${PROJECT}" \
            --workflow.url="${IO_SERVER_URL}/api/workflowengine/" \
            --workflow.version="2021.12.0" \
            --file.change.threshold="10" \
            --sast.rescan.threshold="20" \
            --sca.rescan.threshold="20" \
            --scm.type="github" \
            --scm.owner="jones6951" \
            --scm.repo.name="apache-roller" \
            --scm.branch.name="${BRANCH}" \
            --github.username="jones6951" \
            --github.token="${GITHUB_TOKEN}" \
            --polaris.project.name="${PROJECT}" \
            --polaris.url="${POLARIS_SERVER_URL}" \
            --polaris.token="${POLARIS_ACCESS_TOKEN}" \
            --blackduck.project.name="${PROJECT}:${VERSION}" \
            --blackduck.url="${BLACKDUCK_SERVER_URL}" \
            --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
            --jira.enable="false" \
            --codedx.url="${CODEDX_SERVER_URL}/codedx" \
            --codedx.api.key="${CODEDX_TOKEN}" \
            --codedx.project.id="$projectID" \
            --IS_SAST_ENABLED="false" \
            --IS_SCA_ENABLED="false" \
            --IS_DAST_ENABLED="false"
        '''
        script {
          env.IS_SAST_ENABLED = sh(script:'jq -r ".security.activities.sast.enabled" result.json', returnStdout: true).trim()
          env.IS_SCA_ENABLED = sh(script:'jq -r ".security.activities.sca.enabled" result.json', returnStdout: true).trim()
          env.IS_DAST_ENABLED = sh(script:'jq -r ".security.activities.dast.enabled" result.json', returnStdout: true).trim()
          env.IS_DASTPLUSM_ENABLED = sh(script:'jq -r ".security.activities.dastplusm.enabled" result.json', returnStdout: true).trim()
          env.IS_SASTPLUSM_ENABLED = sh(script:'jq -r ".security.activities.sastplusm.enabled" result.json', returnStdout: true).trim()
          env.IS_IMAGESCAN_ENABLED = sh(script:'jq -r ".security.activities.imagescan.enabled" result.json', returnStdout: true).trim()
        }
      }
    }

    stage('SAST - Coverity on Polaris') {
      when {
        expression { env.IS_SAST_ENABLED == "true" }
      }
      agent { label 'ubuntu' }
      steps {
        sh '''
          #if [ ${env.IS_SAST_ENABLED} = "true" ]; then
            echo "Running Coverity on Polaris"
            rm -fr /tmp/polaris 2>/dev/null
            wget -q ${POLARIS_SERVER_URL}/api/tools/polaris_cli-linux64.zip
            unzip -j polaris_cli-linux64.zip -d /tmp
            rm polaris_cli-linux64.zip
            /tmp/polaris --persist-config --co project.name="${PROJECT}" --co project.branch="${BRANCH}" --co capture.build.buildCommands="null" --co capture.build.cleanCommands="null" --co capture.fileSystem="null" --co capture.coverity.autoCapture="enable"  configure
#            /tmp/polaris analyze -w
          #else
          #  echo "Skipping Coverity on Polaris based on prescription"
          #fi
        '''
      }
    }

    stage ('SCA - Black Duck') {
      when {
        expression { env.IS_SCA_ENABLED == "true" }
      }
      agent { label 'ubuntu' }
      steps {
        sh '''
          echo "Running BlackDuck"
          rm -fr /tmp/detect7.sh
          curl -s -L https://detect.synopsys.com/detect7.sh > /tmp/detect7.sh
          bash /tmp/detect7.sh --blackduck.url="${BLACKDUCK_URL}" --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" --detect.project.name="${PROJECT}" --detect.project.version.name="${VERSION}" --blackduck.trust.cert=true
          # --detect.blackduck.scan.mode=RAPID
        '''
      }
    }

    stage ('IAST - Seeker') {
      when {
        expression { env.IS_DAST_ENABLED == "true" }
      }
      agent { label 'ubuntu' }
      steps {
        sh '''
          cd ${SERVER_WORKINGDIR}
          sh -c "$(wget --no-check-certificate 'http://seeker.lan:8080/rest/api/latest/installers/agents/scripts/JAVA?osFamily=LINUX&downloadWith=curl&projectKey=roller&webServer=JETTY&flavor=DEFAULT&agentName=&accessToken=' -O -)"
          export SEEKER_PROJECT_VERSION=${VERSION}
          export SEEKER_AGENT_NAME=${AGENT}
          export MAVEN_OPTS=-javaagent:seeker/seeker-agent.jar

#         Start the server. Return value is either the PID of the running server process, or an error message
          serverMessage=$(/tmp/serverStart.sh --startCmd="${SERVER_START}" --startedString="${SERVER_STRING}" --project="${PROJECT}" --timeout="60s" &)
          if ( /tmp/isNumeric.sh $serverMessage); then
              echo "Running IAST Tests"

              selenium-side-runner -c "browserName=firefox moz:firefoxOptions.args=[-headless]" --output-directory=${WORKSPACE}/selenium/results ${WORKSPACE}/selenium/Apache-Roller.side

              kill $serverMessage
          else
              echo $serverMessage
          fi
        '''
      }
    }
    stage ('IO Workflow - Code Dx') {
      agent { label 'ubuntu' }
      steps {
        sh '''
          projectID=$(/tmp/getProjectID.sh --url=${CODEDX_SERVER_URL} --apikey=${CODEDX_TOKEN} --project=${PROJECT})
          /tmp/prescription.sh \
            --stage="WORKFLOW" \
            --persona="devsecops" \
            --project.name="${PROJECT}" \
            --io.url="${IO_SERVER_URL}/api/ioiq/" \
            --io.token="${IO_TOKEN}" \
            --manifest.type="json" \
            --asset.id="${PROJECT}" \
            --workflow.url="${IO_SERVER_URL}/api/workflowengine/" \
            --workflow.version="2021.12.0" \
            --file.change.threshold="10" \
            --sast.rescan.threshold="20" \
            --sca.rescan.threshold="20" \
            --scm.type="github" \
            --scm.owner="jones6951" \
            --scm.repo.name="apache-roller" \
            --scm.branch.name="${BRANCH}" \
            --github.username="jones6951" \
            --github.token="${GITHUB_TOKEN}" \
            --polaris.project.name="${PROJECT}" \
            --polaris.url="${POLARIS_SERVER_URL}" \
            --polaris.token="${POLARIS_ACCESS_TOKEN}" \
            --blackduck.project.name="${PROJECT}:${VERSION}" \
            --blackduck.url="${BLACKDUCK_URL}" \
            --blackduck.api.token="${BLACKDUCK_ACCESS_TOKEN}" \
            --jira.enable="false" \
            --codedx.url="${CODEDX_SERVER_URL}/codedx" \
            --codedx.api.key="${CODEDX_TOKEN}" \
            --codedx.project.id="$projectID" \
            --IS_SAST_ENABLED="${IS_SAST_ENABLED}" \
            --IS_SCA_ENABLED="${IS_SCA_ENABLED}" \
            --IS_DAST_ENABLED="${IS_DAST_ENABLED}"

          java -jar WorkflowClient.jar --workflowengine.url="${IO_SERVER_URL}/api/workflowengine/" --io.manifest.path=synopsys-io.json
        '''
      }
    }

    stage('Clean Workspace') {
      agent { label 'ubuntu' }
      steps {
        cleanWs()
      }
    }
  }
}
