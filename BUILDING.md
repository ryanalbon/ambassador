Building Ambassador
===================

If you just want to **use** Ambassador, check out https://www.getambassador.io/! You don't need to build anything, and in fact you shouldn't.

If you want to make a doc change, see the `Making Documentation-Only Changes` section below.

Quick Build
-----------

Building Ambassador is straightforward:

```
git clone https://github.com/datawire/ambassador
cd ambassador
git checkout master
git checkout -b username/feature/my-branch-here
make DOCKER_REGISTRY=<YOUR DOCKER REGISTRY> docker-push
```

This will build a Docker image of Ambassador containing your code changes and push it to your given registry. Once it's pushed, you can deploy the new image onto your cluster.

**It is important to use `make` rather than trying to just do a `docker build`.** Actually assembling a Docker image for Ambassador involves quite a few steps before the image can be built.

Running Tests
-------------

Ambassador is infrastructure software, so robust testing is a must. To build
Ambassador and run all of the regression tests, run `make` with the following 
arguments:

```
make DOCKER_REGISTRY=<YOUR DOCKER REGISTRY> KUBECONFIG=<YOUR KUBE CONFIG>
```

The regression tests need a Kubernetes cluster to run, **and they assume that they have 
total control of that cluster**. Do **not** run the Ambassador test suite against a
cluster with anything important in it!

By default, Datawire developers use our on-demand Kubernetes cluster system, Kubernaut.
Outside Datawire, you'll need to use your own Kubernetes cluster, so point `make` to
the appropriate `KUBECONFIG`. Make sure your Kubernetes cluster can access images on
your Docker registry.

When the tests are run, your local machine will configure the cluster as appropriate
and then use a pod inside the cluster to send a large number of requests to the remote
Ambassador instance which will then be validated as part of the test suite. When run
on the `master` branch from the `datawire/ambassador` GitHub repo, the regression test
should _always_ pass.

Ambassador relies on several different backend test services to run the tests. See
the "Updating the Test Services" section below if you need to update a test service.

### Structure and Branches

The current shipping release of Ambassador lives on the `master` branch. It is tagged with its version (e.g. `v0.78.0`).

Changes on `master` after the last tag have not been released yet, but will be included in the next release of Ambassador.

The documentation in the `docs` directory is actually a Git subtree from the `ambassador-docs` repo. See the `Making Documentation-Only Changes` section below if you just want to change docs.

### Making Code Changes

1. **All development must be on branches cut from `master`**.
   - We recommend that your branches start with your username. 
      - At Datawire we typically use `git-flow`-style naming, e.g. `flynn/dev/telepathic-ui`
   - Please do not use a branch name starting with `release`.

2. If your development takes any significant time, **merge master back into your branch regularly**.
   - Think "every morning" and "right before submitting a pull request."
   - If you're using a branch name that starts with your username, `git rebase` is also OK and no one from Datawire will scream at you for force-pushing.
   - Please do **not** rebase any branch that does **not** start with your username.

3. **Code changes must include relevant documentation updates.** 
   - Make changes in the `docs` directory as necessary, and commit them to your
     branch so that they can be incorporated when the feature is merged into `master`.

4. **Code changes must include passing tests.**
   - See `ambassador/tests/README.md` for more here.
   - Your tests **must** actually test the change you're making.
   - Your tests **must** pass in order for your change to be accepted.

5. When you have things working and tested, **submit a pull request back to `master`**.
   - Make sure your branch is up-to-date with `master` right before submitting the PR!
   - The PR will trigger CI to perform a build and run tests.
   - CI tests **must** be passing for the PR to be merged.

6. When all is well, maintainers will merge the PR into `master`, accepting your
   change for the next Ambassador release. Thanks!

### Making Documentation-Only Changes

If you want to make a change that **only** affects documentation, and is not 
tied to a future feature, you'll need to make your change directly in the
`datawire/ambassador-docs` repository. Clone that repository and check out
its `README.md`. 

(It is technically possible to make these changes from the `ambassador` repo.
Please don't, unless you're fixing docs for an upcoming feature that hasn't yet
shipped.)

Developer Quickstart/Inner Loop
-------------------------------

### Dependencies:

Make sure you have Python 3 with `pip` and `virtualenv` installed on your developer workstation.

