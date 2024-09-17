import groovy.json.JsonSlurper
import java.security.*

def parseJson(jsonString) {
    def lazyMap = new JsonSlurper().parseText(jsonString)
    def m = [:]
    m.putAll(lazyMap)
    return m
}

def parseJsonArray(jsonString){
  def datas = readJSON text: jsonString
  return datas
}

def parseJsonString(jsonString, key){
  def datas = readJSON text: jsonString
  String Values = writeJSON returnText: true, json: datas[key]
  return Values
}

def parseYaml(jsonString) {
  def datas = readYaml text: jsonString
  String yml = writeYaml returnText: true, data: datas['kubernetes']
  return yml
}

def createYamlFile(data,filename) {
  writeFile file: filename, text: data
}



def returnSecret(path,secretValues){
  def secretValueFinal= []
  for(secret in secretValues) {
    def secretValue = [:]
    //secretValue['envVar'] = secret.envVariable
    //secretValue['vaultKey'] = secret.vaultKey
    secretValue.put('envVar',secret.envVariable)
    secretValue.put('vaultKey',secret.vaultKey)
    secretValueFinal.add(secretValue)
  }
  def secrets = [:]
  secrets["path"] = path
  secrets['engineVersion']=  2
  secrets['secretValues'] = secretValueFinal

  return secrets
}

// String str = ''
// loop to create a string as -e $STR -e $PTDF

def dockerVaultArguments(secretValues){
  def data = []
  for(secret in secretValues) {
    data.add('$'+secret.envVariable+' > .'+secret.envVariable)
  }
  return data
}

def dockerVaultArgumentsFile(secretValues){
  def data = []
  for(secret in secretValues) {
    data.add(secret.envVariable)
  }
  return data
}

def pushToCollector(){
  print("Inside pushToCollector...........")
    def job_name = "$env.JOB_NAME"
    def job_base_name = "$env.JOB_BASE_NAME"
    String metaDataProperties = parseJsonString(env.JENKINS_METADATA,'general')
    metadataVars = parseJsonArray(metaDataProperties)
    if(metadataVars.tenant != '' &&
    metadataVars.lazsaDomainUri != ''){
      echo "Job folder - $job_name"
      echo "Pipeline Name - $job_base_name"
      echo "Build Number - $currentBuild.number"
      sh """curl -k -X POST '${metadataVars.lazsaDomainUri}/collector/orchestrator/devops/details' -H 'X-TenantID: ${metadataVars.tenant}' -H 'Content-Type: application/json' -d '{\"jobName\" : \"${job_base_name}\", \"projectPath\" : \"${job_name}\", \"agentId\" : \"${metadataVars.agentId}\", \"devopsConfigId\" : \"${metadataVars.devopsSettingId}\", \"agentApiKey\" : \"${metadataVars.agentApiKey}\", \"buildNumber\" : \"${currentBuild.number}\" }' """
    }
}

def returnVaultConfig(vaultURL,vaultCredID){
  echo vaultURL
  echo vaultCredID
  def configurationVault = [:]
  //configurationVault["vaultUrl"] = vaultURL
  configurationVault["vaultCredentialId"] = vaultCredID
  configurationVault["engineVersion"] = 2
  return configurationVault
}

def waitforsometime() {
  sh 'sleep 5'
}

def checkoutRepository() {
  sh '''sudo chown -R `id -u`:`id -g` "$WORKSPACE" '''

  checkout([
    $class: 'GitSCM',
    branches: [[name: env.TESTCASEREPOSITORYBRANCH]],
    doGenerateSubmoduleConfigurations: false,
    extensions: [[$class: 'CleanCheckout']],
    submoduleCfg: [],
    userRemoteConfigs: [[credentialsId: env.SOURCECODECREDENTIALID,url: env.TESTCASEREPOSITORYURL]]
  ])
}


