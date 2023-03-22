## Does size matter? Lessons from my first Scratch/Distroless image 

### Introduction

One of the first concepts that you can encounter when building a Docker image is, how to reduce (and optimize) the size of it. I mean, it makes sense, less image size equals less building time, less disk space and, in general, faster images to use, push and pull. And about this topic I want to talk today. Because, as always in live, only a Sith deals in absolutes. And the circumstances that apply to developer A are not always the same that apply to developer B, lets deep into it. The code presented today is available in [my github](https://github.com/rv0lt/CurriculumDeployment).

### Big vs Small Images

So, let's start from the beginning. You have just developed your first application (in Python, Node, or whatever), and you decided to bundle it in Docker because you are always hearing about Docker, Kubernetes and such fancy new words. Ok, you search a little bit how to do it and you start with 

```
FROM python
```

```
FROM node
```
 
 Simple and effective. What's the problem then? Well, the following image from [Synk](https://snyk.io/) will tell you

![enter image description here](https://cdn.hashnode.com/res/hashnode/image/upload/v1597078122646/BdLr41ztH.png?auto=compress,format&format=webp)
What this is telling us is, a small specialized image, like node, is much more dangerous that using the Ubuntu OS image as our base?

Well, yes and no. I am being purposely misleading here. See, the root of this issue is the base image each one is using. Let's check that with [Trivy](https://github.com/aquasecurity/trivy)

![enter image description here](https://i.imgur.com/gaG4y38.png)
![enter image description here](https://i.imgur.com/HEZmRG6.png)
Here, it was clearly shown how the base image of Debian is insecure. If, instead, we were using Alpine as the base image for node:

![enter image description here](https://i.imgur.com/1uiMaZ8.png)
Alpine is, generally, very useful and recommended. A lightweight, fast and security-focus Operating System. However, it has its own limitations and issues that one need to be aware of, for starters, uses a different standard GNU library, but that's a story for [another day.](https://wiki.musl-libc.org/functional-differences-from-glibc.html)

Returning to our issue. For node, we could initially solve our concern with this tag. But not every image as an alpine tag. Python does not, neither does gcc (for building C programs). We could make use of multi-stage building and do something like this:

```Dockerfile
FROM alpine AS build
RUN apk add build-base # utilities with basic libraries to compile
COPY code.c .
RUN gcc -o code code.c #compile our code

FROM alpine
COPY --from=build code .
CMD ["./code"]
```
Or install the Python3 and pip packages for [Python.](https://stackoverflow.com/questions/62554991/how-do-i-install-python-on-alpine-linux)

But now, let's go one step forward, introducing scratch and [distroless](https://github.com/GoogleContainerTools/distroless). These are images, which are empty, they have been stripped of everything, including a shell. Which means, that, to use them you need always another image as base/builder.

```Dockerfile
FROM python as base 

FROM SCRATCH 
COPY --from=base /usr/local/lib/ /usr/local/lib/
COPY --from=base /usr/local/bin/python /usr/local/bin/python
COPY --from=base /etc/ld.so.cache /etc/ld.so.cache
COPY script.py .
CMD ["/usr/local/bin/python","script.py"]
```

However, as we will see in the following section, there are a lot more things to consider. First, every shared system library that the code calls needs to be copied as well for it to work (in a compile language like C we can use the -static flag to include the libraries in the binary). Also, as I mentioned, there is no shell, so debugging would actually be pretty difficult. In Kubernetes we could use [ephemeral containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container), however at the writing of this post, that requires access to alpha features. 

So, my take on this is the following: distroless is definitely a go-to. The less attack surface and the less image size the better. But, if you are just starting to learn it can be pretty overwhelming seeing such Dockerfiles as we will see in the next section. Also, without a shell, debugging is a more difficult and tedious task. There are pretty minimalistic images where we can also run our applications such as alpine and busybox, that also include some basic tools, and it should be fine. If you and your application can manage to maintain a longer Dockerfile, you have another way to debug it without a shell (or don't need to do it), then go for it. It will all depend on your use case and circumstances. But is good that, at least, you understand the differences between them

### Nginx on Scratch (really scratched my head)

Now, let's go for the practical case. The idea is pretty simple, we are going to try to move an application to run on Scratch. This would be our bucket list:

 - Because Scratch is empty, we need, firstly, a binary that we will exec. A note here, because we don't have a shell the command to run it should be passed as json. If you try to run the command directly it will fail because, under the hood, docker tries to call sh i.e. a shell
 ```Dockerfile
CMD  /binary -options #no
CMD  ["/binary","-options"] #yes
```
 -  If our binary is a simple app, that's all we need. However, for a bigger app, they probably make use of system libraries or other libraries you have downloaded. I mentioned this in the previous section, a good way to tell which libraries a program uses is [ldd](https://linuxhint.com/use-ldd-command-in-linux/). In the nginx example, an issue I faced is that, because it is built on [modules](https://www.nginx.com/resources/datasheets/nginx-modules/) then there are libraries that are not requiered to execute the binary, but some default modules do.
 -  Apart from libraries, there are other system files and folder that our application may need. For example the [passwd](https://en.wikipedia.org/wiki/Passwd) file stores important information about the users in the system. If we want to run our app as a non-root user (as we should do) then we need this file. 
 - Depending on the application we may need other files, in our nginx example. I run into an issue if I didn't have a /run directory. Because there is where nginx stores it's process.pid

So let's get into it. My idea is a simple nginx that will serve in localhost:8080/cv my resume (I'm looking for jobs, if you like what you are reading and want a Junior Devops, SRE, Data Engineer, you can [contact me](alvaro.revuelta.martinez@gmail.com)). And if we try to access the main page, localhost:8080 it will redirect to this blog.

First, modify the default conf file of nginx to something like this:
 ```Conf
location / {
	rewrite ^/$ https://rv0lt.github.io/ redirect;
}

location /cv {
	alias /usr/share/nginx/resume.pdf;
	default_type application/pdf;
}
```

Now, the funny part:

For our dockerfile, we will be using as base the [ubuntu/nginx](https://hub.docker.com/r/ubuntu/nginx) image. Why? Well, mostly because the official nginx image as a lot of built-in vulnerabilities. Even though the reason of all this is to get rid of them by using Scratch, I feel it is better to use a safer base if we have one available
 ![enter image description here](https://i.imgur.com/cwiqrPc.png)
![enter image description here](https://i.imgur.com/b93HFew.png)
Even the alpine nginx image as several critical vulnerabities!
![enter image description here](https://i.imgur.com/goXy5Qp.png)
Ok, solved this, let's start to write our dockerfile:
 ```Dockerfile
FROM  ubuntu/nginx  as  base

#copy both our resume and the new config files
COPY  ./files/cv/resume.pdf  /usr/share/nginx
COPY  ./files/default.conf  /etc/nginx/conf.d/default.conf

#create empty run directory
RUN  mkdir  -p  /run/run

FROM  scratch

#we need the run directory to store the process.pid file
COPY  --from=base  /run/run/  /run/

#copy the users and groups files
COPY  --from=base  /etc/group  /etc/group
COPY  --from=base  /etc/passwd  /etc/passwd
```

Until here, pretty straightforward. We have copied the files needed, and then, moved the files and folders that the nginx needs to work. Next step are the libraries, here is where I spent more time. Because if we try to use ldd to find the libraries:

 
![enter image description here](https://i.imgur.com/Tsp2pRM.png)
It will tell use, those. However, as I mentioned, nginx is built using modules. So those modules use also more libraries:
![enter image description here](https://i.imgur.com/XqHPrCQ.png)
Another option would have been to download, in a ubuntu base image, the source code and compile it ourselves using the -static flag, so that the binary can contain the libraries already.

So, the second part of the Dockerfile would be:

```Dockerfile
# COPY the libraries needed to run nginx
# Some libraries are obtained using ldd. The rest are requieres libraries used by the actived modules by default

COPY  --from=base  /lib/x86_64-linux-gnu/libgcc_s.so.1  /lib/x86_64-linux-gnu/libgcc_s.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libstdc++.so.6  /lib/x86_64-linux-gnu/libstdc++.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libicudata.so.71  /lib/x86_64-linux-gnu/libicudata.so.71

COPY  --from=base  /lib/x86_64-linux-gnu/libc.so.6  /lib/x86_64-linux-gnu/libc.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/liblzma.so.5  /lib/x86_64-linux-gnu/liblzma.so.5

COPY  --from=base  /lib/x86_64-linux-gnu/libicuuc.so.71  /lib/x86_64-linux-gnu/libicuuc.so.71

COPY  --from=base  /lib/x86_64-linux-gnu/libxml2.so.2  /lib/x86_64-linux-gnu/libxml2.so.2

COPY  --from=base  /lib/x86_64-linux-gnu/libcap-ng.so.0  /lib/x86_64-linux-gnu/libcap-ng.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libaudit.so.1  /lib/x86_64-linux-gnu/libaudit.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libpam.so.0  /lib/x86_64-linux-gnu/libpam.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libcrypt.so.1  /lib/x86_64-linux-gnu/libcrypt.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libpcre.so.3  /lib/x86_64-linux-gnu/libpcre.so.3

COPY  --from=base  /lib/x86_64-linux-gnu/libssl.so.3  /lib/x86_64-linux-gnu/libssl.so.3

COPY  --from=base  /lib/x86_64-linux-gnu/libcrypto.so.3  /lib/x86_64-linux-gnu/libcrypto.so.3

COPY  --from=base  /lib/x86_64-linux-gnu/libz.so.1  /lib/x86_64-linux-gnu/libz.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libc.so.6  /lib/x86_64-linux-gnu/libc.so.6

COPY  --from=base  /lib64/ld-linux-x86-64.so.2  /lib64/ld-linux-x86-64.so.2

COPY  --from=base  /lib/x86_64-linux-gnu/libm.so.6  /lib/x86_64-linux-gnu/libm.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libgcc_s.so.1  /lib/x86_64-linux-gnu/libgcc_s.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libmaxminddb.so.0  /lib/x86_64-linux-gnu/libmaxminddb.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libgd.so.3  /lib/x86_64-linux-gnu/libgd.so.3

COPY  --from=base  /lib/x86_64-linux-gnu/libpng16.so.16  /lib/x86_64-linux-gnu/libpng16.so.16

COPY  --from=base  /lib/x86_64-linux-gnu/libfontconfig.so.1  /lib/x86_64-linux-gnu/libfontconfig.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libfreetype.so.6  /lib/x86_64-linux-gnu/libfreetype.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libjpeg.so.8  /lib/x86_64-linux-gnu/libjpeg.so.8

COPY  --from=base  /lib/x86_64-linux-gnu/libuuid.so.1  /lib/x86_64-linux-gnu/libuuid.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libexpat.so.1  /lib/x86_64-linux-gnu/libexpat.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libwebp.so.7  /lib/x86_64-linux-gnu/libwebp.so.7

COPY  --from=base  /lib/x86_64-linux-gnu/libXpm.so.4  /lib/x86_64-linux-gnu/libXpm.so.4

COPY  --from=base  /lib/x86_64-linux-gnu/libtiff.so.5  /lib/x86_64-linux-gnu/libtiff.so.5

COPY  --from=base  /lib/x86_64-linux-gnu/libbrotlidec.so.1  /lib/x86_64-linux-gnu/libbrotlidec.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libX11.so.6  /lib/x86_64-linux-gnu/libX11.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libzstd.so.1  /lib/x86_64-linux-gnu/libzstd.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libLerc.so.3  /lib/x86_64-linux-gnu/libLerc.so.3

COPY  --from=base  /lib/x86_64-linux-gnu/libjbig.so.0  /lib/x86_64-linux-gnu/libjbig.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libdeflate.so.0  /lib/x86_64-linux-gnu/libdeflate.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libbrotlicommon.so.1  /lib/x86_64-linux-gnu/libbrotlicommon.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libxcb.so.1  /lib/x86_64-linux-gnu/libxcb.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libXau.so.6  /lib/x86_64-linux-gnu/libXau.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libXdmcp.so.6  /lib/x86_64-linux-gnu/libXdmcp.so.6

COPY  --from=base  /lib/x86_64-linux-gnu/libbsd.so.0  /lib/x86_64-linux-gnu/libbsd.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libmd.so.0  /lib/x86_64-linux-gnu/libmd.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libxslt.so.1  /lib/x86_64-linux-gnu/libxslt.so.1

COPY  --from=base  /lib/x86_64-linux-gnu/libexslt.so.0  /lib/x86_64-linux-gnu/libexslt.so.0

COPY  --from=base  /lib/x86_64-linux-gnu/libgcrypt.so.20  /lib/x86_64-linux-gnu/libgcrypt.so.20

COPY  --from=base  /lib/x86_64-linux-gnu/libgpg-error.so.0  /lib/x86_64-linux-gnu/libgpg-error.so.0
  
# copy the files that nginx needs to run

COPY  --from=base  /var/lib/nginx/  /var/lib/nginx/
COPY  --from=base  /usr/lib/nginx/  /usr/lib/nginx/
COPY  --from=base  /usr/share/nginx/  /usr/share/nginx/
COPY  --from=base  /var/log/nginx/  /var/log/nginx/
COPY  --from=base  /etc/nginx/  /etc/nginx/
COPY  --from=base  /sbin/nginx  /sbin/nginx


EXPOSE  8080

# no need to use the USER directive bc the nginx process already runs as a non-priviledged user and we have no shell
CMD  ["/sbin/nginx","-g",  "daemon  off;"]
```

And we have it, it works. The complete file is avaible [here](https://raw.githubusercontent.com/rv0lt/CurriculumDeployment/main/files/Dockerfile)


![enter image description here](https://i.imgur.com/yqbsBCt.png)

For completion, I have also written a YAML file for a Kubernetes deployment using this image. You can check it [here](https://github.com/rv0lt/CurriculumDeployment/blob/main/kube/deployment.yaml). Also, this image is uploaded to my [DockerHub](https://hub.docker.com/repository/docker/rv0lt/mycv/general).

To finalize, let's analyze it:
![enter image description here](https://i.imgur.com/SeKf4N2.png)

There are two last things I want to mention, the first is that, despite we have gained a more secure deployment, you can see that the file is really long and looks a bit wacky. I spent a lot of time fixing the problems with the libraries and other problems with files. Don't get me wrong, I learned a lot with this project, but I can understand that developing such files always not always be optimal. What I have learned is that there is a tradeoff here, you can build a safer environment at the cost of more developing time and a more difficult to maintain file.

The second thing is that, you shouldn't be tricked. On the one hand, the image is safer than our base, of course. We have a minimal image with just the things we need. No shell, no other tools. But, remember that we copied the libraries from our base, if those libraries are vulnerable, so our image will be. The reason trivy wouldn't catch them is that we have no way of listing them, it is a Scratch image. So it is impossible to compare against the CVE database. In this case, it can be considered security through obscurity.

### Bonus: CD with GitHub Actions

Ok, just for fun, let's end this by connecting our deployment with GitHub Actions, for the ones that don't know it. GitHub Actions is a CI/CD pipeline integrated in GitHub. Every time you push a file, it calls the pipeline and executes the steps defined. You can learn the basics of setting it up with this two post, [GitHub Actions](https://docs.docker.com/build/ci/github-actions/) and [Manage Access Tokens](https://docs.docker.com/docker-hub/access-tokens/#create-an-access-token)

In a nutshell, we need to create a folder .github/workflows and define a yaml there. I will also add a step to build our pdf from latex. So now, everytime I commit a change to my resume files. It will automatically build a pdf and update the image. So cool!

This is how our actions.yml will look like:

 ```YAML
name: pipeline

on:
	push:
		branches:
			- "main"

jobs:
	build:
		runs-on: ubuntu-latest
		steps:
			-
				name: Checkout
				uses: actions/checkout@v3
			-
				name: Build-latex
				uses: xu-cheng/latex-action@v2
				with:
					root_file: resume.tex
					working_directory: files/cv/
			-
				name: Login to Docker Hub
				uses: docker/login-action@v2
				with:
					username: ${{ secrets.DOCKERHUB_USERNAME }}
					password: ${{ secrets.DOCKERHUB_TOKEN }}
			-
				name: Set up Docker Buildx
				uses: docker/setup-buildx-action@v2
			-
				name: Build img and push
				uses: docker/build-push-action@v4
				with:
					context: .
					file: ./files/Dockerfile
					push: true
					tags: ${{ secrets.DOCKERHUB_USERNAME }}/mycv:latest

```



![enter image description here](https://i.imgur.com/TcP4byJ.png)

So here you have it. I hope you have understood everything I tried to explain. And if you are hiring, you already have my resume! ;) 
