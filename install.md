<!-- GNU Affero General Public License v3.0 or later (see COPYING or https://www.gnu.org/licenses/agpl.txt) -->

# OCP 4.17 SNO install document
#### Last Updated: 2024-11-04

## Configure DNS on UniFi
```text
TODO: DOCUMENT PROCESS
https://192.168.1.1/network/default/settings/routing/dns

A Records:
*.apps.vulcan.internal
    192.168.1.88
api.vulcan.internal
    192.168.1.88
```

## Configure System Certificate
Linux
```bash
$ oc extract configmap/kube-root-ca.crt -n openshift-config --to=.

$ mv ca.crt /workspaces/openshift/.devcontainer/files/api-vulcan-internal.crt
```
*Note: This supresses `WARNING: Using insecure TLS client config.` when using `oc login`.*

macOS
```zsh
$ openssl s_client -connect console-openshift-console.apps.vulcan.internal:443 -showcerts > ~/Downloads/web-vulcan-internal.pem

$ sudo security add-trusted-cert \
  -d \
  -r trustAsRoot \
  -k /Library/Keychains/System.keychain \
  ~/Downloads/web-vulcan-internal.pem
```
*Note: macOS requires you to enable local network access to view the web console.*

## Configure OAuth (script)
1. sets up htpasswd authentication
2. creates the htpasswd file
3. creates the htpasswd secret
4. outputs command to watch auth pods for restart
5. adds user `admin` as `cluster-admin`
```bash
$ cd /workspaces/openshift/oauth/
$ ./oauth-all-in-one.bash
```

## Configure LVM Storage Operator
```bash
$ cd /workspaces/openshift/lvm-storage-operator/
$ ./0-lvm-storage-setup.bash
```

## Enable default storage class for OpenShift Container Platform and Virtualization
```bash
$ oc patch storageclass $(oc get storageclass -o jsonpath="{.items[].metadata.name}") -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

$ oc patch storageclass $(oc get storageclass -o jsonpath="{.items[].metadata.name}") -p '{"metadata": {"annotations":{"storageclass.kubevirt.io/is-default-virt-class":"true"}}}'
```

## Enable the internal registry
```bash
$ oc apply -f /workspaces/openshift/registry/image-registry-pvc.yaml

$ oc patch configs.imageregistry.operator.openshift.io/cluster --type=json --patch='[
  {"op": "replace", "path": "/spec/managementState", "value": "Managed"},
  {"op": "replace", "path": "/spec/rolloutStrategy", "value": "Recreate"},
  {"op": "replace", "path": "/spec/replicas", "value": 1},
  {"op": "replace", "path": "/spec/storage", "value": {"pvc":{"claim": "image-registry-pvc"}}}
]'
```

## Create route for internal registry
```bash
$ oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

## Get the route
```bash
$ OCP_INT_REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

## Get the certificate of the ingress operator for the internal registry
```bash
$ oc get secret -n openshift-ingress router-certs-default -o go-template='{{index .data "tls.crt"}}' \
| base64 -d \
| tee /workspaces/openshift/.devcontainer/files/${OCP_INT_REGISTRY}.crt > /dev/null
```

## Logout of kubeadmin
```bash
$ oc logout && oc login --user=admin --server=https://api.vulcan.internal:6443 --web
```

## Configure access to internal registry
Read Only
```bash
$ oc policy add-role-to-user registry-viewer $(oc whoami)
```
Read Write
```bash
$ oc policy add-role-to-user registry-editor $(oc whoami)
```

## login to the internal registry using the default route
```bash
$ podman login -u $(oc whoami) -p $(oc whoami -t) "${OCP_INT_REGISTRY}"
```






## OpenShift Virtualization goes here
```bash
$ cd /workspaces/openshift/openshift-cnv-operator/
$ ./0-openshift-cnv-setup.bash
```








## Create a project for the application
```bash
$ oc new-project "${PROJECT_NAME}"
```

## Create an empty image stream in your project for the image using oc create imagestream.
```bash
$ oc create imagestream "${PROJECT_NAME}"
```

## Tag the local image to push with the details of the image registry, your project in OpenShift, the name of the image stream and image version tag.
```bash
$ podman tag loclahost/image "${OCP_INT_REGISTRY}/${PROJECT_NAME}/${IMAGE}:latest"
```

## Push the image to the internal registry
```bash
$ podman push "${OCP_INT_REGISTRY}/${PROJECT_NAME}/${IMAGE}"
```