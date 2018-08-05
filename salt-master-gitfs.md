# How to Set Up a Salt Master with GitFS

## Deploy your Salt master
If you're building a new master, it's easier to use the bootstrap script from Salt. It does all the work for you.

~~~
curl -L https://bootstrap.saltstack.com -o install_salt.sh
sudo sh install_salt.sh -P -M
~~~

## Configure your Salt Master
Install the following packages:
~~~
yum install GitPython
~~~

Edit your /etc/salt/master file accordingly:
~~~
gitfs_provider: GitPython

fileserver_backend:
  - gitfs

gitfs_remotes:
  - git@github.com:xthor0/salt.git

pillar_roots:
  base:
    - /srv/pillar
  dev:
    - /srv/pillar/dev
  prod:
    - /srv/pillar/prod
~~~

## Set up your Git repository
Clone the repo:
~~~
git clone git@github.com:xthor0/salt.git
~~~

Set up the repository:
~~~
cd salt
git checkout -b prod
rm README.md
git push --set-upstream origin prod
git checkout master
git checkout -b dev
rm README.md
git push --set-upstream origin dev
<copy in some states, or create one>
git commit -m 'initial commit'
git push
~~~

Now that your branches are set up, switch back to master:
~~~
git checkout master
~~~

Now, add something like this to your top.sls:
~~~
prod:
  'kernel:linux':
    - match: grain
    - linuxos
  'G@env:prod and G@roles:icinga2':
    - match: compound
    - icinga2

dev:
  'kernel:linux':
    - match: grain
    - linuxos
  'G@env:dev and G@roles:icinga2':
    - match: compound
    - icinga2
~~~

## Generate an SSH key
Generate an SSH key as root, using `ssh-keygen`. Do not set a key (I need to update this documentation when I figure out how to use it with a passphrase).

Make sure you do this as root, as that's what GitPython uses for authentication.

Paste the public key into your repository. On GitHub, click on your repository, and then click `Settings` -> `Deploy Keys` to upload it.

## Make sure the SSH key works
For github, I had to run `ssh git@github.com` and I was presented with this message:

~~~
# ssh git@github.com
The authenticity of host 'github.com (192.30.255.112)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
RSA key fingerprint is MD5:16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.255.112' (RSA) to the list of known hosts.
PTY allocation request failed on channel 0
Hi xthor0/salt-top! You've successfully authenticated, but GitHub does not provide shell access.
~~~

## Test GitFS integration in SaltStack
Restart the salt-master and then run this command:
~~~
salt-run fileserver.file_list
~~~

You should see `top.sls` (or potentially `README.md` if you're using a service that creates one by default).

If you created a branch as I explained above, you can also see the contents of that branch by running the following command:
~~~
salt-run fileserver.file_list saltenv=dev
~~~

If you get no errors, congratulations! You can now start using your Salt master and deploying states with files pulled from Git.

## Why would I want to set this up?
Unless you work in a really small shop (and, frankly, even if you do), using GitFS with Salt gives the following benefits:

- Multiple developers no longer step on each other while developing states
- States can be tested locally, pushed to a dev branch for wider testing, and finally promoted to production. More testing == better product
- Git provides an awesome history of who did what - it'll be easier to determine who made changes, and when, in case something goes wrong
- Finally, you can treat your configuration management like infrastructure as code - and automate a deployment pipeline with the appropriate testing and gates!