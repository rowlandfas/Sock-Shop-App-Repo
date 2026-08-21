pipeline {
  agent any

  triggers {
    pollSCM('* * * * *') // Runs every minute
  }

  environment {
    BASTION_INSTANCE_ID = credentials('bastion-id')
    ANSIBLE_IP = credentials('ansible-ip') // Keeping IP since we need it for port 22 connection
    AWS_REGION = 'eu-west-3'
    APP_REPO_NAME = "Sock-Shop-App-Repo"
    PORT = "30001"
    DEPLOYMENT_MANIFEST = "complete.yaml"
    GIT_REPO_URL = "https://github.com/rowlandfas/Sock-Shop-App-Repo.git"
    STAGE_BRANCH = "stage"
    MAIN_BRANCH = "main"
    APP_DOMAIN_URL = credentials('stage-app-domain-url')
    SLACK_CHANNEL = credentials('slack-cred')
    SLACK_TEAM_DOMAIN = credentials('slack-team-domain')
  }

  stages {
    stage('Deploying to Stage Environment') {
      steps {
        script {
          // Start SSM session to bastion with port forwarding for SSH (port 22),
          // in the background, and wait until the tunnel is actually accepting
          // connections instead of blindly sleeping.
          sh '''
              rm -f /tmp/ssm-session-stage.log /tmp/ssm-session-stage.pid

              nohup aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                > /tmp/ssm-session-stage.log 2>&1 &

              echo $! > /tmp/ssm-session-stage.pid

              # A TCP accept on 9999 only means the local end of the SSM
              # forwarder is up, not that the data channel to the bastion is
              # relaying yet. Wait for the plugin's own readiness message too,
              # otherwise the first ssh through the tunnel can land in that
              # gap and die with "Temporary failure in name resolution".
              echo "Waiting for SSM tunnel on port 9999..."
              for i in $(seq 1 30); do
                  if bash -c "cat < /dev/null > /dev/tcp/localhost/9999" 2>/dev/null \
                      && grep -q "Waiting for connections" /tmp/ssm-session-stage.log 2>/dev/null; then
                      echo "SSM tunnel is ready."
                      break
                  fi
                  sleep 1
              done

              if ! bash -c "cat < /dev/null > /dev/tcp/localhost/9999" 2>/dev/null; then
                  echo "ERROR: Port 9999 did not become available."
                  cat /tmp/ssm-session-stage.log
                  exit 1
              fi
            '''

          // Validate both SSH key credentials before sshagent loads them,
          // so a malformed key fails here with the specific credential ID
          // named, instead of surfacing as a generic "ssh-add ... invalid
          // format" error against an anonymous temp-file path.
          withCredentials([
            sshUserPrivateKey(credentialsId: 'bastion-key', keyFileVariable: 'BASTION_KEY_FILE'),
            sshUserPrivateKey(credentialsId: 'ansible-key', keyFileVariable: 'ANSIBLE_KEY_FILE')
          ]) {
            sh '''
                echo "Validating bastion-key credential..."
                ssh-keygen -y -f "$BASTION_KEY_FILE" > /dev/null || { echo "ERROR: credential 'bastion-key' is not a valid SSH private key"; exit 1; }

                echo "Validating ansible-key credential..."
                ssh-keygen -y -f "$ANSIBLE_KEY_FILE" > /dev/null || { echo "ERROR: credential 'ansible-key' is not a valid SSH private key"; exit 1; }
              '''
          }

          // SSH through the tunnel to Ansible server on port 22.
          // Even once the tunnel accepts connections, the first hop through
          // it can still transiently fail while the SSM data channel settles,
          // so retry a few times before giving up. ansible-playbook is safe
          // to re-run (git checkout + kubectl apply are idempotent).
          // Terraform destroy/apply can recreate the ansible instance (whose
          // key is cached under its own IP) and, since the bastion sits
          // behind an autoscaling group, the bastion too (whose key is
          // cached under "[localhost]:9999", the local SSM tunnel address
          // every session connects through regardless of which real bastion
          // instance answers). Either stale entry causes a hard "REMOTE HOST
          // IDENTIFICATION HAS CHANGED" refusal that StrictHostKeyChecking=no
          // does not override for an existing mismatched entry. Purge both
          // specific entries first rather than disabling host-key checking
          // outright.
          sshagent(['bastion-key', 'ansible-key']) {
            sh '''
                ssh-keygen -R "${ANSIBLE_IP}" 2>/dev/null || true
                ssh-keygen -R "[localhost]:9999" 2>/dev/null || true

                for i in $(seq 1 5); do
                    if ssh -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=10 \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no -o ConnectTimeout=10 ubuntu@localhost -p 9999" \
                        ubuntu@${ANSIBLE_IP} \
                        "ansible-playbook /etc/ansible/playbooks/stage.yml"; then
                        exit 0
                    fi
                    echo "SSH through bastion tunnel failed (attempt $i/5), retrying in 5s..."
                    sleep 5
                done
                echo "ERROR: Could not reach Ansible host through bastion tunnel after 5 attempts."
                exit 1
              '''
          }
        }
      }
      post {
        always {
          // Always tear down the tunnel, even if SSH above failed, so it
          // never leaks into the next build.
          sh '''
              if [ -f /tmp/ssm-session-stage.pid ]; then
                  kill $(cat /tmp/ssm-session-stage.pid) 2>/dev/null || true
                  rm -f /tmp/ssm-session-stage.pid
              fi
              pkill -f "aws ssm start-session.*${BASTION_INSTANCE_ID}" 2>/dev/null || true
            '''
        }
      }
    }


    stage('Slack Notification for stage') {
      steps {
        slackSend channel: env.SLACK_CHANNEL,
        message: "New Stage Deployment completed for ${env.APP_DOMAIN_URL}",
        teamDomain: env.SLACK_TEAM_DOMAIN,
        tokenCredentialId: 'slack'
      }
    }

    stage('DAST Scan') {
      steps {
        sh '''
          sleep 30s
          chmod 777 $(pwd)

          docker run -v $(pwd):/zap/wrk/:rw \
            -t ghcr.io/zaproxy/zaproxy:stable \
            zap-baseline.py \
            -t "$APP_DOMAIN_URL" \
            -g gen.conf \
            -r testreport.html || true
        '''
      }
    }

    stage('Prompt for Approval') {
      steps {
        timeout(activity: true, time: 10) {
          input message: 'Review before approval', submitter: 'admin'
        }
      }
    }

    stage('Update Deployment Manifest in Main Branch') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
            sh """
                        rm -rf ${APP_REPO_NAME} || true
                        git clone ${GIT_REPO_URL}
                        cd ${APP_REPO_NAME}
                        git config --global user.email "jenkins@set30.space"
                        git config --global user.name "Jenkins CI"

                        # Checkout stage and pull latest
                        git checkout ${STAGE_BRANCH}
                        git pull origin ${STAGE_BRANCH} --rebase

                        # Merge stage into main with conflict resolution strategy
                        git checkout ${MAIN_BRANCH}
                        git pull origin ${MAIN_BRANCH}
                        git merge --no-ff -X theirs origin/${STAGE_BRANCH} -m "Automated merge of stage into main by Jenkins \${BUILD_NUMBER}"

                        # Update nodeport from 30000 to 30001 in main branch
                        sed -i 's|nodePort: 30000|nodePort: 30001|g' ${DEPLOYMENT_MANIFEST}

                        git add ${DEPLOYMENT_MANIFEST}
                        if git diff --cached --quiet; then
                            echo "No changes detected in nodeport, skipping commit."
                        else
                            git commit -m "Update nodeport from 30000 to 30001 in main branch"
                        fi

                        # Push the updated main branch
                        REPO_URL_HTTPS=\$(echo "${GIT_REPO_URL}" | sed 's|https://||')
                        git push https://\${GIT_USERNAME}:\${GIT_TOKEN}@\${REPO_URL_HTTPS} ${MAIN_BRANCH}
                    """
          }
        }
      }
    }

    stage('Deploying to Prod Environment') {
      steps {
        script {
          sh '''
              rm -f /tmp/ssm-session-prod.log /tmp/ssm-session-prod.pid

              nohup aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                > /tmp/ssm-session-prod.log 2>&1 &

              echo $! > /tmp/ssm-session-prod.pid

              # A TCP accept on 9999 only means the local end of the SSM
              # forwarder is up, not that the data channel to the bastion is
              # relaying yet. Wait for the plugin's own readiness message too,
              # otherwise the first ssh through the tunnel can land in that
              # gap and die with "Temporary failure in name resolution".
              echo "Waiting for SSM tunnel on port 9999..."
              for i in $(seq 1 30); do
                  if bash -c "cat < /dev/null > /dev/tcp/localhost/9999" 2>/dev/null \
                      && grep -q "Waiting for connections" /tmp/ssm-session-prod.log 2>/dev/null; then
                      echo "SSM tunnel is ready."
                      break
                  fi
                  sleep 1
              done

              if ! bash -c "cat < /dev/null > /dev/tcp/localhost/9999" 2>/dev/null; then
                  echo "ERROR: Port 9999 did not become available."
                  cat /tmp/ssm-session-prod.log
                  exit 1
              fi
            '''

          withCredentials([
            sshUserPrivateKey(credentialsId: 'bastion-key', keyFileVariable: 'BASTION_KEY_FILE'),
            sshUserPrivateKey(credentialsId: 'ansible-key', keyFileVariable: 'ANSIBLE_KEY_FILE')
          ]) {
            sh '''
                echo "Validating bastion-key credential..."
                ssh-keygen -y -f "$BASTION_KEY_FILE" > /dev/null || { echo "ERROR: credential 'bastion-key' is not a valid SSH private key"; exit 1; }

                echo "Validating ansible-key credential..."
                ssh-keygen -y -f "$ANSIBLE_KEY_FILE" > /dev/null || { echo "ERROR: credential 'ansible-key' is not a valid SSH private key"; exit 1; }
              '''
          }

          sshagent(['bastion-key', 'ansible-key']) {
            sh '''
                ssh-keygen -R "${ANSIBLE_IP}" 2>/dev/null || true
                ssh-keygen -R "[localhost]:9999" 2>/dev/null || true

                for i in $(seq 1 5); do
                    if ssh -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=10 \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no -o ConnectTimeout=10 ubuntu@localhost -p 9999" \
                        ubuntu@${ANSIBLE_IP} \
                        "ansible-playbook /etc/ansible/playbooks/prod.yml"; then
                        exit 0
                    fi
                    echo "SSH through bastion tunnel failed (attempt $i/5), retrying in 5s..."
                    sleep 5
                done
                echo "ERROR: Could not reach Ansible host through bastion tunnel after 5 attempts."
                exit 1
              '''
          }
        }
      }
      post {
        always {
          sh '''
              if [ -f /tmp/ssm-session-prod.pid ]; then
                  kill $(cat /tmp/ssm-session-prod.pid) 2>/dev/null || true
                  rm -f /tmp/ssm-session-prod.pid
              fi
              pkill -f "aws ssm start-session.*${BASTION_INSTANCE_ID}" 2>/dev/null || true
            '''
        }
      }
    }

    stage('Slack Notification for prod') {
      steps {
        slackSend channel: env.SLACK_CHANNEL,
        message: "New Production Deployment completed",
        teamDomain: env.SLACK_TEAM_DOMAIN,
        tokenCredentialId: 'slack'
      }
    }
  }
}