def getApplicationUrl(){
  def host
  if (env.DEPLOYMENT_TYPE == 'KUBERNETES'){
    kubernetesJsonString = parseJsonString(env.JENKINS_METADATA,'kubernetes')
    ingressJsonString = parseJsonString(kubernetesJsonString,'ingress')
    ingress = parseJsonArray(ingressJsonString)
    if(ingress['hosts']){
      hostname = ingress.hosts[0]
      print("hostname:"+hostname)
      host = hostname
    }
    else{
      host = metadataVars.ingressAddress
    }
    print("host in kubernetes:"+host)
  }
  else if(env.DEPLOYMENT_TYPE == 'EC2') {
    dockerProperties = parseJsonString(env.JENKINS_METADATA, 'docker')
    dockerData = parseJsonArray(dockerProperties)
    host = DOCKERHOST+':'+dockerData.hostPort
    print("host in ec2:"+host)
  }

  print("SITE_URL under test: http://${host}$CONTEXT/")
  return host;
}

def runSeleniumTest() {
  def host = getApplicationUrl()
  sh env.TESTCASECOMMAND + " -DSITE_URL=http://${host}$CONTEXT/ > maven_test.out || true "
  sh 'cat maven_test.out'
  sh script: """
             #!/bin/bash
             cat maven_test.out | egrep "^\\[(INFO|ERROR)\\] Tests run: [0-9]+, Failures: [0-9]+, Errors: [0-9]+, Skipped: [0-9]+\$" | cut -c 8- | awk 'BEGIN { for(i=1;i<=5;i++) printf "*+"; } {printf "%s",\$0 } END { for(i=1;i<=5;i++) printf "*+"; }'
             """
}


def runCypressTest() {
  def host = getApplicationUrl()

  sh """
  cat <<EOL > test.sh
  #!/bin/bash

  rm -rf node_modules/ mochawesome-report/ cypress/videos/ /cypress/screenshots/
  apt-get update
  apt-get install -y libgbm-dev
  npm install --save-dev mochawesome
  npm install mochawesome-merge --save-dev
  npm install
  case "$env.TESTCASECOMMAND" in
    *env*)
        # Do stuff
        echo 'case env'
        $env.TESTCASECOMMAND applicationUrl=http://${host}$CONTEXT/ --browser $env.BROWSERTYPE --reporter mochawesome --reporter-options overwrite=false,html=false,json=true,charts=true
        ;;
    *)
        echo 'case else'
        $env.TESTCASECOMMAND -- --env applicationUrl=http://${host}$CONTEXT/ --browser $env.BROWSERTYPE --reporter mochawesome --reporter-options overwrite=false,html=false,json=true,charts=true
        ;;
  esac
  npx mochawesome-merge mochawesome-report/*.json > mochawesome-report/output.json
  npx marge mochawesome-report/output.json mochawesome-report  ./ --inline

  EOL
  """
  sh 'docker run -v "$WORKSPACE"/testcaseRepo:/app -w /app cypress/browsers:node14.19.0-chrome100-ff99-edge /bin/sh test.sh > test.out || true'
  sh 'cat test.out'
  sh """
  awk 'BEGIN { for(i=1;i<=5;i++) printf "*+"; } /(^[ ]*✖.+(failed|pended|pending|skipped|skipping)|^[ ]*✔[ ]+All specs passed).+/ {for(i=4;i>=0;i--) switch (i) {case 4: if( \$(NF-i) ~ /^[0-9]/ ){printf aggr "Tests run: "\$(NF-i) aggr ", ";} else{printf "Tests run: 0, " ;}  break; case 3: if( \$(NF-i) ~ /^[0-9]/ ){printf aggr "Passed: "\$(NF-i) aggr ", "} else{printf "Passed: 0, "} break; case 2: if( \$(NF-i) ~ /^[0-9]/ ){printf aggr "Failures: " \$(NF-i) aggr ", "} else{printf "Failures: 0, "}  break; case 1: if( \$(NF-i) ~ /^[0-9]/ ){printf aggr "Pending: " \$(NF-i) aggr ", "} else{printf "Pending: 0, "} break; case 0: if( \$(NF-i) ~ /^[0-9]/ ){printf aggr "Skipped: " \$(NF-i)} else{printf "Skipped: 0"} break; }} END { for(i=1;i<=5;i++) printf "*+"; }' test.out
  """
  sh 'pwd'
}

