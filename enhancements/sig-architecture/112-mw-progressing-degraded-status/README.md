# Define Progressing and Degrade Status in ManifestWork

## Release Signoff Checklist

- [] Enhancement is `implemented`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is to enhance the `ManifestWork` API to report progressing and degraded status. 

## Motivation
Currently, work agent only report `Applied` and `Availabe` status of the `ManifestWork`. However, progressing and degraded 
status is important for the user to know whether the workload deployed by the `ManifestWork` is still rolling out, or in 
a bad state. Also 
[rollingOutStrategy API](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/81-addon-lifecycle).
we have introduced before also requires the `ManifestWork` to provide progressing state so the that addon manager can know whether
the addon is upgrading or not.

### Goals

- Improve `ManifestWork` API to allow defining `Progressing` and `Degraded` status conditions.
- Work agent reports `Progressing` and `Degraded` status conditions based on configurations in the spec.

## Proposal

### Design Details

The is not common approach to define the progressing and degraded state of a `ManifestWork`. It is up to the resource
manifests in the spec of `ManifestWork`. For example, we can define a `Deployment` is in progressing when its `status.readyReplicas`
does not equal to `spec.replicas`, and in degraded state if its `status.unavailbleReplicas` is not zero. It is not true
for another workload resource or a custom resource. Defining these state should be customizable by the consumer of the
`ManifestWork`.

The CEL has been used in CRD validation and `AdmissionValidatingPolicy`, and we propose to use the same mechanism in
`ManifestWork` to define these two states. 

#### API change

In `ManifestConfigOption`, we introduce a new `StatusCheck`

```go

type ManifestConfigOption struct {
	// ResourceIdentifier represents the group, resource, name and namespace of a resoure.
	// iff this refers to a resource not created by this manifest work, the related rules will not be executed.
	// +kubebuilder:validation:Required
	// +required
	ResourceIdentifier ResourceIdentifier `json:"resourceIdentifier"`

	// FeedbackRules defines what resource status field should be returned. If it is not set or empty,
	// no feedback rules will be honored.
	// +optional
	FeedbackRules []FeedbackRule `json:"feedbackRules,omitempty"`

	// this is newly added to define the check to return progressing and degraded status
	StatusCheck []StatusCheck `json:"statusCheck,omitempty"`

	// UpdateStrategy defines the strategy to update this manifest. UpdateStrategy is Update
	// if it is not set.
	// +optional
	UpdateStrategy *UpdateStrategy `json:"updateStrategy,omitempty"`
}

type StatusCheck struct {
	// A list of checkers, if all passed, set progreassing to false, otherwise set
	// progressing to true
	IsProgressing []Checker `json:"isProgressing,omitempty"`
	// A list of checkers, if all passed, set degraded to false, otherwise set
	// degraded to true
	IsDegraded    []Checker `json:"isDegraded,omitempty"`
}

type Checker struct {
    Expression string `json:"expression"`
    // Message represents the message displayed when validation fails. The message is required if the Expression contains
    // line breaks. The message must not contain line breaks.
    // If unset, the message is "failed rule: {Rule}".
    // e.g. "must be a URL with the host matching spec.host"
    // If the Expression contains line breaks. Message is required.
    // The message must not contain line breaks.
    // If unset, the message is "failed Expression: {Expression}".
   // +optional
   Message string `json:"message,omitempty"`
}
```

An example `ManifestWork` with the checker would be like:

```yaml
kind: ManifestWork
metadata:
  name: demo-work1
spec:
  workload:
    manifests:
    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello
        namespace: default
      spec:
        selector:
          matchLabels:
            app: hello
        template:
          metadata:
            labels:
              app: hello
          spec:
            containers:
            - name: hello
              image: quay.io/asmacdo/busybox
              command: ['sh', '-c', 'echo "Hello, World!" && sleep 3600']
  manifestConfigs:
  - resourceIdentifier:
      group: apps
      resource: deployments
      name: hello
      namespace: default
    statusChecker:
      isProgressing:
      - expression: obj.status.readyReplicas != obj.spec.replicas
        message: deployment is still rolling.
      isDegraded:
      - expression: obj.status.unavailableReplicas > 0
        message: some repolicas of the deployment is unavailable.
```

Multiple expressions for a single manifests are `ANDed`. 

### Test Plan

E2E tests will be added to cover cases including:

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

### Version Skew Strategy
- The statusChecker field is optional, and if it is not set, the manifestwork can be correctly treated by work agent with elder version
- The elder version work agent will ignore the statusChecker field.

## Alternatives

