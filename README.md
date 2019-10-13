`paas-grafana-annotation-resource`
-----------------------------

![CI Workflow GitHub Actions Badge](https://github.com/alphagov/paas-grafana-annotation-resource/workflows/ci/badge.svg)
![License GitHub Badge](https://img.shields.io/github/license/alphagov/paas-grafana-annotation-resource?style=plastic)

A resource for adding annotations to Grafana dashboards.

The resource represents a single Grafana (`version >= 6.3`) and this resource
can be used to add and update annotations using tags.

Usage
-----

See the example below for all the pipeline code.

If you create an annotation using with `put: resource-name`, it will be a
point-in-time annotation. This put step will write the ID of the current
resource to `resource-name/id`.

In order to make it the annotation a region annotation, put to the resource
again, but pass the resource name to the path `path` param.  This will update
the annotation, and make it a region beginning when the resource was created,
and ending at the current time.

```
  # Create a point-in-time annotation, e.g. a single event
- put: my-grafana-annotation

- task: do-a-thing-that-takes-some-time

  # Update the point-in-time annotation to be a region
  # The region's duration will be the time taken by the task
- put: my-grafana-annotation
  params:
    path: my-grafana-annotation
```

Example
-------

In the following example, imagine we are running smoke tests from Concourse,
and we want to annotate our Grafana dashboard with a region during which the
smoke tests ran.

First define `grafana-annotation` as a resource type:

```
resource_types:
  - name: grafana-annotation
    type: docker-image
    source:
      repository: tlwr/grafana-annotation-resource
      tag: latest
```

Then declare a `grafana-annotation` resource with a descriptive name:

```
resources:
  - name: run-smoke-tests-annotation
    type: grafana-annotation
    source:
      url: http://grafana:3000
      username: admin
      password: admin
    tags:
      - run-from-concourse
      - smoke-tests
```

Then use the resource:

```
jobs:
  - name: run-smoke-tests
    plan:
      - put: run-smoke-tests-annotation
        params:
          tags:
            - started

      - task: run-smoke-tests
        config:
          platform: linux
          image_resource:
            type: docker-image
            source:
              repository: my-smoke-tests
              tag: latest
          run:
            path: /bin/smoke

      - put: run-smoke-tests-annotation
        params:
          tags:
            - finished

          # The path is the resource name, so the ID is identical to above.
          # This tells the resource that we are updating an existing annotation

          path: run-smoke-tests-annotation
```

Notes
-----

- The tags in source and params are merged

- When an existing annotation is updated, the tags from creation are
overwritten with the tags from the update

- You should ensure `put` steps to Grafana in `try` and `ensure` + `try` task
steps so that failure to create/update regions in Grafana does not impact your
pipeline.

- Currently updating individual dashboards and panels is not supported. Use
tags to view your resources instead.
