# Android Project Setup

## Environment Setup

[Android Studio](http://developer.android.com/sdk/index.html), which is based off of [JetBrains'](https://www.jetbrains.com/) [IntelliJ Idea](https://www.jetbrains.com/idea/), is the official Android IDE and comes bundled with the Android SDK. Prior to installing Android Studio and the Android SDK, ensure that you have installed the latest version of the [Java Development Kit (JDK)](http://www.oracle.com/technetwork/java/javase/downloads/index.html). Further assistance installing Android Studio and the Android SDK can be found [here](https://developer.android.com/sdk/installing/index.html?pkg=studio).

## Build System

Android utilizes [Gradle](https://gradle.org/getting-started-android/) as the official [Android build system](https://developer.android.com/sdk/installing/studio-build.html). Gradle features the ability to manage/download dependencies, customize keystores and manage multiple variants of an app in the form of product flavors. Gradle also offers the ability write and run customizable scripts in the form of tasks. The preferred method of using Gradle is through the [Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html) which will automatically download the correct version of Gradle prior to building your app. More advanced information on the use of Gradle with Android can be found in Android's [Gradle Plugin User Guide](http://tools.android.com/tech-docs/new-build-system/user-guide).

## File Structure

The typical Android file structure looks something like what you see below.

```
├─ app
│  ├─ libs
│  ├─ src
│  │  ├─ androidTest
│  │  │  └─ java
│  │  │     └─ com/originate/project/tests
│  │  └─ main
│  │     ├─ assets
│  │     ├─ java
│  │     │  └─ com/originate/project
│  │     ├─ res
│  │     └─ AndroidManifest.xml
│  ├─ build.gradle
│  └─ proguard-rules.pro
├─ build.gradle
└─ settings.gradle
```

### Root Directory

Most of the files in the root directory are auto generated by Android Studio and require little to no adjustments. The top-level `build.gradle` file is used to define configuration options that are common to all sub-projects while `settings.gradle` maintains references to all of your sub-projects.

```gradle
// build.gradle

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.2.3'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}
```

```gradle
// settings.gradle

include ':app'
```

The `app` directory is a sub-project that contains all of the code relevant to the specific application or library project. The example above includes only one sub-project directory, but you can have multiple directories if your application utilizes your own library projects.

### Gradle Configuration

Inside a subproject directory, you'll find a `libs` directory, `src` directory and a `build.gradle` file. The `libs` & `src` directories are rather self-explanatory as they typically house `.jar` files for external libraries and your sub-project's source code respectively. On the other hand, the `build.gradle` file is a bit more complex and handles a number of things as seen below.

##### Android Plugin

In order to use gradle to build your Android app, you'll first need to apply the Android plugin to this build so that the `android {...}` element becomes available for use.

```gradle
// app/build.gradle

apply plugin: 'com.android.application'
```

Inside the `android {...}` element you'll find all of the Android specific build options. The first of which will be specifying the version of the Android sdk and build tools that your app should be compiled against. Assuming your app supports them, you'll typically want to use the latest versions here.

```gradle
compileSdkVersion 22
buildToolsVersion "22.0.1"
```

###### Default Configs

The `defaultConfig {...}` element houses the configuration for the `main` and `androidTest` source sets. It's typical for your `targetSdkVersion` to match your `compileSdkVersion` from earlier. Additionally, Android package name conventions typically dictate that your application id consists of your reverse domain name followed by the name of your application.

```gradle
// app/build.gradle

defaultConfig {
    testApplicationId "com.originate.project.test"
    testInstrumentationRunner "com.originate.sample.test.ProjectInstrumentationTestRunner"
    testHandleProfiling true
    testFunctionalTest true

    applicationId "com.originate.project"
    minSdkVersion 16
    targetSdkVersion 22
    versionCode 1
    versionName "1.0"
    }
```

###### Signing Configs

Android requires all apps to be digitally signed prior to installation. While debug builds will typically be automatically signed with a debug certificate generated by the Android SDK, release builds should always be signed with your own private keystore prior to distribution. 

A new keystore can be generated via **Build > Generate Signed APK** in Android Studio or via the command line `keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000`.

```
# keystore.properties

storePassword=password
keyAlias=release_key
keyPassword=password
```

It's important that you **never** under any circumstance or form commit your keystore credentials to a code repository even if you think it's private. Instead, keystore credentials can be stored locally in a `keystore.properties` file that is placed inside the root directory of your project and as environment variables in your CI/CD build system. Make sure that you remember to add `keystore.properties` to your `.gitignore` so that it doesn't get accidentally committed to your repo.

```gradle
// app/build.gradle

signingConfigs {
    release {
        storeFile new File("release.keystore")

        if (project.hasProperty("CI") && project.property("CI")) {
            storePassword project.property("storePassword")
            keyAlias project.property("keyAlias")
            keyPassword project.property("keyPassword")
        } else {
            def props = new Properties();
            props.load(new FileInputStream(new File("keystore.properties")))

            storePassword props.getProperty("storePassword")
            keyAlias props.getProperty("keyAlias")
            keyPassword props.getProperty("keyPassword")
        }
    }
}
```
```bash
# Build System
./gradlew -PCI=$CI -PstorePassword=$STORE_PASSWORD -PkeyAlias=$KEY_ALIAS -PkeyPassword=$KEY_PASSWORD assembleRelease

# Locally
./gradlew assembleRelease
```

The above configuration makes use of gradle properties to read credentials in from command-line on builds that occur on your build system, while builds that occur locally utilize your `keystore.properties` file.

###### Build Types

```gradle
// app/build.gradle

buildTypes {
    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        signingConfig signingConfigs.release
    }
    customBuildType.initWith(buildTypes.debug)
}
```
Gradle allows for the creation of new build types and special configuration for specific build types. Additionally for each build type, gradle auto-creates an `assemble<BuildTypeName>` gradle task. Here you'll commonly want to set [Proguard](http://developer.android.com/tools/help/proguard.html) obfuscation rules and signing config settings for release builds.

###### Product Flavors

Gradle also offers the ability to include product flavors which allow a single app to have multiple variants with slight source code changes. Each product flavor will have its own dedicated source directory where flavor specific code is housed. This is a common tactic that is used by apps that have a free version with ads and a paid ad free version.

```gradle
// app/build.gradle

productFlavors {
    free { 
       packageName "com.originate.project.free"
    }
    paid { 
        packageName "com.originate.project.paid"
    }
}
```

##### Dependencies

Application dependencies are outlined outside of the Android plugin inside a separate `dependencies {...}` element. Gradle supports the resolution of remote dependencies via [jCenter](https://bintray.com/bintray/jcenter) or [Maven](http://search.maven.org) repositories. These remote dependencies are declared in the form of `group:name:version` where `version` follows [semantic versioning](http://semver.org/). Semantic versions are written in the format of `MAJOR.MINOR.PATCH` and support the `+` wildcard which automatically resolves to the latest version.

```gradle
compile "com.google.code.gson:gson:2.2.1"
compile "com.jakewharton:butterknife:7.+"
```

Dependencies not housed in a remote repository can be included by storing the corresponding jar into your libs folder and adding the following to your gradle file.

```gradle
compile fileTree(dir: 'libs', include: ['*.jar'])

compile name: "logentries-android-2.1.4" // where logentries is replaced by whatever your jar is
```

Additionally, dependencies can be declared for use in only specific product flavors. This is a commonly used tactic for testing frameworks.

```gradle
testCompile "org.mockito:mockito-core:1.9.5"
``` 

## Running Project

### Physical Device

In order to use you Android phone or tablet for development and testing, you'll need to enable developer options and usb debugging in your settings. Developer options can be found on devices running Android 3.2 or older under **Settings > Applications > Development** and under **Settings > Developer options** on Android 4.0 and newer. On Android 4.2 and newer, Developer options are hidden by default and **Build number** under **Settings > About phone** needs to be tapped seven times to enable them. More information on setting your device up for testing and debugging can be found on the [official Android guide](http://developer.android.com/tools/device.html).

### Emulation

##### Android Virtual Device (AVD)

An Android Virtual Device (AVD) is the emulator configuration for Android's built-in Android Emulator. The easiest way to create and manage Android emulators is through the graphical [AVD Manager](http://developer.android.com/tools/devices/managing-avds.html) that can be accessed from within Android Studio. AVD offers both ARM and Intel x86 based emulators. The system images for the various emulator versions can be downloaded via Android Studio's [SDK Manager](http://developer.android.com/tools/help/sdk-manager.html). If your computer supports it, it's recommended that you use x86 based emulators and select **Use Host GPU** so that you can take advantage of [Intel's hardware acceleration](http://developer.android.com/tools/devices/emulator.html#acceleration). If you need access to Google Maps or other Google services, be sure to use a [Google APIs](https://software.intel.com/en-us/blogs/2014/03/06/now-available-android-sdk-x86-system-image-with-google-apis) system image.

##### Genymotion

Even with hardware acceleration, Android emulators are notoriously slow. Luckily a better and faster Android emulator exists in the form of [Genymotion](https://www.genymotion.com). Genymotion emulators do not come with Google Play Services out of the box, however it is possible to [install it manually](https://stevenkideckel.wordpress.com/2014/12/27/how-to-install-google-play-services-on-genymotion/).

## Additional Resources

* A complete Gradle and .gitignore example can be found [here](/android/files/code/setup).
* Command line interfacing via [Android Debug Bridge (ADB)](http://developer.android.com/tools/help/adb.html)
* [Helpful tips](https://wiki.cyanogenmod.org/w/Doc:_debugging_with_logcat) on using [logcat](http://developer.android.com/tools/help/logcat.html) to capture debug and error logs.
* AVD emulator [setup instructions](Emulator%20Setup.md) for non-developers.