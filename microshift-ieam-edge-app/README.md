# Deploy a sample edge application on Microshift cluster and manage it with IBM Edge Application Manager (IEAM).

The step-by-step guidance for preparing the IBM edge device environment and deploying a sample edge application on Microshift managed by IBM Edge Application Manager 
(IEAM).

## Navigation

- [Prerequisites](#prerequisites)
- [Configure image registry](#optional---deploy-image-registry)

## Prerequisites

1. Have Microshift deployed on the edge device either via [microshift-rhacm](https://github.com/gargpriyank/telco-edge/tree/main/microshift-rhacm) or 
 [microshift-raspberry-pi](https://github.com/gargpriyank/telco-edge/tree/main/microshift-raspberry-pi).
2. Have [IEAM Management Hub](https://www.ibm.com/docs/en/eam/4.4?topic=installation-install-ieam) installed on the managed from Red Hat OpenShift cluster. 
   1. Login to the edge device as root user. Use `oc` command to login to the Red Hat OpenShift cluster where IEAM is deployed. Ensure all pods in "ibm-common-services" 
      namespace are either Running or Completed
      ```markdown
      oc get pods -n ibm-common-services
      ```
   2. Log in, pull and extract the agent bundle with your entitlement key through the [Entitled Registry](https://myibm.ibm.com/products-services/containerlibrary)
      ```markdown
      podman login cp.icr.io --username cp && \
      podman rm -f ibm-eam-4.4.0-bundle; \
      podman pull cp.icr.io/cp/ieam/ibm-eam-bundle:2.29.0-638 && \
      podman create --name ibm-eam-4.4.0-bundle cp.icr.io/cp/ieam/ibm-eam-bundle:2.29.0-638 bash && \
      podman cp ibm-eam-4.4.0-bundle:/ibm-eam-4.4.0-bundle.tar.gz ibm-eam-4.4.0-bundle.tar.gz && \
      tar -zxvf ibm-eam-4.4.0-bundle.tar.gz && \
      cd ~/ibm-eam-4.4.0-bundle/tools
      ```
   3. Validate the installation state
      ```markdown
      ./service_healthcheck.sh
      Output:
      ==Running service verification tests for IBM Edge Application Manager==
      SUCCESS: IBM Edge Application Manager Exchange API is operational
      SUCCESS: IBM Edge Application Manager Cloud Sync Service is operational
      SUCCESS: IBM Edge Application Manager Agbot database heartbeat is current
      SUCCESS: IBM Edge Application Manager SDO API is operational
      SUCCESS: IBM Edge Application Manager UI is properly requiring valid authentication
      ==All expected services are up and running==
      ```
   4. Install the hzn CLI. Use "ppc64le" or "x86_64" directory depending on the hardware architecture
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle/agent && \
      tar -zxvf edge-packages-4.4.0.tar.gz
      
      rpm-ostree install make
      rpm-ostree install ./edge-packages-4.4.0/linux/rpm/x86_64/horizon-cli-*.x86_64.rpm
      systemctl reboot
      ```
   5. Replace "docker" with "podman". Create first "Org" in IEAM for multi-tenancy
      ```markdown
      cd ~/ibm-eam-4.4.0-bundle/tools
      sed -i 's/docker/podman/gi' post_install.sh
      ./post_install.sh <choose-your-org-name>
      ```
   6. Set IEAM cluster URL, user credentials and `podman` authentication environment variables
      ```markdown
      export CLUSTER_URL=https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}' -n ibm-common-services)
      export CLUSTER_USERNAME=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_username}' | base64 --decode)
      export CLUSTER_USERPASS=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 --decode)
      export REGISTRY_USERNAME=cp
      export REGISTRY_PASSWORD=<ENTITLEMENT_KEY>
      ```
   7. Download and install [cloudctl](https://www.ibm.com/docs/en/eam/4.4?topic=cli-installing-cloudctl-kubectl-oc) binary. Use `cloudctl` to login to IEAM management hub
      ```markdown
      cloudctl login -a $CLUSTER_URL -u $CLUSTER_USERNAME -p $CLUSTER_USERPASS --skip-ssl-validation
      ```
   8. Create API key. Find the key value in the output and save it for future use
      ```markdown
      cloudctl iam api-key-create "<choose-an-api-key-name>" -d "<choose-an-api-key-description>"
      ```
   9. Set the Horizon environment variables
      ```markdown
      export HZN_EXCHANGE_USER_AUTH=iamapikey:<iam-api-key>
      export HZN_ORG_ID=<org-id>
      export HZN_EXCHANGE_URL=$CLUSTER_URL/edge-exchange/v1
      export HZN_FSS_CSSURL=$CLUSTER_URL/edge-css/
      ```
   10. Generate the files for edge device installation
       ```markdown
       cd ~/ibm-eam-4.4.0-bundle/agent
       cp /usr/bin/edgeNodeFiles.sh edgeNodeFiles.sh
       sed -i 's/docker/podman/g' ./edgeNodeFiles.sh
       sed -i 's/podman.io/docker.io/g' ./edgeNodeFiles.sh
       HZN_EXCHANGE_USER_AUTH='' ./edgeNodeFiles.sh ALL -c -p edge-packages-4.4.0 -r cp.icr.io/cp/ieam
       ```
## Deploy image registry

1. Set a valid host name
   ```markdown
   hostnamectl set-hostname <your-new-hostname-with-2-dots>
   ```
2. Replace "kubeconfig" to connect back to microshift cluster and check the storage class name. It should be `kubevirt-hostpath-provisioner`
   ```markdown
   cat /var/lib/microshift/resources/kubeadmin/kubeconfig > ~/.kube/config
   oc get storageclasses
   ```
3. Use `image-registry.yaml` provided with this repo to deploy image registry
   ```markdown
   oc apply -f image-registry.yaml
   ```
4. Verify the image registry is deployed
   ```markdown
   oc get pods -n image-registry
   ```
## Install IEAM agent

1. Download `agent-install.sh`   
   ```markdown
   curl -u "$HZN_ORG_ID/$HZN_EXCHANGE_USER_AUTH" -k -o agent-install.sh $HZN_FSS_CSSURL/api/v1/objects/IBM/agent_files/agent-install.sh/data
   chmod +x agent-install.sh
   ```
2. Install edge agent on the microshift cluster
   ```markdown
   sudo -s -E ./agent-install.sh -i 'css:' -p IBM/pattern-ibm.helloworld -w '*' -T 120
   ```
3. 