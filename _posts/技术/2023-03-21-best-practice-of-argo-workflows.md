---
layout: post
title: Best Practice of Using Argo Workflows
category: 技术
---

<aside>
💡 Well, I lied, this is not the best practices, just a summary of the experiences of using Argo Workflows in real environment, rather than an exhaustive list of best practices.

</aside>

## Introduction

Argo Workflow is an open-source container-native workflow engine for orchestrating parallel jobs on Kubernetes. It is designed to run a series of containers and perform computations in a distributed and scalable manner. Argo Workflow helps streamline the process of managing complex workflows and enables automation of critical business processes.

In this blog post, I will share my experience of using Argo Workflow which is based on the successful delivery during a client engagement. I would assume the readers already have a basic understanding and hands-on experience with using Argo Workflows.

## Define common templates

One of the most effective practices for using Argo Workflow is to utilize common templates. By defining reusable components, you can easily integrate them into your workflows. This approach helps you create modular and reusable workflows that can be easily updated and maintained, leading to more consistent results and reduced errors.

In a large organization with restricted permission management, a dedicated team can manage the shared templates. Only workflows with the appropriate permissions can use these templates, which ensures consistency across all workflows.

Once you create shared workflow templates in a common location, it is simple to reference them in other workflows:

```yaml
...
templates:
  - name: main
    steps:
      - - name: step1
          templateRef:
            name: common-tmpls
            template: a-shared-template-name
          arguments:
            ...
```

## Parallelism of workflow steps

When using parallelism in Argo Workflow, it is important to find a balance between maximizing resource utilization and avoiding resource contention. You should carefully consider the resource requirements of each task and ensure that they do not exceed the available resources. Additionally, you should avoid using too many loops in your workflows as this can lead to performance issues and resource contention.

Like any other programming languages, loops in Argo Workflows is achieved by iterating over a set of inputs using `withItems` or `withParam` in the template, e.g.

```yaml
- name: B
  depends: "A"
  template: whalesay
  arguments:
    parameters:
    - {name: message, value: "{{item}}"}
  withItems:
  - foo
  - bar
```

It is important to note that when using `withParam`, a string representation of a JSON array is accepted. However, if the array has a large number of items (e.g. 50 or 100 or more), creating a corresponding number of pods in the cluster can lead to resource constraints and cause issues with the Kubernetes scheduler.

To avoid this, it is recommended to configure the parallelism of the workflow to limit the number of parallel executions of child workflows or templates within a workflow.

First, create a new ConfigMap like below, you can specify any key name in the data field, as long as the same name is used in the template:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wf-parallelism-config
data:
  update-product: "10"
```

Then in your workflow, on the templates that parallel execution should be limited,

```yaml
...
- name: template-in-loop
  synchronization:
    semaphore:
      configMapKeyRef:
        name: wf-parallelism-config
        key: update-product
```

This makes sure that, when the workflow is running, the maximum number of steps created simultaneously by the template `template-in-loop` should be 10.

## Avoid too many nested loops

It is important to carefully design your workflows to avoid using too many loops, especially nested loops. Each loop will create additional steps in the workflow and result in a larger amount of data being stored in the workflow status. This can quickly exceed the storage limit of the object in etcd, leading to errors such as "Request entity too large: limit is 3145728".

Using the Workflow of Workflows pattern is indeed a good solution for handling large workflow and avoiding the "Request entity too large" error. This pattern allows you to split a large workflow into smaller, more manageable workflows. To implement the Workflow of Workflows pattern, you can define a resource type template that creates Workflow custom resource as the child workflow.

```yaml
- name: triggerChildWorkflow
  inputs:
    parameters:
      - name: workflowtemplate
  resource:
    action: create
    manifest: |
      apiVersion: argoproj.io/v1alpha1
      kind: Workflow
      metadata:
        generateName: workflow-of-workflows-1-
      spec:
        workflowTemplateRef:
          name: {{inputs.parameters.workflowtemplate}}
    successCondition: status.phase == Succeeded
    failureCondition: status.phase in (Failed, Error)
```

## Create custom metrics

Creating custom metrics in Argo Workflow can help you monitor the performance of your workflows and identify areas for optimization. You can use Prometheus metrics to monitor things like workflow completion times, resource usage, and error rates. By creating custom metrics, you can gain deeper insight into how your workflows are performing and make informed decisions about how to optimize them. Additionally, we could create alerts based on the metrics, e.g. get email notifications when a workflow fails.

Argo Workflows supports pre-defined Prometheus metrics out of the box for monitoring the state of the controller, the depth of the workflow queue, and the current number of running workflows. However, one important metric that is missing is the one emitted when a workflow fails. In such cases, a custom metric is needed. In the workflow spec, this metric can be defined:

```yaml
metrics:
  prometheus:
    - name: execution_status
      help: The result status of the workflow execution
      labels:
        - key: workflow_name
          value: "{{ workflow.name }}"
        - key: status
          value: "{{ workflow.status }}"
        - key: other_resource_name
          value: xxx
      gauge:
        value: "1"
```

Then in the Prometheus alerting rule, using this metric to decide if there are new workflows failed or not.

## Hooks for workflow/template

Argo Workflow supports lifecycle hook that can be used to execute custom scripts or commands before or after a workflow/template is run. This can be useful for tasks like cleaning up resources, sending notifications, or triggering other workflows. Hooks can be defined in your workflow YAML file and run automatically as part of the workflow.

A common use case for using hooks is to integrate with services like GCP Pub/Sub. By sending events to a Pub/Sub topic at key points during the workflow, third-party applications with the correct permissions can get an overview of how the workflow is executed. This is useful for troubleshooting or auditing purposes.

An example of workflow hook definition:

```yaml
hooks:
  success:
    expression: steps["a-critical-step"].status == "Succeeded"
    templateRef:
      name: common-tmpls
      template: send-pubsub-notification
    arguments:
      parameters:
        - name: token
