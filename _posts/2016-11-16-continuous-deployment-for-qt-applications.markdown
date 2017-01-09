---
layout: post
title:  "Continuous deployment for Qt applications"
description: "Deploy automatically with Travis CI and AppVeyor"
date:   2016-11-16 13:11:51 -0400
categories: qt continuous-deployment
comments: true
---

# Introduction

This article will show you how to set up a continuous deployment pipeline for Windows, macOS, and Linux-- specifically for building and deploying Qt applications. It contains template code and discussion that demonstrates how to automatically deploy your Qt apps.

We'll define continuous deployment as the process of building and deploying your application automatically. It doesn't necessarily mean that every build needs to be deployed.

The templates are configuration files and build scripts for use on [Travis CI][travis] for macOS and Ubuntu and [AppVeyor][appveyor] for Windows.

You can reference the scripts that inspired this article in the [Kryvos][kryvos] ([https://github.com/adolby/Kryvos][kryvos]) and [Dialogue][dialogue] ([https://github.com/adolby/Dialogue][dialogue]) repositories.

# Build script design

[Travis CI][travis] and [AppVeyor][appveyor] support running scripts (shell, batch, Python, etc.) from the project repository, which means that you could set up build scripts that work on an arbitrary deployment service.

The templates demonstrate how to build and deploy an installer and portable archive for Windows and Linux and a dmg archive for macOS.

# Qt specifics

## Installing Qt on your continuous deployment platform

**Travis CI**

[Qt][qt] can be installed with package managers if you are willing to accept the Qt version that is currently provided for the continuous deployment platform's Linux distribution and version or on [Homebrew][homebrew] for macOS. However, these packages are frequently not up to date.

If you want to use the latest version, you'll need to find a build of Qt for your operating system. The Qt Project provides official builds of Qt, but their installers have no command line mode, which means that the latest Qt version can't be installed easily.

You can build Qt yourself or install Qt from the official installer on another system and archive the files. You'll then need to host the file and download and unpack it in your build script. Finally, add the binaries to your source folder and add the binary path to your PATH variable.

Alternatively, Ben Lau details how to execute the Qt Project installers on Linux with the [Qt-CI project][qtci]. It looks like a great alternative solution.

In the templates provided, I've hosted Qt versions on a GitHub repo I use for providing unofficial Qt builds. You could create your own similar repo or use mine, which can be found at https://github.com/adolby/qt-more-builds.

**AppVeyor**

AppVeyor has current versions of Qt installed on their build images, so you'll just need to add the Qt binary path to your PATH variable. You can build with the Visual C++ compiler from multiple Visual Studio versions or with MinGW. If you're building with the Visual C++ compiler, you'll need to call vcvarsall.bat with %PLATFORM% as the argument, where %PLATFORM% is an AppVeyor environment variable.

## Qt dependencies

Qt provides tools that copy dependency files for deployment on macOS and Windows. They work very well, though there is no tool for Linux deployment. The Qt documentation on deployment is a great resource to use when you're setting the command line parameters up for macdeployqt and windeployqt. The parameters are similar between the two tools, but note that the syntax is slightly different.

## QML apps

The deployment tools can also determine QML dependencies from your QML source files if you pass the directory containing the QML files in your project as a parameter. To deploy a QML app on Linux, you'll need to note your dependencies yourself or compare the output from macdeployqt or windeployqt.

The Dialogue repo shows how to copy the necessary files for deploying a QML project on Linux.

## Linux packages

Travis CI runs Ubuntu 12.04 or Ubuntu 14.04; the latter is available by specifying dist: trusty in your .travis.yml configuration file. If you are developing your application on a different version of Ubuntu or another operating system, some software packages may not available in the same version you developed your application with.

To resolve this, you can create your own package repository, build your application's missing dependency from source, or try with an older version.

# Templates

Below are templates of build config files for Travis CI and AppVeyor and build script files that run on them.

You can write your build scripts in whatever language that you prefer, as long as it will run on your build environment of choice. I've written shell scripts for macOS and Linux and batch files for Windows for you to use as templates.

# Travis CI

The Travis CI config file specifies building and deployment on macOS and Ubuntu 14.04. To get started on Travis CI you'll need to create a GitHub (or other Git hosting provider) repository and then enable it on Travis CI.

The config file demonstrates deployment to [GitHub Releases][releases]. There are other deployment targets available. The details are in the Travis CI documentation.

To allow Travis to deploy to GitHub Releases, you'll need to [create a personal access token on GitHub][githubtoken]. Next, you'll use the Travis tool to encrypt (hash) it as a secure environment variable. Then copy the output over the secure key in the deployment section with your encrypted (hashed) personal access key.

This process is detailed under the [Environment Variable section in the Travis CI documentation][encrypttravis].

Alternatively, you could add the unencrypted personal access token as an environment variable through the Travis CI settings page for your repository.

Travis CI is configured to build on all branches and will only deploy on tagged commits.

The templates use the Git tag name as a version number, which allows you to differentiate between deployed builds with [semantic versioning][semantic], or whatever versioning scheme you prefer.

Make sure to replace YourApp with your app's name.

**.travis.yml**

{% highlight yml %}
dist: trusty
sudo: required

git:
  depth: 1

language: cpp

matrix:
  include:
    - os: osx
      compiler: clang
      osx_image: xcode8
    - os: linux
      compiler: gcc
      addons:
        apt:
          sources:
            - ubuntu-toolchain-r-test
          packages:
            - gcc-6
            - g++-6

script:
  - if [[ "${TRAVIS_OS_NAME}" == "osx" ]]; then chmod +x build_macOS.sh ; fi
  - if [[ "${TRAVIS_OS_NAME}" == "osx" ]]; then TAG_NAME=${TRAVIS_TAG} ./build_macOS.sh ; fi
  - if [[ "${TRAVIS_OS_NAME}" == "linux" ]]; then sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-6 100 ; fi
  - if [[ "${TRAVIS_OS_NAME}" == "linux" ]]; then sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-6 100 ; fi
  - if [[ "${TRAVIS_OS_NAME}" == "linux" ]]; then chmod +x build_linux.sh ; fi
  - if [[ "${TRAVIS_OS_NAME}" == "linux" ]]; then TAG_NAME=${TRAVIS_TAG} CXX="g++-6" CC="gcc-6" ./build_linux.sh ; fi

deploy:
  - provider: releases
    api_key:
      secure: NDZ/RLvQzTWOkUnFhRqDFlBUgWJUtre5f12aBcolDMoIxyFfoZ/87GWXBk+OD4zYH7kQa0TY21Ap8zKJOpLW5vqbqPXtHW2cEHYez5l8FBxF1JpEUL0TLplbNbgjGGKkTatXYG9M0jCTKb8cXm3X7er3AAIKxJUhsLUdQDjR5SlJgqdShcOMuha1SXRuOhxV85Tp6ZEwXFnKeBQI9k4v1TyJMHY6bS6Sv8uMmpExVdznFgHNiDrdtLbqPyIyw4cJbOPCxlSmdynUkG7jvm0bHlMqzwsxVrWREL/QX0A1L5Orux+Wc187XCZunKCw7Eg05lesXySj3E3v0DoChjNuTIm7hBR6zCxHkHTgqdyfuiN1fKwLbZ2zHrhxIz206V/wddrFwYhmly243ayAua4ExJbQj+N1s47ttqQWUUJz/sOHorPsT9fG97SneyyEeGwtt/q7DKVpX/g4SA2UIIHxJxLlqw5TVpAatimhd09mg5HFt1i4yZExoBQSNQePCB7bsu5hvd2Wfs9l+3b10VAwbTsT1+n6oEHcjzt25nS7ekGuzSj7LYRUxbqeEdRzKWNGgQ16UEcldgAQlFPeqRcjVS5g+KQwIQwk8IPg5C8b5E4/r+Ig40O51aU2fXI33fMpJhhpQXoDc1id13G6vvuMMAKS/0O/jFP1184dPQ/hjiA=
    file:
      - build/macOS/clang/x86_64/release/YourApp_${TRAVIS_TAG}_macos.zip
    overwrite: true
    skip_cleanup: true
    on:
      tags: true
      condition: $TRAVIS_OS_NAME = osx
  - provider: releases
    api_key:
      secure: NDZ/RLvQzTWOkUnFhRqDFlBUgWJUtre5f12aBcolDMoIxyFfoZ/87GWXBk+OD4zYH7kQa0TY21Ap8zKJOpLW5vqbqPXtHW2cEHYez5l8FBxF1JpEUL0TLplbNbgjGGKkTatXYG9M0jCTKb8cXm3X7er3AAIKxJUhsLUdQDjR5SlJgqdShcOMuha1SXRuOhxV85Tp6ZEwXFnKeBQI9k4v1TyJMHY6bS6Sv8uMmpExVdznFgHNiDrdtLbqPyIyw4cJbOPCxlSmdynUkG7jvm0bHlMqzwsxVrWREL/QX0A1L5Orux+Wc187XCZunKCw7Eg05lesXySj3E3v0DoChjNuTIm7hBR6zCxHkHTgqdyfuiN1fKwLbZ2zHrhxIz206V/wddrFwYhmly243ayAua4ExJbQj+N1s47ttqQWUUJz/sOHorPsT9fG97SneyyEeGwtt/q7DKVpX/g4SA2UIIHxJxLlqw5TVpAatimhd09mg5HFt1i4yZExoBQSNQePCB7bsu5hvd2Wfs9l+3b10VAwbTsT1+n6oEHcjzt25nS7ekGuzSj7LYRUxbqeEdRzKWNGgQ16UEcldgAQlFPeqRcjVS5g+KQwIQwk8IPg5C8b5E4/r+Ig40O51aU2fXI33fMpJhhpQXoDc1id13G6vvuMMAKS/0O/jFP1184dPQ/hjiA=
    file:
      - build/linux/gcc/x86_64/release/YourApp_${TRAVIS_TAG}_linux_x86_64_portable.zip
      - installer/linux/YourApp_${TRAVIS_TAG}_linux_x86_64_installer
    overwrite: true
    skip_cleanup: true
    on:
      tags: true
      condition: $TRAVIS_OS_NAME = linux

notifications:
  email:
    recipients:
      - andrewdolby@gmail.com
    on_success: change
    on_failure: change
{% endhighlight %}

# macOS

This macOS build script builds and run tests for CI, then packages the application as a dmg file archive for deployment. Make sure to replace YourApp with your app's name. $TAG_NAME is a Travis CI specific environment variable containing the tag name of the current build.

**build_macOS.sh**

{% highlight shell %}
#!/bin/bash

set -o errexit -o nounset

# Hold on to current directory
project_dir=$(pwd)

# Output macOS version
sw_vers

# Update platform
echo "Updating platform..."
brew update

# Install p7zip for packaging
brew install p7zip

# Install npm appdmg if you want to create custom dmg files with it
# npm install -g appdmg

# Install Qt
echo "Installing Qt..."
cd /usr/local/
sudo wget https://github.com/adolby/qt-more-builds/releases/download/5.7/qt-opensource-5.7.0-macos-x86_64-clang.zip
sudo 7z x qt-opensource-5.7.0-macos-x86_64-clang.zip &>/dev/null
sudo chmod -R +x /usr/local/Qt-5.7.0/bin/

# Add Qt binaries to path
PATH=/usr/local/Qt-5.7.0/bin/:${PATH}

# Create temporary symlink for Xcode8 compatibility
cd /Applications/Xcode.app/Contents/Developer/usr/bin/
sudo ln -s xcodebuild xcrun

# Build your app
echo "Building YourApp..."
cd ${project_dir}
qmake -config release
make

# Build and run your tests here

# Package your app
echo "Packaging YourApp..."
cd ${project_dir}/build/macOS/clang/x86_64/release/

# Remove build directories that you don't want to deploy
rm -rf moc
rm -rf obj
rm -rf qrc

echo "Creating dmg archive..."
macdeployqt YourApp.app -qmldir=../../../../../src -dmg
mv YourApp.dmg "YourApp_${TAG_NAME}.dmg"

# You can use the appdmg command line app to create your dmg file if
# you want to use a custom background and icon arrangement. I'm still
# working on this for my apps, myself. If you want to do this, you'll
# remove the -dmg option above.
# appdmg json-path YourApp_${TRAVIS_TAG}.dmg

# Copy other project files
cp "${project_dir}/README.md" "README.md"
cp "${project_dir}/LICENSE" "LICENSE"
cp "${project_dir}/Qt License" "Qt License"

echo "Packaging zip archive..."
7z a YourApp_${TAG_NAME}_macos.zip "YourApp_${TAG_NAME}.dmg" "README.md" "LICENSE" "Qt License"

echo "Done!"

exit 0
{% endhighlight %}

# Ubuntu 14.04

This Ubuntu build script builds and run tests for CI, then packages the application as a portable archive and creates an installer executable for deployment. Make sure to replace YourApp with your app's name. $TAG_NAME is a Travis CI specific environment variable containing the tag name of the current build.

**build_linux.sh**

{% highlight shell %}
#!/bin/bash

set -o errexit -o nounset

# Update platform
echo "Updating platform..."

# Install p7zip for packaging archive for deployment
sudo -E apt-get -yq --no-install-suggests --no-install-recommends --force-yes install p7zip-full

# Need to install chrpath to set up proper rpaths. This is necessary
# to allow Qt libraries to be visible to each other. Alternatively,
# you could use qt.conf, which wouldn't require chrpath.
sudo -E apt-get -yq --no-install-suggests --no-install-recommends --force-yes install chrpath

# Hold on to current directory
project_dir=$(pwd)

# Define your Qt install path for later use
qt_install_dir=/opt

# Install Qt
echo "Installing Qt..."
cd ${qt_install_dir}
echo "Downloading Qt files..."
sudo wget https://github.com/adolby/qt-more-builds/releases/download/5.7/qt-opensource-5.7.0-linux-x86_64.7z
echo "Extracting Qt files..."
sudo 7z x qt-opensource-5.7.0-linux-x86_64.7z &> /dev/null

# Install Qt Installer Framework
echo "Installing Qt Installer Framework..."
sudo wget https://github.com/adolby/qt-more-builds/releases/download/qt-ifw-2.0.3/qt-installer-framework-opensource-2.0.3-linux.7z
sudo 7z x qt-installer-framework-opensource-2.0.3-linux.7z &> /dev/null

# Add Qt binaries to path
echo "Adding Qt binaries to path..."
PATH=${qt_install_dir}/Qt/5.7/gcc_64/bin/:${qt_install_dir}/Qt/QtIFW2.0.3/bin/:${PATH}

# Build YourApp
echo "Building YourApp..."
cd ${project_dir}

# Output qmake version info to make sure we have the right install
# directory in the PATH variable
qmake -v

qmake -config release -spec linux-g++-64
make

# Build and run your tests here

# Package YourApp
echo "Packaging YourApp..."
cd ${project_dir}/build/linux/gcc/x86_64/release/YourApp/

# Remove build directories that you don't want to deploy
rm -rf moc
rm -rf obj
rm -rf qrc

echo "Copying files for archival..."

# Copy ICU libraries
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libicui18n.so.56.1" "libicui18n.so.56"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libicuuc.so.56.1" "libicuuc.so.56"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libicudata.so.56.1" "libicudata.so.56"

# Copy YourApp's Qt dependencies, including QML dependencies if your
# app uses QML. If your app doesn't use QML, you'll probably need to
# include libQt5Widgets.so.5. You will need additional library
# dependency files if your app uses Qt features found in other
# modules. You can find out your app's library and QML dependencies by
# checking the Qt docs or by referencing the library and QML files
# that are copied by macdeployqt or windeployqt on macOS or Windows.
mkdir platforms
mkdir -p Qt/labs/

# You'll always need these libraries on Linux.
cp "${qt_install_dir}/Qt/5.7/gcc_64/plugins/platforms/libqxcb.so" "platforms/libqxcb.so"
cp "${qt_install_dir}/Qt/5.7/gcc_64/plugins/platforms/libqminimal.so" "platforms/libqminimal.so"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Core.so.5.7.0" "libQt5Core.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Gui.so.5.7.0" "libQt5Gui.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5DBus.so.5.7.0" "libQt5DBus.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5XcbQpa.so.5.7.0" "libQt5XcbQpa.so.5"

# You may or may not need these, depending on which Qt features you
# use
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Svg.so.5.7.0" "libQt5Svg.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Qml.so.5.7.0" "libQt5Qml.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Quick.so.5.7.0" "libQt5Quick.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5QuickControls2.so.5.7.0" "libQt5QuickControls2.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5QuickTemplates2.so.5.7.0" "libQt5QuickTemplates2.so.5"
cp "${qt_install_dir}/Qt/5.7/gcc_64/lib/libQt5Network.so.5.7.0" "libQt5Network.so.5"

cp -R "${qt_install_dir}/Qt/5.7/gcc_64/qml/Qt/labs/settings/" "Qt/labs/"
cp -R "${qt_install_dir}/Qt/5.7/gcc_64/qml/QtGraphicalEffects/" "."
cp -R "${qt_install_dir}/Qt/5.7/gcc_64/qml/QtQuick/" "."
cp -R "${qt_install_dir}/Qt/5.7/gcc_64/qml/QtQuick.2/" "."

# Copy other project files
cp "${project_dir}/README.md" "README.md"
cp "${project_dir}/LICENSE" "LICENSE"
cp "${project_dir}/Qt License" "Qt License"

# Use chrpath to set up rpaths for Qt's libraries so they can find
# each other
chrpath -r \$ORIGIN/.. platforms/libqxcb.so
chrpath -r \$ORIGIN/.. platforms/libqminimal.so
chrpath -r \$ORIGIN/../.. QtQuick/Controls.2/libqtquickcontrols2plugin.so
chrpath -r \$ORIGIN/../.. QtQuick/Templates.2/libqtquicktemplates2plugin.so

# Copy files to a folder with configuration for creating an installer
# with Qt Installer Framework
echo "Copying files for installer..."
cp -R * "${project_dir}/installer/linux/packages/com.yourappproject.yourapp/data/"

# Copy files to a portable archive inside the containing folder
# created earlier
echo "Packaging portable archive..."
cd ..
7z a YourApp_${TAG_NAME}_linux_x86_64_portable.zip YourApp

# Use the Qt Installer Framework to create an installer executable
echo "Creating installer..."
cd "${project_dir}/installer/linux/"
binarycreator --offline-only -c config/config.xml -p packages YourApp_${TAG_NAME}_linux_x86_64_installer

echo "Done!"

exit 0
{% endhighlight %}

# AppVeyor

The AppVeyor config file specifies building and deployment on Windows. To get started on AppVeyor you'll need to create a GitHub (or other Git hosting provider) repository and then enable it on AppVeyor.

The config file demonstrates deployment to [GitHub Releases][releases]. There are other deployment targets available. The details are in the AppVeyor documentation.

To allow Travis to deploy to GitHub Releases, you'll need to [create a personal access token on GitHub][githubtoken]. Next, you'll use the Encrypt data tool, which can be found on the accounts dropdown menu after signing in on AppVeyor, to encrypt (hash) it as a secure environment variable. Then copy the output over the secure key in the deployment section with your encrypted (hashed) personal access key.

This process is detailed under the [Environment variables section in the AppVeyor documentation][encryptappveyor].

Alternatively, you could add the unencrypted personal access token as an environment variable through the AppVeyor settings page for your repository.

AppVeyor is configured to build on all branches and will only deploy on tagged commits.

The template uses the Git tag name as a version number, which allows you to differentiate between deployed builds with [semantic versioning][semantic], or whatever versioning scheme you prefer.

Make sure to replace YourApp with your app's name.

**appveyor.yml**

{% highlight yml %}
image: Visual Studio 2015

environment:
  matrix:
  - QT: C:\Qt\5.7\msvc2015_64
    PLATFORM: amd64
    COMPILER: msvc
    VSVER: 14

clone_depth: 1

init:
  - set TAG_NAME=%APPVEYOR_REPO_TAG_NAME%

build_script:
  - call "build_windows.cmd"

artifacts:
  - path: build\windows\msvc\x86_64\release\YourApp_$(appveyor_repo_tag_name)_windows_x86_64_portable.zip
    name: portable
  - path: installer\windows\x86_64\YourApp_$(appveyor_repo_tag_name)_windows_x86_64_installer.exe
    name: installer

deploy:
  - provider: GitHub
    tag: $(appveyor_repo_tag_name)
    release: $(appveyor_repo_tag_name)
    description: $(appveyor_repo_tag_name)
    auth_token:
      secure: bOwrg0z7hv/7CnAQD2q+sf74q2vH40mWJLZYc8EzYvqkrXk9KYnd23rVCJ3Fsrqs
    artifact: portable, installer
    draft: false
    prerelease: false
    force_update: true
    on:
      appveyor_repo_tag: true
{% endhighlight %}

# Windows

This Windows build script builds and run tests for CI, then packages the application as a portable archive and creates an installer executable for deployment. Make sure to replace YourApp with your app's name. %APPVEYOR_REPO_TAG_NAME% is an AppVeyor specific environment variable containing the tag name of the current build.

**build_windows.cmd**

{% highlight powershell %}
echo on

SET project_dir="%cd%"

echo Set up environment...
set PATH=%QT%\bin\;C:\Qt\Tools\QtCreator\bin\;C:\Qt\QtIFW2.0.1\bin\;%PATH%
call "C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\vcvarsall.bat" %PLATFORM%

echo Building YourApp...
qmake -spec win32-msvc2015 CONFIG+=x86_64 CONFIG-=debug CONFIG+=release
nmake

echo Running tests...

echo Packaging...
cd %project_dir%\build\windows\msvc\x86_64\release\
windeployqt --qmldir ..\..\..\..\..\src\ YourApp\YourApp.exe

rd /s /q YourApp\moc\
rd /s /q YourApp\obj\
rd /s /q YourApp\qrc\

echo Copying project files for archival...
copy "%project_dir%\README.md" "YourApp\README.md"
copy "%project_dir%\LICENSE" "YourApp\LICENSE.txt"
copy "%project_dir%\Qt License" "YourApp\Qt License.txt"

echo Copying files for installer...
mkdir "%project_dir%\installer\windows\x86_64\packages\com.yourappproject.yourapp\data\"
robocopy YourApp\ "%project_dir%\installer\windows\x86_64\packages\com.yourappproject.yourapp\data" /E

echo Packaging portable archive...
7z a YourApp_%TAG_NAME%_windows_x86_64_portable.zip YourApp

echo Creating installer...
cd %project_dir%\installer\windows\x86_64\
binarycreator.exe --offline-only -c config\config.xml -p packages YourApp_%TAG_NAME%_windows_x86_64_installer.exe
{% endhighlight %}

# Conclusion

I hope that these templates provide a good start for you to set up continuous deployment for your project!

Please contact me (andrewdolby@gmail.com) or leave a comment if you have any questions or suggestions, and I'll be happy to address them in this article. I'll also be happy to help with problems you're having with continuous integration/deployment.

[qt]: https://www.qt.io/
[kryvos]: https://github.com/adolby/Kryvos
[dialogue]: https://github.com/adolby/Dialogue
[travis]: https://travis-ci.org/
[appveyor]: https://www.appveyor.com/
[homebrew]: http://brew.sh/
[semantic]: http://semver.org/
[releases]: https://help.github.com/categories/releases/
[encrypttravis]: https://docs.travis-ci.com/user/environment-variables/#Defining-encrypted-variables-in-.travis.yml
[encryptappveyor]: https://www.appveyor.com/docs/build-configuration/#secure-variables
[githubtoken]: https://help.github.com/articles/creating-an-access-token-for-command-line-use/
[qtci]: https://github.com/benlau/qtci
