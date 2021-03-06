# Setting up EnMasse

## Access permissions

Some permissions need to be granted before setting up the messaging
service.  View permission should be granted to the default
serviceaccount (system:serviceaccount:myproject:default for a project
named myproject). This is needed by the configmap-bridge and the
router agent.

Edit rights must also be granted to the deployer role, used by the
storage-controller.

The permissions can be setup with the following commands:

    oc policy add-role-to-user view system:serviceaccount:$(oc project -q):default
    oc policy add-role-to-user edit system:serviceaccount:$(oc project -q):deployer

## Using the simple template

There is a template that can be used to setup the necessary objects in
openshift. To use it run:

    oc process -f enmasse-template.yaml | oc create -f -

## Using the TLS template

For the messaging service to be externally accessible, a route must be
setup over TLS. This requires qdrouterd containers to have TLS enabled.

To do this we need to create a secret called qdrouterd-certs with the
key and certificate for the routers contained in server-key.pem and
server-cert.pem respectively:

    oc secret new qdrouterd-certs server-cert.pem server-key.pem

Then the default serviceaccount (under which the router containers are
currently run) needs to be allowed to mount this secret:

    oc secret add serviceaccount/default secrets/qdrouterd-certs --for=mount

Then we can use the tls version of the template as follows:

    oc process -f tls-enmasse-template.yaml | oc create -f -

## Manual setup

Alternatively, instead of using the templates, the necessary objects
can be created manually as follows:

### Setting up configuration service

The configmap-bridge requires view permission so that it can read the
config map. See section on access permissions above.

Start the configuration service endpoint and the replication controller:

    oc create -f configuration-service.yaml
    oc create -f configmap-bridge-rc.yaml

The configuration service will watch all configmaps for the project deployed to.

### Creating config map

Create initial config map:

    oc create -f addresses.yaml

Where address.yaml is of the form:

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: maas
data:
  json: |
    {
        "queue1": { "store_and_forward":true, "multicast":false },
        "queue2": { "store_and_forward":true, "multicast":false },
        "anycast": { "store_and_forward":false, "multicast":false },
        "broadcast": { "store_and_forward":false, "multicast":true }
    }
```

After editing that file, update the config map with:

    oc replace -f addresses.yaml

### Setting up the storage controller

In order for the storage controller to work, it must be granted edit rights so that it can 
create replication controllers. See section on access permissions above.

Then start the replication controller:

    oc create -f storage-controller-rc.yaml

Whenever you change the 'maas' config map, the storage-controller will pick up the changes and
delete/create/update broker clusters.

### Setting up the routers

Start the router agent service replication controller:

    oc create -f ragent-service.yaml
    oc create -f ragent-rc.yaml

The router agent(s) update the routers to match the desired addresses
and to maintain full-mesh connectivity.

The router agent requires view permission so that it can determine
other router agent endpoints in the service. See section on access
permissions above.

Start the messaging service and replication-controller:

    oc create -f messaging-service.yaml
    oc create -f qdrouterd-rc.yaml

You can get the service IP either from the web console or by running:

    oc get service messaging

You should now be able to connect to that to send and receive
messages based on the addressing scheme in the config map.

## Benchmarking

EnMasse provides a benchmarking suite, ebench, that can run alongside the EnMasse cluster or
on a separate set of machines. The suite is composed of an agent that sends messages to a specific
address and a collector that aggregates metrics from multiple agents. To start an agent and a
collector:

    oc create -f ebench-agent-rc.yaml
    oc create -f ebench-collector.yaml

The agent and collector is parameterized using environment variables defined in the specification
file. 

You can scale the number of agents by adjusting the number of replicas, and the collector will
automatically pick up the changes and display aggregated results for all agents running.

