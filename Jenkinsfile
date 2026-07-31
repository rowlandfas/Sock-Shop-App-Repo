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
          // Start SSM session to bastion with port forwarding for SSH (port 22)
          sh '''
              aws ssm start-session \
                --target ${BASTION_INSTANCE_ID} \
                --region ${AWS_REGION} \
                --document-name AWS-StartPortForwardingSession \
                --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                &
              sleep 5  # Wait for port forwarding to establish
            '''

          // SSH through the tunnel to Ansible server on port 22
          sshagent(['bastion-key', 'ansible-key']) {
            sh '''
                ssh -o StrictHostKeyChecking=no \
                    -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                    ubuntu@${ANSIBLE_IP} \
                    "ansible-playbook /etc/ansible/playbooks/stage.yml"
              '''
          }

          // Terminate the SSM session
          sh 'pkill -f "aws ssm start-session"'
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
    rm -f ssm.log

    nohup aws ssm start-session \
        --target ${BASTION_INSTANCE_ID} \
        --region ${AWS_REGION} \
        --document-name AWS-StartPortForwardingSession \
        --parameters portNumber=22,localPortNumber=9999 \
        > ssm.log 2>&1 &

    echo "Waiting for SSM tunnel..."

    for i in $(seq 1 30); do
        if ss -tln | grep -q ":9999"; then
            echo "Tunnel established."
            break
        fi
        sleep 1
    done

    echo "===== Tunnel Status ====="
    ss -tln | grep 9999 || true

    echo "===== SSM Log ====="
    cat ssm.log || true
'''

          // Test SSH Agent and Bastion Login
          sshagent(['bastion-key']) {
            sh '''
        echo "======================================"
        echo "SSH Agent Identities"
        echo "======================================"
        ssh-add -l

        echo ""
        echo "======================================"
        echo "SSH_AUTH_SOCK"
        echo "======================================"
        echo $SSH_AUTH_SOCK

        echo ""
        echo "======================================"
        echo "Testing SSH to Bastion"
        echo "======================================"

        ssh -vvv \
            -o StrictHostKeyChecking=no \
            -p 9999 \
            ubuntu@localhost \
            "hostname"

        echo ""
        echo "======================================"
        echo "Testing SSH to Ansible Server"
        echo "======================================"

        ssh -A \
            -vvv \
            -o StrictHostKeyChecking=no \
            -o "ProxyCommand=ssh -A -o StrictHostKeyChecking=no -W %h:%p -p 9999 ubuntu@localhost" \
            ubuntu@${ANSIBLE_IP} \
            "hostname"

        echo ""
        echo "======================================"
        echo "Running Stage Playbook"
        echo "======================================"

        ssh -A \
            -o StrictHostKeyChecking=no \
            -o "ProxyCommand=ssh -A -o StrictHostKeyChecking=no -W %h:%p -p 9999 ubuntu@localhost" \
            ubuntu@${ANSIBLE_IP} \
            "ansible-playbook /etc/ansible/playbooks/stage.yml"
    '''
          }

          sh 'pkill -f "aws ssm start-session"'
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
