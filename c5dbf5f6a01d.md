## Configuring Go Agents in GoCD in Kubernetes: Static and Elastic Agents

# Configuring Go Agents in GoCD in Kubernetes: Static and Elastic Agents


_Recently, a_ [_colleague_](https://www.linkedin.com/in/saranrajsekar/) _and I had to spike on the possibility of migrating our CI/CD infrastructure from a VM based design to a Kubernetes based one._

_While exploring how to do it, we had to spend some time to find out all the information we needed just to install and configure GoCD without the migration. So I decided to write this and a few other posts hoping someone like me would find it helpful. This is the third part on Configuring Go Agents in GoCD in Kubernetes: Static and Elastic Agents. You can find the first part on Installing and Configuring GoCD on GKE using Helm_ [_here_](https://mohamednajiullah.tech/installing-and-configuring-gocd-on-gke-using-helm-cjy1vtdio005jjjs1ivrml8f6) _and the second part on Configuring Google Container Registry (GCR) as an Artifact Store in GoCD_ [_here_](https://mohamednajiullah.tech/configuring-google-container-registry-gcr-as-an-artifact-store-in-gocd-5453f8037e8c).

While carrying out our spike on our CI/CD infrastructure from a VM based design to a Kubernetes based one, we also had to configure Go Agents. According to the official [documentation](https://docs.gocd.org/current/introduction/concepts_in_go.html#agent),

> _GoCD Agents are the workers in the GoCD ecosystem. All tasks configured in the system run on GoCD Agents._

The latest version of GoCD offers two types of GoCD agents, Static and Elastic.

# Static Agents

In our old system, we have a predefined number of Go Agents. These agents are VMs that we had created, provisioned and connected to the Go Server using Infrastructure as Code tools like Puppet. The agents needed a number of tools and packages to do the things we needed them to do. All these tools and packages had to be provisioned into all the agents individually once we had created them.

But for a GoCD setup on Kubernetes, it becomes much simpler to maintain agents as every agent is simply a pod. The GoCD Helm chart allows us to configure always running agents using the values in the values.yaml file I’ve mentioned in my previous post.

This greatly simplifies provisioning the agents as all we have to do is create a Docker image with all the tools and packages installed in it. Once we have this image ready we can specify it in the _agent.image_ section of the _values.yaml_ fileas follows


```
agent:
  image:
    # agent.image.repository is the GoCD Agent image name
    repository: "gocd/gocd-agent-alpine-3.9"# agent.image.tag is the GoCD Agent image's tag
    tag: v19.3
    # agent.image.pullPolicy is the GoCD Agent image's pull policy
    pullPolicy: "IfNotPresent" # agent.replicaCount is the GoCD Agent replicas Count. Specify the number of GoCD agents to run
  replicaCount: 2
```


The above example states that the agent repository is gocd/gocd-agent-alpine-3.9 with the version being v19.3\. You can also the specify the replica count. This is the number of agents that will be running once the values are deployed.

Every time a new tool or package is to be added to the agent, a new image can be created. Then this image can be referenced in the given image fields before being applied. This greatly simplifies the upgrade process.

_Note:_
1.  _The given repository should be public. If a private repository is used, then the Go server should have access to it. I have mentioned how to configure access for a private repository using the Docker Artifact Store plugin here._
2.  _All agents that are created after applying the values will be in a pending state. They have to be manually enabled._

There are many other configuration options available in the [values.yaml](https://github.com/helm/charts/blob/master/stable/gocd/values.yaml) that can be used to configure the agents. One important example is the _security.ssh_ section that allows us to mount a SSH files using a kubernetes secret. This can be used to allow access to various VCS that use SSH to control access. I’ve explained it in a little more detail in the [first post](/@mohammed.najiullah/installing-and-configuring-gocd-on-gke-using-helm-1acf73649dc6) of this series. You can also specify the [storage](https://github.com/helm/charts/blob/480ad2744095a7ca62699c7cd97d20e2491a4852/stable/gocd/values.yaml#L268), [memory as well as maximum CPU allocation](https://github.com/helm/charts/blob/480ad2744095a7ca62699c7cd97d20e2491a4852/stable/gocd/values.yaml#L348)for the agents.

# Elastic Agents:

We also experimented with elastic agents which is a relatively new feature offered by GoCD. According to the official [documentation](https://docs.gocd.org/current/configuration/elastic_agents.html)

> Elastic Agents is an extension-point in GoCD that allows for on-demand agents which are created and provisioned by an elastic-agent plugin when there are jobs to be executed, and terminated when the agents are running idle.

Elastic Agents have a significant advantage in that they get created when there is a pending job and get destroyed when the job gets completed. This can theoretically result in a lot of cost saving as the agents don’t have to keep on waiting for a job but rather are created only when necessary.

I can explain a scenario of how using elastic agents can result in cost savings. You can provision a cluster with a single n1-standard-1 VM (3.5GB RAM, 1vCPU) to start with. You can then enable the auto-scaling feature with a reasonable maximum node limit. This will ensure that when no jobs are running all nodes except the first node will be deleted resulting in cost saving compared to have a fixed number of static Go agents.

GoCD offers a number of [plugins](https://www.gocd.org/plugins/#elastic-agents) that support different Cloud providers like AWS or Azure or different orchestration platforms like DockerSwarm or Kubernetes. Considering that we were trying this out on Kubernetes we used the GoCD Elastic Agent plugin for Kubernetes.

# Elastic Agent Profiles

Elastic Agent Profiles allows you to configure elastic agents. This configuration allows you to specify the following

*   Container Image with its tag
*   Max Memory and CPU
*   Security Context
*   Volume Mounts
*   Environment Variables

The configuration can be specified using the UI or by making a _cURL_ request to the API exposed by the Go Server.

# Configuration Using UI

The UI option can be found under the Elastic Profiles section in the Admin menu of the Go Dashboard. You’ll be able to see a cluster profile called _k8-cluster-profile._ You can see the existing elastic agent profiles if you expand the section. You’ll find an option to add an Elastic Agent Profile. If you select it, you’ll be able to see a pop-up window with 3 options on how to specify the Agent Pod configuration. You can either

*   Use the config properties option to add the configuration using the various fields.
*   Use the Pod Yaml option to specify the pod’s configuration in _yaml._
*   Use the Remote file option to specify the pod’s configuration using an external _json_ or _yaml_ file.

# Configuration Using API POST Request

You can also configure an Elastic Agent profile using the API that the Go server exposes. You can simply make a POST call using cURL like the following


```
curl -vi "[http://$gocd_public_ip/go/api/elastic/profiles](http://$gocd_public_ip/go/api/elastic/profiles)" \
           -u "$USERNAME:$PASSWORD" \
           -H 'Accept: application/vnd.go.cd.v2+json' \
           -H 'Content-Type: application/json' \
           -X POST -d '{
        "id": "goagent-1",
        "plugin_id": "cd.go.contrib.elasticagent.kubernetes",
        "cluster_profile_id": "k8-cluster-profile",
        "properties": [
          {
            "key": "PodConfiguration",
            "value": "apiVersion: v1
kind: Pod
metadata:
  name: gocd-agent-{{ POD_POSTFIX }}
  labels:
    app: web
spec:
  serviceAccountName: default
  containers:
    - name: gocd-agent-container-{{ CONTAINER_POSTFIX }}
      image: <image-name-with-version>
      volumeMounts:
      - name: gocd-agent-ssh
        mountPath: \"/home/go/.ssh\"
        readOnly: true
  volumes:
  - name: gocd-agent-ssh
    secret:
      secretName: gocd-agent-ssh"
          },
          {
            "key": "PodSpecType",
            "value": "yaml"
          },
          {
            "key": "Privileged",
            "value": "true"
          }
        ]
      }'
```


As you can see the pod spec has been specified in YAML. You will have to set the variables  _`$gocd_public_ip`_ , _`$USERNAME`_ and _`$PASSWORD` with the appropriate values. The username and password should be of a user that has admin privileges to run the above command. If you want to configure using other options, you can use the appropriate keys in the properties array.

# Git Repositories Access Using Git

If you want the pod to have SSH access to a Git service, you can create a secret with the SSH files and then mount it like shown in the pod spec above. Doing this should give your elastic agents access to your Git repositories.

So there you go. I hope you find it useful. If you have any questions, please feel free to comment and I’ll answer from whatever I’ve learnt so far.