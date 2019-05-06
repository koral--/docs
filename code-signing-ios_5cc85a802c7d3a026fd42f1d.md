This document describes how to set up code signing in your iOS project on Semaphore CI using `fastlane match` tool.

- [Basic configuration of iOS Projects](#basic-configuration-of-ios-projects)
- [Setting up Fastlane Match](#setting-up-fastlane-match)
- [Sample configuration files](#sample-configuration-files)
- [Example application on GitHub](#example-application-on-github)

If you'd like the Semaphore CI to push new builds for distribution to Hockey App, Fabric Beta, or TestFlight, then the CI must have access to the code signing certificate and provisioning profiles that are required to build the app for such distribution.

In this guide we show you how to do that.

## Basic configuration of iOS Projects
We assume you have a working iOS project configured to run on Semaphore CI (see [Demo project](https://github.com/semaphoreci-demos/semaphore-demo-ios-swift-xcode)). In addition, we assume the `fastlane` tool is installed and configured. Please visit [Fastlane Setup](https://docs.fastlane.tools/getting-started/ios/setup/) to learn how to configure fastlane for your project.

If you are new to code signing, we recommend you to visit the [Code Signing Guide](https://codesigning.guide) and read it. 

The `match` tool has an extensive documentation on how to configure and use it, which you can find at [Fastlane Docs](https://docs.fastlane.tools/actions/match/).

In a nutshell, what we are aiming at is:

- Your iOS project configured with `fastlane` and `match`
- You have a separate private git repository that stores code signing certificates and provisioning profiles (we use `GitHub` repository in this example).
- You configure the 2 repositories above to be accessible from Semaphore CI.

## Setting up Fastlane Match

Before setting up provisioning profiles and distributing builds, you need an App ID for your app and App Store Connect application. If you don't have them yet, you can either create them manually in the Apple developer account and App Store Connect website, or you can use Fastlane's [`produce` action](https://docs.fastlane.tools/actions/produce/) to create both from command line.

When you have your App ID and App Store Connect Application created, you can proceed to configuring code signing for your project.

To set up the `match`, you'll need a private git repository. Follow the [documentation](https://docs.fastlane.tools/actions/match/) and command line prompts from the `init` command:

    # from the root directory of your iOS project
    $> bundle exec fastlane match init

This will configure `match` with your private git URL and password locally on your computer. With this, you are now ready to create provisioning profiles for your app:

    # this will generate 'development' provisioning profile so that you can run 
    # your app on the device connected to Xcode
    $> bundle exec fastlane match development

    # this wil generate 'adhoc' provisioning profile so that other people
    # can run your app on their devices through non-App Store distribution
    # like HockeyApp or Fabric Beta
    $> bundle exec fastlane match adhoc

    # this will generate 'appstore' provisioning profile so that 
    # your app can be distributed through TestFlight and App Store
    $> bundle exec fastlane match appstore

### Preparing your Xcode project for use with Fastlane Match

By default, the Xcode uses "automatic" code signing that uses Xcode's preferences to manage the signing certificates and provisioning profiles. While this works on your local computer, it won't work in the CI environment. That's why the project should switch to the "manual" code signing.

To do this, in the "General" tab of your app target in Xcode, uncheck the "Automatically manage signing" and then select the provisioning profiles generated by match for each "Provisioning Profile" dropdowns for each configuration (Sections "Signing (Debug)" and "Signing (Release)").

### Adding Semaphore Plugin to Fastlane

In order to configure the fastlane for the CI environment, please use the Semaphore plugin. 

    # Install the plugin
    $> bundle exec fastlane add_plugin semaphore

This plugin provides `setup_semaphore` action that configures temporary Keychain and switches `match` to 'readonly' mode.

### Adding Match to the Fastlane Lane

As mentioned earlier, the CI does not have access to your developer account unless you provide the credentials in the configuration for the `match`. To make this work, we need to add a `match` action to the lanes that will publish your app for distribution.

    # fastlane/Fastfile 
    default_platform(:ios)

    platform :ios do
    before_all do
        setup_semaphore
    end

    desc "Build and run tests"
    lane :test do
        scan
    end

    desc "Ad-hoc build"
    lane :adhoc do
        match(type: "adhoc")
        gym(export_method: "ad-hoc")
    end

    desc "TestFlight build"
    lane :build do
        match(type: "appstore")
        gym(export_method: "app-store")
    end
    ...
    end

### Adding a Deploy key to the Semaphore project

Now that we have configured our tools to use appropriate configuration, we must also provide a way for the Semaphore CI to access the Git certificates repo, and the Apple Developer portal.

To allow Semaphore CI download certificates from your private certificates repository, you need to create a deploy key and add the key to the Semaphore secrets. Adding a deploy key is described in the [Semaphore Documentation](https://docs.semaphoreci.com/article/109-using-private-dependencies). 

If you have not installed the `sem` command line tool, it is a good time to install it - see the [documentation](https://docs.semaphoreci.com/article/53-sem-reference).

To store the deploy key as a secret file in the Semaphore environment:

    $> sem create secret ios-cert-repo -f id_rsa_semaphore:/Users/semaphore/.keys/ios-cert-repo

This will create the deploy key file under the `.keys` directory when your build will run on the CI.

### Adding the Match passphrase to a secret

Next, add the URL for the certificates reposiotry and the encryption password as environment variables that will be accessible in the CI (see the [Environment Variables and Secrets guide](https://docs.semaphoreci.com/article/66-environment-variables-and-secrets)). Also, add the App Store developer account's credentials here. :

    $> sem create secret fastlane-env \
        -e MATCH_GIT_URL="<your ssh git url>" \
        -e MATCH_PASSWORD="<password for decryption>" \
        -e FASTLANE_USER="<app store developer's Apple ID>" \
        -e FASTLANE_PASSWORD="<app store developer's password>"

As a security note, it is highly advisable to create a separate app store "developer" account just for use in the CI environment. The same approach is advisable for accessing the private git certificates repository. 

In a similar fashion, you can also add API keys for the distribution platform you use, in case you are using non-TestFlight (like HockeyApp or Fabric Beta). Consult with the respective platform's documentation to see which environment variables or secrets you need to include.

With these secrets and configuration, now the Semaphore CI will be able to access the code signing certificates and provisioning profiles in order to build and distribute your app.

## Sample configuration files

After you have configured `match`, `fastlane`, and environment variables, you can run the build on Semaphore CI. The following are the examples of 'Fastfile' and `semaphore.yml` configuration file that will run test and build on every push to the default branch. 

    # fastlane/Fastfile
    default_platform(:ios)

    platform :ios do

        before_all do
            # install the semaphore plugin with `fastlane add_plugin semaphore`
            setup_semaphore
        end

        desc "Run Tests"
        lane :test do  
            scan
        end

        desc "Build"
        lane :build do
            match(type: "appstore")
            gym(export_method: "app-store")
        end

        desc "Ad-hoc build"
        lane :adhoc do
            match(type: "adhoc")
            gym(export_method: "ad-hoc")
        end

    end

    # .semaphore/semaphore.yml
    version: v1.0
    name: Semaphore iOS example
    agent:
    machine:
        type: a1-standard-4
        os_image: macos-mojave
    blocks:
    - name: Run tests
        task:
        env_vars:
            - name: LANG
            value: en_US.UTF-8
        prologue:
            commands:
            - checkout
            - cache restore gems-$SEMAPHORE_GIT_BRANCH-$(checksum Gemfile.lock),gems-$SEMAPHORE_GIT_BRANCH-,gems-master-
            - bundle install --path vendor/bundle
            - cache store gems-$SEMAPHORE_GIT_BRANCH-$(checksum Gemfile.lock) vendor/bundle
        jobs:
            - name: Fastlane test
            commands:
                - bundle exec fastlane ios test
        secrets:
            - name: fastlane-env

    - name: Build app
        task:
        env_vars:
            - name: LANG
            value: en_US.UTF-8
        prologue:
            commands:
            - checkout
            - cache restore gems-$SEMAPHORE_GIT_BRANCH-$(checksum Gemfile.lock),gems-$SEMAPHORE_GIT_BRANCH-,gems-master-
            - bundle install --path vendor/bundle
        jobs:
            - name: Fastlane build
            commands:
                - chmod 0600 ~/.keys/*
                - ssh-add ~/.keys/*
                - bundle exec fastlane build
        secrets:
            - name: fastlane-env
            - name: ios-cert-repo

## Example application on GitHub

To see an example project of how to configure iOS project for Semaphore CI, visit the [`semaphore-demo-ios-swift-xcode` GitHub repository](https://github.com/semaphoreci-demos/semaphore-demo-ios-swift-xcode).