# Background

> Update Dec 7, 2022: Decrypted value from argocd-repo-server will be stored on redis for caching purposes. This has potential security risk if we don't enable password on redis.
> Read more about this on [this reply](https://twitter.com/crenshawdotdev/status/1600131737190772736) or to the related [GitHub Issue](https://github.com/argoproj/argo-cd/issues/3130)

ArgoCD is a Continuous Delivery system that has excelent sets of features, including an awesome UI and support for GitOps methodology. This condition really bring many users in order to use it as their Continuous Delivery system, including my company. But one crucial things that “missing” from ArgoCD is the way they do secret management, especially in GitOps approach.

Based on their [official documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/) their don’t really have any opinionated way to do this. But they do include some links about how to do secret management, most of them includes 3rd party tooling outside ArgoCD, with their own lifecycle / hacks.

> Argo CD is un-opinionated about how secrets are managed. There's many ways to do it and there's no one-size-fits-all solution. Here's some ways people are doing GitOps secrets:

One project that in my opinion really shines when combined with a full GitOps approach on ArgoCD is SOPS. [SOPS](https://github.com/mozilla/sops) — Secret Operations — a project from Mozilla is a tools for encrypting files (YAML, JSON, ENV, INI and BINARY format) with multiple keys like AWS KMS, GCP KMS, Azure key vault, [age](https://github.com/FiloSottile/age), and [PGP](https://www.openpgp.org/). From the example and how it works, this tools suits my needs very quickly for several reasons:

1. Play nicely with Kubernetes
	Kubernetes use YAML for the configuration format, thus this encryption will work out of the box. We just need to encrypt this before applying it.

2. Preserve structure and keys
	If you take a look on the example, SOPS only do encryption on the values, add signature to signal which key that being used for encryption, and preserve the structure. For my case, this is very ideal because everyone could do some checks to the secret (like is the secret name right? namespace? is there any missing keys, etc), without needing to decrypt the file first.

But one major question on how to integrate SOPS to ArgoCD Workflow?

# Integrate SOPS with ArgoCD

By default, ArgoCD doesn't have sops binary installed. To add it, we could go in multiple routes, for example Add it as [custom tool](https://argo-cd.readthedocs.io/en/stable/operator-manual/custom_tools/), or install it as [ArgoCD Plugins](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/). Custom tool approach suggest to build your own ArgoCD image with custom tooling in it, or do volume mount on initialization, and move the binary to ArgoCD.

But SOPS also could be added via plugins, which considered safer in my opinion. It basically add a sidecar to the ArgoCD, and declare the presence of the plugins via ArgoCD Config. Then, we could enable / disable the plugin via ArgoCD Application declaration. This blog will focus on how to integrate SOPS into ArgoCD via CMP Plugins.

## SOPS Application Structure

Before actually implement SOPS, we need to plan on how to design our ArgoCD Application structure to then reflect it into plugin implementation. SOPS by default has `.sops.yaml` file as [a configuration file](https://github.com/mozilla/sops#using-sops-yaml-conf-to-select-kms-pgp-for-new-files). We could utilize this special file for our `discovery` method on the plugins (will talk about this on the next section). Then, we will write all the yaml inside the directory itself with file name pattern of `*.secret.yaml`.

```bash
❯ tree secrets
secrets
├── .sops.yaml
├── app1.secret.yaml
├── app2.secret.yaml
```

### Configure SOPS

Configure the SOPS to response only files with specific pattern and encrypt partial key rather than all the keys.

```yaml
creation_rules:
  - encrypted_regex: "^(data|stringData)$"
    path_regex: \.secret\.yaml$
    gcp_kms: 'my-gcp-kms-key'
```

From the snippet above, we declare that files with name `*.secret.yaml` will be encrypted by using Google Cloud KMS key of `my-gcp-kms-key`, and only encrypt values under `data` or `stringData`. On Kubernetes Secret context, `data` and `stringData` will contains our sensitive information, while other keys like `metadata.name`, or `type` is not necessary to be encrypted.

## Create CMP Plugins ArgoCD using Sidecar Approach

At the very beginning, we need to declare plugin configuration as a YAML with kind `ConfigManagementPlugin`, then add it to the sidecar image via custom build, or mount it directly.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: sops-plugin
spec:
  version: v1.0
  generate:
    command:
      - bash
      - -c
      - |
        echo "" > output.yaml
        for f in *.secret.yaml
        do
            sops -d $f >> output.yaml
            cat <<EOF >> output.yaml
        ---
        EOF
        done
        cat output.yaml
  discover:
    fileName: ".sops.yaml"
```

from the snippet above, we declare a plugin called `sops-plugin`. There are 2 major keys under `spec` to be configured. `discover` declare how ArgoCD know when to use this plugins. As we declare the `discover` with `fileName`, it means that if ArgoCD Application directory has `.sops.yaml` inside it, it will use this plugins instead.

`generate` key declare what to do if this plugins being used. As you expect, this section is our implementation of SOPS. `generate` must print a valid YAML stream to stdout, with SOPS it means to simply execute `sops -d` command. The directory will have multiple files to decrypt, so `sops -d` isn't enough, we need a way to merge every yamls then output it to the stream.

```bash
# Initialize new output.yaml file
echo "" > output.yaml

# Loop for all files with suffix `.secret.yaml`, then do `sops -d`
for f in *.secret.yaml
do
	sops -d $f >> output.yaml
	cat <<EOF >> output.yaml
---
EOF
done

# read the file a.k.a send it to stdout.
cat output.yaml
```

## [Register the Plugin to ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/#3-register-the-plugin-sidecar)

To register the plugin, we need to configure our argocd-repo-server deployment. `argocd-repo-server` will have the plugin container runs as a sidecar and plugin declaration yaml files. Here are the overlay of `argocd-repo-server` if you use Kustomize to install ArgoCD.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      containers:
      - name: sops
        command:
        - /var/run/argocd/argocd-cmp-server
        image: mozilla/sops:v3 # Use official SOPS image
        env:
          - name: GOOGLE_APPLICATION_CREDENTIALS # For using GCP KMS, we need Service Account configured.
            value: /secrets/my-service-account.json
        securityContext:
          runAsNonRoot: true
          runAsUser: 999 # Read more on https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/#3-register-the-plugin-sidecar
        volumeMounts:
          - mountPath: /var/run/argocd
            name: var-files
          - mountPath: /home/argocd/cmp-server/plugins
            name: plugins
          - mountPath: /home/argocd/cmp-server/config/plugin.yaml # Our plugin declaration goes here.
            subPath: plugin.yaml
            name: cmp-plugin
          - mountPath: /tmp
            name: cmp-tmp
          - mountPath: /secrets/my-service-account.json # For using GCP KMS
            name: argocd-bot-service-account
            subPath: my-service-account.json
      volumes:
      - configMap:
          name: sops-plugin
        name: cmp-plugin
      - emptyDir: {}
        name: cmp-tmp
      - name: argocd-bot-service-account # For using GCP KMS
        secret:
          secretName: secret-containing-service-account-json
```

From the snippet above, we adds another container called `sops`, with the official SOPS docker image, but customize the entrypoint to use `/var/run/argocd/argocd-cmp-server` from default volumes that `argocd-repo-server` has.

Then mount the plugins files into `cmp-plugin` directory, as we are using official image, and then configure the temporary directory (as `cmp-tmp`).

On this overlay, you could also configured any additional configuration for SOPS to working properly, e.g configuring access to your GCP KMS, or mounting AGE private key. We also could configure resource request and limits here just in case the plugins require more compute resources in order to run well.

# Conclusion

ArgoCD doesn't have secret management tools out of the box, while SOPS is a great simple tools to manage secret. In order to configure ArgoCD to use SOPS, we could take several approach, and in my opinion, the safest route is to implement SOPS as CMP Plugin Sidecar. This approach as several benefit such as:

## Isolated lifecycle — Security & Resource isolation

Only the container that do decrypt has access to the keys. Even the ArgoCD container didn’t has it, reducing potential leaks of the access to the key. Sidecar also has it’s own resources, so if our implementation screwed in resource, we could easily detect it. Isolation also help us debugging our plugins because the log stream will be different between ArgoCD and Plugins.

## No need to custom build ArgoCD.

Custom ArgoCD Image means we need to maintain and keep it up to date with the upstream, or even do resolution if our changes has conflict with the upstream branch. Sidecar avoid us to do those stuff.

## Plug & Play

If our plugin has issues, and we want to go back, simply remove the sidecar from the deployment. It still restart ArgoCD Server workload, but it doesn’t changed at all, so we could expect that once you remove the sidecar, ArgoCD will work as expected.