## Build a Jenkins Pipeline to secure your deployments. 

Hello and welcome, the idea of this post is to show how can you deploy a Jenkins pipeline to do CD and secure the deployments of your Kubernetes applications. 

This tutorial assumes:

 - You already have a running kubernetes cluster, you can check the following article to learn how to quickly build one using kubespray [here.](https://forgemia.inra.fr/dipso/kubespray/-/blob/master/docs/setting-up-your-first-cluster.md)
 -  You already have a Jenkins deployment and it is connected with your cluster. If you don't, you can check the following resources: [Setup Jenkins](https://devopscube.com/setup-jenkins-on-kubernetes-cluster/) and [Setup Kubernetes plugin](https://devopscube.com/jenkins-build-agents-kubernetes/)
- Some basic knowledge about Kubernetes and container technology

### Origin of the idea

The story behind this implementation is that, while working on my Thesis project. I came across this document of [Red Hat.](https://www.redhat.com/rhdc/managed-files/cl-state-of-kubernetes-security-report-2022-ebook-f31209-202205-en.pdf) In it, you can find the following image, it is from a survey to DevOps about which tools they use to secure their clusters:

![enter image description here](https://i.imgur.com/tYzxFG9.png)
So, I started playing with them. One thing led to another, and it gave me the idea of implementing this pipeline that will automatically check for vulnerabilities in, both the image use and in the YAML files.

### Image scanner

The first thing, is to check if the image we want to deploy is secure. There are a lot of things to consider here. For example, if nothing is specified in the Dockerfile used to create it, the container will run as root. 

The usual procedure (as far as I know) is to use tools like Claire or Trivy. They scan the libraries that the image uses and check them against the [enter link CVE database ](https://cve.mitre.org/) to find known vulnerabilities, it will also try to find misconfigurations. Some container registries like Harbor already integrates them, and when you upload an image it perfoms the scanning. If you have never used it, they have a test server where you can play around. [(Check here)](https://goharbor.io/docs/1.10/install-config/demo-server/)

So, for this setup, I develop a Jenkinsfile that will run Trivy, perform this scanner and fail the deployment if there are heavy vulnerabilities. I wanted to go one step more, and I will also build the image from a Dockerfile and upload them it to a registry.

First things, first. We need to define the template for our containers:

We are using, on the one hand, [img](https://github.com/genuinetools/img#running-with-kubernetes) which is a Standalone, daemon-less, unprivileged Dockerfile and OCI compatible container image builder. On the other hand, we will be performing the scanning with [trivy](https://github.com/aquasecurity/trivy)

The YAML file that we can define in the Jenkinsfile will look something like this:
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


Now, let's see how do we use them in our pipeline. First, the image is built with the img container. We will generate an oci-compliant tar file, so that we can pass it to Trivy. Trivy has an interesting feature of building HTML with the results. The official container image has a template that we can use to generate the file. So we will call trivy two times; One first to generate a report and publish it on Jenkins, and a second one that will fail if there are CRITICAL vulnerabilities found.

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
                    sh 'trivy image --input build.tar --timeout 30m0s --ignore-unfixed --format template --template "@/contrib/html.tpl" -o reports/report.html'
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
                    sh 'trivy image --timeout 30m0s --input build.tar --ignore-unfixed --exit-code 1 --severity CRITICAL'
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

Let's give a close look to the trivy and img commands. 

```bash
img build -t rv0lt/app -o type=oci,dest=build.tar .
```
For img, we are calling the build function, specifying a tag (if we don't want to push it to a register, then we could omit the tag). The -o flag is used to specify the output. We are doing this because then it will be easier for trivy to scan the image, so we will generate a tar file that is oci-compliant. See the following [post](https://snyk.io/blog/container-image-formats/) for more information on this.

For trivy. The first time we call it, we used:

```bash
trivy image --input build.tar --timeout 30m0s --ignore-unfixed --format template --template "@/contrib/html.tpl" -o reports/report.html
```
We are telling it to use our generated tar file, we define a timeout, in case our cluster is slow. The first time the command is called it will download the CVE Vulnerabilities database. But it shouldn't take more than 10 minutes.  We will ignore such vulnerabilities that are not fixed in another release, use the given template and output the result through the HTML file. The HTML Publisher Plugin will help us visualize the HTML. For the second call:

```bash
trivy image --timeout 30m0s --input build.tar --ignore-unfixed --exit-code 1 --severity CRITICAL
```

The difference here is that we are telling it to fail (exit code 1) if it finds CRITICAL vulnerabilities (we could adapt this to another level, or define it in a variable that we could modify according to our specifications).


So, for this example we would get the following results:

![enter image description here](https://i.imgur.com/RgMjwAu.png)![enter image description here](https://i.imgur.com/a8owkO4.png)

### Syntax linter

The next tool we can use is [Kube-Linter](https://github.com/stackrox/kube-linter). 
It can check the syntax of YAML files, and will exit with an error code if it finds any. We could go one step forward. And check also if the image that is trying to use is vulnerable, for example, we can use the grep tool to find the line(s) in the YAML file that contains the image being used and pass it to trivy. We are going to suppose pod.yaml is the file we are analyzing:

```bash
cat pod.yaml | grep "image:" | sed "s/^.*: //" > images.txt
```
We read the file and send it to grep to find the lines that contain the syntax used to define the images to deploy in Kubernetes. Finally, the sed comand is removing everything after the ":" symbol to retrieve only the image name:

![](https://i.imgur.com/9zD74Ro.png)

Now, we can loop through the file images.txt and pass the same command we already used above. In our Jenkinsfile it would look like this, suppose we already have a file pod.yaml defined:

```jenkinsfile
        stage('Syntax Scanning') {
            steps {
                container('linter') {
                    sh '/kube-linter lint pod.yaml'
                }
            }
             
        }

        stage('image Scanning') {
            steps {
                container('trivy') {
                    sh 'cat pod.yaml | grep "image:" | sed "s/^.*: //" > images.txt'
                    sh 'while read i; do trivy image --timeout 30m0s "$i" --ignore-unfixed --exit-code 1 --severity CRITICAL; done < images.txt'
                }
            }
        }

```

Remember to define your kube-linter container in the YAML file that Jenkins will use as a template.

Finally, we can finish this, deploying the resource to the cluster, We can create a Role and Role Binding for the service account we are using with privileges to deploy resources, and then access the api like this:
```
        stage('Deploy'){
            steps {
                container('curl'){
                    sh 'curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" -X POST https://kubernetes.default.svc/api/v1/namespaces/devops-tools/pods -H "Content-Type: application/yaml" --data "$(cat simple-pod.yaml)"'
                }
            }
        }
```

We have to define a new container that has curl. Because it is not included by default in Busybox which is the base image for most of the containers we are using today. For more information on accesin the API from withing a pod, you can read the official [documentation](https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/)


### Resulting files

To wrap everything up, let's have a complete look of our two Jenkinsfiles:

```jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
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
'''
        }
    }
    stages {
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
                    sh 'trivy image --input build.tar --timeout 30m0s --ignore-unfixed --format template --template "@/contrib/html.tpl" -o reports/report.html'
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
                    sh 'trivy image --timeout 30m0s --input build.tar --ignore-unfixed --exit-code 1 --severity CRITICAL > .tmp'
                }
            }
             
            }
            
        stage('Deploy'){
            // Deploy the image to registry
            steps {
                container('img'){
                    withCredentials([string(credentialsId: 'dockerhub', variable: 'SECRET')]) {
                        sh 'img login -u rv0lt -p ${SECRET}'
                    }
                    sh 'img push rv0lt/app'
                }
            }
        }
    }
        
}

```

```jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
spec:
  containers:
  - name: curl
    image: nixery.dev/shell/curl # just a simple image with a shell and curl
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - infinity
  - name: linter
    image: steampunkfoundry/kube-linter
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - infinity 
  - name: trivy
    image: aquasec/trivy
    imagePullPolicy: IfNotPresent
    command:
    - sleep
    args:
    - infinity 
'''
        }
    }
    stages {
        stage('Syntax Scanning') {
            steps {
                container('linter') {
		                
                    sh '/kube-linter lint pod.yaml'
                }
            }
             
        }

        stage('image Scanning') {
            steps {
                container('trivy') {
                    sh 'cat pod.yaml | grep "image:" | sed "s/^.*: //" > images.txt'
                    sh 'while read i; do trivy image --timeout 30m0s "$i" --ignore-unfixed --exit-code 1 --severity CRITICAL; done < images.txt'
                }
            }
        }
            
        stage('Deploy'){
            steps {
                container('curl'){
                    sh 'curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" -X POST https://kubernetes.default.svc/api/v1/namespaces/devops-tools/pods -H "Content-Type: application/yaml" --data "$(cat simple-pod.yaml)"'
                }
            }
        }
    }
}

```

### Further security

Now that we have this deployment, we can further secure it. There are two tools that I find particulary interesting:
[Kube Bench](https://github.com/aquasecurity/kube-bench) runs a set of test to see if the cluster is configure acording to the [CIS](https://www.cisecurity.org/benchmark/kubernetes) Benchmark. If you run it in a pod it will check for misconfigurations in your node, and if you run it as a binary in the control panel, it will check the whole cluster!
And [Kube Hunter](https://github.com/aquasecurity/kube-hunter) will try to find vulnerabilities in your cluster simulating an attacker that is trying to gain privileges.  Maybe I write about them in the future.

### Conclussion

One thing that I want to highlight is that some of the tools explained today are still experimental, Kubernetes development is still pretty new and much of these tools are maybe not older than 5 years. So they are likely to evolve and change in the future, the set-up today explained may not work in the future, but at least you get the main idea.

This is the first time I write a technical post, hopefully I managed to explain everything clearly. Any suggestions you can try to contact me.

That's all. Happy coding!
