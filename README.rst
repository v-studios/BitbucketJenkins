=====================================
 Bitbucket Triggering Jenkins Builds
=====================================

This repo allows you to easily run Bitbucket and Jenkins in Docker
containers, then walks you through configuring them to integrate the
two. When done, a push to Bitbucket will cause Jenkins to run the
Jenkins file, which simply checks out the code but could be extended
to do builds, run tests, create deploy artifacts, etc.

We want our Bitbucket Server (aka: Stash) to trigger our Jenkins
builds on commit. There is a free plugin from Atlassian for Jenkins
that does this, and it seems to work well once permissions between the
systems are setup. There's a excellent video from their developers,
and this write-up is largely based on that, with some trouble-shooting
that may help you debug problems:

  https://www.youtube.com/watch?v=0-FugzVYJQU


Docker Compose
==============

The ``docker-compose.yml`` file defines BitbucketServer and Jenkins
services. It also defines a volume for each, and a network they need
to talk to each other. Bring it up::

  docker-compose up

This may take a while to download the images the first time. It then
takes a couple minutes for these Java services to start.

Watch the logs for the Jenkins auth key you'll need to login::

  jenkins2_1 | *************************************************************
  jenkins2_1 | Jenkins initial setup is required. An admin user has been created and a password generated.
  jenkins2_1 | Please use the following password to proceed to installation:
  jenkins2_1 |
  jenkins2_1 | 529cd61aba66ef4b6f46ebc5bba88f4d
  jenkins2_1 |
  jenkins2_1 | This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
  jenkins2_1 | *************************************************************

