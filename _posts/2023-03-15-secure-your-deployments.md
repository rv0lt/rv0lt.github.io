## Build a Jenkins Pipeline to secure your deployments. 

Hello and welcome, the idea of this post is to show how can you deploy a Jenkins pipeline to do CD and secure the deployments of your Kubernetes applications. 
This tutorial assumes:

 - You already have a running kubernetes cluster, you can check the following article to learn how to quickly build one using kubespray https://forgemia.inra.fr/dipso/kubespray/-/blob/master/docs/setting-up-your-first-cluster.md
 -  You already have a Jenkins deployment and it is connected with your cluster. If you don't, you can check the following resources: https://devopscube.com/setup-jenkins-on-kubernetes-cluster/
https://devopscube.com/jenkins-build-agents-kubernetes/
- Some basic knowledge about Kubernetes and container technology

### Origin of the idea

The story behind this implementation is that, while working on my Thesis project. I came across this document of [Red Hat.](https://www.redhat.com/rhdc/managed-files/cl-state-of-kubernetes-security-report-2022-ebook-f31209-202205-en.pdf) In it, you can find the following image, it is from a survey to DevOps about which tools they use to secure their clusters:

![enter image description here](https://i.imgur.com/tYzxFG9.png)
So, I started playing with them. One thing led to another, and it gave me the idea of implementing this pipeline that will automatically check for vulnerabilities in, both the image use and in the YAML files.

### Image scanner

The first thing, is to check if the image we want to deploy is secure. There are a lot of things to consider here. For example, if nothing is specified in the Dockerfile used to create it, the container will run as root. 

However, sometimes we are just pulling the image from a registry and have no access to the Dockerfile. The usual procedure here is to use tools like Claire or Trivy. They scan the libraries that the image uses and check them against the CVE database (https://cve.mitre.org/) to find known vulnerabilities. Some container registries like Harbor already uses them when you upload an image to them.

So, for this setup, I develop a Jenkinsfile that will run Trivy, perform this scanner and fail the deployment if there are heavy vulnerabilities. I wanted to go one step more, and I will also build the image from a Dockerfile and upload them it to a registry.

First things, first. We need to define the template for our containers:

We are using, on the one hand, [img](https://github.com/genuinetools/img#running-with-kubernetes) which is a Standalone, daemon-less, unprivileged Dockerfile and OCI compatible container image builder. On the other hand, we will be performing the scanning with [trivy](https://github.com/aquasecurity/trivy)

The YAML file will look something like this:
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/img: unconfined # Necesarry for img
    container.seccomp.security.alpha.kubernetes.io/img: unconfined
spec:
  containers:
  - name: trivy # image scanner
    image: aquasec/trivy
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - infinity
  - name: img # build images
    image: r.j3ss.co/img
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - infinity
```


Now, let's see how do we use them in our Jenkinsfile. First, the image is built with the img container. We will generate an oci-compliant tar file, so that we can pass it to Trivy. Trivy has an interesting feature of building HTML with the results. The official container image has a template that we can use to generate the file. So we will call trivy two times; One first to generate a report and publish it on Jenkins, and a second one that will fail if there are CRITICAL vulnerabilities found.

Finally, to push the image to a registry we defined our credentials as a Jenkins Secret.

```jenkinsfile
        stage('Scanning') {
            steps {
                container('img') {
                    // simple Dockerfile
                    sh 'echo FROM ubuntu > Dockerfile'
                    // build it
                    sh 'img build -t rv0lt/app -o type=oci,dest=build.tar .'
                }
        
                container('trivy') {
                    sh 'mkdir -p reports'
                    // scan to generate report 
                    sh 'trivy image --input build.tar --timeout 30m0s --scanners vuln --ignore-unfixed --format template --template "@/contrib/html.tpl" -o reports/report.html'
                    publishHTML target : [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'reports',
                        reportFiles: 'report.html',
                        reportName: 'Trivy Scan',
                        reportTitles: 'Trivy Scan'
                    ]

                    // Scan again and fail on the defined level
                    sh 'trivy image --timeout 30m0s --input build.tar --scanners vuln --ignore-unfixed --exit-code 1 --severity CRITICAL > .tmp'
                }
            }
             
            }
            
        stage('Deploy'){
            // Deploy the image to registry
            steps {
                container('img'){
                    withCredentials([string(credentialsId: 'dockerhub', variable: 'SECRET')]) {
                        sh 'img login -u rv0lt -p ${SECRET}'
                        sh 'img push rv0lt/app'
                    }
                }
            }
        }
```

NOTE: In a real word set up, we will have this pipeline integrated with a repository like GitHub or GitLab, it will clone the files and, then, build the Dockerfile from it. Maybe I make a tutorial on this in the future, because I encountered some interesting errors that were not covered on tutorials that I saw while trying to implement it on my Kubernetes cluster.

This is the results we will be getting from our example:

![enter image description here](https://i.imgur.com/RgMjwAu.png)![enter image description here](https://i.imgur.com/a8owkO4.png)

### Syntax linter

The next tool we can use is [Kube-Linter](https://github.com/stackrox/kube-linter)

