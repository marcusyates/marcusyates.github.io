---
layout: post
title: "Monorepos with Gradle and Yarn and Webpack"
date: 2020-10-11 22:00:00 +0100
---

I was thinking about a Gradle project that can build Javascript modules. Building Java with Gradle is easy because it already supports it natively, but to build Javascript artifact you will need additional build tools, such as npm and Webpack at hand. 

For a basic do-it-yourself example, I would recommend following Gradle's [Running Webpack with Gradle](https://guides.gradle.org/running-webpack-with-gradle) guide. To make this more interesting, we are going to try something more sophisticated:

1. Gradle to download and install npm and Yarn locally
1. Use Yarn v2 to only download and install Javascript dependencies once
1. Extract common Javascript into module dependencies
1. Use Yarn workspaces to link Javascript modules together
1. Assemble the Javascript bundle artifact with Webpack

So we need an example monorepo with multiple Javascript Gradle modules. Let's start with a `ui` component which could be served up in a web page, and define two local Javascript dependencies: `calculator` and `concatinator`.

So we need an example monorepo with multiple Javascript Gradle modules. Let's start with a `ui` component and define two local Javascript dependencies: `calculator` and `concatinator`. This is a Gradle build and adding a Java webserver project to bundle the `ui` in a web application is pretty easy, but that will make this blog even longer, so not included.

Initially, our folder structure looks like this:

```
├── common
│   └── ui
│       └── calculator (2)
│       └── concatinator (2)
└── ui (1)
```

1. Webpack will build the `ui` module, and include
1. Common re-usable Javascript modules

### Monorepo

In my last two jobs I have worked on large monorepo projects predominantly in Java, with some smaller elements in Javascript and C++. Seeing all your source code in one repository makes it easy to manage project dependencies, reduced refactoring costs, and search and navigation, especially with tools such as Intellij. I spend more time reading and understanding code than writing it these days, so this access is invaluable.

There some are challenges, such how you manage deployment of independent microservices, or when you want to open source a section of your code base. On the whole, monorepos are worth it!

### Gradle Project

For non-trivial multi-language monorepos, I think Buck or Bazel are probably stronger candidates for large monorepos. I use Buck at work, and it is able to handle over 650 modules easily. Buck is great, but Gradle is really popular, so I thought it would be interesting to learn how to integrate Javascript modules in Gradle. 

The outline for the Gradle multi-project repo is:

```
├── settings.gradle.kts
├── build.gradle.kts
├── deps     
│   ├── node  (1)
│   │   └── build.gradle.kts
│   └── yarn  (2)
│       └── build.gradle.kts
├── common
│   └── ui
│       └── calculator
│           └── build.gradle.kts
│       └── concatinator
│           └── build.gradle.kts
└── ui
    └── build.gradle.kts
```

1. Downloads and installs Node and npm.
1. Uses npm (1) to install Yarn.

The build dependencies look something like this:

```
:deps:node -> :deps:yarn -> [ :common:ui:calculator  
                              :common:ui:concatinator ] -> :ui
```

Where the `ui` module source is layed out like this:

```
└── ui
    ├── babel.config.js
    ├── build.gradle.kts
    ├── package.json
    ├── src
    │   └── main.js
    └── webpack.config.js
```

## Yarn for Dependencies

Actually, Yarn 2. Yarn is wonderful at managing Javascipt packaged dependencies which are defined in `package.json` files in each module. We could easily use npm directly, but I think Yarn has some interesting features that will make our monorepo a little less hacky.

 * Workspaces - This Yarn's solution for monorepos and works a bit like linking up local packages, so they look like downloaded ones in the node_modules directory. 
 * Zero copy - Yarn has a mechanism to only download and install one copy of package dependency from npm.

This is going to save a lot of copying of `.js` files. 

Yarn workspaces work really well when you have a `package.json` at, or near, the root directory of the monorepo.

```json
{
  "name": "gradle-yarn",
  "version": "1.0.0",
  "license": "MIT",
  "workspaces": [
    "ui",
    "common/ui/*"
  ]
}
```

The `workspaces` attribute accepts directory names and wildcards. For this example, we will need a `package.json` in each Javascript module:

```
├── package.json (1)
├── common
│   └── ui
│       ├── calculator
│       │   └── package.json (3)
│       └── concatinator
│           └── package.json (3)
└── ui (4)
    └── package.json (2)
```

1. The root package.json
1. Explicit workspace dependency
1. Wildcard workspace dependency
1. A workspace

The `ui/package.json` will include local and external dependencies (like Webpack).

```json
{
  "dependencies": {
    "calculator": "^1.0.0",
    "concatinator": "^1.0.0"
  },
  "devDependencies": {
      "webpack": "^4.44.2",
      "webpack-cli": "^3.3.12"
  }
}
```

To install Javascript dependencies via Yarn is simple, and we'll get Gradle to do the following steps later.

```bash
$ cd $ROOT_DIR
$ export PATH=$PATH:$PATH_TO_YARN
$ yarn install
```

## Building with Webpack

We are going to use Webpack in the `:ui` project to create a Javascript bundle which contains sources from the ui, and it's dependencies.

Seemingly you would not execute a Webpack build for every single dependency as you might with javac when compiling Java dependencies. Instead, you would have one build which assembles everything all in one go. This Webpack build will compile all the Javascript across all the local modules as described by the dependencies of the `package.json`.

With a `package.json` "build" script defined:

