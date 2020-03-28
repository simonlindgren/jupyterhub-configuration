# JupyterHub on Linode

### 1. Prepare the machine
Create an account at [linode.com](http://linode.com) and create a linode with the specs you want (and [can afford](https://www.linode.com/pricing/)). For this tutorial I used Ubuntu 18.04 LTS.

In the dashboard for your linode, under 'Networking', you will be see its ip address.

Connect to the linode:

```
$ ssh root@xxx.xx.xxx.xx 
```

### 2. Set up Python
Set up a Python 3 environment with the help of Anaconda.

Download: `wget https://repo.continuum.io/archive/Anaconda3-2020.02-Linux-x86_64.sh`

Install: `bash Anaconda3-2020.02-Linux-x86_64.sh`

Choose to install under `opt`: `[/root/anaconda3] >>> /opt/anaconda3`

Say `yes` to conda initialisation.

When asked, say `yes` to `conda init`.

Remove the installer: `rm Anaconda3-2020.02-Linux-x86_64.sh`

Activate anaconda: `source /opt/anaconda3/bin/activate`

### 3. Install JupyterHub
```
apt update
apt install python3-pip
conda install nodejs -c conda-forge
conda install jupyterhub
conda install jupyterlab
jupyter labextension install @jupyterlab/hub-extension
```
All users of the hub must have access to `jupyterhub`, so also do: `pip3 install jupyterhub`

Check that it is now also installed in `/usr/local/bin`, using: `which -a jupyterhub`


### 4. Configure JupyterHub
Your domain name (<your.address>, e.g. http://myhubserver.com) must be set up with your hosting provider to point to the static ip address of the server.

We must set up SSL on the server to be able to use https:

```
apt install letsencrypt
certbot certonly
```
Choose: `1: Spin up a temporary webserver (standalone)`, then go through the generation process.

We now have certificates in the `/etc/letsencrypt/live/<your.address>` folder: `fullchain.pem` is the certificate, and `privkey.pem` is the key file.

Now generate a config file for Jupyterhub. I cd’d back up to root (`cd /`), and created it there:

```
jupyterhub --generate-config 
```
Edit the config file:
```
nano jupyterhub_config.py
```
Add these things to it: 

```
# CONFIGURATION FILE FOR JUPYTERHUB
# Set up users
c.Authenticator.admin_users = {'<name-of-your-first-admin-user>'}
# Set up web stuff
c.JupyterHub.ip = '<your.address>'
c.JupyterHub.port = 443
c.JupyterHub.ssl_key = '/etc/letsencrypt/live/<your.address>/privkey.pem'
c.JupyterHub.ssl_cert = '/etc/letsencrypt/live/<your.address>/fullchain.pem'
c.JupyterHub.cleanup_servers = True
# Set up spawner
c.Spawner.default_url = '/lab'
c.Spawner.cmd = ['jupyter-labhub']
c.Spawner.notebook_dir = '~/notebooks'
```

Make the first user:
`useradd -m <name-of-your-first-admin-user>`

`passwd <name-of-your-first-admin-user>`

Now make a `notebooks` directory under your first admin user's home dir, so that this user has a directory where the hub can spawn. Also give the admin user rights to that directory:

```
mkdir /home/<name-of-your-first-admin-user>/notebooks
chown -R <name-of-your-first-admin-user> /home/<name-of-your-first-admin-user>/notebooks
```

The install will default to the `sh` shell (e.g. in users' Terminals in JupyterHub). 
We want the `bash` shell to be default:

```
rm /bin/sh
ln -s /bin/bash /bin/sh
```

### 5. Starting and re-starting the hub
Use `screen` to set up the server as a session that will continue running when closing terminal.

```
screen -S jupyterhub 
jupyterhub
```
Note that `jupyterhub` must be run in the same directory where `jupyterhub_config.py` is.

Exit the `screen` session by ctrl+A+D.

Access JupyterHub in your browser at https://<your.address>/ . 

Stop the hub:

```
screen -r jupyterhub
<press ctrl+C>
```

If stopping the hub, make sure before relaunching it that all instances of jupyterhub and configurable-http-proxy are terminated:

```
ps aux | grep jupyter 
kill <process id, one or several> 

ps aux | grep configurable 
kill <process id, one or several>
```

### Some notes

----

#### Adding users

To set up new users, first create them as users of the Linux server (no need to set a password here). 

```
useradd -m <user>
mkdir /home/<user>/notebooks
chown -R <user> /home/<user>/notebooks
```

----

#### Installing packages

```
$ ssh root@xxx.xx.xxx.xx 
conda install <package name>
```

or:

```
pip <or pip3> install <package name>
```

----

#### Renewing SSL

Stop jupyterhub. Then:

```
sudo certbot renew
```

Then relaunch jupyterhub.

----

#### Updating Jupyter

Update jupyter notebook, lab, hub and other packages through:

```
$ ssh root@xxx.xx.xxx.xx 

conda update --all 
# or conda update —packagename
# conda update jupyter
# conda update jupyterlab
# conda -c conda-forge install jupyterhub
<etc>
```
----

#### Install R kernel
Install R:

```
$ ssh root@xxx.xx.xxx.xx 
apt install r-base r-base-dev libssl-dev libcurl3-dev curl
R
install.packages('IRkernel') 
IRkernel::installspec(user = FALSE)
q()
```
This will install R in `/usr/bin/R` which makes it accessible by typing `R` in Terminal for all users. It also installs R as a selectable kernel in Jupyter.


Installing R packages:

```
$ ssh root@xxx.xx.xxx.xx 
R
install.packages(“<package-name>")
q()
```
----

#### Install Julia kernel

Install julia, by downloading the most recent version. Then untar it, copy to `/opt` and symlink it for all users to access it. Finally remove the installer.

```
wget https://julialang-s3.julialang.org/bin/linux/x64/1.1/julia-1.1.0-linux-x86_64.tar.gz
tar -xvzf julia-1.1.0-linux-x86_64.tar.gz
sudo cp -r julia-1.1.0 /opt/
sudo ln -s /opt/julia-1.1.0/bin/julia /usr/local/bin/julia
rm -rf julia-1.1.0 
rm julia-1.1.0-linux-x86_64.tar.gz
```

Each user can open a Terminal and enter `julia` to run it at command line. Close julia with ctrl+D.

To make Julia appear as a selectable kernel for Jupyter Notebooks on the hub installation, _each individual user_ must (once) open Terminal in the Jupyter GUI and do:

```
julia
import Pkg
Pkg.add("IJulia”)
<press ctrl + D>
```

Wait for it to install. Refresh the browser. Try to create a new notebook, and Julia will be available as a kernel.

----

#### Making custom keyboard commands (as user)

In the jupyter interface (in your browser) make menu selections: Settings -> Advanced Settings Editor -> Keyboard Shortcuts

In the box "User Preferences", add (as an example):

```
{
       "shortcuts": [
        {
            "command": "terminal:create-new",
            "keys": [
                "Shift Accel T"
            ],
            "selector": "body"
        }
    ]
}
```

Click save button. This sets shift + accelerator key (i.e. command) + T to as a shortcut to launch Terminal.
