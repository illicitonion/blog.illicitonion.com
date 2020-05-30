+++
title = "Scripted/Command Line Android Emulator Installation"
+++

If you're anything like me, you tend to script things like installation, because inevitably, you're going to do it again, and you'll never quite remember the right sequence of steps needed. In particular, I was trying to install the Android emulator, such that I can run tests in it without any manual intervention; able to spin up machines and have them configure themselves to Just Work.

It turns out, the Android emulator goes out of its way to make this tricky. You need a handful of components to get the Android emulator running:
* The Platform Tools
* The version-specific SDK
* The ABI/system images

Two of these things are easily installed from the command line! One of them seems only accessible through the UI! (That last one, there)

But fortunately, if you happen to know the URL where the zip file containing the ABI is, you can download it, and unzip it to the right place!

Then, on first launch, you get a modal dialog asking whether you want to send usage statistics to Google. Useful. But annoying, when you're not a user using the tool!

So, here is my script to do all the magic you need to actually get your android emulator working (on Linux):

```bash
wget http://dl.google.com/android/android-sdk_r15-linux.tgz
tar xf android-sdk_r15-linux.tgz
cd android-sdk-linux
tools/android update sdk --no-ui -t android-14,platform-tools
mkdir -p system-images/android-14
cd system-images/android-14
wget https://dl-ssl.google.com/android/repository/sysimg_armv7a-14_r01.zip
unzip sysimg_armv7a-14_r01.zip
mkdir -p ~/.android
echo "pingId=0" > ~/.android/ddms.cfg
```