```
$ python --version
Python 3.7.4

$ pip --version
pip 19.1.1 from /usr/local/lib/python3.7/site-packages/pip (python 3.7)

$ pip install --user pipenv
$ pip install --user virtualenv

$ virtualenv --version
16.7.5
```

Go 1.13 is also required.

```
$ go version
go version go1.13.1 darwin/amd64
```

### Quickstart:

1. `git clone https://github.com/datawire/ambassador`
2. `cd ambassador`
3. `export DOCKER_REGISTRY=$registry`
4. `git checkout -b username/feature/my-branch-here`
5. `make setup-develop`
6. `make test`, or
   - `make test TEST_NAME=$testname` where `$testname` is the name of a single test you want to run. `$testname` cannot currently contain spaces, sorry.
7. Edit code.
8. Go back to 6 as necessary.
9. Commit _to your branch_.
10. Open a pull request against `master`.
11. Maintainers will review and merge the PR. W00t!

### Details:

You can just follow the script above, but it may well be helpful to know some
of the details around the build environment and workflow:

#### `export DOCKER_REGISTRY=$registry`

**This is mandatory.** It sets the registry to which you'll push Docker images, e.g.:

- "dwflynn" will push to Dockerhub with user `dwflynn`
- "gcr.io/flynn" will push to GCR with user `flynn`

If you're using minikube and don't want to push at all, set `DOCKER_REGISTRY` to "-".
(If you're not using minikube, this is probably a _terrible_ idea.)

You can separately tweak the registry from which images will be _pulled_ using
`AMBASSADOR_REGISTRY`. See the files in `templates` for more here.

#### `make setup-develop`

The `make setup-develop` at step 3 will take some time the first time
you run it, because it does a lot of work:

- set up a Python virtualenv with everything Ambassador needs
- build the Go bits for Ambassador
- make sure the Envoy generated code is OK
- etc.

The simplest way to actually use this stuff once it's set up is to 
run the tests with `make test`, or to e.g. run the Ambassador CLI
(see the "ambassador dump" section below).

#### Docker Image Names

Running `make docker-images` or `make docker-push` in development
builds a Docker image that includes the short Git hash of your current
commit in its name (because if the name doesn't change when you commit
new code, it can be very hard to get some Kubernetes environments to
actually pull the new image!).

**Whenever you commit new code, you must rerun `make docker-push` 
before doing things that try to use the image.** Yes, this is annoying,
but the other ways we tried were all worse. Sigh.

#### Dev Shells

If you prefer, you can run `make shell` instead of (or after) `make
setup-develop`. This will do all the `setup-develop` work, then start
a shell with the `PATH` and other Ambassador environment variables
set for you, allowing you to run `pytest` and the `ambassador` CLI 
without using `make` targets.

**If you use a dev shell, you _must_ exit the shell and rerun `make shell`
every time you run `make docker-push`.**

This is because the dev shell caches the current image name in the 
environment. Some of us (e.g. Flynn) have mostly stopped using dev shells
for this reason: it's just easier to use `make` targets than it is to
keep the dev shell up-to-date. (If anyone wants to make this better, it
would be welcome!)

If you do use the dev shell, you can start as many of them as you
want. Exiting and restarting the dev shell should be quick as long as
you don't `make clean` or `make clobber` in between. 

#### `ambassador dump`

After running `make setup-develop` or `make shell`, you'll be able to use the
`ambassador` CLI. The most useful thing it can do is `ambassador dump`, which
can export the most import data structures that Ambassador works with as JSON.
It works from an input which can be either a single file or a directory full of files in the following formats:

- raw Ambassador resources like you'll find in the `demo/config` directory; or
- an annotated Kubernetes resources like you'll find in `/tmp/k8s-AmbassadorTest.yaml` after running `make test`; or
- a `watt` snapshot like you'll find in the `$AMBASSADOR_CONFIG_BASE_DIR/snapshots/snapshot.yaml` (which is a JSON file, I know, it's misnamed).

(If you have a choice, use a `watt` snapshot: it's the input source Ambassador uses when running for real, so it's the most complete type of input.)

Given an input source, running

```
venv/bin/ambassador dump --ir --v2 [$input_flags] $input > test.json
```

will dump the Ambassador IR and v2 Envoy configuration into `test.json`. Here
`$input_flags` will be

- nothing for raw Ambassador resources;
- `--k8s` for Kubernetes resources; or
- `--watt` for a `watt` snapshot.

You can get more information with 