```json
{
 "scripts": {
   "build: "webpack --config webpack.config.js"
 }
}
```

We can use Yarn to run the build from the `ui` workspace:

```bash
$ yarn workspace ui build
```

Webpack likes to be configured. This is pretty standard except that [Webpack needs to understand Yarn 2](https://yarnpkg.com/features/pnp#native-support) via a plugin. Babel ensures that my crumby Javascript will work for older browsers and allows me to use the ES Modules style, 

i.e. `export default {};` and `import sauages from './sausages.js'`. 

```javascript
// webpack 4 support for Yarn PnP
const PnpWebpackPlugin = require(`pnp-webpack-plugin`);
const path = require('path');

module.exports = {
    entry: './src/main.js',
    module: {
        rules: [ {
                test: /\.js$/,
                exclude: /node_modules/,
                use: ["babel-loader"]
            } ]
    },
    output: {
        filename: 'main.js',
        path: path.resolve(__dirname, 'build/dist'),
    },
    resolve: {
        extensions: ['*', '.js'],
        plugins: [ PnpWebpackPlugin, ],
    },
    resolveLoader: {
        plugins: [ PnpWebpackPlugin.moduleLoader(module), ],
    },
};
```

## Yarn Gradle Tasks

Until now, we have been looking mostly at the Javascript build tooling like Yarn and Webpack. Holding these together are the Gradle build scripts and custom build tasks. The `:ui` project will need to install dependencies and run Webpack to assemble the final Javascript bundle. 

All depend on Yarn.

Running a `yarn install` in a Gradle `Exec` task boils down to

```kotlin
val install by tasks.registering(Exec::class) {
    dependsOn(configurations["yarn"])
    val yarnExe = file(configurations["yarn"].singleFile)

    inputs.files(yarnExe, file(projectDir.resolve("package.json")))

    workingDir(rootDir)
    environment("PATH", "${environment["PATH"]}:${yarnExe.parent}")
    commandLine("sh", "-c", "yarn install")
}
```

A `"yarn"` configuration provides access to the yarn script installed by `:deps:yarn`, and accessible in Gradle in a task with `file(configurations["yarn"].singleFile)`. 

```kotlin
configurations {
    create("yarn") {
        isCanBeConsumed = false
        isCanBeResolved = true
    }
}

dependencies {
    add("yarn", project(mapOf(
      "path" to ":deps:yarn", 
      "configuration" to "yarn")))
}
```

Then, running Webpack

```kotlin
val webpack by tasks.registering(Exec::class) {
    dependsOn(configurations["yarn"], install)
    val yarnExe = project.file(configurations["yarn"].singleFile)

    inputs.files(yarnExe)
    inputs.dir("src")
    outputs.files(buildDir.resolve("dist/main.js"))

    workingDir(rootDir)
    environment("PATH", "${environment["PATH"]}:${yarnExe.parent}")
    commandLine("sh", "-c", "yarn workspace ui build")
}

tasks["assemble"].dependsOn(webpack)
```

## Installing Node with Gradle

As a true believer of bootstrapping a build, such as downloading Gradle via `gradlew`, I think it is quite powerful and really convenient to have `node`, `npm` and `yarn` installed locally by a normal local user without the need for root access (via `sudo`). Gradle has a nice Ivy repository plugin, and nodejs has tarballs:

```kotlin
repositories {
    ivy {
        setUrl("https://nodejs.org/dist/")
        patternLayout {
            artifact("/v[revision]/[module]-v[revision]-[classifier].[ext]")
        }
        metadataSources {
            artifact()
        }
    }
}

configurations {
    create("setUp")
}

dependencies {
    add("setUp", "node:node:12.18.4:linux-x64@tar.gz")
}
```

Gradle is less good at unpacking `.tar.gz` files that come with symbolic links. Currently, symlinks are treated as regular files with zero size. Until this is fixed in Gradle, the normal `tar` linux tool is perfectly fine, so:

```bash
$ tar --extract --file node-v12.0.0-linux-x64.tar.gz --directory ./build/unpacked/dist
```

can be executed in Gradle as:

```kotlin
val installNode by tasks.registering(Exec::class) {
    doFirst {
        mkdir(file("${buildDir}/unpacked/dist"))
    }

    val setUp = configurations["setUp"]
    dependsOn(setUp)

    val tarGzFile = setUp.find { file -> file.name.startsWith("node") }!!
    val dep = setUp.dependencies.find { d -> d.name.equals("node") }!!
    val dirName = "${dep.name}-v${dep.version}-linux-x64"
    val binDir = "${buildDir}/unpacked/dist/${dirName}/bin"

    extra["npmPath"] = file("${binDir}/npm") // used by artifacts

    inputs.file(tarGzFile)
    outputs.files("${binDir}/node", "${binDir}/npm", "${binDir}/npx")

    commandLine("tar", "--extract", "--file", file(tarGzFile), "--directory", "${buildDir}/unpacked/dist")
}
```

## Summary

With the help of Yarn's workspaces, it is possible to integrate Javascript build tooling into a Gradle multi-project build so that we can build full java web applications with fancy pants modern web UIs, and all under a single monorepo. 

The simple example I created works, but things get interesting when adding a real UI library like Vue.js or React. It is also quite clear dependencies are in package.json and gradle build files. Dependencies described in one place is better.

There are Gradle plugins offering Yarn integration, but I am unsure they really support monorepos and multi-projects. 

Anyway, somethings are more fun when you do it yourself.
