---
title: "Contributing to a Go Project"
date: 2018-09-08T13:43:59-04:00
draft: true
author: juice
---

I've been using Terraform to spin up infrastructure.

For those of you who don't know, Terraform is a great little tool designed to
provision infrastructure built by HashiCorp.

Terraform stores the state of the world it knows about in a JSON file. That JSON
file can be stored locally, but that makes it next to impossible to collaborate
with other people. To solve this problem they made it possible to store state
in different places, such as S3, GCS or Artifactory.

I've been using Artifactory to store state remotely. It's nice because you get
to specify environment variables like `ARTIFACTORY_URL` to configure it, so it
can pull the configuration from the environment.

There's a catch: it doesn't look to the environment for the repo or the subpath.

There's ways around it, but it would be really nice if it just got everything
from the same place.

Looks like this is a great chance to open a pull request.

## Getting Set Up

The first thing I did was clone the repo.

Then, I forked it. Working with Go projects is a bit different because of the
way import paths work. Normally I would fork first and then clone my own fork,
but to keep everything sane I just added my own fork as a remote.

Thankfully Terraform has _excellent_ documentation for people getting started
with contributing to the project. I just followed the docs to get started.

> You'll need to run `make tools` to install some required tools, then `make`.
This will compile the code and then run the tests. If this exits with exit
status 0, then everything is working!

So I ran make tools and that worked. Then I ran make:
```
~/go/src/github.com/hashicorp/terraform => make tools
go get -u github.com/kardianos/govendor
go get -u golang.org/x/tools/cmd/stringer
go get -u golang.org/x/tools/cmd/cover
~/go/src/github.com/hashicorp/terraform => make
==> Checking that code complies with gofmt requirements...
gofmt needs running on the following files:
./backend/remote-state/s3/backend_test.go
./builtin/provisioners/chef/linux_provisioner_test.go
./command/init.go
./command/meta_backend_test.go
./helper/schema/resource_timeout_test.go
./helper/schema/schema_test.go
./plugin/discovery/get_test.go
You can use the command: `make fmt` to reformat code.
make: *** [fmtcheck] Error 1
```

Looks like `master` was left in a bad state somehow. I decided to see what that
`make fmt` command would do.
```
~/go/src/github.com/hashicorp/terraform => make fmt
gofmt -w $(find . -name '*.go' | grep -v vendor)
~/go/src/github.com/hashicorp/terraform => git diff

A LOT OF OUTPUT

~/go/src/github.com/hashicorp/terraform => git status
On branch artifactory-vars
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

      modified:   backend/remote-state/s3/backend_test.go
      modified:   builtin/provisioners/chef/linux_provisioner_test.go
      modified:   command/init.go
      modified:   command/meta_backend_test.go
      modified:   helper/schema/resource_timeout_test.go
      modified:   helper/schema/schema_test.go
      modified:   plugin/discovery/get_test.go

no changes added to commit (use "git add" and/or "git commit -a")
```

Looks like it changed a bunch of stuff. I ran `make` again
```
~/go/src/github.com/hashicorp/terraform => make

...

FAIL

...
```

and it failed on some tests.

At this point I wanted to see exactly what tests were failing, and once again
the Terraform docs didn't disappoint:

> If you're developing a specific package, you can run tests for just that
package by specifying the `TEST` variable.

So I ran the tests on the `command` package using `make test TEST=./command/...`
and they _passed_. Weird.

Just to make sure I was sane, I tried running `make` again, and it worked too.

Shit happens. ¯\_(ツ)_/¯

At this point I wanted to try and build my own version of it. The docs are 3-3
so far:

> To compile a development version of Terraform and the built-in plugins, run
`make dev`.

```
~/go/src/github.com/hashicorp/terraform => make dev
==> Checking that code complies with gofmt requirements...
go generate ./...
2018/09/08 13:59:49 Generated command/internal_plugin_list.go
==> Removing old directory...
==> Building...
Number of parallel builds: 7

-->    darwin/amd64: github.com/hashicorp/terraform

==> Results:
total 232600
-rwxr-xr-x  1 hugot  USCORP\Domain Users   114M Sep  8 14:00 terraform
```