We'll need the IP addresses of the containers so that Jenkins can
refer to Bitbucket. Our "project" gets its name from the directory in
which we ran ``docker-compose up``, and our network is named
"cicdnet" in our yml file, so inspect the network::

  $ docker network inspect bitbucketjenkins_cicdnet
  ...
  "Name": "bitbucketjenkins_cicdnet",
  "Driver": "bridge",
  "Containers": {
      "c5df07a6f7c7dbb67c2f6c2d7991962e5db9597930dbbc6e939faae9b73dd1e7": {
          "Name": "bitbucketjenkins_bitbucket_1",
          "IPv4Address": "172.22.0.2/16",
      },
      "fdddc9a191d8712aef1be9d29f553b8806577dacb1781371a28ad7add630034c": {
          "Name": "bitbucketjenkins_jenkins_1",
          "IPv4Address": "172.22.0.3/16",

Bitbucket
=========

Go to http://localhost:7990/ and create a user.

Create a Project and Repository, push your code to it.

Change the server's own URL from localhost to use the Docker IP
address: Gear icon -> Server settings::

  Base URL: http://172.22.0.2:7990/

Create yourself an access key which Jenkins will need. Click your face
icon in the top right -> Manage Account -> Personal access tokens ->
Create Token::

  Token name:   BitbucketJenkins
  Projects:     Admin
  Repositories: Admin (inherited)
  Expiry:       no

Create and save the token.


Jenkins
=======

Go to http://localhost:8080/ and enter the admin auth key from the
docker logs, create an eval license, and user.

Customize the plugins to remove the ones you don't want.

Config Jenkins so it knows its Docker IP in the URL, rather than
``localhost``; otherwise it will create webhooks on Bitbucket with
``localhost`` which will cause the webhook to fail to connect to
Jenkins: Jenkins -> manage Jenkins -> Configure System::

  Jenkins URL: http://172.22.0.3:8080/

In Jenkins -> Manage Jenkins -> Manage Plugins, pick tab "Available" and
search for "Bitbucket Server Integration", then "Install without restart".

Add our Bitbucket server with the same Docker IP based URL that we
used to configure Bitbucket above. Manage Jenkins -> Configure System
-> Bitbucket Server integration -> Add a Bitbucket Server instance ->
Instance details::

  Instance name: bitbucket
  Instance URL: http://172.22.0.2:7990/
  Personal access token:
    Jenkins:
      Domain: Global credentials
      Kind: Bitbucket personal access token
      Token: [paste the one you got from Bitbucket]
      Description: ChrisBitbucket
  Credentials (for build auth): none  [leave this empty for now]

Test connection. If it can't connect, check your URL. If you can't
auth, re-do the access token.

Save.

Jenkins project integration with Bitbucket
------------------------------------------

Jenkins ->  New Item: BitbucketJenkins, Pipeline.
Configure::

  [x] Bitbucket Server trigger after push
  Pipeline: Pipeline script from SCM.
  SCM: Bitbucket Server
  Credentials (for build auth): Add -> Jenkins::
    Domain: Global credentials
    Kind: username with password
    Scope: GlobalUsername: use your Bitbucket username+password
    Description: BitbucketPassword

  Bitbucket Server instance: bitbucket
  Project name: TriggerJenkins [should autocomplete if creds are good]
  Repository name: BitbucketJenkins
  Branches: \*/master, \*/develop, \*/feature/\*
  Script Path: Jenkinsfile

Pipeline Syntax: sample Step: bbs_checkout: BitBucketSCMStep. Enter
the same Bitbucket info and creds as before, so it will generate the
incantation we need for our ``Jenkinsfile``. Generate Pipeline Script.
It emits the config incantation.

Include it your ``Jenkinsfile`` below (the asterisks below are only
prefixed by backslash to protect them here, don't include in your
file)::

  node {
      stage "Checkout from Bitbucket"

      bbs_checkout branches: [[name: '\*/master'], [name: '\*/develop'], [name: '\*/feature/\*']],
        credentialsId: 'c1c86c01-e86c-4ee3-8d68-10e0dd0c8531',
        id: '2390541b-8bee-4236-90e1-87f0cf20a74f',
        mirrorName: '',
        projectName: 'TriggerJenkins',
        repositoryName: 'BitbucketJenkins',
        serverId: '10e8046e-c1e7-463e-a38c-8416718eb2ea'
  }

Go back to the Pipeline definition you were creating and hit Save.
This will cause Jenkins use your creds to create a Bitbucket webhook.

Commit and push code. This should trigger you webhook.

Validate in Bitbucket, Troubleshooting, Success
-----------------------------------------------

In your project, Gear icon -> Webhooks. You should see one which
Jenkins with creds created. If you don't, make sure you saved your
Pipeline so the Bitbucket plugin can create the webhook in Bitbucket
Server.

Check the webhook. If it has an Error for last response, View details.
Mine had Response::

  Unable to connect to the URL specified within the timeout, please
  check the host and port are correct and that the URL is accessible
  from the server running this request.

If you go to the Request, we see::

  URL endpoint: http://localhost:8080/bitbucket-server-webhook/trigger

Oops, we can't have ``localhost`` since we're running in Docker:
Bitbucket would be trying to connect to itself.

In Jenkins -> Manage Jenkins -> Bitbucket Server integration, verify
we're not using localhost but the Docker IP in the URL: OK.

In Jenkins -> manage Jenkins -> Base URL, make sure we specify the
Jenkins Docker IP in the URL. I fixed this, then went to my Pipeline,
hit Save, and verified on Bitbucket my webhook now has the proper URL,
not localhost. Cool.

Push code again to test the webhook. Didn't even trigger. What? Delete
the webhook. Jenkins Project Pipeline SAVE to recreate. Verify it's
created in Bitbucket repo, looks good::

  http://172.22.0.3:8080/bitbucket-server-webhook/trigger

Push code again, the new webhook is never fired. Why not?
Tried modding the repo to get it to rescan the webhook, push, no change.
I do NOT see the webhook saying it fired.

But if I go to Jenkins I can see it did build for each of my pushes
after the initial failed webhook! I suspect the UI was not updating
showing the successful webhooks. Some of the Console Output::

  Triggered by Bitbucket webhook due to changes by chris shenton.
  Lightweight checkout support not available, falling back to full checkout.
  Checking out com.atlassian.bitbucket.jenkins.internal.scm.BitbucketSCM into
  /var/jenkins_home/workspace/BitbucketJenkins@script to read Jenkinsfile
  using credential c1c86c01-e86c-4ee3-8d68-10e0dd0c8531
  ...
  Commit message: "Modified bitbucket repo to see if it has to reread config"
  [Pipeline] }
  [Pipeline] // node
  [Pipeline] End of Pipeline
  Posting build status of SUCCESSFUL to bitbucketFinished: SUCCESS
