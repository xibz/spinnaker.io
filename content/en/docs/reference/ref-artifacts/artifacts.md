---
title: "Artifact Resolution"
description: How Spinnaker handles resolution of artifacts 
---

This document will cover various aspects of artifacts and how it pertains to
artifact resolution while giving high level overviews of different aspects of
an artifact, but diving deep into the resolution portions.

# Artifacts

Artifacts in Spinnaker are representation of data that are references to some
external object or the raw contents of the data.

Artifacts contains a `type` to let Spinnaker know on how to handle the
artifact. For instance, `type` may be `docker/image` and the `reference` field
would be something like `gcr.io/project/my-image@sha256:28f82eba`.

So the full representation in `JSON` may look like:
```json
{
  "type": "docker/image",
  "reference": "gcr.io/project/my-image@sha256:28f82eba",
  "name": "gcr.io/project/my-image",
  "version": "sha256:28f82eba"
}
```

However, it is important to note that artifact types will eventually be turned
into an `embedded/base64` type and persisted to the pipeline executions
context. If artifact store is enabled, then these `embedded/base64` will be
converted as a compressed version of this with type `remote/base64`, since
`embedded/base64` contains the full artifact in the pipeline context.

For a list of various artifact types supported, please visit
[here](https://spinnaker.io/docs/reference/ref-artifacts/types/) for more
information.

Spinnaker also has concepts around artifacts and may be indicated from their
key in the pipeline definition or execution context: 

* [Input Artifacts](#input-artifacts)
* [Expected Artifacts](#expected-artifacts)
* [Resolved Expected Artifacts (Bound Artifacts)](#resolved-expected-artifacts---bound-artifacts)
* [Trigger](#trigger)
* [Triggers](#triggers)

## Pipeline Definitions

Pipeline definitions, not executions, are a crucial part in the artifact
resolving process. This section will explain the various important fields
within the definition to help facilitate understanding in the later sections

Here is a subset of the pipeline definition which highlight the important parts
for artifact resolution
<details>
<summary>JSON pipeline definition</summary>
<pre>
{
  "expectedArtifacts": [
    {
      "defaultArtifact": {
        "customKind": true,
        "id": "c622eeb7-36d4-4749-9e38-03274d849eda"
      },
      "displayName": "another-random-name",
      "id": "a7d0c1cf-6a46-46a2-a338-d2255b494ea2",
      "matchArtifact": {
        "artifactAccount": "embedded-artifact",
        "id": "ceeaab7d-cd18-4517-9345-fdf0e4bf8787",
        "name": "helm/my-helm-chart.tgz",
        "type": "embedded/base64"
      },
      "useDefaultArtifact": false,
      "usePriorArtifact": false
    }
  ],
  "keepWaitingPipelines": false,
  "lastModifiedBy": "xibz",
  "limitConcurrent": true,
  "schema": "1",
  "spelEvaluator": "v4",
  "stages": [
    {
      "expectedArtifacts": [
        {
          "defaultArtifact": {
            "customKind": true,
            "id": "51c8be6a-74d2-43df-b2f7-a3335ebb8f17"
          },
          "displayName": "some-random-name",
          "id": "c060cd33-6386-47e9-a5dc-828e34aa5493",
          "matchArtifact": {
            "artifactAccount": "embedded-artifact",
            "customKind": false,
            "id": "592f0be0-8986-49fc-9189-5ae264ab12f0",
            "name": "my-artifact",
            "type": "embedded/base64"
          },
          "useDefaultArtifact": false,
          "usePriorArtifact": false
        }
      ],
      "inputArtifacts": [
        {
          "account": "embedded-artifact",
          "artifact": null,
          "id": "a7d0c1cf-6a46-46a2-a338-d2255b494ea2"
        }
      ],
      "isNew": true,
      "name": "Bake (Manifest)",
      "outputName": "my-artifact",
      "overrides": {},
      "refId": "1",
      "requisiteStageRefIds": [],
      "templateRenderer": "HELM2",
      "type": "bakeManifest"
    }
  ],
  "triggers": [
    {
      "enabled": true,
      "expectedArtifactIds": [
        "a7d0c1cf-6a46-46a2-a338-d2255b494ea2"
      ],
      "type": "email-trigger"
    }
  ]
}
</pre>
</details>

The sections below will describe the purpose of keys that are relevant to
artifact resolution

### Expected Artifacts

Expected artifacts are a list of artifact criteria that are used to match
against incoming artifacts to ensure that a stage or pipeline has received the
correct artifact(s).

The expected artifacts can be defined in stages or in a pipeline. When
`expectedArtifacts` are used for a pipeline, then that key will exist at the
top-level of the pipeline definition. The top-level `expectedArtifacts` are
used to match against any artifact that are sent to the pipeline.

The inner stage level `expectedArtifacts` are used to match against any
produced artifacts from prior stages. This is a clear difference in that one is
about receiving artifacts outside of the pipeline versus produced by the
same pipeline.

### Default and Prior Artifacts

The `useDefaultArtifact` and `usePriorArtifact` are two fields that reside in
the expected artifact class and is another way Spinnaker matches against
artifacts. When resolving artifacts, Spinnaker will iterate through the list of
artifacts, and check to see if an artifact satisfies a set of criteria. If none
of these criteria are met, then resolution will fall back by looking at the
`useDefaultArtifact` and `usePriorArtifact` fields, both booleans.
Both of them are only relevant when Spinnaker was not able to find
a match in the incoming artifacts for the given criteria. When
`useDefaultArtifact` is set, Spinnaker will resolve an artifact
that doesn't have a match by choosing instead a fallback artifact
defined next to the criteria. When `usePriorArtifact` is set,
Spinnaker will look up the previous execution for the same pipeline,
and if that artifact was a match for the same criteria, it will
re-use this artifact from the last execution.

### Triggers

Pipelines can be triggered by external sources like GitHub or Docker. These
triggers can be set up in the pipelines and reside within the `triggers` key in
the example above. Inspecting this element we can see a somewhat familiar key
of `expectedArtifactIds`. Rather than providing the full expected artifacts,
triggers rely on the IDs to match against.

When a trigger is ran, it will provide artifact(s) that will reside in the
pipeline execution context to match against.

## Pipeline Execution

The pipeline execution is data that was used during execution to perform tasks
across stages within a pipeline. When a stage is running it may add to the
execution context to help perform certain actions and also gives any meaningful
information back to the user or future tasks or stages.

Here is a subsection of the pipeline execution which includes what is necessary
for artifact resolving.
<details>
<summary>JSON pipeline execution</summary>
<pre>
"stages" : [{
  "context" : {
    "artifacts" : [{
  	  "customKind" : false,
  	  "metadata" : { },
  	  "name" : "my-artifact",
  	  "reference" : "aGVsbG8gd29ybGQK",
  	  "type" : "embedded/base64"
    }],
    "expectedArtifacts" : [{
  	  "defaultArtifact" : {
  	    "customKind" : true,
  	    "id" : "cbc26e0b-1e95-45c8-a47f-b46483985b06"
  	  },
  	  "displayName" : "sweet-falcon-28",
  	  "id" : "c060cd33-6386-47e9-a5dc-828e34aa5493",
  	  "matchArtifact" : {
  	    "artifactAccount" : "embedded-artifact",
  	    "customKind" : false,
  	    "id" : "592f0be0-8986-49fc-9189-5ae264ab12f0",
  	    "name" : "my-artifact",
  	    "type" : "embedded/base64"
  	  },
  	  "useDefaultArtifact" : false,
  	  "usePriorArtifact" : false
    }],
    "inputArtifacts" : [{
  	  "account" : "embedded-artifact",
  	  "id" : "a7d0c1cf-6a46-46a2-a338-d2255b494ea2"
    }],
    "resolvedExpectedArtifacts" : [{
  	  "boundArtifact" : {
  	    "customKind" : false,
  	    "metadata" : { },
  	    "name" : "my-artifact",
  	    "reference" : "aGVsbG8gd29ybGQK",
  	    "type" : "embedded/base64"
  	  },
  	  "defaultArtifact" : {
  	    "customKind" : true,
  	    "metadata" : {
  	  	"id" : "cbc26e0b-1e95-45c8-a47f-b46483985b06"
  	    }
  	  },
  	  "id" : "c060cd33-6386-47e9-a5dc-828e34aa5493",
  	  "matchArtifact" : {
  	    "artifactAccount" : "embedded-artifact",
  	    "customKind" : false,
  	    "metadata" : {
  	  	"id" : "592f0be0-8986-49fc-9189-5ae264ab12f0"
  	    },
  	    "name" : "my-artifact",
  	    "type" : "embedded/base64"
  	  },
  	  "useDefaultArtifact" : false,
  	  "usePriorArtifact" : false
    }],
    "templateRenderer" : "HELM2"
  },
  "name" : "Bake (Manifest)",
  "outputs" : {
    "artifacts" : [{
  	  "customKind" : false,
  	  "metadata" : { },
  	  "name" : "my-artifact",
  	  "reference" : "aGVsbG8gd29ybGQK",
  	  "type" : "embedded/base64"
    }],
    "cloudProvider" : "kubernetes",
    "resolvedExpectedArtifacts" : [{
  	  "boundArtifact" : {
  	    "customKind" : false,
  	    "metadata" : { },
  	    "name" : "my-artifact",
  	    "reference" : "aGVsbG8gd29ybGQK",
  	    "type" : "embedded/base64"
  	  },
  	  "defaultArtifact" : {
  	    "customKind" : true,
  	    "metadata" : {
  	  	"id" : "cbc26e0b-1e95-45c8-a47f-b46483985b06"
  	    }
  	  },
  	  "id" : "c060cd33-6386-47e9-a5dc-828e34aa5493",
  	  "matchArtifact" : {
  	    "artifactAccount" : "embedded-artifact",
  	    "customKind" : false,
  	    "metadata" : {
  	  	"id" : "592f0be0-8986-49fc-9189-5ae264ab12f0"
  	    },
  	    "name" : "my-artifact",
  	    "type" : "embedded/base64"
  	  },
  	  "useDefaultArtifact" : false,
  	  "usePriorArtifact" : false
    }]
  },
}],
"trigger" : {
  "artifacts" : [{
    "artifactAccount" : "embedded-artifact",
    "customKind" : false,
    "metadata" : {
  	"alias" : "my-artifact",
  	"originalName" : "the-best-artifact-ever.tgz"
    },
    "name" : "my-artifact",
    "reference" : "aGVsbG8gd29ybGQK",
    "type" : "embedded/base64"
  }],
  "expectedArtifacts" : [{
    "boundArtifact" : {
  	"artifactAccount" : "embedded-artifact",
  	"customKind" : false,
  	"metadata" : {
  	  "alias" : "my-artifact",
  	  "originalName" : "the-best-artifact-ever.tgz"
  	},
  	"name" : "my-artifact",
  	"reference" : "aGVsbG8gd29ybGQK",
  	"type" : "embedded/base64"
    },
    "defaultArtifact" : {
  	"customKind" : true,
  	"metadata" : {
  	  "id" : "c622eeb7-36d4-4749-9e38-03274d849eda"
  	}
    },
    "id" : "a7d0c1cf-6a46-46a2-a338-d2255b494ea2",
    "matchArtifact" : {
  	"artifactAccount" : "embedded-artifact",
  	"customKind" : false,
  	"metadata" : {
  	  "id" : "5676661e-2c22-4c8e-b39e-47e28d7d98cf"
  	},
  	"name" : "my-artifact",
  	"type" : "embedded/base64"
    },
    "useDefaultArtifact" : false,
    "usePriorArtifact" : false
  }],
  "resolvedExpectedArtifacts" : [{
    "boundArtifact" : {
  	"artifactAccount" : "embedded-artifact",
  	"customKind" : false,
  	"metadata" : {
  	  "alias" : "my-artifact",
  	  "originalName" : "the-best-artifact-ever.tgz"
  	},
  	"name" : "my-artifact",
  	"reference" : "aGVsbG8gd29ybGQK",
  	"type" : "embedded/base64"
    },
    "defaultArtifact" : {
  	"customKind" : true,
  	"metadata" : {
  	  "id" : "c622eeb7-36d4-4749-9e38-03274d849eda"
  	}
    },
    "id" : "a7d0c1cf-6a46-46a2-a338-d2255b494ea2",
    "matchArtifact" : {
  	"artifactAccount" : "embedded-artifact",
  	"customKind" : false,
  	"metadata" : {
  	  "id" : "5676661e-2c22-4c8e-b39e-47e28d7d98cf"
  	},
  	"name" : "my-artifact",
  	"type" : "embedded/base64"
    },
    "useDefaultArtifact" : false,
    "usePriorArtifact" : false
  }],
  "email-trigger" : {
    "email": "trigger-this-pipeline@trigger-this-pipeline.com"
  },
}
</pre>
</details>

Within this execution JSON context, there are two top level keys that can be
used for artifact resolution: `stages` and `trigger`. It is also important to
note that artifact resolution occurs both in the pipeline definition and
execution context, and are used to match against incoming artifacts.

The next nested sections will provide summaries about the purpose of each
key within the execution context

### Input Artifacts

Input artifacts is a list of incoming artifacts to be used for a given stage.
Incoming artifacts may be from some prior stage or a trigger.

### Trigger

The `artifacts` array in the trigger object is all incoming artifacts from the
trigger jobs itself.  For this particular example we see this trigger called
our pipeline with the `my-artifact` artifact.

The `trigger` key in the execution context is information about the incoming
trigger that caused the start of the pipeline execution. When a trigger is
received within Spinnaker, its payload is something that Spinnaker cannot
understand. Spinnaker must adapt this raw payload to this trigger object so
Spinnaker can do something with it.

### Resolved Expected Artifacts - Bound Artifacts

When a stage or trigger needs to match against an incoming artifact, it does so
based on the configuration specified in the pipeline definition.  When an
artifact matches against that criteria, it is put in the
`resolvedExpectedArtifacts` array.

## Artifact Resolution

With a better understanding of the pieces that are used in artifact resolution,
we can finally explain how resolution occurs. We have observed the importance
of the pipeline definitions and the execution context, and this section will
explain how both are used to resolve artifacts. Further, we can split up
resolution into two key concepts: resolving and matching which will be
discussed below

### Resolving

Artifact resolution is handled primarily in the
[`ArtifactUtils`](https://github.com/spinnaker/orca/blob/master/orca-core/src/main/java/com/netflix/spinnaker/orca/pipeline/util/ArtifactUtils.java#L65)
class in Orca. We have identified artifact resolutions two use cases: on
incoming artifacts, and stage consuming and/or producing artifacts. We will
highlight each resolution case separately to keep things simple. 

Resolution has two entry points for incoming artifacts, the
`OperationsController` or the `DependentPipelineStarter`.  These two entry
points differ in what use case they handle.  The `OperationsController` handle
all triggers except for chaining pipelines.  The `DependentPipelineStarter` is
the other use case of chaining pipelines.  So when a user defines a pipeline to
call another pipeline using the start pipeline stage, then artifact resolution
occurs in the `DependentPipelineStart` and **not** in the
`OperationsController`. Both of these entry points will call the
`resolveArtifacts` method within the `ArtifactUtils` class.

Here's the high level overview on how resolution works:

1. Looks for expected artifact IDs in the `trigger` object and `triggers` array
in the pipeline context.
	- It's important to note that the `triggers` key in the pipeline context is
      not exposed back to the user, and is from the pipeline definition itself.
	  So looking in the execution JSON will not show that key, but instead you
      need to look at the pipeline definition as JSON to see what triggers are
      available.
2. Check the expected artifacts match against any of the expected artifact IDs.
3. If neither trigger or triggers expected IDs matched, then resolution will
fall back to using the default or prior artifact, if specified.

Secondly, this will then match against the expected artifacts with the
artifacts that were built and/or provided in a Spinnaker pipeline.

1. Look for all artifacts that are specified in the `receivedArtifacts` and
`artifacts` key in the pipeline context.
2. Check if any of the received artifacts intersect with any of the expected
artifacts. If any do, we include those artifacts as resolved.

# Matching

Most, if not all, bake tasks rely on the `ArtifactUtils` and `ExpectedArtifact`
class for matching. For example, the `CreateBakeManifestTask` will call the
`getBoundArtifactForStage` which will match against any resolutions from the
entry points, `OperationsController` and `DependentPipelineStarter`.

Here are the steps involved in matching artifacts, at a high level:

1. Retrieve expected artifacts from the pipeline context.
2. Match the expected artifacts from the list of possible artifacts defined in
   the `trigger` portion in the execution context.
   - Matching can be done on the expected artifact's name, type, and/or ID. By
     looking at this ID, `a7d0c1cf-6a46-46a2-a338-d2255b494ea2`, we can see it
     many times in the pipeline execution JSON: `resolvedExpectedArtifacts`,
     `expectedArtifacts`, and `inputArtifacts`.
3. If no matches were found, we will use the original incoming artifact from the
   previous stage, if present.
4. If the stage did not receive an incoming artifact, then no match was found.
   At this point either the default artifact or an artifact from the previous
   execution might be used, if configured to do so.
