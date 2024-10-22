
## Get user credentials for Quay.io

This section is now optional as of this time there is no reason to use this since
required pre-release bits are now GA in OperatorHub.

Go to [Quay.io](https://quay.io) and register for an account or login with an
existing account. Once a username has been created provide that username to
your Red Hat contact who will grant read permission to the private pre-release
repositories.

In the Quay.io interface, in the upper right hand corner, click on your
username and a drop down with *Account Settings* will appear. Go into *Account
Settings*. On the left hand side of *Account Settings*, select the *Gears* icon
for user settings. Near the top left of the settings page, there is a *CLI
Password* label followed by a clickable *Generate Encrypted Password* link.
Click this link and type in your quay.io password. This will generate a
*Credentials for YourUsername* box that will have various credentials for the
user. Select the *Kubernetes Secret* on the left hand side list. Download the
`<username>-secret.yml` file as we will need this for later steps. Note that
`<username>` will be whatever the username that was logged into Quay.io.

## Update the OpenShift cluster pull secret

Now, let's update the Openshift cluster pull secret with the information we obtained above. First let's pull the
current pull secret from the cluster:

~~~bash
$ oc get secret/pull-secret -n openshift-config -o json | jq '.data.".dockerconfigjson"' | tr -d '"' | base64 -d > cluster-pull-secret.json
~~~

Next, we will assign the dockerconfig string from our `<username>-secret.yml`
file we downloaded:

~~~bash
$ export PRERELEASE_PULL=$(grep -o '.dockerconfigjson:.*' bschmaus-secret.yml | cut -f2- -d: | sed 's/^[ \t]*//;s/[ \t]*$//')
~~~

Then, we will change the `quay.io` string in the variable to the right scoped
organization and repository and dump out to a file:

~~~bash
$ echo $PRERELEASE_PULL | base64 -d | sed "s/quay\.io/quay\.io\/redhat_emp1\/ecosys-nvidia/g" | tail -n +3 | head -n -2 > prerelease-secret.json
~~~

Now, we can merge our current cluster secret and our pre-release secret into
one file:

~~~bash
$ cat cluster-pull-secret.json | jq ".auths += {`cat prerelease-secret.json`}" > merged-pull-secret.json
~~~

And finally patch the updated pull secret to our Openshift cluster:

~~~bash
$ oc patch secret/pull-secret -n openshift-config --type merge --patch '{"data":{".dockerconfigjson":"'$(cat merged-pull-secret.json | tr -d '[:space:]' | base64 -w 0)'"}}'
~~~

We will go ahead and validate that our update looks right by pulling the pull
secret from the cluster and passing through `jq`:

~~~bash
$ oc get secret/pull-secret -n openshift-config -o json | jq '.data.".dockerconfigjson"' | tr -d '"' | base64 -d | python3 -m json.tool
~~~
