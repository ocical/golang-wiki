# Go 1.11 Modules

Go 1.11 will add preliminary support for versioned modules as proposed [here](https://golang.org/design/24301-versioned-go).

Go modules will be an [experimental](https://research.swtch.com/vgo-accepted) opt-in feature in Go 1.11, with the hope of incorporating feedback and finalizing the feature for Go 1.12. 

## Current Status

* The recent work by the Go team on versioned Go modules started outside of the main Go repository with the `vgo` tool, but on 2018-07-12 support for versioned Go modules [landed](https://groups.google.com/d/msg/golang-dev/a5PqQuBljF4/61QK4JdtBgAJ) in the main Go repository. 
* Beta support for modules is now also available starting with [Go 1.11 beta 2](https://groups.google.com/d/msg/golang-dev/A6TCp2kCoss/XLQoI4MeBgAJ) (released on 2018-07-20).
* Development work on modules is now [occurring exclusively in the main Go repository](https://groups.google.com/d/msg/golang-dev/a5PqQuBljF4/61QK4JdtBgAJ), with an automatic export from the main Go repository to the vgo repository when there is a snapshot ready for people still using `vgo`.

## Installing

To use modules, [install the Go toolchain from source](https://golang.org/doc/install/source) on the `master` branch, or install the `vgo` binary from the [`vgo` subrepository](https://github.com/golang/vgo) (and replace `go` with `vgo` in the commands below).

You can activate module support in one of three ways:
* Invoke the `go` command in a directory outside of the `$GOPATH/src` tree, with a valid `go.mod` file in the current directory or any parent of it and the environment variable `GO111MODULE` unset (or explicitly set to `auto`).
* Invoke the `go` command with `GO111MODULE=on` in the command environment.
* Invoke the `vgo` binary (built from the subrepository) as a binary _named_ `vgo`.

## New Concepts

### Major Versions

Go modules must be [semantically versioned](https://semver.org/) in the form `v(major).(minor).(patch)` (for example, `v0.1.0`, `v1.2.3`, or `v3.0.1`). If using Git, [tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging) released commits with their versions. (Stand-alone distributed module repositories, such as [Project Athens](https://github.com/gomods/athens), are in the works.)

The major version of a module must for now be included in both the module path and the package import path if the major version is v2 or higher. Module versions of v1 and v0 must not be included in the path. Modules with different paths are different modules. Thus `me.io/mymod` is different then `me.io/mymod/v2` and may import packages from one major version to another major version.

The behavior of modules for existing packages with post-`v1` tags is still in flux; an important related recent change was [issue 26238](https://golang.org/issue/26238), which substantially [improved the behavior](https://github.com/golang/go/issues/25967#issuecomment-407567904) for existing packages with post-`v1` tags.

### Modules

Go packages now live in modules. The path of a module is a URL path where it may be found, and may include a major version at the end of the path (but nowhere else). Module source code moutside of `$GOPATH`. Two example modules are `rsc.io/goversion` and `github.com/kardianos/vtest/v3`.

Modules are defined by a `go.mod` file that lives in the root of the module. Module files may include comments and will look familiar to a go programmer. Here is an example `go.mod` file:
```
module github.com/kardianos/vmain/v3

require (
    github.com/kardianos/vtest/v2 v2.0.2
)
```

There are 4 directives: "module", "require", "exclude", "replace". Module paths may be quoted but are not required to be.

Including major versions in import paths will produce incompatibilities with old versions of Go. To work around this prior versions of the go tool have been updated and released to continue building as before. (See [issue 25069](https://github.com/golang/go/issues/25069).)

`exclude` and `replace` directives only operate on the current (“main”) module. `exclude` and `replace` directives in modules other than the main module are ignored.

TODO: show example exclude and replace directives.

There are two ways to release a v2 (or higher) module version.
 1. Create a `v2` directory and place a new `go.mod` file in that folder. The module path must end with `/v2`. Tag the release with `v2.0.0`.
 2. Update the `go.mod` file to include a `/v2` at the end of the module path. Tag the release with `v2.0.0`.
    * To avoid confusion with this approach, consider putting the `v2.*.*` commits on a separate `v2` branch.

Packages are imported relative to the full module path: `import "me.io/mymod/v2/pkg1"` for package `pkg1` in module `me.io/mymod/v2`, or `import "me.io/mymod/pkg1"` for package `pkg1` in module `me.io/mymod` (v1 or v0).

### Version Selection

The version of each module used in a build is always the semantically highest of the versions explicitly `require`d by the module or one of its dependencies. This effectively locks versions into place until the module author or user chooses a new version explicitly. Use `go list -m` to list selected module versions.

Different major versions are distinct modules. A `/v2` module will never be compared with a `/v3` module, even if the rest of the module path is the same, but `me.io/mymod` may be included alongside `me.io/mymod/v2`. (This allows a v1 module to be implemented in terms of its v2 replacement or vice-versa.)

## Defining a Module

To create a `go.mod` for an existing project, follow the following steps.

1. Navigate to the root of the module's source tree and activate module mode in the `go` command.

   ```
   $ cd $GOPATH/src/<project path>
   $ export GO111MODULE=on
   ```

   or

   ```
   $ cd <project path outside $GOPATH/src>
   ```

2. Create the initial module definition and write it to the `go.mod` file.

   ```
   $ go mod -init
   ```

   This step converts any existing from a [`dep`](https://github.com/golang/dep) `Gopkg.lock` file or other [supported dependency format](https://tip.golang.org/pkg/cmd/go/internal/modconv/?m=all#pkg-variables).

   If `go mod` cannot determine an appropriate package path, or if you need to override that path, use the `-module` flag:

   ```
   $ go mod -init -module example.com/path/to/my/module/v2
   ```


3. Fill in requirements for any missing or unconverted dependencies, and remove modules that are not needed to satisfy any imports. The `-sync` flag synchronizes the `go.mod` file with the source code in the module.

   ```
   $ go mod -sync
   ```

4. Test the module as configured to ensure that it works with the selected versions.

   ```
   $ go test ./...
   ```

5. (Optional) Run the tests for all imported modules to check for incompatibilities.

   ```
   $ go test ...
   ```

### Updating Dependencies

Day-to-day adding, removing, upgrading, and downgrading of dependencies should be done using 'go get', which will automatically update the go.mod file.

To upgrade to the latest version for all transitive dependencies of the current module:
 * run `go get -u` to use newer minor or patch releases
 * run `go get -u=patch` to use newer patch releases

To upgrade or downgrade to a more specific version, 'go get' allows version selection to be overridden by adding an @version suffix or "module query" to the package argument, such as `go get github.com/gorilla/mux@v1.6.2`, `go get github.com/gorilla/mux@e3702bed2`, or `go get github.com/gorilla/mux@'<v1.6.2'`. See the ["Module queries"](https://tip.golang.org/cmd/go/#hdr-Module_queries) and ["Module-aware go get"](https://tip.golang.org/cmd/go/#hdr-Module_aware_go_get) sections of the tip documentation for more information.

After upgrading or downgrading any dependencies, you may then want to run the tests again for all imported modules to check for incompatibilities:
   ```
   $ go test ...
   ```

## Additional Resources

### Documentation and Proposal

* Current [official modules documentation on tip](https://tip.golang.org/cmd/go/#hdr-Modules__module_versions__and_more)
  * For more about modules, see 'go help modules'
  * For more about the 'go mod' command, see 'go help mod'
  * For more about the behavior of 'go get' when in module-aware mode, see 'go help module-get'
* The initial ["Go & Versioning"](https://research.swtch.com/vgo) series of blog posts on `vgo` by Russ Cox (first posted February 20, 2018)
* Official [golang.org blog post introducing the proposal](https://blog.golang.org/versioning-proposal) (March 26, 2018)
  * This provides a more succinct overview of the proposal than the full `vgo` blog series, along with some of the history and process behind the proposal
* Official [Versioned Go Modules Proposal](https://golang.org/design/24301-versioned-go) (last updated March 20, 2018)
* [FAQ](https://github.com/golang/go/issues/24301#issuecomment-371228664) from the official proposal discussion, including answers to common question such as:
  * "Won't minimal version selection keep developers from getting important updates?"
  * "Why are major versions v0, v1 omitted from import paths?"

### Introductory Material

* Introductory blog post ["Taking Go Modules for a Spin"](https://dave.cheney.net/2018/07/14/taking-go-modules-for-a-spin) by Dave Cheney (July 14, 2018)
* Introductory blog post on [how to build go from tip and start using go modules](https://carolynvanslyck.com/blog/2018/07/building-go-from-source/) by Carolyn Van Slyck (July 16, 2018)
* Introductory [Go Meetup slides on modules](https://docs.google.com/presentation/d/1ansfXN8a_aVL-QuvQNY7xywnS78HE8aG7fPiITNQWMM/edit#slide=id.g3d87f3177d_0_0) by Chris Hines (July 16, 2018)
* Introductory video ["The Principles of Versions in Go"](https://www.youtube.com/watch?v=F8nrpe0XWRg&list=PLq2Nv-Sh8EbbIjQgDzapOFeVfv5bGOoPE&index=3&t=0s) from GopherCon Singapore by Russ Cox (May 2, 2018)
  * Succinctly covers the philosophy behind the design of versioned Go modules, including the three core principles of "Compatibility", "Repeatability", and "Cooperation"

### Additional Material

* Blog post ["Using Go modules with Travis CI"](https://dave.cheney.net/2018/07/16/using-go-modules-with-travis-ci) by Dave Cheney (July 16, 2018)
* Blog post ["The vgo proposal is accepted. Now what?"](https://research.swtch.com/vgo-accepted) by Russ Cox (May 29, 2018)
  * Includes summary of what it means that versioned modules are currently an experimental opt-in feature

### Issues

* [Currently open issues](https://golang.org/issues?q=is%3Aopen+is%3Aissue+label:modules)
* Submit a [new issue](https://github.com/golang/go/issues/new?title=cmd%2Fgo%3A%20%3Cfill%20this%20in%3E) using 'cmd/go:' as the prefix