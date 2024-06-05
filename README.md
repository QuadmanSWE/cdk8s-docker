## Initializing a new app

Initializing a new app requires you to mount the target directory into the container like so: `docker run -v /my-new-cdk8s-project:/files dsoderlund/cdk8s init typescript-app`.

## Synthesizing your app

Synthesizing works similar but instead of mounting an empty directory you need to mount the directory containing your cdk8s code. This could look like: `docker run -v /my-existing-cdk8s-project:/files dsoderlund/cdk8s synth`.

In powershell on windows you can use this to escape path characters.

``` PowerShell
$app = "localappdir" | get-item | select -expand fullname
docker run -v "${app}:/files" dsoderlund/cdk8s synth
```

## Synthezising Python code

To be able to synthesize Python cdk8s code you need to use the _dsoderlund/cdk8s:python_ image. It contains additional dependencies like Python 3.7 and pipenv.

## Setting the owner of generated files

No matter if you are initializing a new app or synthesizing your manifests, all generated files will have a root ownership afterwards. This happens cause cdk8s is being executed as root to be able to read and write files from and to the volume due to [this issue](https://github.com/moby/moby/issues/2259). To fix this you can pass the `-u uid=$(id -u)` parameter to the `docker run command`. Tried to prevent executing as root but currently it does not seem possible while keeping a good user experience of the resulting image.

## Installing as a plugin for argocd

Define cdk8s container as a sidecar for argocd-repo-server. If you are installing argocd with helm there are instruction in the values file of the chart. This is also a good reference to learn how the plugins for argocd repository server: https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/

``` yaml
# ... Removed for brevity, imagine an argocd helm values declaration.
configs:
  cmp:
    create: true
    plugins:
      cdk8s-typescript:
        init:
          command: ["sh", "-c"]
          args:
            - >
              echo "init cdk8s-typescript" &&
              npm install
        generate:
          command: ["sh", "-c"]
          args:
            - >
              cdk8s synth > /dev/null &&
              cat dist/*
        discover:
          fileName: "./imports/k8s.ts"
repoServer:
  extraContainers:
    - name: cdk8s-typescript
      command:
        - "/var/run/argocd/argocd-cmp-server"
      image: docker.io/dsoderlund/cdk8s:typescript
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
      volumeMounts:
        - mountPath: /tmp
          name: cmp-tmp
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /home/argocd/cmp-server/config/plugin.yaml
          name: argocd-cmp-cm
          subPath: cdk8s-typescript.yaml
  volumes:
    - name: argocd-cmp-cm
      configMap:
        name: argocd-cmp-cm
    - name: cmp-tmp
      emptyDir: {}
```