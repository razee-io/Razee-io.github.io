# Kapitan

## Overview

Kapitan is a CD 'tool' that runs on a Kubernetes cluster. It is a collection of
CustomResourceDefinitions (CRD) and controllers (Kubernetes Deployments) for those
definitions. Kapitan is designed to be extensible so that any developer can
easily plugin their custom CI process into Kapitan (We have several examples of
extending Kapitan that we use for our own CI process).

### Core Components

- Download:
  - The most essential of all 4 components, Download can be an
  effective first step into using Kapitan, as it doesn't require any of the
  other components to start deploying your resources to a cluster.
  - CRD: store information on where/how to download a resource.
  - Controller: code that uses the CRD information, downloads the resource,
  and applies the resource to the cluster.
- ManagedSet
  - A very simple component, ManagedSets main roll is to be a collection/list
  of resources. It helps to organize and maintain the resources that you want
  applied to a cluster.
  - CRD: list of resources to apply to a cluster.
  - Controller: code that applies the resources to the cluster.
- Template
  - Template is were things can start to get interesting. We use template in 2
  main ways: version control and variable substitution. For the version
  control example, we can substitute the proper version into our Download
  resource to go pull the latest version. For variable substitution, we are
  just using cluster specific variable defined in configmaps/secrets and
  putting them into the resource YAML before applying them to the cluster.
  - CRD: list of environment variables to get from the cluster and a list of
  templates to run against.
  - Controller: code to fetch the environment variables and do the templating
  and applying to the cluster.
- FeatureFlagSet
  - FeatureFlagSet is an advanced feature that allows you to connect a feature
  flagging service into your cluster. By doing this, we can pull more
  environment variables and version control information into our cluster to be
  used by our Template resource.
  - CRD: information on how to connect to the feature flag service.
  - Controller: connects and processes feature flags for the cluster, and
  saves the data into the cluster for use by other resources.

### Controllers

```text
Controller Hierarchy

BaseController
├── CompositeController
|   ├── ManagedSetController
|   ├── BaseDownloadController
|   |   ├── HTTPDownloadController
|   |   └── S3DownloadController
|   |       └── DecryptS3DownloadController
|   └── BaseTemplateController
|       └── MustacheTemplateController
└── BaseFeatureFlagSetController //to be implemented
    ├── LDFeatureFlagSetController
    └── JSONFeatureFlagSetController //to be implemented
```

#### Extensible Controllers

- BaseController: All of our controllers extend the BaseController, a collection of helper
functions to process resource events, apply files to the cluster, and update
resource status.
- CompositeController: Our 3 main controllers extend the CompositeController.
The CompositeController houses the helper functions related to 'children'. Any
resource that is created from a CompositeController is considered a child. This
is especially useful for resource cleanup. When a parent is deleted, all the
children resources it created in its lifetime will also be deleted (unless
otherwise specified).
- BaseDownloadController: BaseDownloadController handles the main functionality
around downloading resources, but not actually downloading. To have a
functioning Download controller (such as HTTPDownloadController), you must
extend BaseDownloadController and implement the download() function.
BaseDownloadController will call this download() function to get the file, and
will handle the rest for you.
- BaseTemplateController: similar to BaseDownloadController,
BaseTemplateController handles the hard stuff, but requires subclasses to
implement the processTemplate(templates, view) function which takes a set of
templates and a set of environment variables and returns the set of processed
templates to be handled and applied to the cluster by BaseTemplateController.

#### Implemented Controllers

- ManagedSetController
- HTTPDownloadController
- S3DownloadController
- DecryptS3DownloadController
- MustacheTemplateController
- LDFeatureFlagSetController: Connects to the feature flag service LaunchDarkly,
evaluates the feature flag rules based on environment data found in the cluster,
and saves the result to the cluster to be used by other resources.
