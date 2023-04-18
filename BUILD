load("@bazel_skylib//rules:write_file.bzl", "write_file")
load("@io_bazel_rules_go//go:def.bzl", "go_binary")
load("@bazel_skylib//rules:common_settings.bzl", "bool_flag")
load("@aspect_bazel_lib//lib:transitions.bzl", "platform_transition_filegroup")


GOLANG_HELLO_WORLD="""
package main
import "fmt"
func main() {
    fmt.Println("hello world")
}
"""

BROKEN_GOLANG_HELLO_WORLD="""
package main
import "fmt"
func main() {
    fmt.println("hello world")
}
"""

write_file(
    name="hello_file",
    out="hello.go",
    content = select({
        "//:broken_build": [BROKEN_GOLANG_HELLO_WORLD],
        "//conditions:default": [GOLANG_HELLO_WORLD],
    })
)

go_binary(
    name = "hello_binary",
    srcs = ["hello.go"],
)

constraint_setting(name = "is_broken")

constraint_value(
    name = "working_build",
    constraint_setting = ":is_broken",
)

constraint_value(
    name = "broken_build",
    constraint_setting = ":is_broken",
)


platform(
    name = "broken_build_platform",
    constraint_values = [
        "//:broken_build",
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
    ],
)

platform_transition_filegroup(
    name = "broken_hello_filegroup",
    srcs = [
        ":hello_binary",
    ],
    target_platform = "//:broken_build_platform",
)
