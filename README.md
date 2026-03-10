# Buildpack: Go

This is a buildpack for [Go][go].

## Getting Started

Follow the guide at <https://doc.scalingo.com/languages/go/start>.

There's also a hello world sample app at <https://github.com/Scalingo/sample-go-martini>.

## Example

```console
$ ls -A1
.git
Procfile
go.mod
main.go

$ scalingo create my-go-app
$ git push scalingo master
-----> Go app detected
-----> Installing go1.25.7... done
-----> Running: go install -v -tags paas .
-----> Discovering process types
       Procfile declares types -> web
        Build complete, shipping your container...
         Waiting for your application to boot...
          <-- https://my-go-app.osc-fr1.scalingo.io -->
```

This buildpack will detect your repository as Go if it contains a
[go.mod][gomodules] file.

This buildpack adds a `paas` [build constraint](https://golang.org/pkg/go/build/), to enable
Scalingo-specific code. See the [App Engine build constraints
article](https://blog.golang.org/the-app-engine-sdk-and-workspaces-gopath) for more.

## Go Module Specifics

When using go modules, this buildpack will search the code base for `main`
packages, ignoring any in `vendor/`, and will automatically compile those
packages. If this isn't what you want you can specify specific package spec(s)
via the `go.mod` file's `// +scalingo install` directive (see below).

The `go.mod` file allows for arbitrary comments. This buildpack utilizes [build
constraint](https://golang.org/pkg/go/build/#hdr-Build_Constraints) style
comments to track Scalingo build specific configuration which is encoded in the
following way:

- `// +scalingo goVersion <version>`: the major version of go you would like
  Scalingo to use when compiling your code. If not specified this defaults to the
  buildpack's [DefaultVersion]. Specifying a version < go1.11 will cause a build
  error because modules are not supported by older versions of go.

  Example: `// +scalingo goVersion go1.11`

- `// +scalingo install <packagespec>[, <packagespec>]`: a space separated list of
  the packages you want to install. If not specified, the buildpack defaults to
  detecting the `main` packages in the code base. Generally speaking this should
  be sufficient for most users. If this isn't what you want you can instruct the
  buildpack to only build certain packages via this option. Other common choices
  are: `./cmd/...` (all packages and sub packages in the `cmd` directory) and
  `./...` (all packages and sub packages of the current directory). The exact
  choice depends on the layout of your repository though.

  Example: `// +scalingo install ./cmd/... ./special`

The Go version can also be overridden using the `$GOVERSION` environment
variable, which takes precedence over the `go.mod` directives. Setting
`$GOVERSION` to a major version will result in the buildpack using the latest
released minor version in that series. Since Go doesn't release `.0` versions,
specifying a `.0` version will pin your code to the initial release of the given
major version (ex `go1.24.0` == `go1.24` w/o auto updating to `go1.24.1` when
it becomes available).

```console
scalingo --app my-app env-set GOVERSION=go1.24   # Will use go1.24.X, where X is the latest minor release in the 1.24 series
scalingo --app my-app env-set GOVERSION=go1.23.4 # Pins to go1.23.4
```

If a top level `vendor` directory exists and the `go.sum` file has a size
greater than zero, `go install` is invoked with `-mod=vendor`, causing the build
to skip downloading and checking of dependencies. This results in only the
dependencies from the top level `vendor` directory being used.

### Pre/Post Compile Hooks

If the file `bin/go-pre-compile` or `bin/go-post-compile` exists and is
executable then it will be executed either before compilation (go-pre-compile)
of the repo, or after compilation (go-post-compile).

These hooks can be used to install additional tools, such as `github.com/golang-migrate/migrate`:

```bash
#!/bin/bash
set -e

go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@v4.15.1
```

Because the buildpack installs compiled executables to `bin`, the
`go-post-compile` hook can be written in go if it's installed by the specified
`<packagespec>` (see above).

Example:

```console
$ cat go.mod
// +scalingo install ./cmd/...
$ ls -F cmd
client/ go-post-compile/ server/
```

## Default Procfile

If there is no Procfile in the base directory of the code being built and the
buildpack can figure out the name of the base package (also known as the
module), then a default Procfile is created that includes a `web` process type
that runs the resulting executable from compiling the base package.

For example, if the package name was `github.com/Scalingo/example`, this buildpack
would create a Procfile that looks like this:

```sh
$ cat Procfile
web: example
```

This is useful when the base package is also the only main package to build.

If you have adopted the `cmd/<executable name>` structure this won't work and
you will need to create a [Procfile].


## Using with cgo

The buildpack supports building with C dependencies via [cgo][cgo]. You can set
config vars to specify CGO flags to specify paths for vendored dependencies. The
literal text of `${build_dir}` will be replaced with the directory the build is
happening in. For example, if you added C headers to an `includes/` directory,
add the following config to your app: `scalingo env-set CGO_CFLAGS='-I${
build_dir}/includes'`. Note the used of `''` to ensure they are not converted to
local environment variables

## Using a development version of Go

The buildpack can install and use any specific commit of the Go compiler when
the specified go version is `devel-<short sha>`. The version can be set either
via the `go.mod` file or via the `$GOVERSION` environment variable. The specific
sha is downloaded from Github w/o git history. Builds may fail if GitHub is
down, but the compiled go version is cached.

When this is used the buildpack also downloads and installs the buildpack's
current default Go version for use in bootstrapping the compiler.

Build tests are NOT RUN. Go compilation failures will fail a build.

No official support is provided for unreleased versions of Go.

## Passing a symbol (and optional string) to the linker

This buildpack supports the go [linker's][go-linker] ability (`-X symbol value`)
to set the value of a string at link time. This can be done by setting
`GO_LINKER_SYMBOL` and `GO_LINKER_VALUE` in the application's config before
pushing code. If `GO_LINKER_SYMBOL` is set, but `GO_LINKER_VALUE` isn't set then
`GO_LINKER_VALUE` defaults to [`$SOURCE_VERSION`][source-version].

This can be used to embed the commit sha, or other build specific data directly
into the compiled executable.


If the source code contains a golanglint-ci configuration file in the root of
the source code (one of `/.golangci.yml`, `/.golangci.toml`, or
`/.golangci.json`) then golanci-lint is run at the start of the test phase.

Use one of those configuration files to configure the golanglint-ci run.

[app-engine-build-constraints]: http://blog.golang.org/2013/01/the-app-engine-sdk-and-workspaces-gopath.html
[build-constraint]: http://golang.org/pkg/go/build/
[buildpack]: http://doc.scalingo.com/buildpacks/
[cgo]: http://golang.org/cmd/cgo/
[curl]: https://curl.haxx.se/
[DefaultVersion]: https://github.com/Scalingo/go-buildpack/blob/master/data.json#L4
[docker]: https://www.docker.com/
[go-linker]: https://golang.org/cmd/ld/
[go]: http://golang.org/
[gomodules]: https://github.com/golang/go/wiki/Modules
[gopgsqldriver]: https://github.com/jbarham/gopgsqldriver
[jq]: https://github.com/stedolan/jq
[make]: https://www.gnu.org/software/make/
[Procfile]: https://doc.scalingo.com/platform/app/procfile
[source-version]: http://doc.scalingo.com/app/build-environment
