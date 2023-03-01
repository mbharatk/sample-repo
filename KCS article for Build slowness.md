# [OCP 4.x] KCS article for Build slowness and how to improve the performance of the builds.

There were several support requests opened about build slowness, especially by users who are migrating from OCP 3.11 to 4.x. This article provides the workaround/actions can be taken by the users to improve the build performance and help the builds run faster:

## a) Remove/disable the Dynatrace OneAgent operator:
Some customers run the Dynatrace OneAgent operator on their clusters. OneAgent by default enables automatic "deep" monitoring of all processes, which causes the performance of OpenShift Builds to degrade significantly (1). Any fix to address the performance degradation would need to be provided by Dynatrace (in partnership with Red Hat if necessary).

###### Workaround:
OpenShift admins/users who install Dynatrace OneAgent can configure Dynatrace to exclude deep monitoring of certain workloads (2). Admins can add a monitoring rule to disable the process monitor for the OpenShift-Build process which excludes all OpenShift Builds, if that is desired.

#### Admins can also use Tolerations and Node Selectors to isolate Builds from nodes that run Dynatrace OneAgent. This could be accomplished as follows:

```
1. Add node labels and taints to the build worker nodes

  a. Taint worker nodes to be used for builds with a desired key, value, and the `NoSchedule` effect:
    
    `$ oc taint node <worker-node> build-node=true:NoSchedule-`
   
  b. Label these worker nodes with a desired key and value. These can be the same as above:
  
    $ oc label node <worker-node> build-node=true

2. Alternatively, add or update the labels and taints on a MachineSet (3)

  apiVersion: machine.openshift.io/v1beta1
  kind: MachineSet
  ...
  spec:
    template: # this is the template for the Machines to be provisioned
      metadata:
        labels:
          build-node: "true"
      ...
      spec:
        metadata: # this is metadata applied to all Nodes underlying the MachineSet
          labels:
            build-node: "true"
      taints: # taints applied to all Nodes underlying the MachineSet
      - effect: NoSchedule
        key: build-node
        value: "true"

3. Set up a cluster-wide BuildOverride that allows builds to tolerate the "build-node" taint and forces builds onto the labeled build-nodes (4).

$ oc edit build.config.openshift.io/cluster

spec:
  buildOverrides:
    nodeSelector:
      build-node: "true"
    tolerations:
    - effect: NoSchedule
      key: build-node
      
4. Deploy Dynatrace OneAgent via Operator Hub. The agents will not tolerate the custom "build-node" taint by default and therefore will not run on these nodes.
```

[1] https://access.redhat.com/solutions/4978291

(2) https://www.dynatrace.com/support/help/shortlink/process-group-monitoring#enable-automatic-deep-monitoring

(3) https://docs.openshift.com/container-platform/4.7/machine_management/creating_machinesets/creating-machineset-aws.html

(4) https://docs.openshift.com/container-platform/4.7/cicd/builds/build-configuration.html

## b) Increase the memory allocated to the build such that it is about equal to the base image size:
Sometimes when the user has default resource limits set, it may lead to cpu/memory constraints when the builds run. To overcome this, the user can set resource requirements on builds and increase the requests and limits to check if the performance and time of the builds are improved.

Support team should work with the user to investiagte the reason for the consumption of cpu/memory and determine a minimal resource the customer builds require to get sufficient performance.



