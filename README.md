# faasd-github-bot
![github-issue-bot](images/faasd-issue-bot.png)

Table of Contents
=================

* [faasd\-github\-bot](#faasd-github-bot)
* [Table of Contents](#table-of-contents)
    * [Description](#description)
    * [Prerequisites](#prerequisites)
        * [arkade](#arkade)
        * [inletsctl](#inletsctl)
        * [inlets\-pro](#inlets-pro)
        * [multipass](#multipass)
        * [faas\-cli](#faas-cli)
    * [Setup exit\-node server on AWS](#setup-exit-node-server-on-aws)
    * [Setting up a new GitHub App](#setting-up-a-new-github-app)
    * [Setup faasd](#setup-faasd)
    * [Build&amp;Deploy function](#builddeploy-function)
    * [Connect your client to the inlets\-pro server using inletsctl](#connect-your-client-to-the-inlets-pro-server-using-inletsctl)
    * [Test](#test)
    * [Cleanup](#cleanup)
* [References](#references)


### Description
In this demo, we are going to develop a Github application using Go, then we deploy it as a serverless function to make use of faasd which is a lightweight & portable faas engine.We are also going to do this demo on our local environment, so we should open our function which runs on our local environment to the Internet so Github can send events to our function.In order to do that we use inlets-pro which provides secure TCP/L4 tunnels.
### Prerequisites
> We are going to run this demo on our macbook, but you can run the demo wherever you want.
#### arkade
You can basically use arkade to install of these following tools, so we need to install arkade first.
arkade provides a portable marketplace for downloading your favourite devops CLIs and installing helm charts, with a single command.
````bash
$ curl -sLS https://dl.get-arkade.dev | sh
$ arkade version
            _             _
  __ _ _ __| | ____ _  __| | ___
 / _` | '__| |/ / _` |/ _` |/ _ \
| (_| | |  |   < (_| | (_| |  __/
 \__,_|_|  |_|\_\__,_|\__,_|\___|

Get Kubernetes apps the easy way

Version: 0.6.35
Git Commit: df53c4f6d9c604186b36aae7f0feb1d39940be8f
````
#### inletsctl
We need inletsctl to automate the task of creating an exit-node on cloud infrastructure.
````bash
$ arkade get inletsctl
$ inletsctl version
 _       _      _            _   _
(_)_ __ | | ___| |_ ___  ___| |_| |
| | '_ \| |/ _ \ __/ __|/ __| __| |
| | | | | |  __/ |_\__ \ (__| |_| |
|_|_| |_|_|\___|\__|___/\___|\__|_|

Version: 0.7.1
Git Commit: 8641a9a09903f0e3b18a4ed1d388a35663a52dc4
````
#### inlets-pro
We need inlets-pro to tunnel out any TCP traffic from an internal network to another network.
```bash
$ inletsctl download --pro
$ inlets-pro version
 _       _      _            _
(_)_ __ | | ___| |_ ___   __| | _____   __
| | '_ \| |/ _ \ __/ __| / _` |/ _ \ \ / /
| | | | | |  __/ |_\__ \| (_| |  __/\ V /
|_|_| |_|_|\___|\__|___(_)__,_|\___| \_/

PRO edition

Version: 0.7.2
Git Commit: 5190eca94b4020ec8de47a22eb411bb17a995399
```
#### multipass
We need to provision ubuntu vm's to host our faasd.In order to do that multipass cli is the best tool.
```bash
$ brew install multipass
$ multipass version
multipass  1.5.0+mac
multipassd 1.5.0+mac
```
#### faas-cli
We need faas-cli to interact with the gateway which runs on faasd.
```bash
$ arkade get faas-cli
$ faas-cli version
  ___                   _____           ____
 / _ \ _ __   ___ _ __ |  ___|_ _  __ _/ ___|
| | | | '_ \ / _ \ '_ \| |_ / _` |/ _` \___ \
| |_| | |_) |  __/ | | |  _| (_| | (_| |___) |
 \___/| .__/ \___|_| |_|_|  \__,_|\__,_|____/
      |_|

CLI:
 commit:  598336a0cad38a79d5466e6a3a9aebab4fc61ba9
 version: 0.12.21
```

Now we are ready to go 🚀

### Setup exit-node server on AWS
In order to do that, we need two things: AWS Account and inletsctl. <br/>
First we need to get our access key ID and the secret access key from our AWS Account. <br/>
To get your access key ID and secret access key:
1. Open the IAM console.
2. From the navigation menu, click Users.
3. Select your IAM user name.
4. Click User Actions, and then click Manage Access Keys.
5. Click Create Access Key.
Your keys will look something like this:
* Access key ID example: AKIAIOSFODNN7EXAMPLE
* Secret access key example: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
6. Click Download Credentials, and store the keys in a secure location.

Then, make use of the environment variables to securely store them.
```bash
$ export AWS_SECRET_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
$ export AWS_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE"
```
Finally, start your exit-node server on the eu-central-1 region.
```bash
$ inletsctl create --provider ec2 --secret-key $AWS_SECRET_KEY --pro --access-token $AWS_ACCESS_KEY --region eu-central-1
Using provider: ec2
Requesting host: charming-perlman2 in eu-central-1, from ec2
Host: i-04510810d5aa67ba2, status: creating
[1/500] Host: i-04510810d5aa67ba2, status: creating
[2/500] Host: i-04510810d5aa67ba2, status: creating
[3/500] Host: i-04510810d5aa67ba2, status: creating
[4/500] Host: i-04510810d5aa67ba2, status: creating
[5/500] Host: i-04510810d5aa67ba2, status: creating
[6/500] Host: i-04510810d5aa67ba2, status: initialising
...
inlets PRO (0.7.0) exit-server summary:
  IP: X.XXX.XXX.XXX
  Auth-token: XXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Command:
  export LICENSE=""
  export PORTS="8000"
  export UPSTREAM="localhost"

  inlets-pro client --url "wss://X.XXX.XXX.XX:8123/connect" \
        --token "XXXXXXXXXXXXXXXXXXXXXXXXXXXXX" \
        --license "$LICENSE" \
        --upstream $UPSTREAM \
        --ports $PORTS

To Delete:
          inletsctl delete --provider ec2 --id "i-04510810d5aa67ba2"


# Check your EC2 instances after the command by using aws-cli
$ aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].InstanceId" --no-cli-pager
i-i-04510810d5aa67ba3
```

### Setting up a new GitHub App

To register a new app, visit the app settings page in your GitHub profile, and click New GitHub App.

![register-github-app](images/register-github-app.png)
You'll see a form where you can enter details about your app. See "Creating a GitHub App" for general information about the fields on this page. For the purposes of this guide, you'll need to enter specific data in a few fields:

For more details you can check it out the [link](https://docs.github.com/en/free-pro-team@latest/developers/apps/setting-up-your-development-environment-to-create-a-github-app#step-2-register-a-new-github-app)

The most important parts of this form are "Webhook URL,Webhook Secret and Private keys":

* Webhook URL --> You should set the IP address of the output of the command above
![webhook-url](./images/webhook-url.png)
  
* Webhook Secret --> Define a secret for your Webhook
![webhook-secret](./images/webhook-secret.png)
  
* Private Key --> Generate and download private key for your Github Application
![private-key](./images/private-keys.png)

* Also, *do not forget to request permissions to Read & Write the repository's issues.*
![issues](./images/permissions.png)
  
Finally, we registered our application
![github-app](./images/app.png)
  
### Setup faasd
In order to get up and running with your own faasd installation on your Mac you can use multipass.

Let's start our Ubuntu VM with multipass <br/>

First, we need a cloud-config.txt to set up faasd while bootstrapping VM
```bash
$ curl -sSLO https://raw.githubusercontent.com/openfaas/faasd/master/cloud-config.txt
```
Then, we need to update the SSH key to match your own, edit cloud-config.txt:
```
$ ssh-keygen -t rsa -b 4096 -C "developerguyn@gmail.com" -f $PWD/id_rsa
```

Replace the _ssh_authorized_keys::ssh-rsa_ value with the contents of `~/.ssh/id_rsa.pub`, which is defined in `cloud-config.txt`.

Finally, boot the VM

```bash
$ multipass launch --cloud-init cloud-config.txt  --name faasd
```

Check the VM if it is working properly
```bash
$ multipass list                                                                                                                                                                                             🐳(docker-desktop)
Name                    State             IPv4             Image
faasd                   Running           192.168.64.25    Ubuntu 20.04 LTS
```

For more details you can check it out the [link](https://github.com/openfaas/faasd/blob/master/docs/MULTIPASS.md).

### Build&Deploy function
In this demo, we are going to use Go to develop our Github Application, in order to do that, first, we need to pull the corresponding function template for the Go.
```bash
# let's look at the available Go function templates within the OpenFaaS store
$ faas-cli template store list | grep -i "go"
go                       openfaas           Classic Golang template
golang-http              openfaas           Golang HTTP template
golang-middleware        openfaas           Golang Middleware template
# We are going to use golang-middleware function template, let's pull it.
$ faas-cli template store pull golang-middleware
# Then, create the function itself.
$ faas-cli new issues-bot --lang golang-middleware --prefix <DOCKER_HUB_ID>
```
After created the function, we need to define some arguments, environments and secrets for the function.
Let's add them:
```yaml
  build_args:
      GO111MODULE: on
    secrets:
      - webhook-secret # your secret goes here
      - private-key-secret # your private key goes here
    environment:
      APP_ID: "" #your app id goes here
```

Finally, we need to create those secrets above with make use of faas-cli.

Let's create our secrets
```bash
$ export WEBHOOK_SECRET="sup3rs3cr3t"
$ faas-cli secret create webhook-secret --from-literal $WEBHOOK_SECRET
# Download the private key to your host
$ faas-cli secret create private-key-secret --from-file <path_to_your_pem_file>.pem
```

We should create a secret in faasd, in order to do that we need to access the Gateway of faasd
```bash
$ export IP=$(multipass info faasd --format json| jq '.info.faasd.ipv4[0]' | tr -d '\"')
# Let's capture the authentication password into a file for use with faas-cli
$ ssh ubuntu@$IP "sudo cat /var/lib/faasd/secrets/basic-auth-password" > basic-auth-password
# Login from your laptop (the host)
$ export OPENFAAS_URL=http://$IP:8080 && \
cat basic-auth-password | faas-cli login -s
```

### Connect your client to the inlets-pro server using inletsctl
We need to establish connection between our client, and the inlets-pro server in order to get events from there
```bash
$ export UPSTREAM=$IP # faasd gateway ip, we have already grap the URL above
$ export PORTS=8080 # faasd gateway port
$ export LICENSE="eyJhbGciOiJFUzI..."

# Notice that this command is the output of the "inletsctl create" command above
$ inlets-pro client --url "wss://XX.XXX.XXX.XX:8123/connect" \
        --token "$TOKEN" \
        --license "$LICENSE" \
        --upstream $UPSTREAM \
        --ports $PORTS
```
### Test
In order to test it we need to install this app to selected repositories. Create a repository called "test-issues-bot", then install this app for it.
![repository-access](./images/repository-access.png)

Then, create an issue for the repository. You will see the message 
> "Hello, issue opened by: developer-guy"

![test-issue-bot](./images/test-issue-bot.png)
![close-issue](./images/close-issue.png)

### Cleanup
```bash
$ multipass delete faasd
$ multipass purge
$ inletsctl delete --provider ec2 --id "YOUR_INSTANCE_ID" --access-token $AWS_ACCESS_KEY --secret-key $AWS_SECRET_KEY --region eu-central-1
```

### Next steps
You can run this Bot on a Raspberry Pi, with faasd and TLS with inlets-pro and Caddy! Checkout this [README](./docs/faasd_on_pi.md) to know how.

Yaaay 🎉🎉🎉🎉🎉🎉🎉
# References
* https://blog.alexellis.io/deploy-serverless-faasd-with-cloud-init/
* https://www.x-cellent.com/blog/automating-github-with-golang-building-your-own-github-bot/
* https://blog.alexellis.io/share-work-using-inlets/
