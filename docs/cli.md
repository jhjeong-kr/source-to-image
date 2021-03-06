# s2i command line interface

This document describes thoroughly all `s2i` subcommands and flags with explanation
of their purpose as well as an example usage.

Currently `s2i` has five subcommands, each of which will be described in the
following sections of this document:

* [create](#s2i-create)
* [build](#s2i-build)
* [rebuild](#s2i-rebuild)
* [usage](#s2i-usage)
* [version](#s2i-version)
* [help](#s2i-help)

Before diving into each of the aforementioned commands, let's have a closer look
at common flags that can be used with all of the subcommands.

| Name                       | Description                                             |
|:-------------------------- |:--------------------------------------------------------|
| `-h (--help)`              | Display help for the specified command |
| `--loglevel`               | Set the level of log output (0-5) (see [Log levels](#log-levels))|
| `-U (--url)`               | URL of the Docker socket to use (default: `unix:///var/run/docker.sock`) |

#### Log levels

There are four log levels:
* Level `0` - produces output from containers running `assemble` script and all encountered errors
* Level `1` - produces basic information about the executed process
* Level `2` - produces very detailed information about the executed process
* Level `3` - produces very detailed information about the executed process, along with listing tar contents
* Level `4` - currently produces same information as level `3`
* Level `5` - produces very detailed information about the executed process, lists tar contents, Docker Registry credentials, and copied source files

**NOTE**: All of the commands and flags are case sensitive!

# s2i create

The `s2i create` command is responsible for bootstrapping a new S2I enabled
image repository. This command will generate a skeleton `.sti` directory and
populate it with sample S2I scripts you can start hacking on.

Usage:

```
$ s2i create <image name> <destination directory>
```

# s2i build

The `s2i build` command is responsible for building the Docker image by combining
the specified builder image and sources. The resulting image will be named according
to the tag parameter.

Usage:
```
$ s2i build <source location> <builder image> [<tag>] [flags]
```
The build command parameters are defined as follows:

1. `source location` - the URL of a GIT repository or a local path to the source code
1. `builder image` - the Docker image to be used in building the final image
1. `tag` - the name of the final Docker image (if provided)

If the build image is compatible with incremental builds, `s2i build` will look for
an image tagged with the same name. If an image is present with that tag and a
`save-artifacts` script is present in the scripts directory, `s2i build` will save the build artifacts from
that image and add them to the tar streamed to the container into `/artifacts`.

#### Build flags

| Name                       | Description                                             |
|:-------------------------- |:--------------------------------------------------------|
| `--callback-url`           | URL to be invoked after a successful build (see [Callback URL](#callback-url)) |
| `-d (--destination)`       | Location where the scripts and sources will be placed prior doing build (see [S2I Scripts](https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md#s2i-scripts)) |
| `--dockercfg-path`         | The path to the Docker configuration file |
| `--incremental`            | Try to perform an incremental build |
| `-e (--env)`               | Environment variables to be passed to the builder eg. `NAME=VALUE,NAME2=VALUE2,...` |
| `-E (--environment-file)`               | Specify the path to the file with environment |
| `--force-pull`             | Always pull the builder image, even if it is present locally (defaults to true) |
| `--run`             | Launch the resulting image after a successful build. All output from the image is being printed to help determine image's validity. In case of a long running image you will have to Ctrl-C to exit both s2i and the running container.  (defaults to false) |
| `-r (--ref)`               | A branch/tag that the build should use instead of MASTER (applies only to GIT source) |
| `--rm`                     | Remove the previous image during incremental builds |
| `--save-temp-dir`          | Save the working directory used for fetching scripts and sources |
| `--context-dir`            | Specify the directory containing your application (if not located within the root path) |
| `-s (--scripts-url)`       | URL of S2I scripts (see [S2I Scripts](https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md#s2i-scripts)) |
| `--recursive`              | Perform recursive git clone when getting sources using git|
| `-q (--quiet)`             | Operate quietly, suppressing all non-error output |

#### Context directory

In the case where your application resides in a directory other than your repository root
folder, you can specify that directory using the `--context-dir` parameter. The
specified directory will be used as your application root folder.

#### Callback URL

Upon completion (or failure) of a build, `s2i` can execute a HTTP POST to a URL with information
about the build:

* `success` - flag indicating the result of the build process (`true` or `false`)
* `payload` - list of messages from the build process

Example: data posted will be in the form:
```
{
    "payload": "A string containing all build messages",
    "success": true
}
```

#### Example Usage

Build a Ruby application from a GIT source, using the official `ruby-20-centos7` builder
image, the resulting image will be named `ruby-app`:

```
$ s2i build git://github.com/mfojtik/sinatra-app-example openshift/ruby-20-centos7 ruby-app
```

Build a Node.js application from a local directory, using a local image, the resulting
image will be named `nodejs-app`:

```
$ s2i build --force-pull=false /home/user/nodejs-app local-nodejs-builder nodejs-app
```

In case of building from the local directory, the sources will be copied into
the builder images using plain filesystem copy if the Git binary is not
available. In that case the output image will not have the Git specific labels.
Use this method only for development or local testing.

Build a Java application from a GIT source, using the official `wildfly-8-centos`
builder image but overriding the scripts URL from local directory.  The resulting
image will be named `java-app`:

```
$ s2i build --scripts-url=file://s2iscripts git://github.com/bparees/openshift-jee-sample openshift/wildfly-8-centos java-app
```

Build a Ruby application from a GIT source, specifying `ref`, and using the official
`ruby-20-centos7` builder image.  The resulting image will be named `ruby-app`:

```
$ s2i build --ref=my-branch git://github.com/mfojtik/sinatra-app-example openshift/ruby-20-centos7 ruby-app
```

***NOTE:*** If the ref is invalid or not present in the source repository then the build will fail.

Build a Ruby application from a GIT source, overriding the scripts URL from a local directory,
and specifying the scripts and sources be placed in `/opt` directory:

```
$ s2i build --scripts-url=file://s2iscripts --destination=/opt git://github.com/mfojtik/sinatra-app-example openshift/ruby-20-centos7 ruby-app
```

# s2i rebuild

The `s2i rebuild` command is used to rebuild an image already built using S2I,
or the image that contains the required S2I labels.
The rebuild will read the S2I labels and automatically set the builder image,
source repository and other configuration options used to build the previous
image according to the stored labels values.

Optionally, you can set the new image name as a second argument to the rebuild
command.

Usage:

```
$ s2i rebuild <image name> [<new-tag-name>]
```


# s2i usage

The `s2i usage` command starts a container and runs the `usage` script which prints
information about the builder image. This command expects `builder image` name as
the only parameter.

Usage:
```
$ s2i usage <builder image> [flags]
```

#### Usage flags

| Name                       | Description                                             |
|:-------------------------- |:--------------------------------------------------------|
| `-d (--destination)`       | Location where the scripts and sources will be placed prior invoking usage (see [S2I Scripts](https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md#s2i-scripts))|
| `-e (--env)`               | Environment variables passed to the builder eg. `NAME=VALUE,NAME2=VALUE2,...`) |
| `--force-pull`             | Always pull the builder image, even if it is present locally |
| `--save-temp-dir`          | Save the working directory used for fetching scripts and sources |
| `-s (--scripts-url)`       | URL of S2I scripts (see [Scripts URL](https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md#s2i-scripts))|

#### Example Usage

Print the official `ruby-20-centos7` builder image usage:
```
$ s2i usage openshift/ruby-20-centos7
```


# s2i version

The `s2i version` command prints the version of S2I currently installed.


# s2i help

The `s2i help` command prints help either for the `s2i` itself or for the specified
subcommand.

### Example Usage

Print the help page for the build command:
```
$ s2i help build
```

***Note:*** You can also accomplish this with:
```
$ s2i build --help
```
