## What is RattlesnakeOS
RattlesnakeOS is privacy focused Android OS based on [AOSP](https://source.android.com/) for Google Pixel phones. It is my migration strategy away from [CopperheadOS](https://en.wikipedia.org/wiki/CopperheadOS) (hence the name similarity) which is no longer maintained. RattlesnakeOS is stock AOSP with no Google apps and a few additional features: [verified boot](https://source.android.com/security/verifiedboot/) with your own signing keys, OTA updates, latest Chromium ([webview](https://www.chromium.org/developers/how-tos/build-instructions-android-webview) + browser), and latest [F-Droid](https://f-droid.org/) (with [privileged extension](https://gitlab.com/fdroid/privileged-extension)).

## What is rattlesnakeos-stack
Rather than providing random binaries of RattlesnakeOS to install on your phone, I've gone the route of creating a cross platform tool, `rattlesnakeos-stack`, that provisions all of the [AWS](https://aws.amazon.com/) infrastructure needed to continuously build your own personal RattlesnakeOS, with your own signing keys, and your own OTA updates. It uses [AWS Lambda](https://aws.amazon.com/lambda/features/) to provision [EC2 Spot Instances](https://aws.amazon.com/ec2/spot/) that build RattlesnakeOS and upload artifacts to [S3](https://aws.amazon.com/s3/). Resulting OS builds are configured to receive over the air updates from this environment.

## Features
* Based on latest AOSP 9.0 (Android P)
* Support for <b>Google Pixel, Pixel XL, Pixel 2, Pixel 2 XL</b>
* Updates and monthly security fixes delivered through OTA updates - no need to manually flash your device
* Maintain [verified boot](https://source.android.com/security/verifiedboot/) with a locked bootloader just like official Android but with your own personal signing keys
* Latest Chromium [browser](https://www.chromium.org) and [webview](https://www.chromium.org/developers/how-tos/build-instructions-android-webview)
* Latest [F-Droid](https://f-droid.org/) client and [privileged extension](https://gitlab.com/fdroid/privileged-extension)
* No Google apps pre-installed
* Full end to end setup of build environment for RattlesnakeOS in AWS
* Costs a few dollars a month to run (see FAQ for additional cost breakdown)

## Prerequisites
* An AWS account - you can [create an AWS account](https://portal.aws.amazon.com/billing/signup) if you don't have one. 
  * <b>If this is a new AWS account, make sure you launch at least once paid instance before running through these steps.</b>  To do this you can navigate to the [EC2 console](https://us-west-2.console.aws.amazon.com/ec2/), click `Launch instance`, select any OS, pick a `c4.4xlarge`, and click `Review and launch`. After it launches you can terminate the instance through the console.
* You'll need AWS credentials with `AdministratorAccess` access. If you're not sure how to do that, you can follow [this step by step guide](https://serverless-stack.com/chapters/create-an-iam-user.html).
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html) for your platform and [configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) it to use these credentials by default.
* [Setup an SSH keypair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) in the EC2 console and download the key. You'll use this keypair name when deploying your stack and you'll use this key if you want to SSH into the launched EC2 spot instances.

## Installation of rattlesnakeos-stack tool
The easiest way is to download a pre-built binary from the [Github Releases](https://github.com/dan-v/rattlesnakeos-stack/releases) page. The other option is to compile from source (see `Build from Source` section).

## Deployment of stack using rattlesnakeos-stack tool
The `rattlesnakeos-stack` tool will handle deploying your stack - which is just all the required AWS infrastructure needed to run ongoing builds of RattlesnakeOS. After initial deployment, your first build will automatically start; by default it is configured to build on a weekly basis after this (see the FAQ for details on how to modify build schedule). When deploying your stack with `rattlesnakeos-stack`:
* Pick a unique name to replace `rattlesnakeos-<yourstackname>` in the commands below. <b>Note: this name has to be unique or it will fail to provision.</b>
* Provide the SSH keypair name that you created in the prerequisite steps to replace `<yourkeyname>` in commands below.

### Example commands
Deploy stack with default options for your specific device

```sh 
# Pixel XL (marlin)
./rattlesnakeos-stack --region us-west-2 --name rattlesnakeos-<yourstackname> --device marlin --ssh-key <yourkeyname>

# Pixel (sailfish)
./rattlesnakeos-stack --region us-west-2 --name rattlesnakeos-<yourstackname> --device sailfish --ssh-key <yourkeyname>

# Pixel 2 XL (taimen)
./rattlesnakeos-stack --region us-west-2 --name rattlesnakeos-<yourstackname> --device taimen --ssh-key <yourkeyname>

# Pixel 2 (walleye)
./rattlesnakeos-stack --region us-west-2 --name rattlesnakeos-<yourstackname> --device walleye --ssh-key <yourkeyname>
```

To see full list of options you can pass rattlesnake-stack you can use the help flag (-h)

```sh
...

Flags:
    --ami string          ami id to use for build environment. this is optional as correct ubuntu ami for region will be chosen by default.
-d, --device string       device you want to build for: 'marlin' (Pixel XL), 'sailfish' (Pixel), 'taimen' (Pixel 2 XL), 'walleye' (Pixel 2)
    --force               build even if there are no changes in available version of AOSP, Chromium, or F-Droid.
-h, --help                help for rattlesnakeos-stack
-n, --name string         name for stack. note: this must be a valid/unique S3 bucket name.
    --prevent-shutdown    for debugging purposes only - will prevent ec2 instance from shutting down after build.
-r, --region string       aws region for deployment (e.g. us-west-2)
    --remove              cleanup/destroy all deployed aws resources.
    --schedule string     cron expression that defines when to kick off builds. note: if you give invalid expression it will fail to deploy stack. see: https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html#CronExpressions (default "rate(7 days)")
    --spot-price string   max ec2 spot instance bid. if this value is too low, you may not obtain an instance or it may terminate during a build. (default "1.00")
    --ssh-key string      aws ssh key to add to ec2 spot instances. this is optional but is useful for debugging build issues on the instance.
    --version             version for rattlesnakeos-stack
```

## First Time Setup After Deployment
* Setup email notifications for builds:
  * Go to the [SNS console](https://us-west-2.console.aws.amazon.com/sns/v2/home?region=us-west-2#/topics)
  * Click on the topic named `rattlesnakeos-<yourstackname>`
  * Click on `Create subscription` button
  * In `Create subscription` dialog, in `Protocol` dropdown select `Email`
  * For `Endpoint`, enter your email address
  * Click `Create subscription` button
  * You should get an email link that you need to click in order to subscribe to messages in this topic
* After initial setup with `rattlesnakeos-stack` tool, a build should have automatically kicked off. You can check this by going to the [EC2 console](https://us-west-2.console.aws.amazon.com/ec2/v2/home) and verifying there is an EC2 instance running. If a build hasn't kicked off, check out the FAQ for how to manually start a build.
* The <b>initial build will likely take 5+ hours to complete</b>. Looking at the EC2 instance metrics like CPU, etc is NOT a good way to determine if the build is progressing. If you want to see live build progress, go to FAQ section `How can I connect to the EC2 instance and see the build status?`.
* After the build finishes, a factory image should be uploaded to the S3 bucket that you can download:
  * Go to the [S3 console](https://s3.console.aws.amazon.com/s3/buckets/)
  * Click on `rattlesnakeos-<yourstackname>-release` bucket.
  * From this bucket, download the file `<device>-factory-latest.tar.xz`
* Use this factory image and [follow the instructions on flashing your device carefully](FLASHING.md).
* You followed the instructions until the end and you re-locked your bootloader and disabled OEM unlocking after flashing right? If not, go do that!
* After successfully flashing your device, you will now be running RattlesnakeOS and all future updates will happen through the built in OTA updater.
* <b>I highly suggest backing up your generated signing keys</b>. To do this:
  * Go to the [S3 Console](https://s3.console.aws.amazon.com/s3/buckets/)
  * Click on `rattlesnakeos-<yourstackname>-keys` bucket.
  * Download all these keys there and store them in a safe place

## How to update rattlesnakeos-stack
* Just download the new version of rattlesnakeos-stack and run the same command used previously (e.g. `rattlesnakeos-stack --region us-west-2 --name rattlesnakeos-<yourstackname> --device <yourdevicename>  --ssh-key <yourkeyname>`) to apply the updates

## FAQ
1. <b>Should I use rattlesnakeos-stack?</b> Use at your own risk.
2. <b>How much does this cost?</b> The costs are going to be variable by AWS region and by day and time you are running your builds as spot instances have a variable price depending on market demand. Below is an example scenario that should give you a rough estimate of costs:
   * The majority of the cost will come from builds on EC2. It currently launches spot instances of type c4.4xlarge which average maybe $.30 an hour in us-west-2 (will vary by region) but can get up over $1 an hour depending on the day and time. The `rattlesnakeos-stack` tool allows you define a maximum bid price (`--spot-price`) you are willing to pay and if market price exceeds that then your instance will be terminated. Builds can take anywhere from 2-6 hours depending on if Chromium needs to be built. So let's say you're doing a weekly build at $0.50 an hour and it is taking on average 4 hours - you'd pay ~$8 in EC2 costs per month. You could reduce this to a monthly build (see section how to change build frequency) and then you'd be looking at ~$2 in EC2 costs per month.
   * The other very minimal cost would be S3. Storage costs are almost non existent as a stack will only store about 3GB worth of files (factory image, ota file, target file) and at $0.023 per GB you're looking at $0.07 per month in S3 storage costs. The other S3 cost would be for data transfer out for OTA updates - let's say you are just downloading an update per week (~500MB file) at $0.09 per GB you're looking at $0.20 per month in S3 network costs.
3. <b>How do I change build frequency?</b> The current default is to do builds on a weekly basis. With `rattlesnakeos-stack` tool there is an option to specify how frequently builds are kicked off with option `--schedule`. For example you could set `--schedule "rate(30 days)"` to only build every 30 days. Also note, the default behavior is to only run a build if there have been version updates in AOSP build, Chromium version, or F-Droid versions.
4. <b>How do I manually start a build?</b>
   * Go to the [AWS Lambda](https://us-west-2.console.aws.amazon.com/lambda/) console
   * Click on the function named 'rattlesnakeos-\<yourstackname>-build'
   * Click on the 'Test' button
   * In 'Configure test event dialog', set event name to 'rattlesnakeos', keep the defaults, and click 'Create' button.
   * Click the 'Test' button again to kick off the build
5. <b>Where do I find logs for a build?</b> On build failure/success, the instance should terminate and upload its logs to S3 bucket called `<stackname>-logs` and it's in a file called `<device>/<timestamp>`.
6. <b>How can I connect to the EC2 instance and see the build status?</b> There are a few steps required to be able to do this:
   * In the [default security group](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html#DefaultSecurityGroup), you'll need to [open up SSH access](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html).
   * You should be able to SSH into the instance (can get IP address from EC2 console): `ssh -i yourkeypairname.pem ubuntu@yourinstancepublicip`
   * Tail the logfile to view progress `tail -f /var/log/cloud-init-output.log`
7. <b>How can I prevent the EC2 instance from immediately terminating on error so I can debug?</b> There is a flag you can pass `rattlesnakeos-stack` called `--prevent-shutdown`. Note that this will keep the instance online for 12 hours or until you manually terminate it.
8. <b>Why did my EC2 instance randomly terminate?</b> If there wasn't an error notification, this is likely because the [Spot Instance](https://aws.amazon.com/ec2/spot/) bid was not high enough at this specific time. You can see historical spot instance pricing in the [EC2 console](https://console.aws.amazon.com/ec2sp/v1/spot/home). Click `Pricing History`, select c4.4xlarge for `Instance Type` and pick a date range. If you want to avoid having your instance terminated, you can pass an additional flag to `rattlesnakeos-stack` with a higher than default bid: `--spot-price 1.50`
9. <b>How do OTA updates work?</b> If you go to `Settings->System update settings` you'll see the updater app settings. The updater app will check S3 to see if there are updates and if it finds one will download and apply it your device. There is no progress indicator unfortunately - you'll just got a notification when it's done and it will ask you to reboot. If you want to force a check for OTA updates, you can toggle the `Require battery above warning level` setting and it will check for a new build in your S3 bucket.
10. <b>What network carriers are supported?</b> I only have access to a single device and carrier to test this on, so I can't make any promises about it working with your specific carrier. Confirmed working: T-Mobile, Rogers, Cricket. Likely not to work: Sprint (has requirements about specific carrier app being on phone to work), Project Fi.

## Uninstalling
### How to uninstall rattlesnakeos-stack
If you decide this isn't for you and you want to remove all the provisioned AWS resources, there's a command for that. Note: if you've already done a full build, you'll need to manually remove all of the files from S3 buckets before running this cleanup command.

```sh
./rattlesnakeos-stack --remove --region us-west-2 --name rattlesnakeos-<yourstackname> --ssh-key <yourkeyname>
```

### How to revert back to stock Android
For Pixel and Pixel XL, just unlock your bootloader and flash stock factory image.

For Pixel 2 and Pixel 2 XL, you'll need to clear the configured AVB public key after unlocking the bootloader and before locking it again with the stock factory images.

```sh
fastboot erase avb_custom_key
```

## Powered by
* Huimin Zhang - author of the original underlying build script that was written for CopperheadOS.
* [anestisb/android-prepare-vendor](https://github.com/anestisb/android-prepare-vendor)
* [terraform](https://www.terraform.io/)

## Build from Source
 * To compile from source you'll need to install Go (https://golang.org/) for your platform
  ```sh
  go get github.com/dan-v/rattlesnakeos-stack
  cd $GOPATH/src/github.com/dan-v/rattlesnakeos-stack/
  make tools
  make
  ```