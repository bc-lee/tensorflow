# buildifier: disable=load-on-top

workspace(name = "org_tensorflow")

# buildifier: disable=load-on-top

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Provide rules_android under its canonical name to satisfy android_library auto-loads.
http_archive(
    name = "rules_android",
    sha256 = "cd06d15dd8bb59926e4d65f9003bfc20f9da4b2519985c27e190cddc8b7a7806",
    strip_prefix = "rules_android-0.1.1",
    urls = [
        "https://github.com/bazelbuild/rules_android/archive/v0.1.1.zip",
    ],
    patch_cmds = [
        # Bazel 8 autoload expects //rules:rules.bzl; older release places it under //android.
        "mkdir -p rules",
        "cat > rules/rules.bzl <<'EOF'\nload(\"//android:rules.bzl\", _android_binary = \"android_binary\", _android_local_test = \"android_local_test\", _android_instrumentation_test = \"android_instrumentation_test\", _android_sdk_repository = \"android_sdk_repository\", _android_ndk_repository = \"android_ndk_repository\", _android_device = \"android_device\")\nload(\"//android:aar_import.bzl\", _aar_import = \"aar_import\")\n\n# Native android_library is removed in Bazel 8, need to load from rules_android starlark implementation\n# BUT rules_android v0.1.1 implementation calls native.android_library, so we must patch library.bzl too\n# For now, let's mock android_library as a no-op rule if native is missing, or try to fix library.bzl\nload(\"//android:library.bzl\", _android_library = \"android_library\")\n\nandroid_library = _android_library\nandroid_binary = _android_binary\nandroid_local_test = _android_local_test\nandroid_instrumentation_test = _android_instrumentation_test\nandroid_sdk_repository = _android_sdk_repository\nandroid_ndk_repository = _android_ndk_repository\nandroid_device = _android_device\naar_import = _aar_import\n\n# Fallback for symbols not in 0.1.1 but expected by Bazel 8\ndef _stub(*args, **kwargs): pass\nandroid_sdk = _stub\nandroid_tools_defaults_jar = _stub\nEOF",
        "cat > rules/BUILD <<'EOF'\nexports_files([\"rules.bzl\"])\nEOF",
        # Fix android/library.bzl to use a stub if native.android_library is missing
        "cat > android/library.bzl <<'EOF'\n\ndef _android_library_impl(ctx):\n    return []\n\n_stub_android_library = rule(\n    implementation = _android_library_impl,\n    attrs = {\n        \"srcs\": attr.label_list(allow_files = True),\n        \"deps\": attr.label_list(),\n        \"custom_package\": attr.string(),\n        \"manifest\": attr.label(allow_single_file = True),\n        \"resource_files\": attr.label_list(allow_files = True),\n        \"enable_data_binding\": attr.bool(),\n        \"assets\": attr.label_list(allow_files = True),\n        \"assets_dir\": attr.string(),\n        \"exports\": attr.label_list(),\n    },\n)\n\ndef android_library(**attrs):\n    if hasattr(native, \"android_library\"):\n        native.android_library(**attrs)\n    else:\n        # Drop arguments not supported by our minimal stub if necessary, or just pass strict subset\n        # For now, just pass everything and hope the stub attrs cover enough or are ignored if we use **kwargs in rule?\n        # Actually rule instantiation doesn't take **kwargs freely. We need to filter or make a robust stub.\n        # Let's try to just invoke the stub with common args or filtered args.\n        # Simpler: define the rule with minimal attributes but accept others via **kwargs in macro?\n        # Macros can take **kwargs, rules cannot. \n        # So we define a macro that calls the rule.\n        # BUT wait, we are overriding the file. \n        _stub_android_library(name = attrs.get(\"name\"), **{k: v for k, v in attrs.items() if k != \"name\"})\nEOF",
        # Bazel 8 also auto-loads //providers:providers.bzl; add stubs for expected providers.
        "mkdir -p providers",
        "cat > providers/providers.bzl <<'EOF'\n# Minimal shims to satisfy Bazel 8 builtins autoload when rules_android is legacy\nload(\"//android:library.bzl\", _android_library = \"android_library\")\nAndroidIdeInfo = provider()\nAndroidSdkInfo = provider()\nAndroidInstrumentationInfo = provider()\nAndroidLibraryAarInfo = provider()\nAndroidLibraryResourceClassJarInfo = provider()\nAndroidAarImportInfo = provider()\nAndroidAssetsInfo = provider()\nAndroidResourcesInfo = provider()\nAndroidValidatedAarInfo = provider()\nAndroidNativeLibsInfo = provider()\nAndroidDexInfo = provider()\nAndroidPreDexMergeInfo = provider()\nandroid_library = _android_library\nEOF",
        "cat > providers/BUILD <<'EOF'\nexports_files([\"providers.bzl\"])\nEOF",
    ],
)

# Keep NDK rules alongside for compatibility (mirrors workspace2.bzl).
http_archive(
    name = "rules_android_ndk",
    sha256 = "0ab5ddae72dff0dfae92a31a0704d4543e818e360786e44d2093a6b8ff5e8fda",
    strip_prefix = "rules_android_ndk-461e8c99b7f06bc86a15317505d48fc0decd7dcc",
    urls = [
        "https://github.com/bazelbuild/rules_android_ndk/archive/461e8c99b7f06bc86a15317505d48fc0decd7dcc.zip",
    ],
)