```

Some pitfalls when using hooks:

1. The template (`send-pubsub-notification` in the above) in the hook can’t have access to the outputs of the step referenced in the expression field.
2. When hook is defined in a step which has `withParam`, the step name in the loop is not available to the hook template, because Argo Workflows generates a random name for the steps in the loop. To work around this, we could add an extra “wrapper” step as the loop step, e.g. instead of doing:

    ```yaml
    templates:
      - name: main
        steps:
          - - name: parent-step
              withParam: "{{ workflow.parameters.items }}"
              template: child-step
              arguments:
                parameters:
                  - name: param1
                    value: "{{ item }}"
              hooks:
                success:
                  expression: steps["child-step"].status == "Succeeded" # can't use child-step here
                  templateRef:
                    name: common-tmpls
                    template: send-pubsub-notification
    ```

    we could define a new wrapper template:

    ```yaml
    templates:
      - name: main
        steps:
          - - name: parent-step
              template: child-step-wrapper
              arguments:
                parameters:
                  - name: items
                    value: "{{ workflow.parameters.items }}"
              hooks:
                success:
                  expression: steps["child-step-wrapper"].status == "Succeeded"
                  templateRef:
                    name: common-tmpls
                    template: send-pubsub-notification

      - name: child-step-wrapper
        inputs:
          parameters:
            - name: items
        steps:
          - - name: child-step
              withParam: "{{ inputs.parameters.items }}"
              template: child-step
              arguments:
                parameters:
                  - name: param1
                    value: "{{ item }}"
    ```

3. Do not define hooks in the shared templates, the step name is not deterministic before running.

## Reliability of long running workflow

One of the biggest challenges when using Argo Workflow is ensuring the reliability of long running workflows.

For each step in the workflow, Argo Workflows creates a new Pod to execute the task. However, Kubernetes is optimized for stateless and scalable web applications, where if a process fails, another process can quickly take its place. Kubernetes has no guarantees for the longevity of your Pods, especially those without Quality of Service (QoS) guarantees, and may terminate them for various reasons.

To make sure that your workflows run reliably and do not fail due to transient errors or outages, here are some options you can follow:

1. Implement retry logic and ensure that your workflows are designed to handle failures gracefully. To make this process easier, it's best to design tasks that are idempotent, meaning that they can be run multiple times without changing the result. This ensures that if a task fails and needs to be retried, it can be rerun without causing any unintended side effects or data inconsistencies.

    ```yaml
    spec:
      templates:
        - name: main
          retryStrategy:
            limit: "2"
            # Only continue retrying if the last exit code is greater than 1 and the input parameter is true
            expression: "asInt(lastRetry.exitCode) > 1 && {{inputs.parameters.safe-to-retry}} == true"
            retryPolicy: Always # By default is OnFailure
            backoff:
              duration: "5s"
              factor: "1.5"
              maxDuration: "3m"
    ```

2. Define pod disruption budget in the workflow template

    ```yaml
    spec:
      podDisruptionBudget:
        minAvailable: "9999"   # Provide arbitrary big number if you don't know how many pods the workflow creates
    ```

3. You can also break a large task into small ones, so that the impact of the disruption of pod deletion is smaller.

## Re-run failed workflow

When workflow is failed, we usually need to re-run after root cause is spotted and fixed. The `argo` CLI is used to interact with Argo Workflows service, it provides two subcommands to re-run the workflow:

- **resubmit**: A Workflow execution has been completed, and you would like to submit it again. This is essentially an alias for running argo submit again, so a new workflow will be created.
- **retry**: Rerun a failed Workflow. The same Workflow object is re-run and all of the steps that failed or errored are marked as pending and then executed as normal. No new Workflows are created.

When doing resubmit or retry, make sure all the things that the workflow is interacting with are expected, e.g. if the workflow parameters are still relevant, the resources this workflow is reading is still up-to-date, etc.

If you want to change some parameters of the failed workflow, you can run:

`argo resubmit $wf --parameter "key=value"`

Or if you want to retry the failed workflow but also need to re-run some of the succeeded steps, run:

`argo retry $wf --restart-successful --node-field-selector=displayName=xxx`

the node display name can be found in the YAML manifest of the workflow.

## Integration with Argo Events

Argo Events is often considered a crucial component when working with Argo Workflows, especially in scenarios where event-driven architecture is preferred. In fact, for some organizations, the combination of Argo Events and Argo Workflows can be a more efficient solution compared to Kubernetes operators. Additionally, Argo Events supports a wide range of event sources out of the box, making it a versatile and powerful tool for event-driven workflows.

Using fields filtering in EventSource and script filtering in Sensor dependencies can provide a more flexible and stable resource filtering mechanism, e.g.

```yaml
# In the EventSource
filter:
  labels:
    - key: app
      operation: "=="
      value: my-workflow
  fields:
    - key: metadata.name
      operation: ==
      value: my-workflow
# In the Sensor
spec:
  dependencies:
    - name: test-dep
      eventSourceName: custom-resource-events
      eventName: custom-resource-change
      filters:
        script: |-
          if event.type == "UPDATE" and event.body.metadata.generation == event.oldBody.metadata.generation then return false else return true end
```

## Conclusion

Argo Workflow is a powerful tool for managing complex workflows in a distributed and scalable manner. The definition of workflow templates is highly flexible, and there are many advanced use cases that I have yet to explore due to the restrictive policies in the environment, e.g the artifacts, workflow pod sidecars, daemon containers, etc. Those practices in this article may have worked well in your environment or maybe not, you have to evaluate your own specific needs and constraints to determine what practices work best for your own use cases.