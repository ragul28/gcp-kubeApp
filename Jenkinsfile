node {
  def project = 'arimac-devops'
  def appName = 'gceme'
  def feSvcName = "${appName}-frontend"
  def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
  def kubepath = '/home/ragul/drive/project/google-cloud-sdk/bin/'
  def gcpauth = '/home/ragul/arimac-devops-70e35c8c2085.json'
  
  checkout scm

  stage 'Build image'
  sh("docker build -t ${imageTag} .")

  stage 'Run Go tests'
  sh("docker run ${imageTag} go test")

  stage 'Push image to registry'
  withDockerRegistry(url: 'https://gcr.io/', credentialsId: 'gcr:arimac-devops'){
      sh("docker push ${imageTag}")
  }
  
  stage "Deploy Application"
  switch (env.BRANCH_NAME) {
    // Roll out to canary environment
    case "canary":
        // Change deployed image in canary to the one we just built
        //sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/canary/*.yaml")
        sh("${kubepath}gcloud auth activate-service-account --key-file ${gcpauth}")
        sh("${kubepath}gcloud config set project arimac-devops")
        sh("${kubepath}gcloud info")
        sh("${kubepath}gcloud container clusters get-credentials cluster-3 --zone us-central1-a --project arimac-devops")
        sh("${kubepath}kubectl get nodes")
        sh("${kubepath}kubectl --namespace=production get pods")
        sh("${kubepath}kubectl --namespace=production get service")
        sh("${kubepath}kubectl --namespace=production apply -f k8s/services/")
        sh("${kubepath}kubectl --namespace=production apply -f k8s/canary/")
        sh("echo http://`${kubepath}kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out to production
    case "master":
        // Change deployed image in canary to the one we just built
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
        sh("${kubepath}kubectl --namespace=production apply -f k8s/services/")
        sh("${kubepath}kubectl --namespace=production apply -f k8s/production/")
        sh("echo http://`${kubepath}kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
        break

    // Roll out a dev environment
    default:
        // Create namespace if it doesn't exist
        sh("${kubepath}kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
        sh("${kubepath}kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
        sh("${kubepath}kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
        echo 'To access your environment run `kubectl proxy`'
        echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
  }
}