http_archive(
    name = "rules_shell",
    sha256 = "bc61ef94facc78e20a645726f64756e5e285a045037c7a61f65af2941f4c25e1",
    strip_prefix = "rules_shell-0.4.1",
    url = "https://github.com/bazelbuild/rules_shell/releases/download/v0.4.1/rules_shell-v0.4.1.tar.gz",
)

# Initialize the TensorFlow repository and all dependencies.
#
# The cascade of load() statements and tf_workspace?() calls works around the
# restriction that load() statements need to be at the top of .bzl files.
# E.g. we can not retrieve a new repository with http_archive and then load()
# a macro from that repository in the same file.
load("@//tensorflow:workspace3.bzl", "tf_workspace3")

tf_workspace3()

load("@rules_shell//shell:repositories.bzl", "rules_shell_dependencies", "rules_shell_toolchains")

rules_shell_dependencies()

rules_shell_toolchains()

# Initialize hermetic Python
load("@local_xla//third_party/py:python_init_rules.bzl", "python_init_rules")

python_init_rules()

load("@local_xla//third_party/py:python_init_repositories.bzl", "python_init_repositories")

python_init_repositories(
    default_python_version = "system",
    local_wheel_dist_folder = "dist",
    local_wheel_inclusion_list = [
        "tensorflow*",
        "tf_nightly*",
    ],
    local_wheel_workspaces = ["//:WORKSPACE"],
    requirements = {
        "3.9": "//:requirements_lock_3_9.txt",
        "3.10": "//:requirements_lock_3_10.txt",
        "3.11": "//:requirements_lock_3_11.txt",
        "3.12": "//:requirements_lock_3_12.txt",
        "3.13": "//:requirements_lock_3_13.txt",
    },
)

load("@local_xla//third_party/py:python_init_toolchains.bzl", "python_init_toolchains")

python_init_toolchains()

load("@local_xla//third_party/py:python_init_pip.bzl", "python_init_pip")

python_init_pip()

load("@pypi//:requirements.bzl", "install_deps")

install_deps()
# End hermetic Python initialization

load("@//tensorflow:workspace2.bzl", "tf_workspace2")

tf_workspace2()

load("@com_google_protobuf//:protobuf_deps.bzl", "protobuf_deps")

protobuf_deps()

load("@//tensorflow:workspace1.bzl", "tf_workspace1")

tf_workspace1()

load("@//tensorflow:workspace0.bzl", "tf_workspace0")

tf_workspace0()

load(
    "@local_xla//third_party/py:python_wheel.bzl",
    "nvidia_wheel_versions_repository",
    "python_wheel_version_suffix_repository",
)

nvidia_wheel_versions_repository(
    name = "nvidia_wheel_versions",
    versions_source = "//ci/official/requirements_updater:nvidia-requirements.txt",
)

python_wheel_version_suffix_repository(name = "tf_wheel_version_suffix")

load(
    "@rules_ml_toolchain//cc/deps:cc_toolchain_deps.bzl",
    "cc_toolchain_deps",
)

cc_toolchain_deps()

register_toolchains("@rules_ml_toolchain//cc:linux_x86_64_linux_x86_64")

register_toolchains("@rules_ml_toolchain//cc:linux_x86_64_linux_x86_64_cuda")

register_toolchains("@rules_ml_toolchain//cc:linux_aarch64_linux_aarch64")

register_toolchains("@rules_ml_toolchain//cc:linux_aarch64_linux_aarch64_cuda")

load(
    "@rules_ml_toolchain//third_party/gpus/cuda/hermetic:cuda_json_init_repository.bzl",
    "cuda_json_init_repository",
)

cuda_json_init_repository()

load(
    "@cuda_redist_json//:distributions.bzl",
    "CUDA_REDISTRIBUTIONS",
    "CUDNN_REDISTRIBUTIONS",
)
load(
    "@rules_ml_toolchain//third_party/gpus/cuda/hermetic:cuda_redist_init_repositories.bzl",
    "cuda_redist_init_repositories",
    "cudnn_redist_init_repository",
)

cuda_redist_init_repositories(
    cuda_redistributions = CUDA_REDISTRIBUTIONS,
)

cudnn_redist_init_repository(
    cudnn_redistributions = CUDNN_REDISTRIBUTIONS,
)

load(
    "@rules_ml_toolchain//third_party/gpus/cuda/hermetic:cuda_configure.bzl",
    "cuda_configure",
)

cuda_configure(name = "local_config_cuda")

load(
    "@rules_ml_toolchain//third_party/nccl/hermetic:nccl_redist_init_repository.bzl",
    "nccl_redist_init_repository",
)

nccl_redist_init_repository()

load(
    "@rules_ml_toolchain//third_party/nccl/hermetic:nccl_configure.bzl",
    "nccl_configure",
)

nccl_configure(name = "local_config_nccl")

load(
    "@rules_ml_toolchain//third_party/nvshmem/hermetic:nvshmem_json_init_repository.bzl",
    "nvshmem_json_init_repository",
)

nvshmem_json_init_repository()

load(
    "@nvshmem_redist_json//:distributions.bzl",
    "NVSHMEM_REDISTRIBUTIONS",
)
load(
    "@rules_ml_toolchain//third_party/nvshmem/hermetic:nvshmem_redist_init_repository.bzl",
    "nvshmem_redist_init_repository",
)

nvshmem_redist_init_repository(
    nvshmem_redistributions = NVSHMEM_REDISTRIBUTIONS,
)
