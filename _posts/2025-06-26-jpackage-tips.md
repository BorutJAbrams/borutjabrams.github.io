---
layout: post
title: JPackage tips
---

I've been tasked with updating a Java 8 with JavaFX application to Java 24.

Many moons have passed since Java 8 was released, so I've decided to use two "new" Java features:
- [JPackage](https://docs.oracle.com/en/java/javase/24/docs/specs/man/jpackage.html)
- [Modularity](https://www.baeldung.com/java-modularity)

<!--more-->

It seemed like a good idea, for many advantages:
- Modularity will add the advantage of creating a smaller application, saving bandwidth and disk space
  (and attack surface)
- JPackage will create an executable or an installer containing the application and all the dependencies
- No need to separately install and manage the JRE
- Every time you ship a new version of your app, the user will automatically have the latest JRE version...
 if you update it (don't worry, I'm not judgin you...)

Since they're not the target of this article, I'm skipping the Java 8 to 24 and JavaFX migrations, 
let's talk about the tips to use JPackage.

Oh, and the goal was to obtain a Windows installer so that's the direction of this article. There's a chance that
I'll update it also for Mac and Linux, if I'll have to create installers for those platforms too.



## tl;dr (everybody loves tl;dr)

From the root of your Maven project run:

```bash
mvn clean install

rm -rf my-app/target/classes
rm -rf my-app/target/generated-sources
rm -rf my-app/target/maven-archiver
rm -rf my-app/target/maven-status

mvn dependency:copy-dependencies -DoutputDirectory=libs 

jpackage \
    --name "MyApp" \
    --type app-image \                          # Or "--type msi" for a Windows installer
    --module com.example.app/com.example.app.Main \
    --module-path "my-app/target;my-app/libs"
```

No luck with the tl;dr? OK, let's dig deeper!



## Prerequisites

Well, not much. If you're not using JavaFX, your favourite JDK will be enough.

If you need JavaFX, you can download a JDK and JavaFX separately like I did, or you can download a JDK like
[Zulu](https://www.azul.com/downloads), which already contains JavaFX.

> First tip: When packaging your app and using JavaFX, you *might* need the JavaFX `.jmod` files, which are not present
> in the Zulu JDK or the JavaFX SDK. I've downloaded them from [Gluon](https://gluonhq.com/products/javafx/),
> and I've put them in a `jmod` directory in my JavaFX installation.
>
> Another good location is in the Java home dir, so you can reference it using environment variables.
 
> Yes, I said *might*: since we're packing a modular application, the dependencies in the `pom.xml` and the `requires`
> statements in the `module-info.java` might be enough. Maybe a simple JDK with no JavaFX might be enough.
> More on that later.

If you need to create a Windows installer, you also need the [WiX Toolset](https://github.com/wixtoolset/wix), but
be careful, because (citing the [Wix Toolset ReadMe](https://github.com/wixtoolset/issues/blob/main/README.md)):

> To ensure the long-term sustainability of this project, use of the WiX Toolset requires an Open Source Maintenance 
> Fee. While the source code is freely available under the terms of the LICENSE, all other aspects of the 
> project--including opening or commenting on issues, participating in discussions and downloading releases--require 
> adherence to the Maintenance Fee.

Which means that to use their binaries you need to pay the Maintenance Fee. Otherwise: fork/clone the repo and build
it by yourself. By reading the [Open Source Maintenance Fee FAQs](https://opensourcemaintenancefee.org/consumers/faq/),
it's my understanding that this should be OK:

> Q: What if I don’t want to pay the Maintenance Fee? 
>
> That’s fine. You can download the project’s source code and follow the Open Source license for the software.
>
> Do not open issues. Do not ask questions. Do not download releases. Do not reference packages via a package manager.
> Do not use anything other than the source code released under the Open Source license.
>
> Also, if you choose to not pay the Maintenance Fee, but find yourself returning to check on the status of issues
> or review answers to questions others ask, you are still using the project and need to pay the Maintenance Fee.



## Developing/building

YMMV here, but some assumptions are:
- You have a Maven project with one or more modules
- Your main class is called `Main` and:
  - It's located in the package `com.example.app`
  - The Maven module which contains it is called `my-app`
  - The Maven module which contains it has a module descriptor 
(the `module-info.java` file) containing something like this:
    
    ```java
    module com.example.app {
    
        [...]
    
        exports com.example.app;
    }
    ```
- After launching `mvn clean install` we expect to have a `my-app-1.2.3.jar` file in the `target` directory.



## Preparing to package



### Removing class files

Before running JPackage, remember that it gets confused when the same directory contains both the jar file and the
target classes. So start with moving your jar to another dir, or deleting what's not needed:

```bash
rm -rf my-app/target/classes
rm -rf my-app/target/generated-sources
rm -rf my-app/target/maven-archiver
rm -rf my-app/target/maven-status

# Yeah, who cares? Let's delete everything!
```

If you forget to delete the `.class` files, JPackage will give you an error like this:

```
Bundler Windows Application Image skipped because of a configuration problem: 
java.lang.module.FindException: Two versions of module com.example.app found in 
my-app\target (my-app-1.2.3.jar and classes)
```



### Including dependencies

Then, you need to gather the .jar dependencies. There's not a task more boring than this, apart from writing meaningful
commit messages. That's why we can use a nice Maven plugin:

```bash
mvn dependency:copy-dependencies -DoutputDirectory=libs
```
which will copy all the dependencies .jar files to the `my-app/libs` directory. Very convenient!



## Installer or app-image?
Ooh, now comes the fun part!

> So, do you want to create an installer?
>
> Well, don't.

Start by creating an app-image and make that work. In case of problems, directly creating an installer will lead you to
not knowing if the app isn't working due to a problem during the installer creation, or due to the app-image itself.
Been there, done that, don’t put the cart before the horse.



## What about JLink?

JLink is the tool that we can use to create a custom JRE which will be then used by our nice and shiny packaged app.
But if we don't have particular needs, we can completely skip using it. JPackage will internally invoke it on our 
behalf, greatly simplifying our work.

> Life is already quite complicated, why make it worse?
> 
> Unless you really need to (performance, size, extra modules), skip JLink.
> 
> JPackage will run it for you, and probably do a better job. Definitely better than I would.



## The parameters

The most important JPackage parameters are:

- `--module` the module/full classpath of your main class,
- `--module-path` the paths to the directories containing the `.jar` and `.jmod` files,
- `--add-modules` which other modules should be included when building your app (and the custom JRE).

> You may be tempted to create a big fat command line blob with all the modules and paths to all your jars and
> dependencies.
> 
> Again: don't.

> The good news is that when packaging a modular application, the `--add-modules` parameter should not be needed.
> This is because this info is already present in the `module-info.java` files.
>
> Another good news is that probably also the JavaFX `.jmod` files are not needed.

JPackage is a command that can become very messy, very confusing, very quickly. For the first run go minimal and keep 
just what you know you need: the `--module` one, and the `--module-path` to your jar file and libs. You'll add the
others incrementally, based on the error messages you're getting.



## Packaging! <3

In case of a simple application, this should be enough and produce a directory `MyApp` containing your packaged
application, wrapped a native executable. Yay!

```bash
$ jpackage \
    --name "MyApp" \
    --type app-image \
    --module com.example.app/com.example.app.Main
    --module-path "my-app/target;my-app/libs"
```

If you're lucky... that's it! Try running the `MyApp/MyApp.exe` program. If it works, you can now replace the
`--type app-image` with `--type msi` and you'll get a nice installer. You can also create installers for Linux and
MacOS too, check the JPackage docs.



## Useful installer parameters

I won't list all the possible 
[command-line parameters](https://docs.oracle.com/en/java/javase/24/docs/specs/man/jpackage.html#jpackage-options).
I encourage you to check them, as you can nicely tweak the installer behavior.

But there's one that's particularly interesting: `--win-upgrade-uuid`.

Windows Installer uses two UUIDs:
- Product Code. JPackage will calculate it using the version you provided when creating the msi file. If you generate 
  an MSI installer multiple times using the same `--app-version`, the result will be the same Product Code. If we change
  the `--app-version`, the Product Code changes too. We cannot  install two programs with the same Product Code,
  that's why JPackage generates it based on the app version we provide and we can't manually change it.
- Upgrade Code. This is the important one. I suggest you to manually generate it
  (there are some [online tools](https://www.uuidgenerator.net/)). This is unique to your application and stays the same
  even when the Product Code changes. The Upgrade Code uniquely identifies your application and allows your MSI 
  Installer to check if your application is already installed in order to understand if it's a new install or an update.

The Upgrade Code mechanism is very useful also from a tooling point of view: you can switch to another packaging tool,
copy the Upgrade Code from the old tool to the new one, and your installer made with the new tool will be able to find
and upgrade your application packaged with the old one.

This is what I did when switching from the old tool to JPackage, and it worked nicely! Now I can upgrade my old Java 8
app (packed with the old tool) with the migrated Java 24 app (made with JPackage) and everything works flawlessly.



### Caveats



#### Generating an EXE installer

Good luck with that. If you're successful please come back to me and tell me how you did it.

The installer gets generated, but I always get this error:

```bash
WaitForSingleObject() failed. System error [6](system error 6 (The handle is invalid))
```

Extensive Googling and ChatGpting lead me to exactly zero results and zero luck with this error.

It's not a big deal for me, since my goal was to create an MSI installer, but maybe for you it is.



#### Upgrading from a 32 to a 64-bits application

Yeah, well, no caveats no party, but it's not a big deal.

The pre-migration app was Java 8 based and was a 32-bits application. The post-migration is a Java 24 based,
64-bits application.

The old application was installed in the `C:/Program Files (x86)` directory. Interestingly, the first upgrade will
install the app still in the `Program Files (x86)` directory.

When upgrading for the second time, eventually the app will be correctly deleted from the `Program Files (x86)`
directory and installed in the `Program Files` one.

Anyway I've tested it quite extensively and I didn't find any problem during the execution, but I still felt that it
could be useful to note it.



## Bonus 1: all-inclusive Maven templates

If you prefer a magical `pom.xml` that does everything, you can find them here:
- [JPackageScriptFX](https://github.com/dlemmermann/JPackageScriptFX): note that this one doesn't use modularization
- [maven-jpackage-template](https://github.com/wiverson/maven-jpackage-template): from the ReadMe file: "Java + Maven +
  GitHub Actions = Native Desktop Apps"

Check which one best suits your needs. Personally I prefer the manual way, using shell scripts, at least for now.



## Bonus 2: adding a splash screen

Who doesn't love splash screens? They make our computer so... so... 90-ish...

A splash screen is useful in Java desktop applications, since they're sloooooow to load. The splash screen gives you the
impression that something is happening.

Now the problem is that the splash screen is also slooow to load, just a little bit less than the main app window, but
hey... when your boss wants a splash screen, you give him a spash screen!

We need to do two main activities here:
1. Copy the splash screen to a place where JPackage can find it
2. Tell JPackage to use it

I solved the first step using the `maven-resource-plugin`:

```xml
<plugin>
    <!-- Copies the splash.png file to the target dir, so it will get packaged with JPackage -->
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-resources</id>
            <phase>package</phase>
            <goals>
                <goal>copy-resources</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/jpackage-resources</outputDirectory>
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <includes>
                            <include>splash.png</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```
This will copy the `src/main/resources/splash.png` file to the `my-app/target/jpackage-resources` directory.

And the second step is solved by adding these two parameters to the jpackage command:

```bash
--input my-app/target/jpackage-resources
--java-options "-splash:app/splash.png"
```

- `--input` instructs JPackage to copy the content of the `my-app/target/jpackage-resources` directory to the
  target app-image directory.
- `--java-options` adds the splash command line option to the java command line which will execute your app.
  This will slowly show your fancy splash screen before even more slowly showing your even more fancy app!



## App not packaging



### Missing module-path, no `--add-modules` used

If you get an error like:

```bash
jlink failed with: Error: Module jakarta.xml.bind not found, required by com.example.app
```

You forgot to put the `my-app/libs` path in the --module-path argument.



### Missing module-path while using `--add-modules`

> If you're not sure if you need or not the `--add-modules` parameter, probably you don't. Try deleting it.

But if you need to manually add an extra module using this parameter, you might get this error:

> jlink failed with: Error: Module javafx.fxml not found

As you can see, if we compare this error with the previous one, the `required by com.example.app` part is missing.
JPackage is telling you that yeah, you provided him a module to add, but it's not present in the `--module-path` you
gave him.

Add this module's directory path to the `--module-path` and the error should be solved.



## App not starting

When running the app, if using JavaFX, you might get this error:

```bash
Error initializing QuantumRenderer: no suitable pipeline found
```

This is the universe telling you that 
[you need to include](https://github.com/javafxports/openjdk-jfx/issues/237#issuecomment-426909275) the `.jmods` 
when building the app-image. So try with this command:

```bash
$ jpackage \
    --name "MyApp" \
    --type app-image \
    --module com.example.app/com.example.app.Main
    --module-path "my-app/target;my-app/libs;C:/Java/javafx-sdk-24.0.1/jmods"
```

Replacing `C:/Java/javafx-sdk-24.0.1/jmods` with the directory containing your JavaFX `.jmod` files.



## MSI not starting

This is not a JPackage specific error. If you try to run your `.msi` installer, but you get the following error:

```
This installation package could not be opened. Contact the application vendor to verify
that this is a valid Windows Installer package.
```

This probably means that you're running your installer from a 
[directory junction](https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions#junctions).
Open the real directory and try again.
