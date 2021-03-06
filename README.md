## API Development with API Connect and OpenShift (***DRAFT***)


Prerequisites:
* Docker 1.12+
* OpenShift 1.5+
* DataPower Docker 7.6 Image (includes non-root support)
* APIC OVA

### Overview

In this tutorial, you will develop and publish an API with  API Connect and DataPower running on OpenShift. This tutorial assumes prior knowledge with Docker and Kubernetes. If you do not have prior experience with Kubernetes concepts, I recommend reading the [Getting Started with DataPower in Kubernetes](https://developer.ibm.com/datapower/2017/02/27/getting-started-datapower-kubernetes/) tutorial first.

By the end of the guide, you will perform the following:


1. **Deploy DataPower gateway on OpenShift**
2. **Deploy and configure API Connect to use the DataPower container**
3. **Create an API using IBM API Connect**


### Setup

The overall flow will look as follows:


### 1. Deploy DataPower gateway on OpenShift

In this tutorial I am running an Ubuntu 16.04 VM with Docker 1.12 and OpenShift 1.5 (running as a docker image). I've also made sure that this machine has enough memory and compute resources to deploy my containers.

There are a couple options available when deploying a DataPower container to achieve the ultimate goal of availability and reproducibility so that a DataPower Pod outage does not result in manual steps to re-deploy and re-join the gateway to the API management server or in major loss of state. I will be using `hostPath` volumes (TODO: and data containers) for simplicity and demonstration but you may and should use other types as appropriate for the deployment. The volumes will store the DataPower configuration in the `config:` directory to persist the connection information to the API management server as well as other artifacts and crypto-material in the `local:` and `sharedcerts:` directories.

#### A) Using hostPath volumes.

In this scenario I am using hostPath volumes for simplicity and illustration purposes; however, you will need to relax the OpenShift security context constraints (SCC) so that Pods are allowed to use the hostPath volume plug-in without granting everyone access to the privileged SCC. To do this, first switch to the administrative user, the default command is:

    $  oc login -u system:admin

Next, edit the **restricted** SCC:

    $ oc edit scc restricted

then change `alloHostDirVolumePlugin:` to `true` and save. Now you should be able to specify hostPath directories as volumes for your containers.

Note:
a) Directories created on the underlying host using the hostPath volume plugin can only be written to by root or by modifying the file permissions on the host to be able to write to a hostPath volume.
b) On some systems, there are also SELinux considerations to get various Volume Plug-ins to function properly such as attaching the right volume label to the directory [1]

Now that you've enabled hostPath volumes, you are ready to deploy the DataPower Pod. To do this, simply change directory to the GitHub project directory and edit the Deployment config file so that the volume paths reflect your local workspace. For example, my Deployment config file, reflecting my home directory of `/home/jpmatamo/` is as follows:

    $ cat kubernetes/deployments/datapower-deployment.yaml
    ...
    volumes:
    - name: config-volume
      hostPath:
        path: /home/jpmatamo/datapower-apic-openshift-demo/datapower/config
    - name: local-volume
      hostPath:
        path: /home/jpmatamo/datapower-apic-openshift-demo/datapower/local
    - name: usrcerts-volume
      hostPath:
        path: /home/jpmatamo/datapower-apic-openshift-demo/datapower/usrcerts
        $ oc create -f kubernetes/deployments/datapower-deployment.yaml

Once you have adapted the config file for your own environment, simply type:

    $ oc create -f kubernetes/deployments/datapower-deployment.yaml
    deployment "datapower" created

To make sure the deployment was created succesfully, you can issue:

    $ oc get deployment
    NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    datapower   1         1         1            1           46m


This will create the datapower deployment which initializes volumes and exposes necessary ports for test and development.
Note: non-root users cannot bind to low ports so make sure you are using ports higher than 1024 or grant capabilities to bind to lower ports.

Next, you will want to make sure to publish some of the ports publicly so that they can be accessed outside of the cluster. To do this, simply create the service as follows:

    $ oc create -f kubernetes/services/datapower-service.yaml
    service "datapower" created

To make sure the service is up, you can issue the following:

    $ oc get service
    NAME        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                                   AGE
    datapower   172.30.61.66   <nodes>       9090:30087/TCP,5550:32643/TCP,5554:31320/TCP,8443:32612/TCP,443:30416/TCP,8000:30360/TCP,8001:32142/TCP   6m


Notice how OpenShift will map the ports that were exposed by the Pod. For example, note how in the example above you will need to reach out to port 30087 on the OpenShift node in order to connect to the DataPower web-management port 9090. Make sure to use the mapped ports from now now when attempting to create connections from outside the cluster such as when adding the DataPower gateway container to the APIC Cloud Manager.
Also note how your firewall settings on your host may restrict certain inbound or outbound traffic so make sure you relax your firewall rules accordingly.

Now that both the `datapower` Deployment and Service are up, it is time to add the container to the cluster.

### 2. Deploy and configure API Connect to use the DataPower container

As always, head to the Services tab of the APIC Cloud Manager and add a DataPower Service. A modal window will ask you for the `Address` of the Datapower Service. For this guide this is the address of the single node OpenShift cluster (not the private IP assigned to the DataPower Pod!).
Next, it will ask you for a port to set as DataPower config. As before, note that the container is running as non-root so you cannot bind to low ports, use port `8443` since this is the port that has been exposed in the Deployment and Service files included in this tutorial for that purpose but you can edit those files and redeploy to use your own values.
Finally, select `External or no load balancer` and click `Save`.
Next, click on `Add Server` on the CMC and provide the public address of the DataPower service as made available by OpenShift, since I have a single node cluster and the Service ports are of type `NodePort` the address is the same as my OpenShift node address. Under the `Port` section, provide the public port that corresponds to the xml-mgmt interface, as seen in the previous section I chose to use port `5550` in DataPower which was mapped to port `32643` by OpenShift, I therefore enter port `32643` in the `Port` field.
Finally, provide the `Username` and `Password` and enter `0.0.0.0` for the `Network Interface` and click `Create`.











## References:

[1] https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/getting_started_with_kubernetes/get_started_provisioning_storage_in_kubernetes
