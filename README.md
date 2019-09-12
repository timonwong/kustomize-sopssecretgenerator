# kustomize-sopssecret-plugin

An external plugin for [kustomize](https://github.com/kubernetes-sigs/kustomize)
that generates Secrets from files encrypted with [sops](https://github.com/mozilla/sops).


## Installation

Download the `SopsSecret` binary from the
[GitHub releases page](https://github.com/goabout/kustomize-sopssecret-plugin/releases) and
move it to `$XDG_CONFIG_HOME/kustomize/plugin/sopssecret`. (By default,
`$XDG_CONFIG_HOME` points to `~/.config` on Linux and OS X and `%LOCALAPPDATA%` on Windows.)

    VERSION=1.0.0 PLATFORM=linux ARCH=amd64
    wget https://github.com/goabout/kustomize-sopssecret-plugin/releases/download/v$VERSION/SopsSecret_$VERSION_$PLATFORM_$ARCH -O SopsSecret
    chmod +x SopsSecret
    mkdir -p $XDG_CONFIG_HOME/kustomize/plugin/sopssecret
    mv SopsSecret $XDG_CONFIG_HOME/kustomize/plugin/sopssecret


## Usage

Create some secrets using `sops`:

    echo FOO=secret > secret-vars.env
    sops -e -i secret-vars.env
    
    echo secret secret-file.txt
    sops -e -i secret-file.txt

Add a generator resourcesto your kustomization:

    cat <<. >kustomization.yaml
    generators:
      - generator.yaml
    .

    cat <<. >generator.yaml
    apiVersion: goabout.com/v1beta1
    kind: SopsSecret
    metadata:
      name: my-secret
    envs:
      - secret-vars.env
    files:
      - secret-file.txt
    .
      
Run `kustomize build` with the `--enable_alpha_plugins` flag:

    kustomize build --enable_alpha_plugins
    
The output is a Kubernetes secret containing the decrypted data:

    apiVersion: v1
    data:
      FOO: c2VjcmV0
      secret-file.txt: c2VjcmV0Cg==
    kind: Secret
    metadata:
      annotations:
        kustomize.config.k8s.io/needs-hash: "true"
      name: my-secret

The `kustomize.config.k8s.io/needs-hash` annotation uses a feature from
[kustomize #1473](https://github.com/kubernetes-sigs/kustomize/pull/1473) to add the content
hash as a suffix to the Secret name, just as the builtin secretGenerator plugin does.
If/when that PR is merged, annotations generated when using the `behavior` and
`disableNameSuffixHash` options will also be supported.

An example showing all options:

    apiVersion: goabout.com/v1beta1
    kind: SopsSecret
    metadata:
      name: my-secret
      labels:
        app: my-app
      annotations:
        create-by: me
    behavior: create
    disableNameSuffixHash: true
    envs:
      - secret-vars.env
      - secret-vars.yaml
      - secret-vars.json
    files:
      - secret-file1.txt
      - secret-file2.txt=secret-file2.sops.txt
    type: Oblique


## Build

Create a binary for your system:

    make
    
The resulting executable will be named `SopsPlugin`.

Make a release for all supported platforms:

    make release
    
Binaries can be found in `releases/`.