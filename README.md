## kustomize-sops

This is a *very* simple wrapper to allow use of [sops](https://github.com/mozilla/sops)
encoded secrets within [kustomize](https://github.com/kubernetes-sigs/kustomize).

It assumes that there exists a single `secrets.enc.yaml` file, and in it there is a
1-deep YAML representation of `SECRET: VALUE`.

Assume you had a _raw_ secrets as secrets.yaml:

```
CAT: ferocious
DOG: tame
```

You would then encrypt it something like:

```
sops --encrypt --gcp-kms projects/MYPROJECT/locations/global/keyRings/sops/cryptoKeys/sops-key secrets.yaml > secrets.enc.yaml
```

You would use a `kustomization.yaml` file as:

```
---
apiVersion: kustomize-sops/v1
kind: SopsSecret
name: my-secret
namespace: bar
source: secrets.enc.yaml
metadata:
  name: not-used
keys:
  - CAT
```

If keys is empty (e.g. `keys: []`), then all keys are imported.

And then running `kustomize build --enable_alpha_plugins .` would yield:

```
apiVersion: v1
data:
  CAT: ZmVyb2Npb3Vz
kind: Secret
metadata:
  name: my-secret-hkbkhc8h2b
  namespace: bar
type: Opaque
```

You may wish to try:
`type: kubernetes.io/dockerconfigjson` if using a docker config.


More information is in the [blog](https://www.agilicus.com/safely-secure-secrets-a-sops-plugin-for-kustomize/) post.

### Install Pre-requisites

### Build & Install plugin

This is a bit complex since Go plugins are *unbelievably* brittle, all packages in both sides must be identical.
Effectively they must be built in the same tree at the same time.

To build, run `make`. Note: the result is both `kustomize` and the plugin, you must use the `kustomize`
that this builds.

It will output two files:
```
~/bin/kustomize
~/.config/kustomize/plugin/kustomize-sops/v1/sopssecret/SopsSecret.so
```

If you have ~/bin on your path, it should work from here.

I have build this with 
```
$ go version
go version go1.13.7 linux/amd64
```

I am not sure how sensitive it is to changes.

### Test/Run

```
kustomize build --enable_alpha_plugins .
```

### Setup encrypted secrets

```
gcloud auth application-default login
gcloud kms keyrings create sops --location global
gcloud kms keys create sops-key --location global --keyring sops --purpose encryption
gcloud kms keys list --location global --keyring sops
# NAME                                                                      PURPOSE          LABELS  PRIMARY_ID  PRIMARY_STATE
# projects/MYPROJECT/locations/global/keyRings/sops/cryptoKeys/sops-key  ENCRYPT_DECRYPT          1           ENABLED

sops --encrypt --gcp-kms projects/MYPROJECT/locations/global/keyRings/sops/cryptoKeys/sops-key secrets.yaml > secrets.enc.yaml
```

### Notes

The interface in `kustomize` for plugins is extremely brittle. They effectively
don't work unless compiled at the same time as kustomize.

See [kustomize-plugins](https://github.com/Agilicus/kustomize-plugins/) for exec-based
plugins that achieve the same purpose as above, but are not as brittle.

The patch... see https://github.com/kubernetes-sigs/kustomize/pull/1075#issuecomment-504551553
