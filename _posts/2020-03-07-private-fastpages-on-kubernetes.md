---
title: "Private FastPages on Kubernetes"
summary: "How I deployed FastPages as a tool for our data tribe to share information and insights on the private Kubernetes cluster at my workplace"
toc: true
categories: [fastpages, kubernetes]
---

# Private FastPages on K8S
> How I deployed FastPages as a tool for our data tribe to share information and insights on the private Kubernetes cluster at my workplace
> 

## Context
> Note: if you are not part of a mid-large company IT department, the following paragraph might sound like a bunch of corporate lingo that makes no sense. It is not completely true, but if you have no idea what I am talking about, just skip it.
> 
As a mathematician turned knowledge engineer turned product manager turned data scientist with a pench for learning new stuff, one of the many hats I tend to wear quite often is that of knowledge sharer. Since in my current role I am helping setting up a federated team of data analysts, one of the problems we need to solve is breaking the information siloes among data people. This way we can avoid duplicate work and we can piggyback on each others' research in order to extract more advanced insights. 

Since I am a bit of a Fast.ai fanboy and I am using Jupyter a lot in my data work, when fast_template/FastPages was presented, I started thinking about how to deploy it for my use case. The main blocker was that GitHub pages are always public, even on private repos, which would have been a showstopper, as we don't want to police a self serving tool to check that no important information is being shared publicly.

## Summary (and/or TL;DR)

In order to deploy FastPages privately on a Kubernetes cluster you need to:
1. Change the branch to which the website built by Jekyll is pushed
2. Deploy a static webpage server (I used Nginx) with a Git-sync sidecar to keep the served site up to date
3. Makesure the sidecar can pull the fastpages repo with a read only deploy key
4. Expose the server internally with appropriate network policies and ingress configurations

While steps 1-3 are fairly straightforward, the last step depends heavily on your cluster configuration and policies. I will give a very high level description of what I did, but you will have to do your own research to make it work in your case. 

### Requirements
This guide is not too beginner friendly and it is not meant to be: since the risk of getting it wrong is exposing private information, you need to have some working knowledge of what you are doing or at least some way of making sure that you are not doing huge mistakes (like a SRE/DevOps person to review what you are doing). Other than that you need
- Access and admin privileges to private GitHub repos
- The ability of running GitHub actions on the private repos
- A Kubernetes cluster set up with appropriate network policies and DNS
- A namespace that can access the private GitHub repo. This is usually the case, but if your cluster is super locked down, it might not be possible

> Important: to make the deployment, you need to know your way around Kubernetes. You just need to know how to use it and deploy/manage resources into it, you don't need cluster admin knowledge of any kind
> 

## Setting up FastPages

