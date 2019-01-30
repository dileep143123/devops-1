# End-to-End Testing in Kubernetes


**Table of Contents**

- [End-to-End Testing in Kubernetes](#end-to-end-testing-in-kubernetes)
    - [Goals of e2e tests](#goals-of-e2e-tests)
    - [Debugging](#debugging)
    - [Ability to run in non-dedicated test clusters](#ability-to-run-in-non-dedicated-test-clusters)
    - [Speed](#speed)
    - [Resiliency](#resiliency)
            - [Resilience to relatively rare, temporary infrastructure glitches or delays](#resilience-to-relatively-rare-temporary-infrastructure-glitches-or-delays)
    - [Essential Tools](#essential-tools)
        - [E2e Framework:](#e2e-framework)
        - [E2e utils library:](#e2e-utils-library)
        - [Follow the examples of stable, well-written tests](#follow-the-examples-of-stable-well-written-tests)
        - [Ginkgo Test Framework:](#ginkgo-test-framework)
        - [Cleaning up](#cleaning-up)
    - [Advanced testing](#advanced-testing)
        - [Installing/updating kubetest](#installingupdating-kubetest)
        - [Extracting a specific version of kubernetes](#extracting-a-specific-version-of-kubernetes)
        - [Bringing up a cluster for testing](#bringing-up-a-cluster-for-testing)
        - [Reproducing failures in flaky tests](#reproducing-failures-in-flaky-tests)
        - [Federation e2e tests](#federation-e2e-tests)
            - [Configuring federation e2e tests](#configuring-federation-e2e-tests)
            - [Image Push Repository](#image-push-repository)
            - [Build](#build)
            - [Deploy federation control plane](#deploy-federation-control-plane)
            - [Run the Tests](#run-the-tests)
            - [Teardown](#teardown)
            - [Shortcuts for test developers](#shortcuts-for-test-developers)
        - [Debugging clusters](#debugging-clusters)
        - [Local clusters](#local-clusters)
            - [Testing against local clusters](#testing-against-local-clusters)
        - [Version-skewed and upgrade testing](#version-skewed-and-upgrade-testing)
            - [Test jobs naming convention](#test-jobs-naming-convention)
    - [Kinds of tests](#kinds-of-tests)
        - [Viper configuration and hierarchichal test parameters.](#viper-configuration-and-hierarchichal-test-parameters)
        - [Conformance tests](#conformance-tests)
    - [Continuous Integration](#continuous-integration)
        - [What is CI?](#what-is-ci)
        - [What runs in CI?](#what-runs-in-ci)
            - [Non-default tests](#non-default-tests)
        - [The PR-builder](#the-pr-builder)
        - [Adding a test to CI](#adding-a-test-to-ci)
        - [Moving a test out of CI](#moving-a-test-out-of-ci)
    - [Performance Evaluation](#performance-evaluation)
    - [One More Thing](#one-more-thing)

## Goals of e2e tests 

<details><summary>show</summary>
<p>


Beyond the obvious goal of providing end-to-end system test coverage,
there are a few less obvious goals that you should bear in mind when
designing, writing and debugging your end-to-end tests.  In
particular, "flaky" tests, which pass most of the time but fail
intermittently for difficult-to-diagnose reasons are extremely costly
in terms of blurring our regression signals and slowing down our
automated merge queue.  Up-front time and effort designing your test
to be reliable is very well spent.  Bear in mind that we have hundreds
of tests, each running in dozens of different environments, and if any
test in any test environment fails, we have to assume that we
potentially have some sort of regression. So if a significant number
of tests fail even only 1% of the time, basic statistics dictates that
we will almost never have a "green" regression indicator.  Stated
another way, writing a test that is only 99% reliable is just about
useless in the harsh reality of a CI environment.  In fact it's worse
than useless, because not only does it not provide a reliable
regression indicator, but it also costs a lot of subsequent debugging
time, and delayed merges.

</p>
</details>




## Debugging

<details><summary>show</summary>
<p>

If your test fails, it should provide as detailed as possible reasons
for the failure in it's output. "Timeout" is not a useful error
message. "Timed out after 60 seconds waiting for pod xxx to enter
running state, still in pending state" is much more useful to someone
trying to figure out why your test failed and what to do about it.
Specifically,
[assertion](https://onsi.github.io/gomega/#making-assertions) code
like the following generates rather useless errors:

```
Expect(err).NotTo(HaveOccurred())
```

Rather
[annotate](https://onsi.github.io/gomega/#annotating-assertions) your assertion with something like this:

```
Expect(err).NotTo(HaveOccurred(), "Failed to create %d foobars, only created %d", foobarsReqd, foobarsCreated)
```

On the other hand, overly verbose logging, particularly of non-error conditions, can make
it unnecessarily difficult to figure out whether a test failed and if
so why?  So don't log lots of irrelevant stuff either.

</p>
</details>

## Ability to run in non-dedicated test clusters 

<details><summary>show</summary>
<p>


To reduce end-to-end delay and improve resource utilization when
running e2e tests, we try, where possible, to run large numbers of
tests in parallel against the same test cluster.  This means that:

1. you should avoid making any assumption (implicit or explicit) that
your test is the only thing running against the cluster.  For example,
making the assumption that your test can run a pod on every node in a
cluster is not a safe assumption, as some other tests, running at the
same time as yours, might have saturated one or more nodes in the
cluster.  Similarly, running a pod in the system namespace, and
assuming that that will increase the count of pods in the system
namespace by one is not safe, as some other test might be creating or
deleting pods in the system namespace at the same time as your test.
If you do legitimately need to write a test like that, make sure to
label it ["\[Serial\]"](e2e-tests.md#kinds_of_tests) so that it's easy
to identify, and not run in parallel with any other tests.
1. You should avoid doing things to the cluster that make it difficult
for other tests to reliably do what they're trying to do, at the same
time.  For example, rebooting nodes, disconnecting network interfaces,
or upgrading cluster software as part of your test is likely to
violate the assumptions that other tests might have made about a
reasonably stable cluster environment.  If you need to write such
tests, please label them as
["\[Disruptive\]"](e2e-tests.md#kinds_of_tests) so that it's easy to
identify them, and not run them in parallel with other tests.
1. You should avoid making assumptions about the Kubernetes API that
are not part of the API specification, as your tests will break as
soon as these assumptions become invalid.  For example, relying on
specific Events, Event reasons or Event messages will make your tests
very brittle.

</p>
</details>



## Speed

<details><summary>show</summary>
<p>


We have hundreds of e2e tests, some of which we run in serial, one
after the other, in some cases.  If each test takes just a few minutes
to run, that very quickly adds up to many, many hours of total
execution time.  We try to keep such total execution time down to a
few tens of minutes at most.  Therefore, try (very hard) to keep the
execution time of your individual tests below 2 minutes, ideally
shorter than that.  Concretely, adding inappropriately long 'sleep'
statements or other gratuitous waits to tests is a killer.  If under
normal circumstances your pod enters the running state within 10
seconds, and 99.9% of the time within 30 seconds, it would be
gratuitous to wait 5 minutes for this to happen.  Rather just fail
after 30 seconds, with a clear error message as to why your test
failed ("e.g. Pod x failed to become ready after 30 seconds, it
usually takes 10 seconds").  If you do have a truly legitimate reason
for waiting longer than that, or writing a test which takes longer
than 2 minutes to run, comment very clearly in the code why this is
necessary, and label the test as
["\[Slow\]"](e2e-tests.md#kinds_of_tests), so that it's easy to
identify and avoid in test runs that are required to complete
timeously (for example those that are run against every code
submission before it is allowed to be merged).
Note that completing within, say, 2 minutes only when the test
passes is not generally good enough.  Your test should also fail in a
reasonable time.  We have seen tests that, for example, wait up to 10
minutes for each of several pods to become ready.  Under good
conditions these tests might pass within a few seconds, but if the
pods never become ready (e.g. due to a system regression) they take a
very long time to fail and typically cause the entire test run to time
out, so that no results are produced.  Again, this is a lot less
useful than a test that fails reliably within a minute or two when the
system is not working correctly.

</p>
</details>



## Resiliency

#### Resilience to relatively rare, temporary infrastructure glitches or delays ####

<details><summary>show</summary>
<p>

Remember that your test will be run many thousands of
times, at different times of day and night, probably on different
cloud providers, under different load conditions.  And often the
underlying state of these systems is stored in eventually consistent
data stores.  So, for example, if a resource creation request is
theoretically asynchronous, even if you observe it to be practically
synchronous most of the time, write your test to assume that it's
asynchronous (e.g. make the "create" call, and poll or watch the
resource until it's in the correct state before proceeding).
Similarly, don't assume that API endpoints are 100% available.
They're not.  Under high load conditions, API calls might temporarily
fail or time-out. In such cases it's appropriate to back off and retry
a few times before failing your test completely (in which case make
the error message very clear about what happened, e.g. "Retried
http://... 3 times - all failed with xxx".  Use the standard
retry mechanisms provided in the libraries detailed below.

</p>
</details>


## Essential Tools


<details><summary>show</summary>
<p>

### [E2e Framework](../../test/e2e/framework/framework.go):
   Familiarise yourself with this test framework and how to use it.
   Amongst others, it automatically creates uniquely named namespaces
   within which your tests can run to avoid name clashes, and reliably
   automates cleaning up the mess after your test has completed (it
   just deletes everything in the namespace).  This helps to ensure
   that tests do not leak resources. Note that deleting a namespace
   (and by implication everything in it) is currently an expensive
   operation.  So the fewer resources you create, the less cleaning up
   the framework needs to do, and the faster your test (and other
   tests running concurrently with yours) will complete. Your tests
   should always use this framework.  Trying other home-grown
   approaches to avoiding name clashes and resource leaks has proven
   to be a very bad idea.

   </p>
</details>


### [E2e utils library](../../test/e2e/framework/util.go):

<details><summary>show</summary>
<p>

   This handy library provides tons of reusable code for a host of
   commonly needed test functionality, including waiting for resources
   to enter specified states, safely and consistently retrying failed
   operations, usefully reporting errors, and much more.  Make sure
   that you're familiar with what's available there, and use it.
   Likewise, if you come across a generally useful mechanism that's
   not yet implemented there, add it so that others can benefit from
   your brilliance.  In particular pay attention to the variety of
   timeout and retry related constants at the top of that file. Always
   try to reuse these constants rather than try to dream up your own
   values.  Even if the values there are not precisely what you would
   like to use (timeout periods, retry counts etc), the benefit of
   having them be consistent and centrally configurable across our
   entire test suite typically outweighs your personal preferences.

   </p>
</details>


### Follow the examples of stable, well-written tests
Some of our existing end-to-end tests are better written and more reliable than
   others.  A few examples of well-written tests include:
   
   * [Replication Controllers](../../test/e2e/rc.go),
   * [Services](../../test/e2e/service.go),
   * [Reboot](../../test/e2e/reboot.go).

### [Ginkgo Test Framework](https://github.com/onsi/ginkgo): 

<details><summary>show</summary>
<p>

This is the test library and runner upon which our e2e tests are built.  Before
you write or refactor a test, read the docs and make sure that you
understand how it works.  In particular be aware that every test is
uniquely identified and described (e.g. in test reports) by the
concatenation of it's `Describe` clause and nested `It` clauses.
So for example `Describe("Pods",...).... It(""should be scheduled
with cpu and memory limits")` produces a sane test identifier and
descriptor `Pods should be scheduled with cpu and memory limits`,
which makes it clear what's being tested, and hence what's not
working if it fails.  Other good examples include:

```
   CAdvisor should be healthy on every node
```

and

```
   Daemon set should run and stop complex daemon
```

   On the contrary
(these are real examples), the following are less good test
descriptors:

```
   KubeProxy should test kube-proxy
```

and

```
Nodes [Disruptive] Network when a node becomes unreachable
[replication controller] recreates pods scheduled on the
unreachable node AND allows scheduling of pods on a node after
it rejoins the cluster
```

An improvement might be

```
Unreachable nodes are evacuated and then repopulated upon rejoining [Disruptive]

</p>
</details>





## Overview

<details><summary>show</summary>
<p>


End-to-end (e2e) tests for Kubernetes provide a mechanism to test end-to-end
behavior of the system, and is the last signal to ensure end user operations
match developer specifications. Although unit and integration tests provide a
good signal, in a distributed system like Kubernetes it is not uncommon that a
minor change may pass all unit and integration tests, but cause unforeseen
changes at the system level.

The primary objectives of the e2e tests are to ensure a consistent and reliable
behavior of the kubernetes code base, and to catch hard-to-test bugs before
users do, when unit and integration tests are insufficient.

The e2e tests in kubernetes are built atop of
[Ginkgo](http://onsi.github.io/ginkgo/) and
[Gomega](http://onsi.github.io/gomega/). There are a host of features that this
Behavior-Driven Development (BDD) testing framework provides, and it is
recommended that the developer read the documentation prior to diving into the
 tests.

The purpose of *this* document is to serve as a primer for developers who are
looking to execute or add tests using a local development environment.

Before writing new tests or making substantive changes to existing tests, you
should also read [Writing Good e2e Tests](writing-good-e2e-tests.md)

## Building Kubernetes and Running the Tests

There are a variety of ways to run e2e tests, but we aim to decrease the number
of ways to run e2e tests to a canonical way: `hack/e2e.go`.

You can run an end-to-end test which will bring up a master and nodes, perform
some tests, and then tear everything down. Make sure you have followed the
getting started steps for your chosen cloud platform (which might involve
changing the --provider flag value to something other than "gce").

You can quickly recompile the e2e testing framework via `go install ./test/e2e`.
This will not do anything besides allow you to verify that the go code compiles.
If you want to run your e2e testing framework without re-provisioning the e2e setup,
you can do so via `make WHAT=test/e2e/e2e.test`, and then re-running the ginkgo tests.

To build Kubernetes, up a cluster, run tests, and tear everything down, use:

```sh
go run hack/e2e.go -- --build --up --test --down
```

If you'd like to just perform one of these steps, here are some examples:

```sh
# Build binaries for testing
go run hack/e2e.go -- --build

# Create a fresh cluster.  Deletes a cluster first, if it exists
go run hack/e2e.go -- --up

# Run all tests
go run hack/e2e.go -- --test

# Run tests matching the regex "\[Feature:Performance\]" against a local cluster
# Specify "--provider=local" flag when running the tests locally
go run hack/e2e.go -- --test --test_args="--ginkgo.focus=\[Feature:Performance\]" --provider=local

# Conversely, exclude tests that match the regex "Pods.*env"
go run hack/e2e.go -- --test --test_args="--ginkgo.skip=Pods.*env"

# Run tests in parallel, skip any that must be run serially
GINKGO_PARALLEL=y go run hack/e2e.go -- --test --test_args="--ginkgo.skip=\[Serial\]"

# Run tests in parallel, skip any that must be run serially and keep the test namespace if test failed
GINKGO_PARALLEL=y go run hack/e2e.go -- --test --test_args="--ginkgo.skip=\[Serial\] --delete-namespace-on-failure=false"

# Flags can be combined, and their actions will take place in this order:
# --build, --up, --test, --down
#
# You can also specify an alternative provider, such as 'aws'
#
# e.g.:
go run hack/e2e.go -- --provider=aws --build --up --test --down

# -ctl can be used to quickly call kubectl against your e2e cluster. Useful for
# cleaning up after a failed test or viewing logs. 
# kubectl output is default on, you can use --verbose-commands=false to suppress output.
go run hack/e2e.go -- -ctl='get events'
go run hack/e2e.go -- -ctl='delete pod foobar'
```

The tests are built into a single binary which can be used to deploy a
Kubernetes system or run tests against an already-deployed Kubernetes system.
See `go run hack/e2e.go -- --help` (or the flag definitions in `hack/e2e.go`) for
more options, such as reusing an existing cluster.

### Cleaning up

During a run, pressing `control-C` should result in an orderly shutdown, but if
something goes wrong and you still have some VMs running you can force a cleanup
with this command:

```sh
go run hack/e2e.go -- --down
```

</p>
</details>



## Advanced testing

### Installing/updating kubetest

<details><summary>show</summary>
<p>


The logic in `e2e.go` moved out of the main kubernetes repo to test-infra.
The remaining code in `hack/e2e.go` installs `kubetest` and sends it flags.
It now lives in [kubernetes/test-infra/kubetest](https://git.k8s.io/test-infra/kubetest).
By default `hack/e2e.go` updates and installs `kubetest` once per day.
Control the updater behavior with the `--get` and `--old` flags:
The `--` flag separates updater and kubetest flags (kubetest flags on the right).

```sh
go run hack/e2e.go --get=true --old=1h -- # Update every hour
go run hack/e2e.go --get=false -- # Never attempt to install/update.
go install k8s.io/test-infra/kubetest  # Manually install
go get -u k8s.io/test-infra/kubetest  # Manually update installation

```
</p>
</details>



### Extracting a specific version of kubernetes

<details><summary>show</summary>
<p>


The `kubetest` binary can download and extract a specific version of kubernetes,
both the server, client and test binaries. The `--extract=E` flag enables this
functionality.

There are a variety of values to pass this flag:

```sh
# Official builds: <ci|release>/<latest|stable>[-N.N]
go run hack/e2e.go -- --extract=ci/latest --up  # Deploy the latest ci build.
go run hack/e2e.go -- --extract=ci/latest-1.5 --up  # Deploy the latest 1.5 CI build.
go run hack/e2e.go -- --extract=release/latest --up  # Deploy the latest RC.
go run hack/e2e.go -- --extract=release/stable-1.5 --up  # Deploy the 1.5 release.

# A specific version:
go run hack/e2e.go -- --extract=v1.5.1 --up  # Deploy 1.5.1
go run hack/e2e.go -- --extract=v1.5.2-beta.0  --up  # Deploy 1.5.2-beta.0
go run hack/e2e.go -- --extract=gs://foo/bar  --up  # --stage=gs://foo/bar

# Whatever GKE is using (gke, gke-staging, gke-test):
go run hack/e2e.go -- --extract=gke  --up  # Deploy whatever GKE prod uses

# Using a GCI version:
go run hack/e2e.go -- --extract=gci/gci-canary --up  # Deploy the version for next gci release
go run hack/e2e.go -- --extract=gci/gci-57  # Deploy the version bound to gci m57
go run hack/e2e.go -- --extract=gci/gci-57/ci/latest  # Deploy the latest CI build using gci m57 for the VM image

# Reuse whatever is already built
go run hack/e2e.go -- --up  # Most common. Note, no extract flag
go run hack/e2e.go -- --build --up  # Most common. Note, no extract flag
go run hack/e2e.go -- --build --stage=gs://foo/bar --extract=local --up  # Extract the staged version
```

</p>
</details>


### Bringing up a cluster for testing

<details><summary>show</summary>
<p>


If you want, you may bring up a cluster in some other manner and run tests
against it. To do so, or to do other non-standard test things, you can pass
arguments into Ginkgo using `--test_args` (e.g. see above). For the purposes of
brevity, we will look at a subset of the options, which are listed below:

```
--ginkgo.dryRun=false: If set, ginkgo will walk the test hierarchy without
actually running anything.

--ginkgo.failFast=false: If set, ginkgo will stop running a test suite after a
failure occurs.

--ginkgo.failOnPending=false: If set, ginkgo will mark the test suite as failed
if any specs are pending.

--ginkgo.focus="": If set, ginkgo will only run specs that match this regular
expression.

--ginkgo.noColor="n": If set to "y", ginkgo will not use color in the output

--ginkgo.skip="": If set, ginkgo will only run specs that do not match this
regular expression.

--ginkgo.trace=false: If set, default reporter prints out the full stack trace
when a failure occurs

--ginkgo.v=false: If set, default reporter print out all specs as they begin.

--host="": The host, or api-server, to connect to

--kubeconfig="": Path to kubeconfig containing embedded authinfo.

--provider="": The name of the Kubernetes provider (gce, gke, local, vagrant,
etc.)

--repo-root="../../": Root directory of kubernetes repository, for finding test
files.
```

Prior to running the tests, you may want to first create a simple auth file in
your home directory, e.g. `$HOME/.kube/config`, with the following:

```
{
  "User": "root",
  "Password": ""
}
```

As mentioned earlier there are a host of other options that are available, but
they are left to the developer.

**NOTE:** If you are running tests on a local cluster repeatedly, you may need
to periodically perform some manual cleanup:

  - `rm -rf /var/run/kubernetes`, clear kube generated credentials, sometimes
stale permissions can cause problems.

  - `sudo iptables -F`, clear ip tables rules left by the kube-proxy.


</p>
</details>


### Reproducing failures in flaky tests

<details><summary>show</summary>
<p>

You can run a test repeatedly until it fails. This is useful when debugging
flaky tests. In order to do so, you need to set the following environment
variable:
```sh
$ export GINKGO_UNTIL_IT_FAILS=true
```

After setting the environment variable, you can run the tests as before. The e2e
script adds `--untilItFails=true` to ginkgo args if the environment variable is
set. The flags asks ginkgo to run the test repeatedly until it fails.

</p>
</details>


### Federation e2e tests

<details><summary>show</summary>
<p>


By default, `e2e.go` provisions a single Kubernetes cluster, and any `Feature:Federation` ginkgo tests will be skipped.

Federation e2e testing involve bringing up multiple "underlying" Kubernetes clusters,
and deploying the federation control plane as a Kubernetes application on the underlying clusters.

The federation e2e tests are still managed via `e2e.go`, but require some extra configuration items.

</p>
</details>


#### Configuring federation e2e tests

<details><summary>show</summary>
<p>


The following environment variables will enable federation e2e building, provisioning and testing.

```sh
$ export FEDERATION=true
$ export E2E_ZONES="us-central1-a us-central1-b us-central1-f"
```

A Kubernetes cluster will be provisioned in each zone listed in `E2E_ZONES`. A zone can only appear once in the `E2E_ZONES` list.

</p>
</details>



#### Image Push Repository

<details><summary>show</summary>
<p>


Next, specify the docker repository where your ci images will be pushed.

* **If `--provider=gce` or `--provider=gke`**:

  If you use the same GCP project where you to run the e2e tests as the container image repository,
  FEDERATION_PUSH_REPO_BASE environment variable will be defaulted to "gcr.io/${DEFAULT_GCP_PROJECT_NAME}".
  You can skip ahead to the **Build** section.

	You can simply set your push repo base based on your project name, and the necessary repositories will be
  auto-created when you first push your container images.

	```sh
	$ export FEDERATION_PUSH_REPO_BASE="gcr.io/${GCE_PROJECT_NAME}"
	```

	Skip ahead to the **Build** section.

* **For all other providers**:

	You'll be responsible for creating and managing access to the repositories manually.

	```sh
	$ export FEDERATION_PUSH_REPO_BASE="quay.io/colin_hom"
	```

	Given this example, the `federation-apiserver` container image will be pushed to the repository
	`quay.io/colin_hom/federation-apiserver`.

	The docker client on the machine running `e2e.go` must have push access for the following pre-existing repositories:

	* `${FEDERATION_PUSH_REPO_BASE}/federation-apiserver`
	* `${FEDERATION_PUSH_REPO_BASE}/federation-controller-manager`

	These repositories must allow public read access, as the e2e node docker daemons will not have any credentials. If you're using
	GCE/GKE as your provider, the repositories will have read-access by default.

</p>
</details>


#### Build

<details><summary>show</summary>
<p>

* Compile the binaries and build container images:

  ```sh
  $ KUBE_RELEASE_RUN_TESTS=n KUBE_FASTBUILD=true go run hack/e2e.go -- -build
  ```

* Push the federation container images

  ```sh
  $ federation/develop/push-federation-images.sh
  ```

</p>
</details>


#### Deploy federation control plane

<details><summary>show</summary>
<p>

The following command will create the underlying Kubernetes clusters in each of `E2E_ZONES`, and then provision the
federation control plane in the cluster occupying the last zone in the `E2E_ZONES` list.

```sh
$ go run hack/e2e.go -- --up
```

</p>
</details>


#### Run the Tests

<details><summary>show</summary>
<p>

This will run only the `Feature:Federation` e2e tests. You can omit the `ginkgo.focus` argument to run the entire e2e suite.

```sh
$ go run hack/e2e.go -- --test --test_args="--ginkgo.focus=\[Feature:Federation\]"
```

<details><summary>show</summary>
<p>

#### Teardown

<details><summary>show</summary>
<p>


```sh
$ go run hack/e2e.go -- --down
```

</p>
</details>


#### Shortcuts for test developers

<details><summary>show</summary>
<p>

* To speed up `--up`, provision a single-node kubernetes cluster in a single e2e zone:

  `NUM_NODES=1 E2E_ZONES="us-central1-f"`

  Keep in mind that some tests may require multiple underlying clusters and/or minimum compute resource availability.

* If you're hacking around with the federation control plane deployment itself,
  you can quickly re-deploy the federation control plane Kubernetes manifests without tearing any resources down.
  To re-deploy the federation control plane after running `--up` for the first time:

  ```sh
  $ federation/cluster/federation-up.sh
  ```

</p>
</details>


### Debugging clusters

<details><summary>show</summary>
<p>

If a cluster fails to initialize, or you'd like to better understand cluster
state to debug a failed e2e test, you can use the `cluster/log-dump.sh` script
to gather logs.

This script requires that the cluster provider supports ssh. Assuming it does,
running:

```sh
$ federation/cluster/log-dump.sh <directory>
```

will ssh to the master and all nodes and download a variety of useful logs to
the provided directory (which should already exist).

The Google-run Jenkins builds automatically collected these logs for every
build, saving them in the `artifacts` directory uploaded to GCS.

</p>
</details>


### Local clusters

<details><summary>show</summary>
<p>

It can be much faster to iterate on a local cluster instead of a cloud-based
one. To start a local cluster, you can run:

```sh
# The PATH construction is needed because PATH is one of the special-cased
# environment variables not passed by sudo -E
sudo PATH=$PATH hack/local-up-cluster.sh
```

This will start a single-node Kubernetes cluster than runs pods using the local
docker daemon. Press Control-C to stop the cluster.

You can generate a valid kubeconfig file by following instructions printed at the
end of aforementioned script.

<details><summary>show</summary>
<p>

#### Testing against local clusters

<details><summary>show</summary>
<p>

In order to run an E2E test against a locally running cluster, first make sure
to have a local build of the tests:

```sh
go run hack/e2e.go -- --build
```

Then point the tests at a custom host directly:

```sh
export KUBECONFIG=/path/to/kubeconfig
go run hack/e2e.go -- --provider=local --test
```

To control the tests that are run:

```sh
go run hack/e2e.go -- --provider=local --test --test_args="--ginkgo.focus=Secrets"
```

You will also likely need to specify `minStartupPods` to match the number of
nodes in your cluster. If you're testing against a cluster set up by
`local-up-cluster.sh`, you will need to do the following:

```sh
go run hack/e2e.go -- --provider=local --test --test_args="--minStartupPods=1 --ginkgo.focus=Secrets"
```

<details><summary>show</summary>
<p>

### Version-skewed and upgrade testing

<details><summary>show</summary>
<p>

We run version-skewed tests to check that newer versions of Kubernetes work
similarly enough to older versions.  The general strategy is to cover the following cases:

1. One version of `kubectl` with another version of the cluster and tests (e.g.
   that v1.2 and v1.4 `kubectl` doesn't break v1.3 tests running against a v1.3
   cluster).
2. A newer version of the Kubernetes master with older nodes and tests (e.g.
   that upgrading a master to v1.3 with nodes at v1.2 still passes v1.2 tests).
3. A newer version of the whole cluster with older tests (e.g. that a cluster
   upgraded---master and nodes---to v1.3 still passes v1.2 tests).
4. That an upgraded cluster functions the same as a brand-new cluster of the
   same version (e.g. a cluster upgraded to v1.3 passes the same v1.3 tests as
   a newly-created v1.3 cluster).

[kubetest](https://git.k8s.io/test-infra/kubetest) is
the authoritative source on how to run version-skewed tests, but below is a
quick-and-dirty tutorial.

```sh
# Assume you have two copies of the Kubernetes repository checked out, at
# ./kubernetes and ./kubernetes_old

# If using GKE:
export CLUSTER_API_VERSION=${OLD_VERSION}

# Deploy a cluster at the old version; see above for more details
cd ./kubernetes_old
go run ./hack/e2e.go -- --up

# Upgrade the cluster to the new version
#
# If using GKE, add --upgrade-target=${NEW_VERSION}
#
# You can target Feature:MasterUpgrade or Feature:ClusterUpgrade
cd ../kubernetes
go run ./hack/e2e.go -- --provider=gke --test --check-version-skew=false --test_args="--ginkgo.focus=\[Feature:MasterUpgrade\]"

# Run old tests with new kubectl
cd ../kubernetes_old
go run ./hack/e2e.go -- --provider=gke --test --test_args="--kubectl-path=$(pwd)/../kubernetes/cluster/kubectl.sh"
```

If you are just testing version-skew, you may want to just deploy at one
version and then test at another version, instead of going through the whole
upgrade process:

```sh
# With the same setup as above

# Deploy a cluster at the new version
cd ./kubernetes
go run ./hack/e2e.go -- --up

# Run new tests with old kubectl
go run ./hack/e2e.go -- --test --test_args="--kubectl-path=$(pwd)/../kubernetes_old/cluster/kubectl.sh"

# Run old tests with new kubectl
cd ../kubernetes_old
go run ./hack/e2e.go -- --test --test_args="--kubectl-path=$(pwd)/../kubernetes/cluster/kubectl.sh"
```
</p>
</details>


#### Test jobs naming convention

<details><summary>show</summary>
<p>

**Version skew tests** are named as
`<cloud-provider>-<master&node-version>-<kubectl-version>-<image-name>-kubectl-skew`
e.g: `gke-1.5-1.6-cvm-kubectl-skew` means cloud provider is GKE;
master and nodes are built from `release-1.5` branch;
`kubectl` is built from `release-1.6` branch;
image name is cvm (container_vm).
The test suite is always the older one in version skew tests. e.g. from release-1.5 in this case.

**Upgrade tests**:

If a test job name ends with `upgrade-cluster`, it means we first upgrade
the cluster (i.e. master and nodes) and then run the old test suite with new kubectl.

If a test job name ends with `upgrade-cluster-new`, it means we first upgrade
the cluster (i.e. master and nodes) and then run the new test suite with new kubectl.

If a test job name ends with `upgrade-master`, it means we first upgrade
the master and keep the nodes in old version and then run the old test suite with new kubectl.

There are some examples in the table,
where `->` means upgrading; container_vm (cvm) and gci are image names.

| test name | test suite | master version (image) | node version (image) | kubectl
| --------- | :--------: | :----: | :---:| :---:
| gce-1.5-1.6-upgrade-cluster | 1.5 | 1.5->1.6 | 1.5->1.6 | 1.6
| gce-1.5-1.6-upgrade-cluster-new | 1.6 | 1.5->1.6 | 1.5->1.6 | 1.6
| gce-1.5-1.6-upgrade-master | 1.5 | 1.5->1.6 | 1.5 | 1.6
| gke-container_vm-1.5-container_vm-1.6-upgrade-cluster | 1.5 | 1.5->1.6 (cvm) | 1.5->1.6 (cvm) | 1.6
| gke-gci-1.5-container_vm-1.6-upgrade-cluster-new | 1.6 | 1.5->1.6 (gci) | 1.5->1.6 (cvm) | 1.6
| gke-gci-1.5-container_vm-1.6-upgrade-master | 1.5 | 1.5->1.6 (gci) | 1.5 (cvm) | 1.6

</p>
</details>


## Kinds of tests

<details><summary>show</summary>
<p>

We are working on implementing clearer partitioning of our e2e tests to make
running a known set of tests easier (#10548). Tests can be labeled with any of
the following labels, in order of increasing precedence (that is, each label
listed below supersedes the previous ones):

  - If a test has no labels, it is expected to run fast (under five minutes), be
able to be run in parallel, and be consistent.

  - `[Slow]`: If a test takes more than five minutes to run (by itself or in
parallel with many other tests), it is labeled `[Slow]`. This partition allows
us to run almost all of our tests quickly in parallel, without waiting for the
stragglers to finish.

  - `[Serial]`: If a test cannot be run in parallel with other tests (e.g. it
takes too many resources or restarts nodes), it is labeled `[Serial]`, and
should be run in serial as part of a separate suite.

  - `[Disruptive]`: If a test restarts components that might cause other tests
to fail or break the cluster completely, it is labeled `[Disruptive]`. Any
`[Disruptive]` test is also assumed to qualify for the `[Serial]` label, but
need not be labeled as both. These tests are not run against soak clusters to
avoid restarting components.

  - `[Flaky]`: If a test is found to be flaky and we have decided that it's too
hard to fix in the short term (e.g. it's going to take a full engineer-week), it
receives the `[Flaky]` label until it is fixed. The `[Flaky]` label should be
used very sparingly, and should be accompanied with a reference to the issue for
de-flaking the test, because while a test remains labeled `[Flaky]`, it is not
monitored closely in CI. `[Flaky]` tests are by default not run, unless a
`focus` or `skip` argument is explicitly given.

  - `[Feature:.+]`: If a test has non-default requirements to run or targets
some non-core functionality, and thus should not be run as part of the standard
suite, it receives a `[Feature:.+]` label, e.g. `[Feature:Performance]` or
`[Feature:Ingress]`. `[Feature:.+]` tests are not run in our core suites,
instead running in custom suites. If a feature is experimental or alpha and is
not enabled by default due to being incomplete or potentially subject to
breaking changes, it does *not* block the merge-queue, and thus should run in
some separate test suites owned by the feature owner(s)
(see [Continuous Integration](#continuous-integration) below).

Every test should be owned by a [SIG](/sig-list.md),
and have a corresponding `[sig-<name>]` label.

</p>
</details>


### Viper configuration and hierarchichal test parameters.

<details><summary>show</summary>
<p>

The future of e2e test configuration idioms will be increasingly defined using viper, and decreasingly via flags.

Flags in general fall apart once tests become sufficiently complicated.  So, even if we could use another flag library, it wouldn't be ideal.

To use viper, rather than flags, to configure your tests:

- Just add "e2e.json" to the current directory you are in, and define parameters in it... i.e. `"kubeconfig":"/tmp/x"`.

Note that advanced testing parameters, and hierarchichally defined parameters, are only defined in viper, to see what they are, you can dive into [TestContextType](https://git.k8s.io/kubernetes/test/e2e/framework/test_context.go).

In time, it is our intent to add or autogenerate a sample viper configuration that includes all e2e parameters, to ship with kubernetes.

</p>
</details>


### Conformance tests

<details><summary>show</summary>
<p>

Finally, `[Conformance]` tests represent a subset of the e2e-tests we expect to
pass on **any** Kubernetes cluster. The `[Conformance]` label does not supersede
any other labels.

For more information on Conformance tests please see the [Conformance Testing](conformance-tests.md)

</p>
</details>

***

## Continuous Integration


A quick overview of how we run e2e CI on Kubernetes.

### What is CI?

<details><summary>show</summary>
<p>


We run a battery of `e2e` tests against `HEAD` of the master branch on a
continuous basis, and block merges via the [submit
queue](http://submit-queue.k8s.io/) on a subset of those tests if they fail (the
subset is defined in the [munger config](https://git.k8s.io/test-infra/mungegithub/mungers/submit-queue.go)
via the `jenkins-jobs` flag; note we also block on	`kubernetes-build` and
`kubernetes-test-go` jobs for build and unit and integration tests).

CI results can be found at [ci-test.k8s.io](http://ci-test.k8s.io), e.g.
[ci-test.k8s.io/kubernetes-e2e-gce/10594](http://ci-test.k8s.io/kubernetes-e2e-gce/10594).

</p>
</details>


### What runs in CI?

<details><summary>show</summary>
<p>

We run all default tests (those that aren't marked `[Flaky]` or `[Feature:.+]`)
against GCE and GKE. To minimize the time from regression-to-green-run, we
partition tests across different jobs:

  - `kubernetes-e2e-<provider>` runs all non-`[Slow]`, non-`[Serial]`,
non-`[Disruptive]`, non-`[Flaky]`, non-`[Feature:.+]` tests in parallel.

  - `kubernetes-e2e-<provider>-slow` runs all `[Slow]`, non-`[Serial]`,
non-`[Disruptive]`, non-`[Flaky]`, non-`[Feature:.+]` tests in parallel.

  - `kubernetes-e2e-<provider>-serial` runs all `[Serial]` and `[Disruptive]`,
non-`[Flaky]`, non-`[Feature:.+]` tests in serial.

We also run non-default tests if the tests exercise general-availability ("GA")
features that require a special environment to run in, e.g.
`kubernetes-e2e-gce-scalability` and `kubernetes-kubemark-gce`, which test for
Kubernetes performance.

<details><summary>show</summary>
<p>

#### Non-default tests

<details><summary>show</summary>
<p>

Many `[Feature:.+]` tests we don't run in CI. These tests are for features that
are experimental (often in the `experimental` API), and aren't enabled by
default.

</p>
</details>


### The PR-builder

<details><summary>show</summary>
<p>

We also run a battery of tests against every PR before we merge it. These tests
are equivalent to `kubernetes-gce`: it runs all non-`[Slow]`, non-`[Serial]`,
non-`[Disruptive]`, non-`[Flaky]`, non-`[Feature:.+]` tests in parallel. These
tests are considered "smoke tests" to give a decent signal that the PR doesn't
break most functionality. Results for your PR can be found at
[pr-test.k8s.io](http://pr-test.k8s.io), e.g.
[pr-test.k8s.io/20354](http://pr-test.k8s.io/20354) for #20354.

</p>
</details>


### Adding a test to CI

<details><summary>show</summary>
<p>

As mentioned above, prior to adding a new test, it is a good idea to perform a
`-ginkgo.dryRun=true` on the system, in order to see if a behavior is already
being tested, or to determine if it may be possible to augment an existing set
of tests for a specific use case.

If a behavior does not currently have coverage and a developer wishes to add a
new e2e test, navigate to the ./test/e2e directory and create a new test using
the existing suite as a guide.

TODO(#20357): Create a self-documented example which has been disabled, but can
be copied to create new tests and outlines the capabilities and libraries used.

When writing a test, consult #kinds-of-tests above to determine how your test
should be marked, (e.g. `[Slow]`, `[Serial]`; remember, by default we assume a
test can run in parallel with other tests!).

When first adding a test it should *not* go straight into CI, because failures
block ordinary development. A test should only be added to CI after is has been
running in some non-CI suite long enough to establish a track record showing
that the test does not fail when run against *working* software. Note also that
tests running in CI are generally running on a well-loaded cluster, so must
contend for resources; see above about [kinds of tests](#kinds_of_tests).

Generally, a feature starts as `experimental`, and will be run in some suite
owned by the team developing the feature. If a feature is in beta or GA, it
*should* block the merge-queue. In moving from experimental to beta or GA, tests
that are expected to pass by default should simply remove the `[Feature:.+]`
label, and will be incorporated into our core suites. If tests are not expected
to pass by default, (e.g. they require a special environment such as added
quota,) they should remain with the `[Feature:.+]` label, and the suites that
run them should be incorporated into the
[munger config](https://git.k8s.io/test-infra/mungegithub/mungers/submit-queue.go)
via the `jenkins-jobs` flag.

Occasionally, we'll want to add tests to better exercise features that are
already GA. These tests also shouldn't go straight to CI. They should begin by
being marked as `[Flaky]` to be run outside of CI, and once a track-record for
them is established, they may be promoted out of `[Flaky]`.

</p>
</details>


### Moving a test out of CI

<details><summary>show</summary>
<p>

If we have determined that a test is known-flaky and cannot be fixed in the
short-term, we may move it out of CI indefinitely. This move should be used
sparingly, as it effectively means that we have no coverage of that test. When a
test is demoted, it should be marked `[Flaky]` with a comment accompanying the
label with a reference to an issue opened to fix the test.

</p>
</details>


## Performance Evaluation

<details><summary>show</summary>
<p>

Another benefit of the e2e tests is the ability to create reproducible loads on
the system, which can then be used to determine the responsiveness, or analyze
other characteristics of the system. For example, the density tests load the
system to 30,50,100 pods per/node and measures the different characteristics of
the system, such as throughput, api-latency, etc.

For a good overview of how we analyze performance data, please read the
following [post](http://blog.kubernetes.io/2015/09/kubernetes-performance-measurements-and.html)

For developers who are interested in doing their own performance analysis, we
recommend setting up [prometheus](http://prometheus.io/) for data collection,
and using [grafana](https://prometheus.io/docs/visualization/grafana/) to
visualize the data.  There also exists the option of pushing your own metrics in
from the tests using a
[prom-push-gateway](http://prometheus.io/docs/instrumenting/pushing/).
Containers for all of these components can be found
[here](https://hub.docker.com/u/prom/).

For more accurate measurements, you may wish to set up prometheus external to
kubernetes in an environment where it can access the major system components
(api-server, controller-manager, scheduler). This is especially useful when
attempting to gather metrics in a load-balanced api-server environment, because
all api-servers can be analyzed independently as well as collectively. On
startup, configuration file is passed to prometheus that specifies the endpoints
that prometheus will scrape, as well as the sampling interval.

```
#prometheus.conf
job: {
  name: "kubernetes"
  scrape_interval: "1s"
  target_group: {
    # apiserver(s)
    target: "http://localhost:8080/metrics"
    # scheduler
    target: "http://localhost:10251/metrics"
    # controller-manager
    target: "http://localhost:10252/metrics"
  }
}
```

Once prometheus is scraping the kubernetes endpoints, that data can then be
plotted using promdash, and alerts can be created against the assortment of
metrics that kubernetes provides.

</p>
</details>

***

## One More Thing

You should also know the [testing conventions](../guide/coding-conventions.md#testing-conventions).

**HAPPY TESTING!**