---
title: Deploy a self-hosted gateway to Kubernetes
description: Learn how to deploy a self-hosted gateway component of Azure API Management to Kubernetes
author: dlepow
manager: gwallace
ms.service: api-management
ms.workload: mobile
ms.topic: article
ms.author: danlep
ms.date: 05/25/2021
---
# Deploy a self-hosted gateway to Kubernetes

This article describes the steps for deploying the self-hosted gateway component of Azure API Management to a Kubernetes cluster.

> [!NOTE]
> You can also deploy self-hosted gateway to an [Azure Arc enabled Kubernetes cluster](how-to-deploy-self-hosted-gateway-azure-arc.md) as a [cluster extension](../azure-arc/kubernetes/extensions.md).

## Prerequisites

- Complete the following quickstart: [Create an Azure API Management instance](get-started-create-service-instance.md).
- Create a Kubernetes cluster.
   > [!TIP]
   > [Single-node clusters](https://kubernetes.io/docs/setup/#learning-environment) work well for development and evaluation purposes. Use [Kubernetes Certified](https://kubernetes.io/partners/#conformance) multi-node clusters on-premises or in the cloud for production workloads.
- [Provision a self-hosted gateway resource in your API Management instance](api-management-howto-provision-self-hosted-gateway.md).

## Deploy to Kubernetes

1. Select **Gateways** under **Deployment and infrastructure**.
2. Select the self-hosted gateway resource that you want to deploy.
3. Select **Deployment**.
4. An access token in the **Token** text box was auto-generated for you, based on the default **Expiry** and **Secret key** values. If needed, choose values in either or both controls to generate a new token.
5. Select the **Kubernetes** tab under **Deployment scripts**.
6. Select the **\<gateway-name\>.yml** file link and download the YAML file.
7. Select the **copy** icon at the lower-right corner of the **Deploy** text box to save the `kubectl` commands to the clipboard.
8. Paste commands to the terminal (or command) window. The first command creates a Kubernetes secret that contains the access token generated in step 4. The second command applies the configuration file downloaded in step 6 to the Kubernetes cluster and expects the file to be in the current directory.
9. Run the commands to create the necessary Kubernetes objects in the [default namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) and start self-hosted gateway pods from the [container image](https://aka.ms/apim/sputnik/dhub) downloaded from the Microsoft Container Registry.
10. Run the following command to check if the deployment succeeded. Note that it might take a little time for all the objects to be created and for the pods to initialize.

    ```console
    kubectl get deployments
    NAME             READY   UP-TO-DATE   AVAILABLE   AGE
    <gateway-name>   1/1     1            1           18s
    ```
11. Run the following command to check if the service was successfully created. Note that your service IPs and ports will be different.

    ```console
    kubectl get services
    NAME             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    <gateway-name>   LoadBalancer   10.99.236.168   <pending>     80:31620/TCP,443:30456/TCP   9m1s
    ```
1. Go back to the Azure portal and select **Overview**.
1. Confirm that **Status** shows a green check mark, followed by a node count that matches the number of replicas specified in the YAML file. This status means the deployed self-hosted gateway pods are successfully communicating with the API Management service and have a regular "heartbeat."

    ![Gateway status](media/how-to-deploy-self-hosted-gateway-kubernetes/status.png)

> [!TIP]
> Run the `kubectl logs deployment/<gateway-name>` command to view logs from a randomly selected pod if there's more than one.
> Run `kubectl logs -h` for a complete set of command options, such as how to view logs for a specific pod or container.

## Production deployment considerations

### Access token
Without a valid access token, a self-hosted gateway can't access and download configuration data from the endpoint of the associated API Management service. The access token can be valid for a maximum of 30 days. It must be regenerated, and the cluster configured with a fresh token, either manually or via automation before it expires.

When you're automating token refresh, use [this management API operation](/rest/api/apimanagement/2020-12-01/gateway/generate-token) to generate a new token. For information on managing Kubernetes secrets, see the [Kubernetes website](https://kubernetes.io/docs/concepts/configuration/secret).

### Namespace
Kubernetes [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) help with dividing a single cluster among multiple teams, projects, or applications. Namespaces provide a scope for resources and names. They can be associated with a resource quota and access control policies.

The Azure portal provides commands to create self-hosted gateway resources in the **default** namespace. This namespace is automatically created, exists in every cluster, and can't be deleted.
Consider [creating and deploying](https://www.kubernetesbyexample.com/) a self-hosted gateway into a separate namespace in production.

### Number of replicas
The minimum number of replicas suitable for production is two.

By default, a self-hosted gateway is deployed with a **RollingUpdate** deployment [strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy). Review the default values and consider explicitly setting the [maxUnavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable) and [maxSurge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge) fields, especially when you're using a high replica count.

### Container resources
By default, the YAML file provided in the Azure portal doesn't specify container resource requests.

It's impossible to reliably predict and recommend the amount of per-container CPU and memory resources and the number of replicas required for supporting a specific workload. Many factors are at play, such as:

- Specific hardware that the cluster is running on.
- Presence and type of virtualization.
- Number and rate of concurrent client connections.
- Request rate.
- Kind and number of configured policies.
- Payload size and whether payloads are buffered or streamed.
- Backend service latency.

We recommend setting resource requests to two cores and 2 GiB as a starting point. Perform a load test and scale up/out or down/in based on the results.

### Container image tag
The YAML file provided in the Azure portal uses the **latest** tag. This tag always references the most recent version of the self-hosted gateway container image.

Consider using a specific version tag in production to avoid unintentional upgrade to a newer version.

You can [download a full list of available tags](https://mcr.microsoft.com/v2/azure-api-management/gateway/tags/list).

### DNS policy
DNS name resolution plays a critical role in a self-hosted gateway's ability to connect to dependencies in Azure and dispatch API calls to backend services.

The YAML file provided in the Azure portal applies the default [ClusterFirst](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy) policy. This policy causes name resolution requests not resolved by the cluster DNS to be forwarded to the upstream DNS server that's inherited from the node.

To learn about name resolution in Kubernetes, see the [Kubernetes website](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service). Consider customizing [DNS policy](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy) or [DNS configuration](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config) as appropriate for your setup.

### External traffic policy
The YAML file provided in the Azure portal sets `externalTrafficPolicy` field on the [Service](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#service-v1-core) object to `Local`. This preserves caller IP address (accessible in the [request context](api-management-policy-expressions.md#ContextVariables)) and disables cross node load balancing, eliminating network hops caused by it. Be aware, that this setting might cause asymmetric distribution of traffic in deployments with unequal number of gateway pods per node.

### Custom domain names and SSL certificates

If you use custom domain names for the API Management endpoints, especially if you use a custom domain name for the Management endpoint, you might need to update the value of `config.service.endpoint` in the **\<gateway-name\>.yaml** file to replace the default domain name with the custom domain name. Make sure that the Management endpoint can be accessed from the pod of the self-hosted gateway in the Kubernetes cluster.

In this scenario, if the SSL certificate that's used by the Management endpoint isn't signed by a well-known CA certificate, you must make sure that the CA certificate is trusted by the pod of the self-hosted gateway.

### Configuration backup
To learn about self-hosted gateway behavior in the presence of a temporary Azure connectivity outage, see [Self-hosted gateway overview](self-hosted-gateway-overview.md#connectivity-to-azure).

Configure a local storage volume for the self-hosted gateway container, so it can persist a backup copy of the latest downloaded configuration. If connectivity is down, the storage volume can use the backup copy upon restart. The volume mount path must be <code>/apim/config</code>. See an example on [GitHub](https://github.com/Azure/api-management-self-hosted-gateway/blob/master/examples/self-hosted-gateway-with-configuration-backup.yaml).
To learn about storage in Kubernetes, see the [Kubernetes website](https://kubernetes.io/docs/concepts/storage/volumes/).

### Local logs and metrics
The self-hosted gateway sends telemetry to [Azure Monitor](api-management-howto-use-azure-monitor.md) and [Azure Application Insights](api-management-howto-app-insights.md) according to configuration settings in the associated API Management service.
When [connectivity to Azure](self-hosted-gateway-overview.md#connectivity-to-azure) is temporarily lost, the flow of telemetry to Azure is interrupted and the data is lost for the duration of the outage.
Consider [setting up local monitoring](how-to-configure-local-metrics-logs.md) to ensure the ability to observe API traffic and prevent telemetry loss during Azure connectivity outages.

## Next steps

* To learn more about the self-hosted gateway, see [Self-hosted gateway overview](self-hosted-gateway-overview.md).
* Learn [how to deploy API Management self-hosted gateway to Azure Arc enabled Kubernetes clusters](how-to-deploy-self-hosted-gateway-azure-arc.md).
