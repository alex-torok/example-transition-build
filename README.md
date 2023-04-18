This repo shows how it can be difficult to re-run a build to reproduce a failing build.

```console
$ bazel build ...

Starting local Bazel server and connecting to it...
INFO: Analyzed 5 targets (56 packages loaded, 10053 targets configured).
INFO: Found 5 targets...
ERROR: /home/alex.torok/code/bzl-transition-example/BUILD:38:10: GoCompilePkg hello_binary.a failed: (Exit 1): builder failed: error executing command (from target //:hello_binary) bazel-out/k8-opt-exec-2B5CBBC6-ST-e846b08c7501/bin/external/go_sdk/builder_reset/builder compilepkg -sdk external/go_sdk -installsuffix linux_amd64 -src ... (remaining 19 arguments skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging
bazel-out/k8-fastbuild-ST-0b574a6423b1/bin/hello.go:5:9: undefined: fmt.println
compilepkg: error running subcommand external/go_sdk/pkg/tool/linux_amd64/compile: exit status 2
INFO: Elapsed time: 10.241s, Critical Path: 0.13s
INFO: 3 processes: 3 internal.
FAILED: Build did NOT complete successfully
```

From this log output, someone would likely try to build //:hello_binary, but that passes:
```console
$ bazel build //:hello_binary
INFO: Analyzed target //:hello_binary (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //:hello_binary up-to-date:
  bazel-bin/hello_binary_/hello_binary
INFO: Elapsed time: 0.201s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```

The only way to detect the target label that should be built to reproduce the error is to parse the build event log:

```console
$ bazel build --build_event_json_file=bes.json //...
...

$ jq 'select(.id.targetCompleted != null) | select(.completed.failureDetail != null)' bes.json
{
  "id": {
    "targetCompleted": {
      "label": "//:broken_hello_filegroup",
      "configuration": {
        "id": "6a1fc98b22e6e815a60a63d52626f85d15b8a83339bfb1740d0c4d67db078e13"
      }
    }
  },
  "children": [
    {
      "actionCompleted": {
        "primaryOutput": "bazel-out/k8-fastbuild-ST-0b574a6423b1/bin/hello_binary.a",
        "label": "//:hello_binary",
        "configuration": {
          "id": "2b9f03aa8e43887b2fdc1a5d659e0bb39d7e876607c5a49fa5b41b17fed1be62"
        }
      }
    }
  ],
  "completed": {
    "failureDetail": {
      "message": "builder failed: error executing command (from target //:hello_binary) bazel-out/k8-opt-exec-2B5CBBC6-ST-e846b08c7501/bin/external/go_sdk/builder_reset/builder compilepkg -sdk external/go_sdk -installsuffix linux_amd64 -src ... (remaining 19 arguments skipped)\n\nUse --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging",
      "spawn": {
        "code": "NON_ZERO_EXIT",
        "spawnExitCode": 1
      }
    }
  }
}
```

We can see that the real target to build is `//:broken_hello_filegroup`:

```console
$ bazel build //:broken_hello_filegroup

INFO: Analyzed target //:broken_hello_filegroup (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
ERROR: /home/alex.torok/code/bzl-transition-example/BUILD:38:10: GoCompilePkg hello_binary.a failed: (Exit 1): builder failed: error executing command (from target //:hello_binary) bazel-out/k8-opt-exec-2B5CBBC6-ST-e846b08c7501/bin/external/go_sdk/builder_reset/builder compilepkg -sdk external/go_sdk -installsuffix linux_amd64 -src ... (remaining 19 arguments skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging
bazel-out/k8-fastbuild-ST-0b574a6423b1/bin/hello.go:5:9: undefined: fmt.println
compilepkg: error running subcommand external/go_sdk/pkg/tool/linux_amd64/compile: exit status 2
Target //:broken_hello_filegroup failed to build
Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 0.198s, Critical Path: 0.02s
INFO: 2 processes: 2 internal.
FAILED: Build did NOT complete successfully
```
