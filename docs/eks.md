# Install Percona Distribution for MySQL on Amazon Elastic Kubernetes Service (EKS)

This guide shows you how to deploy Percona Operator for MySQL on Amazon Elastic Kubernetes Service (EKS). The document assumes some experience with Amazon EKS. For more information on the EKS, see the [Amazon EKS official documentation](https://aws.amazon.com/eks/).

## Prerequisites

The following tools are used in this guide and therefore should be preinstalled:

1. **AWS Command Line Interface (AWS CLI)** for interacting with the different
    parts of AWS. You can install it following the [official installation instructions for your system](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

2. **eksctl** to simplify cluster creation on EKS. It can be installed
    along its [installation notes on GitHub](https://github.com/weaveworks/eksctl#installation).

3. **kubectl**  to manage and deploy applications on Kubernetes. Install
    it [following the official installation instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

Also, you need to configure AWS CLI with your credentials according to the [official guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

## Create the EKS cluster

To create your cluster, you will need the following data:

* name of your EKS cluster,
* AWS region in which you wish to deploy your cluster,
* the amount of nodes you would like tho have,
* the desired ratio between [on-demand](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-on-demand-instances.html)
    and [spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
    instances in the total number of nodes.

!!! note

    [spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
    instances are not recommended for production environment, but may be useful
    e.g. for testing purposes.

The most easy and visually clear way is to describe the desired cluster in YAML
and to pass this configuration to the `eksctl` command.

The following example configures a EKS cluster with one [managed node group](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html):

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
    name: test-cluster
    region: eu-west-2

nodeGroups:
    - name: ng-1
      minSize: 3
      maxSize: 5
      instancesDistribution:
        maxPrice: 0.15
        instanceTypes: ["m5.xlarge", "m5.2xlarge"] # At least two instance types should be specified
        onDemandBaseCapacity: 0
        onDemandPercentageAboveBaseCapacity: 50
        spotInstancePools: 2
      tags:
        'iit-billing-tag': 'cloud'
      preBootstrapCommands:
          - "echo 'OPTIONS=\"--default-ulimit nofile=1048576:1048576\"' >> /etc/sysconfig/docker"
          - "systemctl restart docker"
```

!!! note

    `preBootstrapCommands` section is used in the
    above example to increase the limits for the amount of opened files:
    this is important and shouldn’t be omitted, taking into account the
    default EKS soft limit of 65536 files.

When the cluster configuration file is ready, you can actually create your cluster
by the following command:

```{.bash data-prompt="$"}
$ eksctl create cluster -f ~/cluster.yaml
```

## Install the Operator and deploy your MySQL cluster

1. Create a namespace and set the context for the namespace. The resource names must be unique within the namespace and provide a way to divide cluster resources between users spread across multiple projects.

    So, create the namespace and save it in the namespace context for subsequent commands as follows (replace the `<namespace name>` placeholder with some descriptive name):

    ```{.bash data-prompt="$"}
    $ kubectl create namespace <namespace name>
    $ kubectl config set-context $(kubectl config current-context) --namespace=<namespace name>
    ```

    At success, you will see the message that namespace/<namespace name> was created, and the context was modified.

2. Use the following `git clone` command to download the correct branch of the percona-server-mysql-operator repository:

    ```{.bash data-prompt="$"}
    $ git clone -b v{{ release }} https://github.com/percona/percona-server-mysql-operator
    ```

    After the repository is downloaded, change the directory to run the rest of the commands in this document:

    ```{.bash data-prompt="$"}
    $ cd percona-server-mysql-operator
    ```

3. Deploy the Operator [using](https://kubernetes.io/docs/reference/using-api/server-side-apply/) the following command:

    ```{.bash data-prompt="$"}
    $ kubectl apply --server-side -f deploy/bundle.yaml
    ```

    The following confirmation is returned:

    ```{.text .no-copy}
    customresourcedefinition.apiextensions.k8s.io/perconaserverformysqlbackups.ps.percona.com created
    customresourcedefinition.apiextensions.k8s.io/perconaserverformysqlrestores.ps.percona.com created
    customresourcedefinition.apiextensions.k8s.io/perconaserverformysqls.ps.percona.com created
    serviceaccount/percona-server-for-mysql-operator created
    role.rbac.authorization.k8s.io/percona-server-for-mysql-operator-leader-election-role created
    role.rbac.authorization.k8s.io/percona-server-for-mysql-operator-role created
    rolebinding.rbac.authorization.k8s.io/percona-server-for-mysql-operator-leader-election-rolebinding created
    rolebinding.rbac.authorization.k8s.io/percona-server-for-mysql-operator-rolebinding created
    configmap/percona-server-for-mysql-operator-config created
    deployment.apps/percona-server-for-mysql-operator created
    ```

4. The operator has been started, and you can create the Percona Distribution
    for MySQL cluster:

    ```{.bash data-prompt="$"}
    $ kubectl apply -f deploy/cr.yaml
    ```

    The process could take some time.
    The return statement confirms the creation:

    ```{.text .no-copy}
    perconaserverformysql.ps.percona.com/cluster1 created
    ```

5. During previous steps, the Operator has generated several [secrets](https://kubernetes.io/docs/concepts/configuration/secret/), including the password for the `root` user, which you will need to access the cluster.

    Use `kubectl get secrets` command to see the list of Secrets objects (by default Secrets object you are      interested in has `my-cluster-secrets` name). Then `kubectl get secret cluster1-secrets -o yaml` will return the YAML file with generated secrets, including the root password which should look as follows:

    ```yaml
    ...
    data:
      ...
      root: cm9vdF9wYXNzd29yZA==
    ```

    Here the actual password is base64-encoded, and `echo 'cm9vdF9wYXNzd29yZA==' | base64 --decode` will bring it back to a human-readable form (in this example it will be a `root_password` string).

6. Now you can check wether you are able to connect to MySQL from the outside
    with the help of the `kubectl port-forward` command as follows:

    ```{.bash data-prompt="$"}
    $ kubectl port-forward svc/cluster1-mysql-primary 3306:3306 &
    $ mysql -h 127.0.0.1 -P 3306 -uroot -proot_password
    ```
