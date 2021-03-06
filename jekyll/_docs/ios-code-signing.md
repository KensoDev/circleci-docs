---
layout: classic-docs
title: Set up code signing for iOS projects
short-title: iOS code signing
categories: [how-to]
last_updated: March 2nd, 2016
---

## Quick start guide

To get code signing working for your iOS app using our automated code
signing support, you would need to do the following:

1. Upload your `.P12` file in **Project Settings > iOS Code Signing**.
2. Add your provisioning profile (`.mobileprovision`) file to your repo.
3. Set `GYM_CODE_SIGNING_IDENTITY` to match your code-signing identity, i.e.
`iPhone Distribution: Acme Inc.`.
4. Build with `gym` and deploy with `ipa`.

## Background: Code Signing Process

The code-signing process for iOS apps can seem difficult at first, but
once familiar with the concepts the process is less daunting.

There are 2 necessary items for signing an app:

* A provisioning profile (`.mobileprovision`)
* A corresponding developer private-key (`.p12`).

With these 2 files, and an Xcode project the goal is to produce 2 files:

* A signed application (`.ipa`)
* A debug symbols archive (`.app.dSYM.zip`).

The `.ipa` file is the application file that can be distributed to your
testers using a service such as
[Crashlytics Beta](http://try.crashlytics.com/beta/)
or [Hockey App](https://hockeyapp.net/).
The debug symbols file will be used by your distribution service of choice
to produce call-stacks helping you to debug any crashes in your app that
testers encounter.

### OS X circle Keychain
When signing an app, Xcode will look in any
[keychains](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/security.1.html)
on the machine to find the necessary credentials.

When you build on CircleCI, a new keychain named `circle.keychain` is
created. The password for this keychain is `circle`. This keychain is
unlocked for the duration of the build, and it is added to the default
search path, so any credentials stored in this keychain are available to
Xcode.

Any P12 certificates that you upload to CircleCI in Project Settings
will be added to `circle.keychain` before your build begins. Any
Provisioning Profiles files that are found in your repository will also
be added to `circle.keychain`.

### 1. Install Fastlane tools locally
This guide assumes that you are developing on a Mac running OS X. The
first step is to install the Fastlane tools13. If you are using the
system Ruby, you can install using

```
sudo gem install fastlane
```

Or if you are using a Ruby installed using [Homebrew](http://brew.sh/)
you can install as the current user:

```
gem install fastlane
```

The tools take a few minutes to install, so a little patience is
required. For more information check out the [Fastlane installation
guide](https://github.com/fastlane/fastlane/blob/master/docs/Guide.md#fastlane)

### 2. Create and Upload a Code Signing Certificate

To create a code signing certificate and keys we will use the `cert`
Fastlane tool. Open a terminal in the root of the repository and run the
following commands:

    $ mkdir certificates
    $ cert --output_path certificates

This will create 3 files in the `certificates` directly, all named after
the certificate. In my case the certificate is named `NK5Z971JCM` but
this will be different for each project.

    $ ls certificates

    NK5Z971JCM.cer
    NK5Z971JCM.certSigningRequest
    NK5Z971JCM.p12

The `.certSigningRequest` and `.cer` files are not needed to sign your
app using CircleCI. You can delete these files if you have no need for
them. The `.p12` file is the private ket for your code-signing identity.

You can now upload the `.p12` file to your project on CircleCI in
`Project Settings` > `iOS Code Signing`. When your build runs this `p12`
file will be added to the `circle` keychain automatically.

![The code signing section in the project settings]({{ site.baseurl }}/assets/img/docs/code-signing-settings-section.png)

![The code signing welcome screen]({{ site.baseurl }}/assets/img/docs/code-signing-splash-screen.png)

![Uploaded key on the code signing page]({{ site.baseurl }}/assets/img/docs/code-signing-key-uploaded.png)

### 3. Create A Provisioning Profile

Use the `sigh` tool to create a new provisioning profile.

    $ sigh

This will create a provisioning profile (a file with the
`.mobileprovision` extension) in the root of your repository. You can
move this file to any folder that you like, and then add the file to
your repository.

During the build on CircleCI the provisioning profile will be found and
installed for use by Xcode.

### 4. Find Your Code Signing Identity name

One your local machine run the following command to list the current
code-signing identities installed on your machine.

    $ security find-identity -p codesigning

You should see output like the following:

```
Policy: Code Signing
  Matching identities
  1) 0620A954DE3B589E378435B38B8D87B6C0436BB0 "iPhone Distribution: Acme
Inc. (GL31ZZ3256)"
     1 identities found

  Valid identities only
  1) 0620A954DE3B589E378435B38B8D87B6C0436BB0 "iPhone Distribution: Acme
Inc. (GL31ZZ3256)"
     1 valid identities found
```

The string with the form `"iPhone Distribution: Acme Inc. (GL31ZZ3256)"`
is your code-signing identity. You might have more than one identity on
your local machine, so choose the correct one for your app. You should
add this to your `circle.yml` file as follows, so that it's available
during the build:

```
machine:
  environment:
    GYM_CODE_SIGNING_IDENTITY: "iPhone Distribution: Acme Inc.
(GL31ZZ3256)"
```

### 5. Sign Your App on CircleCI

We recommend using [Fastlane Gym](https://github.com/fastlane/gym) to
build a signed app. `gym` is pre-installed on our containers, so it's
easy to set up. The first step is to add a new [deployment
command]({{ site.baseurl }}/configuration#deployment) in your
`circle.yml` file.

```
machine:
  environment:
    GYM_CODE_SIGNING_IDENTITY: "iPhone Distribution: Acme Inc.
(GL31ZZ3256)"

deployment:
  beta_distribution:
    branch: master
     commands:
       - gym
```

You should take a few minutes to read [the documentation on deployments
using CircleCI]({{ site.baseurl }}/configuration#deployment).
The deployment stanza above instructs CircleCI to run the `gym` command
on each successful build of the `master` branch. The `beta_distribution`
is just a name for the deployment. You can use any name here.

You might need to pass additional parameters to `gym` to specify exactly
which configuration to build, but in most cases these are not necessary.
You should run `gym` locally specifying `GYM_CODE_SIGNING_IDENTITY ` or
`--codesigning_identity` until you get a signed app.

```
deployment:
  beta_distribution:
    branch: master
     commands:
       - gym --scheme "App" --workspace "App.xcworkspace"
```

## Troubleshooting

In this section we show a few of the most common errors that can be seen
with the code signing, and offer ways to work around these issues.

### No valid signing identities

If you get an error like

    Code Sign error: No code signing identities found: No valid signing
identities (i.e. certificate and private key pair) matching the team ID
“3EF4ATW3JB” were found.
or

    Code Sign error: No code signing identities found: No valid signing
identities (i.e. certificate and private key pair) were found.

Then Xcode was not able to find the correct keychain to use. The first
step to take is to example the output of the "Install Code Signing
Credentials" step of the build on CircleCI. This is listed under
"machine". Expand the output and you should see output like the
following:

![The output of the code signing step that ran correctly]({{ site.baseurl }}/assets/img/docs/code-signing-correct-step-output.png)

In this example there is one valid code-signing identity, `"iPhone
Distribution: UTAH STREET LABS INC (GL92ZZ6423)"`. The goal is to pass
this literal string to Xcode. If you are building with Fastlane and
`gym`, then you should ensure that you have your code-signing identity
string in your `circle.yml` like this:

```
machine:
  environment:
    GYM_CODE_SIGNING_IDENTITY: "iPhone Distribution: Acme Inc.
(GL31ZZ3256)"
```

Or that you pass this string to `gym` with the `--codesigning_identity`
flag.

If you are calling `xcodebuild` directly you should pass your
code-signing identity as the `CODE_SIGN_IDENTITY=` parameter, for
example `CODE_SIGN_IDENTITY="iPhone Distribution: Acme Inc.
(GL31ZZ3256)"`.


### No provisioning profiles matching an applicable signing identity were found

If you get an error message that says:

    Code Sign error: No matching provisioning profiles found: No
    provisioning profiles matching an applicable signing identity were
    found.

This means that you have a valid P12 file, but there was no matching
provisioning profile found. The first thing to check is that you have a
valid provisioning profile file (`.mobileprovision`) checked in to your
repository. Then take a look at the output from the "Install Code
Signing Credentials" step of the build. It should no longer be saying
`"No provisioning profiles found in repository."` The output shout give
a list of the provisioning profiles that were found as follows:

    Installed provisioning profile 616323a4-5216-42b0-b150-a128674ec52f
    (UTAH STREET LABS INC - CircleCI iOS Game - ["GL92ZZ6423"])
    1 certificate imported.


### My build fails with a timeout during code-signing

If your build fails with a timeout during the code-signing process, the
issue is usually that the correct keychains are not unlocked, or the
keychains are not added to the system search path. The first thing to
check is the output of `security list-keychains`. You should add a step
to your build to run `security list-keychains` immediately before the
command that times out.

You should see the following:

    $ security list-keychains
        "/Users/distiller/Library/Keychains/login.keychain"
        "/Users/distiller/Library/Keychains/circle.keychain"
        "/Library/Keychains/System.keychain"

You need to ensure that `login.keychain` and `circle.keychain` are in
this list. If you are adding a custom keychain using your own commands,
make sure that you are unlocking the keychain too. Make sure you are
doing something like the following:

    security unlock-keychain       -p circle circle.keychain
    security set-keychain-settings -lut 7200 circle.keychain


### What Does "Install Code Signing Credentials" Do?

The "Install Code Signing Credentials" step of OSX builds is where we
prepare the container for code-signing before your builds runs.

1. We create a new keychain called `circle.keychain` with password
   `circle`. We unlock this keychain and add it to the system search
path so Xcode can access it.
1. We install any P12 certificates that you have uploaded in project
   settings to circle.keychain
1. We look through your repo and find any provisioning profiles you
   have. We copy any profiles to `~/Library/MobileDevice/Provisioning
Profiles`, and we add any certificates in the provisioning profiles to
circle.keychain (the public key in the provisioning profile should match
the private key in the P12 file).
1. We call `security find-identity -p codesigning` to list the keychains
   in the build output.
