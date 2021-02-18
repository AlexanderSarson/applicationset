# Templates

The template fields of the `ApplicationSet` `spec` are used to generate Argo CD `Application` resources, based on the contents of that template. 

## Template fields

An Argo CD Application is created by combining the parameters from the generator, and rendering them into the template (via `{{values}}`), and from that a concrete `Application` resource is generated and applied to the cluster.

A template from a Cluster generator:
```yaml
# (...)
 template:
   metadata:
     name: '{{cluster}}-guestbook'
   spec:
     source:
       repoURL: https://github.com/infra-team/cluster-deployments.git
       targetRevision: HEAD
       path: guestbook/{{cluster}}
     destination:
       server: '{{url}}'
       namespace: guestbook
```

The template subfields correspond directly to [the spec of an Argo CD `Application` resource](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#applications):
- `project` refers to the [Argo CD Project](https://argoproj.github.io/argo-cd/user-guide/projects/) in use (`default` may be used here to utilize the default Argo CD Project)
- `source` defines from which Git repostory to extract the desired Application manifests
  - **repoURL**: URL of the repository (eg `https://github.com/argoproj/argocd-example-apps.git`)
  - **targetRevision**: Revision (tag/branch/commit) of the repository (eg `HEAD`)
  - **path**: Path within the repository where Kubernetes manifests (and/or Helm, Kustomize, Jsonnet resources) are located 
- `destination`: Defines which Kubernetes cluster/namespace to deploy to
  - **name**: Name of the cluster (within Argo CD) to deploy to
  - **server**: API Server URL for the cluster (Example: `https://kubernetes.default.svc`)
  - **namespace**: Target namespace in which to deploy the manifests from `source` (Example: `my-app-namespace`)
  - Note:
    - Referenced clusters must already be defined in Argo CD, for ApplicationSet controller to use them
    - Only **one** of `name` or `server` may be specified: if both are specified, an error is returned.

The `metadata` field of template may also be used to set an Application `name`, or to add labels or annotations to the Application. 
    
While the ApplicationSet spec provides a basic form of templating, it is not intended to replace the full-fledged configuration management capabilities of tools such as Kustomize, Helm, or Jsonnet.

## Generator templates

In addition to specifying a template within the `.spec.template` of the `ApplicationSet` resource, templates may also be specified within generators. This is useful for overriding the values of the `spec`-level template. 

The generator's `template` fields takes precedence over the `spec`'s template fields:
- if both templates contain the same field, the generator's field value will be used. 
- If only one of those fields has a value, that value will be used. 

Generator templates can thus be seen as patches against the outer `spec`-level template fields.

In this example, the ApplicationSet controller will generate an `Application` resource using the `path` generated by the List generator, rather than the `path` value defined in `.spec.template`.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - list:
      elements:
        - cluster: engineering-dev
          url: https://kubernetes.default.svc
      template:
        metadata: {}
        spec:
          project: "default"
          source:
            revision: HEAD
            repoURL: https://github.com/argoproj-labs/applicationset.git
            # New path value is generated here
            path: 'examples/template-override/{{cluster}}-override'
          destination: {}

  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj-labs/applicationset.git
        targetRevision: HEAD
        # This 'default' value is not used: it is is replaced by the generator's template path, above
        path: examples/template-override/default
      destination:
        server: '{{url}}'
        namespace: guestbook
```
(*The full example can be found [here](https://github.com/argoproj-labs/applicationset/tree/master/examples/template-override).*)