First of all you have to setup your FastPages repo. Just follow the [latest instructions](https://github.com/fastai/fastpages#setup-instructions) (keep in mind that the tool is under active development, so those might change quite often); this will trigger a GitHub action that will open a PR. Before following the instructions in the PR, there is a few things to do to avoid fastpages publishing private stuff by mistake.
1. Set a [branch protection rule](https://help.github.com/en/enterprise/2.15/admin/developer-workflow/configuring-protected-branches-and-required-status-checks) to make sure that at least two approving reviews are required to push to the `gh-pages`. This is because by default, as soon as something is pushed to the `gh-pages` branch, it gets published on GitHub pages. As far as I know, there is no way to prevent this behaviour.
2. Checkout into the branch that has been opened by the Setup action (the one that the open PR is attempting to merge into master) and modify the `.github/workflows/ci.yaml` from this
```yaml
    - name: Deploy
      if: github.event_name	== 'push'
      uses: peaceiris/actions-gh-pages@v3
      with:
        deploy_key: ${{ secrets.SSH_DEPLOY_KEY }}
        publish_dir: ./_site
```
to something like this (you can pick whatever branch name you want)
```yaml
    - name: Deploy
      if: github.event_name	== 'push'
      uses: peaceiris/actions-gh-pages@v3
      with:
        deploy_key: ${{ secrets.SSH_DEPLOY_KEY }}
        publish_branch: private-website-branch
        publish_dir: ./_site
```

After this you are can continue setting up the repo according to the instructions in the PR.

This is all that is needed on the FastPages side of things.

### Gotchas and other optional steps
If you want the website to function in any useful way, you will have to set up an internal domain. How to do so depends heavily on how the DNS and network policies are setup in your Kubernetes cluster. If your domain is going to be, for example `shiny.private.website`, you want to make sure to  that
- Your CNAME file (so that the categories work properly, for example) contains `shiny.private.website`
- The `_config.yml` file contains
```yaml
url: "https://shiny.private.website" # the base hostname & protocol for your site, e.g. http://example.com
baseurl: ""
```

Before you merge the setup PR (or right after that), it is also a good idea to create an `upstream` branch and to set its remote to the [original FastPages repo](https://github.com/fastai/fastpages.git), so that you can keep your deployment in sync with the development of FastPages. 

Don't do it later as I did. It's a pain to solve the merge conflicts.

## Deploying the server
Since Jekyll builds a complete static website and the github action pushes it to the branch we have set up in the step above, we only need something capable of serving it. I have used a basic [Nginx alpine](https://hub.docker.com/_/nginx) image, but there are probably a thousand different options. In order to avoid having to redeploy the server manually everytime, furthermore, we want to add a [git-sync](https://github.com/kubernetes/git-sync) sidecar that pulls the website branch into the served folder of the Nginx container and keeps it up to date. There are a few possible variations but this is how more or less how the manifest would look like

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-namespace
spec:
  selector:
      matchLabels:
          app: fastpages
  replicas: 1 # You can set this to something higher if needed
  template:
    metadata:
        labels:
            app: fastpages
    spec:
        restartPolicy: Always
        securityContext:
            fsGroup: 65533 # to make SSH key readable 
        containers:
        - name: fastpages
          image: nginx:alpine
          imagePullPolicy: Always
          volumeMounts:
          - name: site
            mountPath: /usr/share/nginx/html
          - mountPath: /etc/nginx/conf.d
            name: fastpages-conf
          ports:
          - containerPort: 80
            protocol: TCP
          resources:
            limits:
              memory: 20Mi
        - name: git-sync
          image: k8s.gcr.io/git-sync
          imagePullPolicy: IfNotPresent
          env:
            - name: GIT_SYNC_REPO
              value: "git@github.com:your-githubname/yourprivaterepo.git"
            - name: GIT_SYNC_DEST
              value: "www"
            - name: GIT_SYNC_ROOT
              value: "/site"
            - name: GIT_SYNC_SSH
              value: "true"
            - name: GIT_SYNC_MAX_SYNC_FAILURES
              value: "5"
          resources:
            requests:
              cpu: 0m
              memory: 0Mi
            limits:
              memory: 200Mi
          securityContext:
            runAsUser: 65533 # git-sync user
          volumeMounts:
          - name: git-secret
            mountPath: /etc/git-secret
          - name: site
            mountPath: /site
        volumes:
        - name: site
          emptyDir: {}
        - configMap:
            defaultMode: 420
            name: fastpages-conf
          name: fastpages-conf
        - name: git-secret
          secret:
            secretName: fastpages-git-ssh
            defaultMode: 0400
---
apiVersion: v1
kind: Service
metadata:
  name: fastpages
spec:
  selector:
    app: fastpages
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
    name: fastpages-conf
    namespace: your-namespace
data:
    default.conf: |-
        server {
            listen       80;
            server_name _;
            root /usr/share/nginx/html/www/;
            access_log /dev/stdout;
        }
---
```

Before deploying the above, generate a [new SSH key pair](https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key) and create a [secret with the private key](
https://kubernetes.io/docs/concepts/configuration/secret/#use-case-pod-with-ssh-keys) in your namespace called `fastpages-git-ssh`. After that use the **public** key of the pair to create a new read only [deploy key](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys) on your FastPages repo.

Once all this is done, you can deploy the above and you webserver container should start happily. Sadly, you will not be able to access it yet if your namespace has properly set policies.

> Important: Please be careful of how you manage your secrets, don't put them on GitHub unencrypted and don't do anything weird.
> 

## Making the server accessible
In order to be expose the served webpages there are probably a couple more resources to be deployed. Both of these are dependent on how your Kubernetes cluster has been set up, so there is not much I can do to help you. In order to make everything work you need
1. An [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) to expose the HTTP route to the DNS and within the cluster
2. A [Netork Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) to make sure that the deployed resources are allowed to communicate among them, if this is not already possible by default in your namespace

Once again, if you don't knwo how to do this, you will have to consult your SRE/DevOps/SystemAdmin team or whoever is maintaining the cluster. Please don't follow the advice of a random Data Scientist to set up network policies for potentially sensitive resources.

## Parting thoughts
If you are reading this, you should at this point have a deployed fastpages powered private website on your Kubernetes cluster. Although the recipe is quite verbose, depending on your experience with Kubernetes, the steps might be all quite familiar. If that is not the case, I hope you managed to at least an idea of what is going on.
