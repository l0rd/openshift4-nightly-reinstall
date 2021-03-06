# OpenShift Install

## Pre-Requirements

A public web server setup, gpg setup.

## Steps

* Download and extract the openshift installer from
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/ to
`./binaries/openshift-install`

* Setup a **PROFILE** in `./config/` taking an example from `configs/config.example.yaml` as for eg: `configs/user.yaml`

* Replace the *%VARIABLE%* in there with the right ones.

  - `%CLUSTER_NAME%`: Whatever name you want to give to that cluster, (or `sed "${RANDOM}q;d" /usr/share/dict/words`)
  - `%REGISTRY_TOKEN%`: the registry token you get from https://try.openshift.com/
  - `%SSH_KEY%` is your public SSH key.

* Add a `local.sh` at the top dir with your profile name to gpg key in bash hashtable and the WEB variable pointing to your local apache root (or subdir), i.e:

```bash
declare -A PROFILE_TO_GPG=(
    ["user"]="gpgkey@user.com"
)

```
When the reinstall is done, it will encrypt the access files and upload it somewhere, it supports different way to upload : 

* `WEB=/var/www/html/` - copied to a local directory which then be served by a web server

* `WEB_PROTECTED_URL=http://server/upload and WEB_PROTECTED=username:password`: 
Tthis will upload to the `WEB_PROTECTED_USER` with the username/password of `WEB_PROTECTED` using the form arguments `file` for the uploaded file and `path` for the target path. 
This usually used with this project : [go-simple-uploader](https://github.com/chmouel/go-simple-uploader). If you setup this project, **make sure you password/ip protect the /upload url endpoint** with the front webserver or you will end up with a bunch of warez and other weird files from the internet.

* `S3_UPLOAD_BUCKET="teambucket"` - Uploaded to this S3 bucket, you need to make sure the aws cli is installed and configured properly. The buckets would be accessible as :

  `https://${S3_UPLOAD_BUCKET}.s3.$S3_REGION_GET_IT_FROM_CONSOLE.amazonaws.com/${USER}/kubeconfig.gpg`

  Those urls would get the public ACL.

## Steps to add a user
  You will need to adjust the other examples of this doscument with this url structure.

* Ask the user for her/his GPG key and import it: `gpg --import gpgkey@user.com.pubkey.asc` or `gpg
   --recv-keys gpgkey@user.com` if it's uploaded on the public GPG servers.

* Trust the key : https://stackoverflow.com/a/17130637/145125

* Setup a cron for that profile to run every night (ask for the most convenient user TZ when she/he is not working) :

`00 06 * * * $PATH_TO/openshift4-nightly-reinstall/install.sh user >>/tmp/install.log`

* Let the user setup a function to resync its cluster key while enjoying a tiny sip of her/his double expresso latté ☕️, i.e:

```bash
function sync-os4() {
    local profile=profilename
    curl -s yourwebserver.com/${profile}.kubeconfig.gpg | gpg --decrypt > ${HOME}/.kube/config.os4
    export KUBECONFIG=${HOME}/.kube/config.os4
    oc version
}
```

## Hooks support

You can do some custom actions (ie save stuff from the already installed cluster) if you like with the hook support. Here is the list of functions that would be executed if present :

* `pre_delete_${profile}`
* `post_delete_${profile}`
* `pre_create_${profile}`
* `post_create_${profile}`
* `pre_encrypt_${profile}`
* `post_encrypt_${profile}`

So if you have a functions like this in your `local.sh` :

```bash
function pre_delete_myprofile() {
    oc get all --all -n mynamespace > /tmp/a.yaml
}
function post_create_myprofile() {
    oc new-project mynamespace
    oc apply -f /tmp/a.yaml
}

```

it will save/restore stuff before installling in the `mynamespace` namespace.


## Web Access

Web access is available with :

```shell
$ curl -s yourwebserver.com/${profile}.kubeadmin.password.gpg |gpg --decrypt
```

With the full url to access :

```shell
curl -s yourwebserver.comf/${profile}.webaccess.gpg |gpg --decrypt
```

## User creation automations

You can automatically add new users to your clusters, so you don't have to have a different webaccess every time :


1. Create first an htpasswd-file with your username/passwd using the [htpasswd](https://httpd.apache.org/docs/current/programs/htpasswd.html) utility :

   ```shell
   $ htpasswd -c /path/to/htpasswd username
   ```

2. Add this function to your zshrc and call it from sync-os4 function
``` shell
function os4_add_htpasswd_auth() {
    oc create secret generic htpasswd-secret --from-file=htpasswd=/path/to/htpasswd -n openshift-config
    oc patch oauth cluster -n openshift-config --type merge --patch "spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: htpasswd-secret
    mappingMethod: claim
    name: htpasswd
    type: HTPasswd
"
    # oc adm policy add-cluster-role-to-user cluster-admin ${your_username_used_to_login}
}
```
