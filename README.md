# Rucio for Project 8
## NOTE: This is an example for the purpose of demonstrating Rucio's ease of installation and exploring its capabilties. Thus, no assumptions about the security of this prototype should be made, for now.

# Set up

* Clone this repository to your local machine (Unix Based OS preferred).

* Install kubectl [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

* Install helm [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

* Install minikube [https://kubernetes.io/docs/tasks/tools/install-minikube/](https://kubernetes.io/docs/tasks/tools/install-minikube/)

* Start minikube and set the RAM:

      minikube start --memory='4096mb'

* Add Helm chart repositories:

      helm repo add stable https://charts.helm.sh/stable
      helm repo add bitnami https://charts.bitnami.com/bitnami
      helm repo add rucio https://rucio.github.io/helm-charts


# Installation of Rucio + FTS + Storage

_NOTE: Change directory to the cloned repo location._

_NOTE: In some of the commands below, you will have to replace the pod IDs with the ones from your instance using the ``kubectl get pods`` command._


* Install the Rucio database.

      helm install postgres bitnami/postgresql -f database.yaml

* Wait for the previous command to finish starting. Output of ``kubectl get pods`` should be STATUS:Running.

* Run the init container once the database container is running:

      kubectl apply -f init.yaml

* Watch the output of the init container to check if everything is fine. Pod should finish with STATUS:Completed

      kubectl logs -f init

* Install the Rucio server and wait for it to come online:

      helm install server rucio/rucio-server -f server.yaml
      
      kubectl logs -f server-rucio-server-[replace-id]-[replace-id] rucio-server

* Prepare a client container for interactive use:

      kubectl apply -f client.yaml

* Once the client container is in STATUS:Running, jump into it and check if the clients are working:

      kubectl exec -it client /bin/bash

      rucio whoami

* Install and start three XRootD storage system instances.

      kubectl apply -f xrd.yaml

* Install the FTS database and wait for it to come online.

      kubectl apply -f ftsdb.yaml
      
      kubectl logs -f fts-mysql-[replace-id]-[replace-id]

* Install the FTS once the FTS database container is up and running:

      kubectl apply -f fts.yaml
      
      kubectl logs -f fts-server-[replace-id]-[replace-id]

* Install the Rucio daemons:

      helm install daemons rucio/rucio-daemons -f daemons.yaml

* Run FTS storage authentication delegation once:

      kubectl create job renew-manual-1 --from=cronjob/daemons-renew-fts-proxy

## Rucio usage

* Jump into the client container

      kubectl exec -it client /bin/bash

* Create the Rucio Storage Elements (RSEs)

      rucio-admin rse add XRD1
      rucio-admin rse add XRD2
      rucio-admin rse add XRD3

* Add the protocol definitions for the storage servers

      rucio-admin rse add-protocol --hostname xrd1 --scheme root --prefix //rucio --port 1094 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD1
      rucio-admin rse add-protocol --hostname xrd2 --scheme root --prefix //rucio --port 1094 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD2
      rucio-admin rse add-protocol --hostname xrd3 --scheme root --prefix //rucio --port 1094 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD3

* Enable FTS

      rucio-admin rse set-attribute --rse XRD1 --key fts --value https://fts:8446
      rucio-admin rse set-attribute --rse XRD2 --key fts --value https://fts:8446
      rucio-admin rse set-attribute --rse XRD3 --key fts --value https://fts:8446

* Set a full mesh network

      rucio-admin rse add-distance --distance 1 --ranking 1 XRD1 XRD2
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD1 XRD3
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD2 XRD1
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD2 XRD3
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD3 XRD1
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD3 XRD2

* Set the RSE quota for root (-1 = unlimited)

      rucio-admin account set-limits root XRD1 -1
      rucio-admin account set-limits root XRD2 -1
      rucio-admin account set-limits root XRD3 -1

* Create a scope for testing

      rucio-admin scope add --account root --scope test

* Create testing data files

      dd if=/dev/urandom of=file1 bs=10M count=1
      dd if=/dev/urandom of=file2 bs=10M count=1
      dd if=/dev/urandom of=file3 bs=10M count=1
      dd if=/dev/urandom of=file4 bs=10M count=1

* Upload the files to the RSEs

      rucio upload --rse XRD1 --scope test file1
      rucio upload --rse XRD1 --scope test file2
      rucio upload --rse XRD2 --scope test file3
      rucio upload --rse XRD2 --scope test file4

For information about Rucio, visit [http://rucio.cern.ch/documentation/](http://rucio.cern.ch/documentation/).