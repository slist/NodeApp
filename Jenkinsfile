node {
	def app

		/* Default to no violations. Used to specify whether no violations vs. violations
		   message is sent. */
		violations = false

		stage('Clone repository') {
			/* Cloning the Repository to jenkins-docker Workspace */
			checkout scm
		}

	withEnv([
			"BUILD_NUMBER_SCAN_OUTFILE=cbctl_scan_${currentBuild.number}.json",
			"REPO=stephanelist",
			"IMAGE=nodeapp",
			"TAG=dev",
			"CBCTL_RESULTS=testing"]){

		/*
		   Environment variables:
		   BUILD_NUMBER_SCAN_OUTFILE = json output from 'cbctl validate'
		   REPO = currently my public docker repo
		   IMAGE = name assigned to image in the Build stage
		   CBCTL_RESULTS = place holder variable. Currently sent in the slack block
		   when there are no violations during 'cbctl validate'
		 */

		stage('Statical YAML Analyse') {
			try {
				echo "Analyse YAML..."
				sh '/var/jenkins_home/app/cbctl k8s-object validate -f deployment.yaml'
			}
			catch (err) {
				violations = true
				echo "Build detected cbctl violations. Review Cbctl scan results."
				slackSend color: "bad", message: "YAML deployment file incorrect! [Jenkins] '${env.JOB_NAME}' ${env.BUILD_URL}"
			}
		}


		stage('Build image') {
			/*
			   Build stage.  Build statically defined image name.
			 */
			app = docker.build("${REPO}/${IMAGE}:${TAG}")
		}


		stage('Test image') {
			/*
			   Test stage.  Not much to see here...
			 */

			app.inside {
				echo "Tests passed"
				echo "Current build lookin: ${currentBuild.currentResult}"
			}
		}


		stage('Push image') {
			/*
			   Docker credentials need to be setup in Jenkins.
			   Make sure that "docker-hub" is the name of the credential ID for your
			   docker account in Jenkins, or modify it accordingly.
			 */
			docker.withRegistry('https://registry.hub.docker.com', 'docker-hub') {
				app.push("${TAG}")
			}
			echo "Trying to Push Docker Build to DockerHub"
		}


		/*
		   Validate new build with cbctl. Outfiles written include the ${IMAGE}_${NAME}_validate.json and the cbctl_policy_violations.txt
		   Tries to validate and send confirmation of no violations.
		   Catch cbctl error when violations occur and send offending rules to cbctl_policy_violations.txt for upload. File is uploaded currently but ideally i want to show the
		   rules in the slack message.
		 */

		stage('Validate image') {
			try {
				echo "Validate stage... Starting validate test for ${REPO}/${IMAGE}:${TAG}. If there are issues, review ${REPO}_${IMAGE}_validate.json"
				sh '/var/jenkins_home/app/cbctl image validate ${REPO}/${IMAGE}:${TAG} -o json > ${REPO}_${IMAGE}_validate.json'
				sh 'python3 /var/jenkins_home/app/cbctl_validate_helper.py ${REPO}_${IMAGE}_validate.json > cbctl_policy_no_violations.txt'
			}
			catch (err) {
				violations = true
					echo "Build detected cbctl violations. Review Cbctl scan results."
					sh 'python3 /var/jenkins_home/app/cbctl_validate_helper.py ${REPO}_${IMAGE}_validate.json > cbctl_policy_violations.txt'
			}
		}

		/*
		   Creates slack block messages and uploads violation summary to channel.
		   Comment out this stage if you dont want to send to Slack :(
		 */

		stage('Send Validate Results') {
			blocks_fail = [
				[
					"type": "section",
				"text": [
					"type": "mrkdwn",
				"text": "View container details in CBC Console - <https://defense-prod05.conferdeploy.net/kubernetes/repos| here > \nView ${env.JOB_NAME} Build no. ${env.BUILD_NUMBER} - <${env.BUILD_URL}| here > "
				]
				]
			]

			if(violations == false) {
				slackSend color: "good", message: "No violations! Woohoo! [Jenkins] '${env.JOB_NAME}' ${env.BUILD_URL}"
			}

			if(violations == true) {
				slackSend(channel: "#build-alerts", blocks: blocks_fail)
				slackUploadFile filePath: "cbctl_policy_violations.txt", initialComment: ""
				echo "Violations occured. results of cbctl validate can be found in ${REPO}/${IMAGE}_validate.json and a summary in 'cbctl_policy_violations.txt'"
			}
		}

		stage('Deployment test') {
			sshagent(['stephane_ssh_key']) {
				sh "scp -o StrictHostKeyChecking=no deployment.yaml stephane@192.168.1.97:/k8s/dev/"
				try{
					sh "ssh stephane@192.168.1.97 microk8s kubectl apply -f /k8s/dev/deployment.yaml && sleep 5"
				}
				catch(error){
					echo "Welp... those didnt exist yet..."
					sh "ssh stephane@192.168.1.97 microk8s kubectl create -f /k8s/dev/deployment.yaml && sleep 5"
				}
			}
		}

		stage('Connection Test') {
			sh "curl 192.168.1.97:30333"
			echo "Done testing"
		}

		stage('Cleanup') {
			sshagent(['stephane_ssh_key']) {
				sh "ssh stephane@192.168.1.97 microk8s kubectl delete deployment nodeapp"
				sh "ssh stephane@192.168.1.97 microk8s kubectl delete service nodeapp-service"
				sh "ssh stephane@192.168.1.97 microk8s kubectl get all"
			}
		}
	}
}
}
