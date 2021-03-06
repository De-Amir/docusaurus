---
title: Deployment: View
sidebar_label: View
---

OPQView is a [Meteor](http://meteor.com) application.  For general information on deploying Meteor applications, see the [Meteor Guide chapter on deployment](https://guide.meteor.com/deployment.html) and the [meteor build command documentation](https://docs.meteor.com/commandline.html#meteorbuild).

There are basically two steps to deploying OPQView: building the production bundle (a nodejs application) on a development machine, then running the bundle on the server.

## Developer system deployment tasks

First, make sure you have [set up opquser ssh access](deploy-initial-configuration.html#set-up-opquser-ssh-access). 

Second, invoke `meteor npm install` to verify that you have installed the latest versions of all libraries.

Third, invoke `meteor npm run test-all` to verify that the test cases pass.

Fourth, invoke `meteor npm run start` and then go to http://localhost:3000 to verify that the system runs without problems.

### Build the production bundle

In general, you will build the production bundle from the master branch of OPQView. 

In your development environment, be sure you are in the master branch, then change directories into the `opq/view/app/` directory. 

Now invoke `meteor npm run build`:

```
$ meteor npm run build

> opqview@ build /Users/philipjohnson/github/openpowerquality/opq/view/app
> meteor build ../deploy --architecture os.linux.x86_64

Node#moveTo was deprecated. Use Container#append.
```

The contents of the `opq/view/deploy/` directory should now contain the following files:

  * app.tar.gz: The file you just built containing the production version of the app (approximately 45 MB). 
  * deploy-run.sh:  A script for running the deployment on the server.
  * deploy-transfer.sh: A script for copying the appropriate files to the server. 
  
Note that *.gz files in the deploy directory are gitignored.

### Copy deployment files to the server

To copy deployment files to the server, cd to the opq/view/deploy directory, and invoke the `deploy-transfer.sh` script. This script creates a new directory whose name is the current timestamp, copies app.tar.gz, deploy-run.sh, and settings.development.json into it, then gzips that directory and secure copies it to emilia.

Here is what the invocation of this command should look like:

```
./deploy-transfer.sh 
++ date +%Y%m%d_%H%M%S
+ timestamp=20180408_083036
+ mkdir 20180408_083036
+ cp app.tar.gz 20180408_083036
+ cp deploy-run.sh 20180408_083036
+ cp ../config/settings.development.json 20180408_083036
+ cd 20180408_083036
+ tar xf app.tar.gz
+ rm app.tar.gz
+ cd ..
+ tar czf 20180408_083036.tar.gz 20180408_083036
+ rm -rf 20180408_083036
+ scp -P 29862 20180408_083036.tar.gz opquser@emilia.ics.hawaii.edu:view
20180408_083036.tar.gz                                                                                                100%   43MB   1.5MB/s   00:29    
+ set +x
```

## Server-side deployment tasks

Now ssh to the server to do the remainder of the deployment:

```
ssh -p 29862 opquser@emilia.ics.hawaii.edu
```

### Unpack tar file with latest release

Change directories into the view/ subdirectory, and list the files:

```
$ cd view
$ ls -la
ls -la
total 43572
drwxr-xr-x  3 opquser opquser     4096 Apr  8 08:55 .
drwxr-xr-x 10 opquser opquser     4096 Apr  8 08:50 ..
-rw-r--r--  1 opquser opquser 44601734 Apr  8 08:55 20180408_085449.tar.gz
```

You should see one (or more) timestamped tar.gz files (and potentially directories). The most recent timestamped tar.gz file is the one you just copied over.  Expand it into a directory as follows:

```
$ tar xf 20180408_085449.tar.gz
```  

The cd into that directory, and list the contents:

```
$ cd 20180408_085449/
opquser@emilia:~/view/20180408_085449$ ls -la
total 24
drwxr-xr-x 3 opquser opquser 4096 Apr  8 08:56 .
drwxr-xr-x 3 opquser opquser 4096 Apr  8 08:55 ..
drwxr-xr-x 4 opquser opquser 4096 Apr  8 08:55 bundle
-rwxr-xr-x 1 opquser opquser  322 Apr  8 08:54 deploy-run.sh
-rw-r--r-- 1 opquser opquser 2279 Apr  8 08:54 settings.development.json
```

### Kill the current OPQView processes

Find the PID of the two currently running OPQView processes this way:

```
$ ps -ef | grep node
opquser   2048     1  0 16:21 ?        00:00:00 /home/opquser/.meteor/packages/meteor-tool/.1.6.1_1.c76v6p.io9tw++os.linux.x86_64+web.browser+web.cordova/mt-os.linux.x86_64/dev_bundle/bin/node --expose-gc /home/opquser/.meteor/packages/meteor-tool/.1.6.1_1.c76v6p.io9tw++os.linux.x86_64+web.browser+web.cordova/mt-os.linux.x86_64/tools/index.js node bundle/main
opquser   2069  2048  0 16:21 ?        00:00:10 /home/opquser/.meteor/packages/meteor-tool/.1.6.1_1.c76v6p.io9tw++os.linux.x86_64+web.browser+web.cordova/mt-os.linux.x86_64/dev_bundle/bin/node bundle/main
opquser   4791  4609  0 16:40 pts/3    00:00:00 grep node
```

In this case, the PIDs are 2048 and 2069. Kill those processes with the following commands:

```
$ kill -9 2048
$ kill -9 2069
```

### Verify that the server is running the correct version of node

The file `bundle/.node_version.txt` indicates the required version of node. Print it with the following command:

```
$ more bundle/.node_version.txt 
v8.11.1
```

Now check that the currently installed version of node matches the required version:

```
$ node --version
v8.11.1
```

If these don't match, then update node to the appropriate version before proceeding.

### Run the new version of OPQView

To run the new version of OPQView, invoke the deploy-run.sh script:

```
$ ./deploy-run.sh 
Make sure node version is v8.11.1

> fibers@2.0.0 install /home/opquser/view/20180408_085449/bundle/programs/server/node_modules/fibers
> node build.js || nodejs build.js

`linux-x64-57` exists; testing
Binary is fine; exiting

> meteor-dev-bundle@ install /home/opquser/view/20180408_085449/bundle/programs/server
> node npm-rebuild.js


> bcrypt@1.0.3 install /home/opquser/view/20180408_085449/bundle/programs/server/npm/node_modules/bcrypt
> node-pre-gyp install --fallback-to-build

[bcrypt] Success: "/home/opquser/view/20180408_085449/bundle/programs/server/npm/node_modules/bcrypt/lib/binding/bcrypt_lib.node" is installed via remote
@babel/runtime@7.0.0-beta.44 /home/opquser/view/20180408_085449/bundle/programs/server/npm/node_modules/@babel/runtime
  :
  (lots of lines deleted)
  :
{
  "npm": "5.6.0",
  "ares": "1.10.1-DEV",
  "cldr": "32.0",
  "http_parser": "2.8.0",
  "icu": "60.1",
  "modules": "57",
  "nghttp2": "1.25.0",
  "node": "8.11.1",
  "openssl": "1.0.2o",
  "tz": "2017c",
  "unicode": "10.0",
  "uv": "1.19.1",
  "v8": "6.2.414.50",
  "zlib": "1.2.11"
}
added 136 packages in 9.609s
$
```

This script brings up OPQView in the background (so that you can exit this terminal session), and sends all output to the file logfile.txt. To check that the system came up normally, you can print out the contents of the logfile, which should look similar to this:

```
$ more logfile.txt 
Starting SyncedCron to update System Stats every 10 seconds.
Initializing 4 user profiles.
Initializing 5 OPQ boxes.
```

## Verify that the new OPQView appears in your browser

As a final check, retrieve OPQView for example: [http://emilia.ics.hawaii.edu](http://emilia.ics.hawaii.edu). You might want to check that whatever recent features you developed are appearing in this system. That verifies that you deployed the intended version.


 
 

