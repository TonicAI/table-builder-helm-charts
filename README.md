# table-builder-helm-charts
This repository contains a Helm chart which can be used to install Tonic Table Builder via `helm install`.

Project structure:
```
.
├── templates
      └── <All template files>
├── values.sample.yaml
└── README.md
```

## Configuration

### values.yaml
Before deploying this setup, rename [values.sample.yaml](values.sample.yaml) to `values.yaml` and configure the following values.
### Environment name

- `environmentName`: E.g. "my-company-name", or if deploying multiple Table Builder instances, "my-company-name-dev" or "my-company-name-prod to differentiate instances.

### Application database
The connection details for the Postgres metadata/application database which holds Table Builder's state (user accounts, workspaces, etc.).

``` yaml
tonicdb:
  host:
  port:
  dbName:
  user:
  password:
  sslMode:
```

### Log collection
Tonic never collects your sensitive data. Enabling this option securely and safely shares logs with Tonic's engineering team. We recommend that you enable this option. See: https://docs.tonic.ai/app/admin/sharing-logs-with-tonic

- `enableLogCollection`: "false" (default) or "true"

### Authorization to access Tonic application Docker images
Tonic hosts our application images on a private [quay.io](https://quay.io) repository. Authorization is required to pull the images.

- `dockerConfigAuth`: This value will be provided to you by Tonic and will allow you to authenticate against our private docker image repository.


### Version
You can set this to a specific Tonic Table Builder version number if you wish to ensure you always get the same version. Otherwise you will always deploy the latest version of Tonic Table Builder.

- `tableBuilderVersion`: "latest" or a specific version tag. Tonic's tag convention is just the release number, e.g. "123". Release notes are available at [doc.tonic.ai](https://docs.tonic.ai/app/resources/release-notes).

### Ingress
The Helm charts include default annotations for internal-facing load balancers for AWS and Azure. You can change to your preferred ingress method by modifying [tonic-web-server-service.yaml](tonic-web-server-service.yaml).

### Resource requests and limits
Each of the deployment YAML template files contains resource requests and limits. In some cases these may need to be modified for your environment.

## Deploy
To install Table Builder, execute the following commands.

Create a namespace: `kubectl create namespace <namespace_name>`
``` shell
$ kubectl create namespace my-table-builder-namespace
namespace/my-table-builder-namespace created
```

Deploy Table Builder: `helm install <name_of_release> -n <namespace_name> <path-to-helm-chart>`
``` shell
$ helm install my-table-builder-release -n my-table-builder-namespace .
NAME: my-table-builder-release
LAST DEPLOYED: Fri Jun 10 11:31:31 2022
NAMESPACE: my-table-builder-namespace
STATUS: deployed
REVISION: 1
```


## Validate the deployment

Use `kubectl get all -n <namespace_name>` to check that the Tonic Table Builder pods are running:

The deployment may take a few minutes with pods in the `ContainerCreating` status. Re-run the command to get an updated status. Once all pods have a status of `Running` and deployments show `READY` as `1/1`, Table Builder should be available shortly after via browser at the URL/IP listed in the `EXTERNAL-IP` field next to the load balancer service. If you have modified the Helm chart ingress configuration, then this will vary. While not required, it's recommended to set up a more user-friendly domain routing to the Table Builder web application.

``` shell
❯ kubectl get all -n my-table-builder-namespace
NAME                                       READY   STATUS              RESTARTS   AGE
pod/tonic-table-builder-556c8766b-npw49    0/1     ContainerCreating   0          3s

NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/tonic-table-builder   LoadBalancer   10.100.130.32   <pending>     443:32530/TCP   3s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/tonic-table-builder   1/1     1            1           28h

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/tonic-table-builder-556c8766b    1         1         1       5m20s
```

You can validate that Table Builder has fully started up and is in a healthy state by running `kubectl logs deployment/tonic-table-builder -n <namespace_name> | grep listening` and checking for the following output.

``` shell
❯ kubectl logs deployment/tonic-table-builder -n my-table-builder-namespace | grep listening
[2022-06-10T16:48:29+00:00 INF Microsoft.Hosting.Lifetime] Now listening on: http://0.0.0.0:80
[2022-06-10T16:48:29+00:00 INF Microsoft.Hosting.Lifetime] Now listening on: https://0.0.0.0:443
```

### If the Table Builder UI does not load
1. Check that Table Builder is successfully connecting to the application database.
Run `kubectl logs deployment/tonic-table-builder -n <namespace_name> | grep "Failed to connect"`. If you see a `Failed to connect to db during startup.  Retrying in 5 seconds...` message like below, Table Builder is not able to connect to your Postgres application database. Please verify the network path between Table Builder and the database as well as the connection parameters.

``` shell
❯ kubectl logs deployment/tonic-table-builder -n my-table-builder-namespace | grep "Failed to connect"
[2022-06-10T16:50:40+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-06-10T16:50:45+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-06-10T16:50:50+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
[2022-06-10T16:50:55+00:00 WRN ] Failed to connect to db during startup.  Retrying in 5 seconds...
```

2. If Table Builder appears to be running and in a healthy state but you are unable to load the UI, verify that Table Builder is reachable. Common issues may be a requirement to be on a VPN , firewall rules preventing access, or another issue with the ingress configuration used to expose your cluster to external user traffic.