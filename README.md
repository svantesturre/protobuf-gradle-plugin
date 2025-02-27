:exclamation: Please [read release notes](https://github.com/google/protobuf-gradle-plugin/releases)
before upgrading the plugin, as usage or compatibility requirements may change.

# Protobuf Plugin for Gradle [![Status](https://travis-ci.org/google/protobuf-gradle-plugin.svg?branch=master)](https://travis-ci.org/google/protobuf-gradle-plugin)
The Gradle plugin that compiles Protocol Buffer (aka. Protobuf) definition
files (``*.proto``) in your project. There are two pieces of its job:
 1. It assembles the Protobuf Compiler (``protoc``) command line and use it to
    generate Java source files out of your proto files.
 2. It adds the generated Java source files to the input of the corresponding
    Java compilation unit (_sourceSet_ in a Java project; _variant_ in an
    Android project), so that they can be compiled along with your Java sources.

For more information about the Protobuf Compiler, please refer to
[Google Developers Site](https://developers.google.com/protocol-buffers/docs/reference/java-generated?csw=1).

## Latest Version
The latest version is ``0.8.10``. It requires at least __Gradle 3.0__ and __Java 8__.
It is available on Maven Central. To add dependency to it:
```gradle
buildscript {
  repositories {
    mavenCentral()
  }
  dependencies {
    classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.10'
  }
}
```

To try out the head version, you can download the source and build it
with ``./gradlew install -x test`` (we skip tests here because they
require Android SDK), then:

```gradle
buildscript {
  repositories {
    mavenLocal()
  }
  dependencies {
    classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.11-SNAPSHOT'
  }
}
```

## Examples

Stand-alone examples are available for each of gradle's supported languages.
 * [Groovy](https://github.com/google/protobuf-gradle-plugin/tree/master/examples/exampleProject) ___(Default)___
   * Run `../../gradlew build` under the example directory to test it out.
 * [Kotlin](https://github.com/google/protobuf-gradle-plugin/tree/master/examples/exampleKotlinDslProject) ___(Experimental)___
   * Run `./gradlew build` under the Kotlin example directory to test it out. This example is set up with Gradle 4.10, the minimum required version.


Directories that start with `testProject` can also serve as usage
examples for advanced options, although they cannot be compiled as
individual projects.


## Adding the plugin to your project
This plugin must work with either the Java plugin or the Android plugin.


### Using the `apply` method
The Java plugin or the Android plugin must be applied before the Protobuf plugin:

```gradle
apply plugin: 'java'
apply plugin: 'com.google.protobuf'
```

```gradle
apply plugin: 'com.android.application'  // or 'com.android.library'
apply plugin: 'com.google.protobuf'
```

The experimental Android plugin is not supported yet (#85).

### Using the Gradle plugin DSL
The order of the plugins doesn't matter:

```gradle
plugins {
  id "com.google.protobuf" version "0.8.10"
  id "java"
}
```


## Configuring Protobuf compilation

The Protobuf plugin assumes Protobuf files (``*.proto``) are organized in the
same way as Java source files, in _sourceSets_. The Protobuf files of a
_sourceSet_ (or _variant_ in an Android project) are compiled in a single
``protoc`` run, and the generated files are added to the input of the Java
compilation run of that _sourceSet_ (or _variant_).

### Customizing source directories
The plugin adds a new sources block named ``proto`` alongside ``java`` to every
sourceSet. By default, it includes all ``*.proto`` files under
``src/$sourceSetName/proto``. You can customize it in the same way as you would
customize the ``java`` sources.

**Java** projects: use the top-level ``sourceSet``:

```gradle
sourceSets {
  main {
    proto {
      // In addition to the default 'src/main/proto'
      srcDir 'src/main/protobuf'
      srcDir 'src/main/protocolbuffers'
      // In addition to the default '**/*.proto' (use with caution).
      // Using an extension other than 'proto' is NOT recommended,
      // because when proto files are published along with class files, we can
      // only tell the type of a file from its extension.
      include '**/*.protodevel'
    }
    java {
      ...
    }
  }
  test {
    proto {
      // In addition to the default 'src/test/proto'
      srcDir 'src/test/protocolbuffers'
    }
  }
}
```

**Android** projects: use ``android.sourceSets``:

```gradle
android {
  sourceSets {
    main {
      proto {
        ...
      }
      java {
        ...
      }
    }
  }
}
```

### Customizing Protobuf compilation
The plugin adds a ``protobuf`` block to the project. It provides all the
configuration knobs.


#### Locate external executables

By default the plugin will search for the ``protoc`` executable in the system
search path. We recommend you to take the advantage of pre-compiled ``protoc``
that we have published on Maven Central:

```gradle
protobuf {
  ...
  // Configure the protoc executable
  protoc {
    // Download from repositories
    artifact = 'com.google.protobuf:protoc:3.0.0'
  }
  ...
}
```

You may also specify a local path.
```gradle
protobuf {
  ...
  protoc {
    path = '/usr/local/bin/protoc'
  }
  ...
}
```

Multiple assignments are allowed in the ``protoc`` block. The last one wins.

You may also run ``protoc`` with codegen plugins.  For a codegen
plugin named as "foo", ``protoc`` will by default use
``protoc-gen-foo`` from system search path.  You can also specify a
downloadable artifact or a local path for it in the ``plugins`` block,
in the same syntax as in the ``protoc`` block above. This will __not__
apply the plugins. You need to configure the tasks in the
``generateProtoTasks`` block introduced below to apply the plugins
defined here.

```gradle
protobuf {
  ...
  // Locate the codegen plugins
  plugins {
    // Locate a plugin with name 'grpc'. This step is optional.
    // If you don't locate it, protoc will try to use "protoc-gen-grpc" from
    // system search path.
    grpc {
      artifact = 'io.grpc:protoc-gen-grpc-java:1.0.0-pre2'
      // or
      // path = 'tools/protoc-gen-grpc-java'
    }
    // Any other plugins
    ...
  }
  ...
}
```

The syntax for `artifact` follows [Artifact Classifiers](https://docs.gradle.org/3.3/userguide/dependency_management.html#sub:classifiers)
where the default classifier is `project.osdetector.classifier` (ie
`"${project.osdetector.os}-${project.osdetector.arch}"`) and the default extension is `"exe"`.
Non-C++ implementations of codegen plugins can be used if a constant
`classifier` is specified (eg `"com.example:example-generator:1.0.0:-jvm8_32"`).

#### Customize code generation tasks

The Protobuf plugin generates a task for each ``protoc`` run, which is for a
sourceSet in a Java project, or a variant in an Android project. The task has
configuration interfaces that allow you to control the type of outputs, the
codegen plugins to use, and parameters.

You must configure these tasks in the ``generateProtoTasks`` block, which
provides you helper functions to conveniently access tasks that are tied to a
certain build element, and also ensures you configuration will be picked up
correctly by the plugin.

**DONOTs**:
 - DO NOT assume the names of the tasks, as they may change.
 - DO NOT configure the tasks outside of the ``generateProtoTasks`` block,
   because there are subtle timing constraints on when the tasks should be
   configured.

```gradle
protobuf {
  ...
  generateProtoTasks {
    // all() returns the collection of all protoc tasks
    all().each { task ->
      // Here you can configure the task
    }

    // In addition to all(), you may select tasks by various criteria:

    // (Java-only) returns tasks for a sourceSet
    ofSourceSet('main')

    // (Android-only selectors)
    // Returns tasks for a flavor
    ofFlavor('demo')
    // Returns tasks for a buildType
    ofBuildType('release')
    // Returns tasks for a variant
    ofVariant('demoRelease')
    // Returns non-androidTest tasks
    ofNonTest()
    // Return androidTest tasks
    ofTest()
  }
}
```


Each code generation task has two collections:
 - `builtins`: code generators built in `protoc`, e.g., `java`, `cpp`,
   `python`.
 - `plugins`: code generation plugins that work with `protoc`, e.g.,
   `grpc`. They must be defined in the `protobuf.plugins` block in
   order to be added to a task.


#### Configure what to generate

Code generation is done by protoc builtins and plugins.  Each
builtin/plugin generate a certain type of code.  To add or configure a
builtin/plugin on a task, list its name followed by a braces block.
Put options in the braces if wanted.  For example:

```gradle
task.builtins {
  // This yields
  // "--java_out=example_option1=true,example_option2:/path/to/output"
  // on the protoc commandline, which is equivalent to
  // "--java_out=/path/to/output --java_opt=example_option1=true,example_option2"
  // with the latest version of protoc.
  java {
    option 'example_option1=true'
    option 'example_option2'
  }
  // Add cpp output without any option.
  // DO NOT omit the braces if you want this builtin to be added.
  // This yields
  // "--cpp_out=/path/to/output" on the protoc commandline.
  cpp { }
}

task.plugins {
  // Add grpc output without any option.  grpc must have been defined in the
  // protobuf.plugins block.
  // This yields
  // "--grpc_out=/path/to/output" on the protoc commandline.
  grpc { }
}
```

#### Default outputs

**Java** projects: the `java` builtin is added by default: without any further specification, Java classes will be generated during the build process.

**Python** output can be generated by adding the `python` builtin:

```gradle
protobuf {
  generatedFilesDir = "$projectDir/generated"

  generateProtoTasks {
    all().each { task ->
      task.builtins {
        // Generates Python code in the output folder:
        python { }

        // If you wish to avoid generating Java files:
        remove java
      }
    }
  }
}
```

See [this section](#change-where-files-are-generated) for details about where the code will be generated.


**Android** projects: no default output will be added.  Since Protobuf
3.0.0, [protobuf-lite](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22protobuf-lite%22)
is the recommended Protobuf library for Android, and you will need to
add it as a codegen plugin.  For example:

```gradle

dependencies {
  // You need to depend on the lite runtime library, not protobuf-java
  compile 'com.google.protobuf:protobuf-lite:3.0.0'
}

protobuf {
  protoc {
    // You still need protoc like in the non-Android case
    artifact = 'com.google.protobuf:protoc:3.0.0'
  }
  plugins {
    javalite {
      // The codegen for lite comes as a separate artifact
      artifact = 'com.google.protobuf:protoc-gen-javalite:3.0.0'
    }
  }
  generateProtoTasks {
    all().each { task ->
      task.builtins {
        // In most cases you don't need the full Java output
        // if you use the lite output.
        remove java
      }
      task.plugins {
        javalite { }
      }
    }
  }
}
```

#### Generate descriptor set files

```gradle
{ task ->
  // If true, will generate a descriptor_set.desc file under
  // $generatedFilesBaseDir/$sourceSet. Default is false.
  // See --descriptor_set_out in protoc documentation about what it is.
  task.generateDescriptorSet = true

  // Allows to override the default for the descriptor set location
  task.descriptorSetOptions.path =
    "${projectDir}/build/descriptors/{$task.sourceSet.name}.dsc"

  // If true, the descriptor set will contain line number information
  // and comments. Default is false.
  task.descriptorSetOptions.includeSourceInfo = true

  // If true, the descriptor set will contain all transitive imports and
  // is therefore self-contained. Default is false.
  task.descriptorSetOptions.includeImports = true
}
```

#### Change where files are generated

By default generated Java files are under
``$generatedFilesBaseDir/$sourceSet/$builtinPluginName``, where
``$generatedFilesBaseDir`` is ``$buildDir/generated/source/proto`` by default,
and is configurable. E.g.,

```gradle
protobuf {
  ...
  generatedFilesBaseDir = "$projectDir/src/generated"
}
```

The subdirectory name, which is by default ``$builtinPluginName``, can also be
changed by setting the ``outputSubDir`` property in the ``builtins`` or
``plugins`` block of a task configuration within ``generateProtoTasks`` block
(see previous section). E.g.,

```gradle
{ task ->
  task.plugins {
    grpc {
      // Write the generated files under
      // "$generatedFilesBaseDir/$sourceSet/grpcjava"
      outputSubDir = 'grpcjava'
    }
  }
}
```

### Protos in dependencies

If a Java project contains proto files, they will be packaged in the jar files
along with the compiled classes.

Protos in dependencies (e.g. upstream jars) can be put in either in the ``compile``
configuration or the ``protobuf`` configuration.

If the dependency is put in the ``compile`` configuration, the proto files are
extracted to an ``extracted-include-protos`` directory and added to the ``--proto_path``
flag of the protoc command line, so that they can be imported by the proto files
of the current project. The imported proto files will not be compiled since
they have already been compiled in their own projects. Example:

```gradle
dependencies {
  compile project(':someProjectWithProtos')
  testCompile files("lib/some-testlib-with-protos.jar")
}
```

If the dependency is put in the ``protobuf`` configuration, the proto files are
extracted to a ``extracted-protos`` directory and added to the protoc command
line as files to compile, in the same protoc invocation as the current project's
proto files (if any). Example:

```gradle
dependencies {
  // protos can be from a local package,
  protobuf files('lib/protos.tar.gz')
  // ... a local directory,
  protobuf files('ext/')   // NEVER use fileTree(). See issue #248.
  // ... or an artifact from a repository
  testProtobuf 'com.example:published-protos:1.0.0'
}
```

## Pre-compiled ``protoc`` artifacts
This [Maven Central directory](https://repo1.maven.org/maven2/com/google/protobuf/protoc/)
lists pre-compiled ``protoc`` artifacts that can be used by this plugin.

## Tips for IDEs

### IntelliJ IDEA

Be sure to enable delegate IDE build/run actions to Gradle so
that Intellij does not use its internal build mechanism to
compile source code. This plugin ensures that code generation
happens before Gradle's build step. If the setting is off,
Intellij's own build system will be used instead of Gradle.

Enable the setting with:
```
Settings -> Build, Execution, Deployment
  -> Build Tools -> Gradle -> Runner
  -> Delegate IDE build/run actions to gradle.
```

This plugin integrates with the ``idea`` plugin and automatically
registers the proto files and generated Java code as sources.


```gradle
apply plugin: 'idea'

protobuf {
    ...
    generatedFilesBaseDir = "$projectDir/gen"
}

clean {
    delete protobuf.generatedFilesBaseDir
}

idea {
    module {
        // proto files and generated Java files are automatically added as
        // source dirs.
        // If you have additional sources, add them here:
        sourceDirs += file("/path/to/other/sources");
    }
}
```


## Testing the plugin

``testProject*`` are testing projects that uses this plugin to compile
``.proto`` files. Because the tests include an Android project, you
need to install
[Android SDK Tools](https://developer.android.com/sdk/index.html#Other).

After you made any change to the plugin, be sure to run these tests.
```
$ ./gradlew test
```