Sure enough I had my own build in my `$GOPATH/bin`.

```
~/go/src/github.com/hashicorp/terraform => terraform version
Terraform v0.11.9-dev (d622ade94f422daee68e2c5aefda42bc2761f20f+CHANGES)
```

Being through with the setup, I figured it would be a good time to commit the
formatting changes before moving on. You can see them
[here](https://github.com/juicemia/terraform/commit/d622ade94f422daee68e2c5aefda42bc2761f20f).

## Making the Change

I saw that there were no tests for detecting configuration, so I decided to add
some first.

> Good devs copy, great devs paste. - Kelsey Hightower

I copied over the original test and modified it to test that the configuration
could be set by environment variables.

```
func TestArtifactoryFactoryEnvironmentConfig(t *testing.T) {
    // This test just instantiates the client. Shouldn't make any actual
    // requests nor incur any costs.

    config := make(map[string]string)

    _, err := artifactoryFactory(config)
    if err.Error() != "missing 'username' configuration or ARTIFACTORY_USERNAME environment variable" {
        t.Fatal("missing ARTIFACTORY_USERNAME should be error")
    }
    os.Setenv("ARTIFACTORY_USERNAME", "test")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'password' configuration or ARTIFACTORY_PASSWORD environment variable" {
        t.Fatal("missing ARTIFACTORY_PASSWORD should be error")
    }
    os.Setenv("ARTIFACTORY_PASSWORD", "testpass")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'url' configuration or ARTIFACTORY_URL environment variable" {
        t.Fatal("missing ARTIFACTORY_URL should be error")
    }
    os.Setenv("ARTIFACTORY_URL", "http://artifactory.local:8081/artifactory")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'repo' configuration or ARTIFACTORY_REPO environment variable" {
        t.Fatal("missing ARTIFACTORY_REPO should be error")
    }
    os.Setenv("ARTIFACTORY_REPO", "terraform-repo")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'subpath' configuration or ARTIFACTORY_SUBPATH environment variable" {
        t.Fatal("missing ARTIFACTORY_SUBPATH should be error")
    }
    os.Setenv("ARTIFACTORY_SUBPATH", "myproject")
    err = nil

    client, err := artifactoryFactory(config)
    if err != nil {
        t.Fatalf("Error for valid config: %v", err)
    }

    artifactoryClient := client.(*ArtifactoryClient)

    if artifactoryClient.nativeClient.Config.BaseURL != "http://artifactory.local:8081/artifactory" {
        t.Fatalf("Incorrect url was populated")
    }
    if artifactoryClient.nativeClient.Config.Username != "test" {
        t.Fatalf("Incorrect username was populated")
    }
    if artifactoryClient.nativeClient.Config.Password != "testpass" {
        t.Fatalf("Incorrect password was populated")
    }
    if artifactoryClient.repo != "terraform-repo" {
        t.Fatalf("Incorrect repo was populated")
    }
    if artifactoryClient.subpath != "myproject" {
        t.Fatalf("Incorrect subpath was populated")
    }
}
```

Sure enough, running the tests failed.

```
~/go/src/github.com/hashicorp/terraform => make test TEST=./state/remote
==> Checking that code complies with gofmt requirements...
go generate ./...
2018/09/08 14:11:15 Generated command/internal_plugin_list.go
go list ./state/remote | xargs -t -n4 go test  -timeout=60s -parallel=4
go test -timeout=60s -parallel=4 github.com/hashicorp/terraform/state/remote
--- FAIL: TestArtifactoryFactoryEnvironmentConfig (0.00s)
    artifactory_test.go:88: missing ARTIFACTORY_REPO should be error
2018/09/08 14:11:24 [DEBUG] New state was assigned lineage "f7027038-9195-b1d4-bc34-4a1dfa7d9de7"
2018/09/08 14:11:24 [TRACE] Preserving existing state lineage "f7027038-9195-b1d4-bc34-4a1dfa7d9de7"
2018/09/08 14:11:24 [DEBUG] New state was assigned lineage "b707f76e-e0c4-23e0-4255-b88eb1667fab"

A LOT OF TRACE LOGS
```

They only failed for `ARITFACTORY_REPO` because `ARTIFACTORY_USERNAME`,
`ARTIFACTORY_PASSWORD`, and `ARTIFACTORY_URL` were already designed to do this.

Making the code pass the tests was just a matter of copying and pasting what
was already there for `repo` and `subpath`.

```
    ... func artifactoryFactory

    url, ok := conf["url"]
    if !ok {
        url = os.Getenv("ARTIFACTORY_URL")
        if url == "" {
            return nil, fmt.Errorf(
                "missing 'url' configuration or ARTIFACTORY_URL environment variable")
        }
    }
    repo, ok := conf["repo"]
    if !ok {
        repo = os.Getenv("ARTIFACTORY_REPO")
        if repo == "" {
            return nil, fmt.Errorf(
                "missing 'repo' configuration or ARTIFACTORY_REPO environment variable")
        }
    }
    subpath, ok := conf["subpath"]
    if !ok {
        subpath = os.Getenv("ARTIFACTORY_SUBPATH")
        if subpath == "" {
            return nil, fmt.Errorf(
                "missing 'subpath' configuration or ARTIFACTORY_SUBPATH environment variable")
        }
    }

    ...
```

Running the tests again worked.

```
~/go/src/github.com/hashicorp/terraform => make test TEST=./state/remote
==> Checking that code complies with gofmt requirements...
go generate ./...
2018/09/08 22:51:30 Generated command/internal_plugin_list.go
go list ./state/remote | xargs -t -n4 go test  -timeout=60s -parallel=4
go test -timeout=60s -parallel=4 github.com/hashicorp/terraform/state/remote
ok    github.com/hashicorp/terraform/state/remote
```

Just to be sure, I ran `make` again to see if the whole process worked and it
did.

I'm going to be a bit handwavy with how I actually tested this feature because
it's a pain to demonstrate. Just take my word for it. I set all my environment
variables and ran `terraform init` and it worked.

Now that the feature was implemented and working, I decided to commit before
refactoring. The commit can be found
[here](https://github.com/juicemia/terraform/commit/2c8f8fdb0f0e00857d4d0ecc5a5f6c0b3441a9f4).

## Refactoring

Let's take another look at the tests:

```
    ...

    _, err := artifactoryFactory(config)
    if err.Error() != "missing 'username' configuration or ARTIFACTORY_USERNAME environment variable" {
        t.Fatal("missing ARTIFACTORY_USERNAME should be error")
    }
    os.Setenv("ARTIFACTORY_USERNAME", "test")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'password' configuration or ARTIFACTORY_PASSWORD environment variable" {
        t.Fatal("missing ARTIFACTORY_PASSWORD should be error")
    }
    os.Setenv("ARTIFACTORY_PASSWORD", "testpass")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'url' configuration or ARTIFACTORY_URL environment variable" {
        t.Fatal("missing ARTIFACTORY_URL should be error")
    }
    os.Setenv("ARTIFACTORY_URL", "http://artifactory.local:8081/artifactory")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'repo' configuration or ARTIFACTORY_REPO environment variable" {
        t.Fatal("missing ARTIFACTORY_REPO should be error")
    }
    os.Setenv("ARTIFACTORY_REPO", "terraform-repo")
    err = nil

    _, err = artifactoryFactory(config)
    if err.Error() != "missing 'subpath' configuration or ARTIFACTORY_SUBPATH environment variable" {
        t.Fatal("missing ARTIFACTORY_SUBPATH should be error")
    }
    os.Setenv("ARTIFACTORY_SUBPATH", "myproject")
    err = nil

    ...
```

These ~30 lines of code are basically the same thing repeated over and over
again. It worked, but I didn't like it.

With some help of the `strings` package, it was easy to just consolidate every
check into one function.

```
func testArtifactoryVar(t *testing.T, vr string) {
    envvr := fmt.Sprintf("ARTIFACTORY_%v", strings.ToUpper(vr))
     config := make(map[string]string)

     // This config only needs to be valid as far as actually having values.
    config["url"] = "foo"
    config["repo"] = "foo"
    config["subpath"] = "foo"
    config["username"] = "foo"
    config["password"] = "foo"
    delete(config, vr)

     errmsg := fmt.Sprintf(
        "missing '%v' configuration or %v environment variable",
        vr,
        envvr,
    )

     _, err := artifactoryFactory(config)
    if err.Error() != errmsg {
        t.Fatalf("missing %v should be error", envvr)
    }
}
```

Using this function, the tests look like this:

```
    ...

    testArtifactoryVar(t, "username")
    testArtifactoryVar(t, "password")
    testArtifactoryVar(t, "url")
    testArtifactoryVar(t, "repo")
    testArtifactoryVar(t, "subpath")

    ...
```

Cleaning this up also made me realize there was a bug. After the cleanup, the
code looked like this:

```
func TestArtifactoryFactoryEnvironmentConfig(t *testing.T) {
    // This test just instantiates the client. Shouldn't make any actual
    // requests nor incur any costs.

    config := make(map[string]string)

    testArtifactoryVar(t, "username")
    testArtifactoryVar(t, "password")
    testArtifactoryVar(t, "url")
    testArtifactoryVar(t, "repo")
    testArtifactoryVar(t, "subpath")

    os.Setenv("ARTIFACTORY_USERNAME", "test")
    os.Setenv("ARTIFACTORY_PASSWORD", "testpass")
    os.Setenv("ARTIFACTORY_URL", "http://artifactory.local:8081/artifactory")
    os.Setenv("ARTIFACTORY_REPO", "terraform-repo")
    os.Setenv("ARTIFACTORY_SUBPATH", "myproject")

    client, err := artifactoryFactory(config)
    if err != nil {
        t.Fatalf("Error for valid config: %v", err)
    }

    artifactoryClient := client.(*ArtifactoryClient)

    if artifactoryClient.nativeClient.Config.BaseURL != "http://artifactory.local:8081/artifactory" {
        t.Fatalf("Incorrect url was populated")
    }
    if artifactoryClient.nativeClient.Config.Username != "test" {
        t.Fatalf("Incorrect username was populated")
    }
    if artifactoryClient.nativeClient.Config.Password != "testpass" {
        t.Fatalf("Incorrect password was populated")
    }
    if artifactoryClient.repo != "terraform-repo" {
        t.Fatalf("Incorrect repo was populated")
    }
    if artifactoryClient.subpath != "myproject" {
        t.Fatalf("Incorrect subpath was populated")
    }
}
```

Can you spot it?

The test is setting environment variables. These environment variables will stay
set for the current process. This doesn't affect anything if the test is being
run in isolation, but if all tests are being run it can cause problems because
tests will be affected by the environment.

Fixing this bug was simple enough.

```
    ...

    os.Setenv("ARTIFACTORY_USERNAME", "test")
    os.Setenv("ARTIFACTORY_PASSWORD", "testpass")
    os.Setenv("ARTIFACTORY_URL", "http://artifactory.local:8081/artifactory")
    os.Setenv("ARTIFACTORY_REPO", "terraform-repo")
    os.Setenv("ARTIFACTORY_SUBPATH", "myproject")

    // Clean up so that information about the test isn't leaked to other tests
    // through the environment.
    defer func() {
        os.Unsetenv("ARTIFACTORY_USERNAME")
        os.Unsetenv("ARTIFACTORY_PASSWORD")
        os.Unsetenv("ARTIFACTORY_URL")
        os.Unsetenv("ARTIFACTORY_REPO")
        os.Unsetenv("ARTIFACTORY_SUBPATH")
    }()

    ...
```

To test my tests, I reran `make test TEST=./state/remote` once per environment
variable, making sure to comment out the functionality that was being tested to
trigger a failure. Refactoring does no good if it comes at the expense of making
your test into a false positive.

Sure enough, everything worked.

At this point I was thinking about the actual code, and I realized that there
was a lot of repetition in there too.

```
    ...

    userName, ok := conf["username"]
    if !ok {
        userName = os.Getenv("ARTIFACTORY_USERNAME")
        if userName == "" {
            return nil, fmt.Errorf(
                "missing 'username' configuration or ARTIFACTORY_USERNAME environment variable")
        }
    }
    password, ok := conf["password"]
    if !ok {
        password = os.Getenv("ARTIFACTORY_PASSWORD")
        if password == "" {
            return nil, fmt.Errorf(
                "missing 'password' configuration or ARTIFACTORY_PASSWORD environment variable")
        }
    }
    url, ok := conf["url"]
    if !ok {
        url = os.Getenv("ARTIFACTORY_URL")
        if url == "" {
            return nil, fmt.Errorf(
                "missing 'url' configuration or ARTIFACTORY_URL environment variable")
        }
    }
    repo, ok := conf["repo"]
    if !ok {
        repo = os.Getenv("ARTIFACTORY_REPO")
        if repo == "" {
            return nil, fmt.Errorf(
                "missing 'repo' configuration or ARTIFACTORY_REPO environment variable")
        }
    }
    subpath, ok := conf["subpath"]
    if !ok {
        subpath = os.Getenv("ARTIFACTORY_SUBPATH")
        if subpath == "" {
            return nil, fmt.Errorf(
                "missing 'subpath' configuration or ARTIFACTORY_SUBPATH environment variable")
        }
    }

    ...
```

This ~40 lines of code could easily be consolidated just like the tests were.

In the original code, it made some sense to keep everything laid out that way
becuase there were two cases to handle.

There was the case where an environment variable could be used, and then there
was the case where it couldn't.

Now that they were all being handled the same way, they could also be made into
one function.

```
func getArtifactoryVar(vr string, conf map[string]string) (string, error) {
    envvr := fmt.Sprintf("ARTIFACTORY_%v", strings.ToUpper(vr))

    val, ok := conf[vr]
    if !ok {
        val = os.Getenv(envvr)
        if val == "" {
            return val, fmt.Errorf(
                "missing '%v' configuration or %v environment variable",
                vr,
                envvr,
            )
        }
    }

    return val, nil
}
```

This made the code in `func artifactoryFactory` much cleaner.

```
    ...

    userName, err := getArtifactoryVar("username", conf)
    if err != nil {
        return nil, err
    }

    password, err := getArtifactoryVar("password", conf)
    if err != nil {
        return nil, err
    }

    url, err := getArtifactoryVar("url", conf)
    if err != nil {
        return nil, err
    }

    repo, err := getArtifactoryVar("repo", conf)
    if err != nil {
        return nil, err
    }

    subpath, err := getArtifactoryVar("subpath", conf)
    if err != nil {
        return nil, err
    }

    ...
```

Rerunning the tests again confirmed that the code was still correct.

```
~/go/src/github.com/hashicorp/terraform => make test TEST=./state/remote
==> Checking that code complies with gofmt requirements...
go generate ./...
2018/09/09 12:54:31 Generated command/internal_plugin_list.go
go list ./state/remote | xargs -t -n4 go test  -timeout=60s -parallel=4
go test -timeout=60s -parallel=4 github.com/hashicorp/terraform/state/remote
ok      github.com/hashicorp/terraform/state/remote
```

At this point I felt pretty confident, so I submitted the pull request. You can
see it for yourself
[here](https://github.com/hashicorp/terraform/pull/18824/commits).
