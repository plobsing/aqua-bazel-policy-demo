# Aqua policy + Bazel example

This repo serves as a demonstration of an interaction between [Aqua](https://aquaproj.github.io/)
(a commandline tool version manager) and [Bazel](https://bazel.build/) (a build tool).

To be specific, this is a demonstration of a failure caused by the interaction between Aqua's policy enforcement mechanism,
triggered by the use of a custom package registry, and the execution environment provided by Bazel when invoking
tools through `bazel run`.

## Setup

Prereq: this repo assumes that Aqua has been set up on your machine. If it hasn't
follow the directions at https://aquaproj.github.io/docs/install/.

We have a small shell script, `hello_world.sh`, that is implemented in part using a tool
supplied through Aqua, `cowsay`. This tool is somewhat niche and hasn't been packaged in
Aqua's `standard` registry; this repo provides a suitable package definition for it using
a custom, supplemental registry.

For [security reasons](https://aquaproj.github.io/docs/guides/policy-as-code/), Aqua requries
that users explicitly opt-in before downloading or running packages from custom registries.
To opt-in users acknowledge the policy described in `aqua-policy.yaml` using the command

```
aqua policy allow aqua-policy.yaml         
```

After which they will be able to install the packages provided by the custom registry and necessary
to the execution of our script of interest.

```
aqua install
```

In addition to direct invocation of the shell script, we also provide for invocation of the script by
way of Bazel's command-execution verb

```
bazel run :hello_world
```

Using [Bazelisk](https://bazel.build/install/bazelisk) to mediate versions,
this becomes

```
bazelisk run :hello_world
```

Bazelisk is availabel through Aqua, and has been installed in this registry;
we will be using it to invoke Bazel from here on.


## Results

### Without Bazel

When directly as a shell script, the script functions as intended.
The `cowsay` tool is located through Aqua and executed flawlessly.

```
❯ ./hello_world.sh 
 ______________ 
< Hello World! >
 -------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

### With Bazel

By contrast, when we invoke the exact same script that just worked, but through
Bazel, we experience Aqua rejecting our policy and refusing to execute the
`cowsay` tool.

```
❯ bazelisk run :hello_world
INFO: Invocation ID: 0fd43c61-e10b-4d93-888b-4bbd154e298a
INFO: Analyzed target //:hello_world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:hello_world up-to-date:
  bazel-bin/hello_world
INFO: Elapsed time: 0.088s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/hello_world
WARN[0000] The policy file is ignored unless it is allowed by "aqua policy allow" command.

$ aqua policy allow "/private/var/tmp/_bazel_peter/e270edd33a131681f9346244c348d651/execroot/_main/aqua-policy.yaml"

If you want to keep ignoring the policy file without the warning, please run "aqua policy deny" command.

$ aqua policy deny "/private/var/tmp/_bazel_peter/e270edd33a131681f9346244c348d651/execroot/_main/aqua-policy.yaml"

   aqua_version=2.28.1 doc="https://aquaproj.github.io/docs/reference/codes/003" env=darwin/arm64 exe_name=cowsay package_name=Code-Hex/Neo-cowsay package_version=v2.0.4 policy_file=/private/var/tmp/_bazel_peter/e270edd33a131681f9346244c348d651/execroot/_main/aqua-policy.yaml program=aqua
FATA[0000] aqua failed                                   aqua_version=2.28.1 doc="https://aquaproj.github.io/docs/reference/codes/002" env=darwin/arm64 error="install the package: this package isn't allowed" exe_name=cowsay package_name=Code-Hex/Neo-cowsay package_version=v2.0.4 program=aqua
```

Why did this happen!? If we look closely at the failure, we see the policy reported as rejected
is in some weird temp directory `/private/var/tmp/_bazel_peter/e270edd33a131681f9346244c348d651/execroot/_main/aqua-policy.yaml`.

Why would we be looking there!? It turns out this is what Bazel calls the
[execroot](https://bazel.build/remote/output-directories#:~:text=The%20working%20directory%20for%20all%20actions.),
and it is where Bazel spawns its subprocesses. It does this mainly for sandboxing build work, but it spawns the
`bazel run` process there as well.

So what's this unrecognized policy file? Why are Aqua policies we didn't author showing up on my machine? Seems
sketchy, right? Well actually, this is the policy file from our repo; the same one that we acknowledged to get our 
tools installed. That's just a bit hidden behind a symlink or two.

```
❯ realpath /private/var/tmp/_bazel_peter/e270edd33a131681f9346244c348d651/execroot/_main/aqua-policy.yaml
/Users/peter/aqua-bazel-policy-demo/aqua-policy.yaml
```

### With Bazel + explicit working directory

We can make Aqua happy by having Bazel spawn our program in an explicitly chosen directory: the current workign directory.
When we do this, Aqua finds a policy it considers accepted again and allows execution of the `cowsay` program.

```
❯ bazelisk run --run_under="cd $(pwd); exec" :hello_world 
INFO: Invocation ID: 495c8d84-c400-4c6e-a1d4-fb14f4bb3b07
INFO: Analyzed target //:hello_world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:hello_world up-to-date:
  bazel-bin/hello_world
INFO: Elapsed time: 0.085s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: /bin/bash -c 'cd /Users/peter/aqua-bazel-policy-demo; exec bazel-bin/hello_world '
 ______________ 
< Hello World! >
 -------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

## So what?

As you might have already guessed, this was never really about verbose bovines.
This is the reduction to a toy example of a real situation that might arise in real programs,
when run under Bazel, and when the system supplies tool dependencies through Aqua.

In real usage, invocation through Bazel is not nearly this optional. Usually, invocation this way is
done because some or all of the program is built through Bazel immediately preceding its execution;
there would not be a full program to invoke directly without going through Bazel.

Likewise, in reality, we do not always have the easy option to change our working directory and have
everything continue to function. Bazel starting programs in the execroot can be very convenient because
it preserves the relative locations of files within the build system; by changing directory, we give
that up and as a result might experience failures due to missing library files (dynamic languages), missing
JARs (Java), missing shared libraries (C/C++), and missing data files (all).

At present, this appears to be an unresolved defect in the interaction between these two tools.
It would be nice to have an understanding about how it should be treated.