def publishResults(reportDir, reportFiles, reportName, reportTitles) {
  publishHTML([allowMissing: true,
               alwaysLinkToLastBuild: true,
               keepAll: true,
               reportDir: reportDir,
               reportFiles: reportFiles,
               reportName: reportName,
               reportTitles: reportTitles
  ])

  junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
}

def testngPublishResults() {
  step([$class: 'Publisher', reportFilenamePattern: '**/testng-results.xml'])
}

def agentLabel = "${env.JENKINS_AGENT == null ? "":env.JENKINS_AGENT}"

pipeline {
	agent { label agentLabel }
  	environment {
        BRANCHES = "${env.GIT_BRANCH}"
        COMMIT = "${env.GIT_COMMIT}"
        RELEASE_NAME = "mysql"
        SERVICE_PORT = "${APP_PORT}"
        DOCKERHOST = "${DOCKERHOST_IP}"
        REGISTRY_URL = "${DOCKER_REPO_URL}"

		DEPLOYMENT_TYPE = "${DEPLOYMENT_TYPE == ""? "EC2":DEPLOYMENT_TYPE}"
		KUBE_SECRET = "${KUBE_SECRET}"
		MYSQL_IMAGE = "bitnami/mysql:8.4-debian-12"
		OC_IMAGE_VERSION = "quay.io/openshift/origin-cli:4.9.0" //https://quay.io/repository/openshift/origin-cli?tab=tags
		HELM_IMAGE_VERSION = "alpine/helm:3.8.1"
    }
    stages {
		stage('Initialization') {
			steps {
				script {
					def listValue = "$env.LIST"
                                 def list = listValue.split(',')
                                 print(list)
                                 // For Context Path read
                                  String metaDataProperties = parseJsonString(env.JENKINS_METADATA,'general')
                                  metadataVars = parseJsonArray(metaDataProperties)
                                  if(metadataVars.repoName == ''){
                                      metadataVars.repoName = env.RELEASE_NAME
                                  }
                                                  //Getting Container Registry URL
                                    env.REGISTRY_URL = metadataVars.containerImagePath
                                    env.BUILD_TAG = metadataVars.containerImageTag
                                    env.KUBE_SECRET = metadataVars.kubernetesSecret
                                    env.ARTIFACTORY_CREDENTIALS = metadataVars.artifactorySecret
                                    env.ARTIFACTORY = metadataVars.artifactory

                                    env.STAGE_FLAG = metadataVars.stageFlag
                                    env.DOCKERHOST = metadataVars.dockerHostIP
                                    env.RELEASE_NAME = metadataVars.name


                               if (env.DEPLOYMENT_TYPE == 'KUBERNETES' || env.DEPLOYMENT_TYPE == 'OPENSHIFT') {
                    env.helmReleaseName = "${metadataVars.helmReleaseName}"
                                    String kubeProperties = parseJsonString(env.JENKINS_METADATA,'kubernetes')
                                    String helmProperties = parseJsonString(env.JENKINS_METADATA,'helm')
                                    helmVars = parseJsonArray(helmProperties)
                                    kubeVars = parseJsonArray(kubeProperties)

                                    if(kubeVars['vault']){
                                        String kubeData = parseJsonString(kubeProperties,'vault')
                                        def kubeValues = parseJsonArray(kubeData)
                                        def tempMap = [vault : [vault : kubeValues]]
                                        String vaultString = writeJSON returnText: true, json: tempMap
                                        if(kubeValues.type == 'vault'){
                                            String vault_file = parseYaml(vaultString, 'vault')
                                            String helm_file = parseYaml(env.JENKINS_METADATA)
                                            echo helm_file
                                            echo vault_file
                                            createYamlFile(vault_file,"vault.yaml")
                                            createYamlFile(helm_file,"Helm.yaml")
                                        }
                                    }else {
                                        sh 'touch "vault.yaml"'
                                        String helm_file = parseYaml(env.JENKINS_METADATA)
                                        echo helm_file
                                        createYamlFile(helm_file,"Helm.yaml")
                                    }
                                }
                                def job_name = "$env.JOB_NAME"

                               print(job_name)
                               print(metadataVars.helmReleaseName)
                               def namespace = ''
                               if (env.DEPLOYMENT_TYPE == 'KUBERNETES' || env.DEPLOYMENT_TYPE == 'OPENSHIFT'){
                                   if (kubeVars.namespace != null && kubeVars.namespace != '') {
                                       namespace = kubeVars.namespace
                                   }else{
                                       echo "namespace not received"
                                   }
                               }
                               print("kube namespace: $namespace")
                               env.namespace_name = namespace



				}
			}
		}
        stage('Deploy') {
            steps {
				script {
					if (env.DEPLOYMENT_TYPE == 'EC2') {
					    String dockerProperties = parseJsonString(env.JENKINS_METADATA, 'docker')
                        dockerData = parseJsonArray(dockerProperties)

						sh """ ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker pull $MYSQL_IMAGE" """
						sh """ ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker stop "${metadataVars.repoName}" || true && docker rm "${metadataVars.repoName}" || true" """
						sh """ ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "mkdir -p /home/ciuser/"${metadataVars.repoName}" && chown -R 1001:1001 /home/ciuser/"${metadataVars.repoName}" && docker run -d --restart always --name "${metadataVars.repoName}" -p $SERVICE_PORT:3306 -v /home/ciuser/"${metadataVars.repoName}":/bitnami/mysql/data -e MYSQL_ROOT_PASSWORD='password'  $MYSQL_IMAGE " """

					}
					if (env.DEPLOYMENT_TYPE == 'KUBERNETES' || env.DEPLOYMENT_TYPE == 'OPENSHIFT') {
                    env.helmReleaseName = "${metadataVars.helmReleaseName}"
                        sh """
                             sed -i s+#SERVICE_NAME#+"$helmReleaseName"+g ./helm_chart/values.yaml ./helm_chart/Chart.yaml

                        """

						withCredentials([file(credentialsId: "$KUBE_SECRET", variable: 'KUBECONFIG')]) {
              env.helmReleaseName = "${metadataVars.helmReleaseName}"

                if (env.DEPLOYMENT_TYPE == 'OPENSHIFT') {
                    sh '''
                        COUNT=$(grep 'serviceAccount' Helm.yaml | wc -l)
                        if [[ $COUNT -gt 0 ]]
                        then
                            ACCOUNT=$(grep 'serviceAccount:' Helm.yaml | tail -n1 | awk '{ print $2}')
                            echo $ACCOUNT
                        else
                            ACCOUNT='default'
                        fi
                        docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps $OC_IMAGE_VERSION oc adm policy add-scc-to-user anyuid -z $ACCOUNT -n "$namespace_name"
                      '''
                }

								sh """
									ls -lart
                                    cat Helm.yaml

									docker run --rm  --user root -v "$KUBECONFIG":"$KUBECONFIG" -e KUBECONFIG="$KUBECONFIG" -v "$WORKSPACE":/apps -w /apps alpine/helm:3.8.1 upgrade --install "$helmReleaseName" -n "$namespace_name"  helm_chart  --create-namespace --atomic --timeout 300s --set image.tag=8.4-debian-12  --set auth.rootPassword="password" --set primary.persistence.size="6Gi" --set service.port=$SERVICE_PORT -f Helm.yaml

									sleep 10
								"""

						}
					}
				}
            }
        }
    }

}