```
venv/bin/ambassador dump --help
```

(Note that if you're in a dev shell, you can just run `ambassador` instead of
`venv/bin/ambassador`.)

#### The Test Suite and Your Cluster

**The test suite assumes that it has complete control of your cluster.**

Let me repeat that: **The test suite assumes that it has complete control
of your cluster.**

When the test suite runs, it applies _lots_ of things to your cluster,
including RBAC setup, some namespaces, lots of deployments and pods, the
works. You are **strongly** advised not to run the test suite against a
cluster with anything important in it: we don't do this at Datawire so we
never test this case at all.

If you use `make test`, you'll get a warning about taking over the cluster
that you have to acknowledge. We encourage using `make test`.

The first time you run the tests, applying everything takes awhile. Be
patient: the test suite tries hard not to do work it doesn't need to, so
it will be much faster the second time.

If you're using Kubernaut, you can release the Kubernaut cluster with
`make clean-test`. This will happen automatically when you run `make clean`.
The next `make test` run will claim a new Kubernaut cluster.

Type Hinting
------------

Ambassador uses Python 3 type hinting and the `mypy` static type checker to
help find bugs before runtime. If you haven't worked with hinting before, a
good place to start is
[the `mypy` cheat sheet](https://mypy.readthedocs.io/en/latest/cheat_sheet_py3.html).

New code must be hinted, and the build process will verify that the type
check passes when you `make docker-images`. Fair warning: this means that
PRs will not pass CI if the type checker fails.

We strongly recommend using an editor that can do realtime type checking
(at Datawire we tend to use PyCharm and VSCode a lot, but many many editors
can do this now) and also running the type checker by hand before submitting
anything: 

- make sure you've done `make setup-develop` or `make shell` to get
everything set up, then
- `make mypy` will start the [mypy daemon](https://mypy.readthedocs.io/en/latest/mypy_daemon.html) and check all the Ambassador code.

Since `make mypy` uses the daemon for caching, it should be very fast after
the first run. Ambassador code should produce _no_ warnings and _no_ errors.

If you're concerned that the cache is somehow wrong (or if you just want the
daemon to not be there any more), `make mypy-server-stop` will stop the daemon
and clear the cache.

**Note well** that at present, `make mypy` will ignore missing imports. We're
still sorting out how to best wrangle the various third-party libraries we use,
so this seems to make sense for right now -- suggestions welcome on this front!

Tests
-----

CI runs Ambassador's test suite on every build. **You will be asked to add tests when you add features, and you should never ever commit code with failing unit tests.**

For more information on the test suite, see [its README](ambassador/tests/README.md).

#### Running tests locally (in minikube)
Tests consume quite a lot of resources, so make sure you allocate them accordingly to your minikube instance. (Honestly, if you're on a Mac, running the full test suite in minikube is likely to be a lost cause. Running a smaller subset can work great though.)

1. Start minikube
2. To build images directly into your minikube instance, set `DOCKER_REGISTRY=-` and point your docker client to docker daemon running inside minikube by running the command `eval $(minikube docker-env)`
3. Point `KUBECONFIG` to minikube, generally using `export KUBECONFIG=~/.kube/config`
4. Since everything is being run locally, you don't need a Kubernetes cluser using kubernaut. Set `USE_KUBERNAUT=false`.

That's it! Now simply run `make clean docker-push test` for the first time. In the following iterations, you can drop `clean` or `docker-push` depending on the nature of test run.

Updating Ambassador's Envoy
---------------------------

Ambassador currently relies on a custom Envoy build. This build lives in
`https://github.com/datawire/envoy`, which is a fork of
`https://github.com/envoyproxy/envoy`, and it'll need to be updated at
least as often as Envoy releases happen. 

Ambassador's current release engineering works by using `git clone`ing the
`datawire/envoy` tree into the `envoy-src` subdirectory of Ambassador's
source tree. **Pay attention** to your working directory as you work through
the update procedure! There are two separate repos involved, even though one
appears as a subdirectory of the other.

 0. In Ambassador's `Makefile`:

    - `$ENVOY_REPO` is the URL of the Envoy source repo. Typically this is
      the `datawire/envoy` repo, shown above; and

    - `$ENVOY_COMMIT` is the commit within `$ENVOY_REPO` from which
      Ambassador will build Envoy. Typically this is the tip of the
      `rebase/master` branch in the `datawire/envoy` repo.

    You **must** edit `$ENVOY_COMMIT` as part of the updating procedure. 
    Think hard before changing `$ENVOY_REPO`!

 1. Within your `datawire/ambassador` clone, use `make envoy-src` to clone
    `$ENVOY_REPO` to `./envoy-src`.  It will check out `$ENVOY_COMMMIT`
    (instead of `master`):

    ```
    $ make envoy-src
    git init envoy
    …
    HEAD is now at a484da25f updated legacy RLS name
    ```

 2. You'll need to manipulate branches in `$ENVOY_REPO`, so

    ```
    $ cd envoy-src
    ```

    to be in the correct place.

 2. You'll need to have the latest commits from the `envoyproxy/envoy` repo
    available so that you can pull the latest changes:

    ```
    $ git remote add upstream git://github.com/envoyproxy/envoy.git
    $ git fetch upstream master
    ```

 3. Since `$ENVOY_COMMIT` typically points at the tip of the `rebase/master`
    branch, that's usually a good branch to work on:

    ```
    $ git checkout rebase/master
    Branch 'rebase/master' set up to track remote branch 'rebase/master' from 'origin'.
    Switched to a new branch 'rebase/master'
    ```

 4. Once on the correct branch, `git rebase` the commit you want for the new
    `$ENVOY_COMMIT`:

    ```
    $ git rebase $NEW_ENVOY_COMMIT
    …
    ```

 5. Deal with any merge conflicts. Sigh.

 6. Switch back to your enclosing Ambassador repo:

    ```
    $ cd ..
    ```

 7. Try compiling your new Envoy from the sources you've rebased locally:

    ```
    $ ENVOY_COMMIT=- make envoy-bin/envoy-static
    ```

    If there are problems with the build, you can run 

    ```
    $ ENVOY_COMMIT=- make envoy-shell
    ```

    to get a shell in to the Docker image where Envoy is compiled.  Any changes
    you make in the Docker image WILL be copied back to the host, potentially
    OVERWRITING changes you made  in the host's `./envoy-src/` directory.

 8. Once you have a clean compile, run

    ```
    $ ENVOY_COMMIT=- make check-envoy
    ```

    to make sure that your new Envoy commit passes Envoy's own test-suite.

 9. Finally, push your new Envoy commit _from the `envoy-src` directory_:

    ```
    $ cd envoy-src
    $ git tag "datawire-$(git describe --tags)"
    $ git push --tags
    …
    $ git push -f origin rebase/master
    …
    ```

 10. Edit `ENVOY_COMMIT ?=` in the Makefile to point to your new Envoy commit.  Follow the instructions in the Makefile when doing so:

    a. Then run `make docker-update-base` to compile Envoy, and build+push new docker base images incorporating that Envoy binary.  This will also update the `go/apis/envoy/` directory if any of Envoy's protobuf definitions have changed; make sure to commit those changes when you commit the change to `ENVOY_COMMIT`.

Updating the Test Services
--------------------------

When running the tests, Ambassador relies on several different backend test services.
These all live in the `test-services` directory, and can be rebuild with 

```
make test-services
```

which will build the test services, then push them to `$(DOCKER_REGISTRY)`.

**Using locally-built tests:**

To use a single test service that you've built locally, set the environment variable
`TEST_SERVICE_$svc` to point to the image you've just built and pushed, e.g.

```
export TEST_SERVICE_AUTH=dwflynn/test_services:test-auth-v0.80.0-28-g3ed96316
```

before running `make test`. The different `$svc` possibilities are `auth`, `auth-tls`,
`ratelimit`, `shadow`, and `stats`.

To use your copy of _all_ the test services:

```
export TEST_SERVICE_REGISTRY=$registry
export TEST_SERVICE_VERSION=$version
```

before running `make test`, e.g.

```
export TEST_SERVICE_REGISTRY=dwflynn/test_services
export TEST_SERVICE_VERSION=v0.80.0-28-g3ed96316
```

to match the example above.

**Updating the official tests:**

The official versions of the test services live in the `quay.io/datawire/test_services` registry.
To update those (which will require you to work at Datawire!), update `TEST_SERVICE_VERSION`
in the Makefile and then `make test-services-release`.

Version Numbering
-----------------

Version numbers will be determined by Git tags for actual releases. You are free to pick whatever version numbers you like when doing test builds.
