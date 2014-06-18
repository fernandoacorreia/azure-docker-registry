# azure-docker-registry

Tools to install and run a production-quality Docker private registry on Azure.

THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND. SEE THE LICENSE FILE FOR MORE INFORMATION.

## Features

* Automated deployment
* Custom domain
* HTTP Basic Authentication
* HTTPS
* Locally and geographically redundant storage
* Separation of data, application and web server in individual containers

## Instructions

These scripts and instructions have been tested on Linux Mint 16. They should work on Ubuntu distributions. If you're running Windows, OS X, or other OS they may require adjustments. Alternatively, you could run them inside a VirtualBox VM running Ubuntu Server, or under Cygwin.

### Step 1: Certificates

The repository will be accessible exclusively via HTTPS and protected via HTTP Basic Authentication. This requires a SSL certificate.

You can aquire a certificate with a provider of your choice, request a free one at [StartSSL](http://www.startssl.com/), or create a self-signed certificate. This last option may have some issues, unless you configure the operating system of your client computer to trust the self-signed certificate. Since this project is targeted at production-quality deployments, it will assume that there is a valid SSL certificate for the domain chosen for the registry server.

Either way, save a copy of your certificate (concatenated with the proper CA bundle) and its respective unencrypted key to these exact locations:

* Certificate: `/tmp/docker-registry/ssl.crt`
* Key: `/tmp/docker-registry/ssl.key`

**Important:** For greater security, set the owner of these temporary files to `root` and the mode to `0600`.

At the end of the deployment process, these files will be removed.

### Step 2: Work environment

The automation in this project depends on several tools and configuration files. Instead of trying to provide instructions to setup your host with these dependencies, a  Docker image is provided with a preconfigured work environment. This project's scripts are assumed to be executed in this work environment.

Create the work environment image with this command:

```
$ scripts/build-workenv
```

This will take a few minutes, but since you'll be entering your Azure credentials and production keys and certificates in this environment, it's more secure to build it yourself than to rely on a published image. The work environment will be based on the generic [phusion/baseimage-docker](https://registry.hub.docker.com/u/phusion/baseimage/) image.

Run the work environment in a Docker container with this command:

```
$ scripts/run-workenv
```

To exit the work environment, simply type `exit` at the prompt. Use the `run-workenv` command again to run the same container, with the files just as they were when you exited it.

### Step 3: Azure authentication

In the work environment, connect to your Azure subscription.

If you have an organizational account with Azure, use the following command:

```
# azure login [username] [password]
```

If your Azure credentials are based on a Microsoft account, use the following command to download your publishing settings:

```
# azure account download
```

Open in your browser the URL mentioned in the output from the previous command
(e.g. http://go.microsoft.com/fwlink/?LinkId=254432). Authenticate with Azure and download your
publishing profile.

On your host machine (not in the work environment), move the downloaded publishing profile to the `/tmp` directory:

```
$ mv ~/Downloads/*.publishsettings /tmp
```

Back in the work environment, import the publish settings file. Notice that the `/tmp` directory on your host machine is mapped to `/mnt/tmp` in the work environment:

```
# azure account import /mnt/tmp/*.publishsettings
```

Finally, for security reasons, remove the publishing profile from the file system:

```
# rm /mnt/tmp/*.publishsettings
```

### Step 4: Configuration

List the Azure subscriptions that can be managed by your credentials:

```
# azure account list
```

If there is more than one, pick the one that you want to use.

Copy the subscription ID to the clipboard.

Edit the configuration file:

```
# vi /root/config
```

Insert the azure subscription ID at the proper place, and set the other configuration options as explained in the configuration file's comments.

Save the configuration file (ESC :wq).

### Step 5: Provisioning

Create the Azure virtual machine and related assets with this command:

```
# /mnt/tools/scripts/create-registry-server
```

This process should take a few minutes. It may fail, depending on network connectivity issues and on the availability and responsiveness of Azure services.

If it does fail, before retrying you will have to manually clean up, deleting the resources that were created. To do this, use the Azure management portal
(http://manage.windowsazure.com) and look for resources with names starting with the name you specified in the VM_NAME configuration option.
This tool does not attempt to automatically clean up partially created environments, because this would be error-prone and risky.

**Important:** When the process finishes, follow the instructions displayed to move the key files with the new virtual machine's credentials from their temporary
location into your `.ssh` directory. Be sure to backup these files, since they are required to manage the virtual machine.

To check if the new VM has booted up and is ready to be set up, use this command:

```
# azure vm list
```

The VM will be ready when its status is "ReadyRole".

### Step 6: Setup

Once the virtual machine is provisioned, it has to be set up. This will be done using Ansible in the work environment to run commands on the Azure virtual machine
via ssh.

To start the setup process execute this command:

```
# /mnt/tools/scripts/setup-registry-server
```

This may take about half an hour. When it finishes successfully, it should output a message like this:

```
PLAY RECAP ********************************************************************
vmname.cloudapp.net : ok=31   changed=27   unreachable=0    failed=0
```

### Step 7: Verification and cleanup

Exit the work environment by typing `exit` at the prompt, and execute the following operations on your host:

* Check that the new Docker registry is working in the domain that you specified by opening its URL in a browser and entering the authentication credentials that you specified in the configuration. It should display a message like this:

```
"docker-registry server (production) (v0.7.2)"
```

* Check that you can work with the registry server:

```
$ docker login https://domain.example.com
$ docker pull busybox
$ docker tag busybox domain.example.com/busybox
$ docker push domain.example.com/busybox
$ docker rmi domain.example.com/busybox
$ docker pull domain.example.com/busybox
```

* Check that you can access the registry virtual machine via ssh with the key that you moved to the `~/.ssh` directory in your host:

```
$ ssh -i ~/.ssh/vmname.key vmuser@vmname.cloudapp.net
```

If you get the error "Agent admitted failure to sign using the key" try using `ssh-add` to authorize the key.

* Ensure that the certificate and key files that you copied to `/tmp/docker-registry` in the host have been deleted:

```
# ls /tmp/docker-registry
```

* Delete the temporary work environment container from your host, so that the keys and secrets that were stored in it are not kept around:

```
$ docker rm azure-docker-registry-work-environment
```

### Step 8: Administration

To manage the registry host, connect to it via ssh, using the private key that was generated during the deployment process:

```
$ ssh -i ~/.ssh/vmname.key vmuser@vmname.cloudapp.net
```

You can use regular Ubuntu and Docker administrative commands. Additionally, some scripts are provided as shortcuts:

* `sudo /opt/registry/start`: Starts the registry container.
* `sudo /opt/registry/stop`: Stops the registry container.
* `sudo /opt/nginx/start`: Starts the nginx container.
* `sudo /opt/nginx/stop`: Stops the nginx container.
* `sudo /opt/nginx/ssh`: Connects to the nginx container via ssh.

For instance, to view nginx error logs:

```
$ ssh -i ~/.ssh/vmname.key vmuser@vmname.cloudapp.net
$ sudo /opt/nginx/ssh
# tail /var/log/nginx/error.log
```

Since the deployment is done with independent containers, you can upgrade the registry server by replacing the `docker_registry` container with a new image, without losing the data stored in a volume in the `docker_registry_data` container, and without affecting nginx configurations stored in the `docker_registry_nginx` container. See the script at `/opt/registry/create-registry-container` for an example of how to recreate the container. As usual, be sure to backup the data before maintenance.

The registry data is stored in the `/vol/docker-registry` volume of the `docker_registry_data` container.

Docker files are placed in `/mnt/data/docker`, which is stored in an Azure data disk, redundant both locally (multiple copies in the same datacenter) and geographically (multiple copies in a datacenter in a different geographic region).

It is recommended to backup the `/mnt/data/docker` directory for additional protection, such as the ability to restore deleted images.
