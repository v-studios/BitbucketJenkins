=====================================
 Bitbucket Triggering Jenkins Builds
=====================================

Docker Compose
==============

The ``docker-compose.yml`` defines a BitbucketServer and Jenkins
service. It also defines a volume for each, and a network they need to
talk to each other. Bring it up::

  docker-compose up

This may take a while to download the images the first time. It takes
a couple minutes for these Java services to start.

Watch the logs for the Jenkins auth key you'll need to login::

  jenkins2_1 | *************************************************************
  jenkins2_1 | Jenkins initial setup is required. An admin user has been created and a password generated.
  jenkins2_1 | Please use the following password to proceed to installation:
  jenkins2_1 |
  jenkins2_1 | 46ebc5bba88f4d529cd61aba66ef4b6f
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

BitBucket
=========

Go to http://localhost:7990/ and create a user.

Create a Project and Repository, push your code to it.



Create yourself an access key which Jenkins will need. Hit your face icon in the top left

Jenkins
=======

Go to http://localhost:8080/ and enter the admin auth key from the
docker logs, create an eval license, and user.

Customize the plugins to remove the ones you don't want.

In Jenkins -> Manage Jenkins -> Manage Plugins, pick tab Available and
search for "Bitbucket Server Integration", then "Install without restart".

Manage Jenkins -> Configure System -> Bitbucket Server integration ->
Add a Bitbucket Server instance -> Instance details:

Instance name: bitbucket
Instance URL: http://172.22.0.2/
Personal access token: [create and select]

Jenkins project integration with Bitbucket
------------------------------------------

Jenkins ->  New Item: samplecode, Pipeline.
Configure
[x] Bitbucket Server trigger after push
Pipeline: Pipeline script from SCM.
SCM: Bitbucket Server
Credentials (for build auth)

TODO I htink I am missing something

samplecode
==========

Doesn't really matter what it is, goal is to trigger Jenkins on commit
to BitBucket. It should have a sane Jenkins file so we can see Jenkins
go; plagiarize the one from avail_fe.

