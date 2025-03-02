<!-- GNU Affero General Public License v3.0 or later (see COPYING or https://www.gnu.org/licenses/agpl.txt) -->

## Check Nodes NOTE: THIS DOES NOT WORK YET
```bash
oc get nodes | grep --extended-regexp --invert-match 'Ready'
```
Example output:
```text
NAME         STATUS     ROLES           AGE   VERSION
172.1.10.4   NotReady   master,worker   20d   v1.26.15+4818370
```

## Check for failed upgrades:
```bash
oc get clusterversions.config.openshift.io --output json | jq '(.items[].status.history[] | select(.state != "Completed"))'
```
Example Output:
```text
{
  "completionTime": "2024-08-07T14:53:04Z",
  "image": "quay.io/openshift-release-dev/ocp-release@sha256:24ea553ce2e79fab0ff9cf2917d26433cffb3da954583921926034b9d5d309bd",
  "startedTime": "2024-08-07T14:19:54Z",
  "state": "Partial",
  "verified": true,
  "version": "4.16.6"
}
```

## Get all cluster operators that are not on the desired version:
```bash
oc get clusteroperators | awk -v desired_version="$(oc get clusterversion --output json | jq -r '.items[].status.desired.version')" 'NR == 1 { print; first_line = 1 } NR > 1 && $2 < desired_version { print }'
```
Example Output:
```text
NAME      VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
dns       4.12.40   True        False         False      250d
network   4.12.40   True        False         False      250d
```

## Find all pods with errors:
```bash
oc get pods --all-namespaces | grep --extended-regexp --invert-match 'Running|Complete'
```
Example Output:
```text
NAMESPACE                           NAME                                READY   STATUS             RESTARTS         AGE
openshift-kube-apiserver            installer-9-tcloud-2024-07-24.lan   0/1     Error              0                5d3h
openshift-kube-controller-manager   installer-9-tcloud-2024-07-24.lan   0/1     Error              0                5d3h
openshift-monitoring                alertmanager-main-0                 5/6     CrashLoopBackOff   181 (36s ago)    19d
strawberry                          httpd-example-5855477686-592mb      0/1     CrashLoopBackOff   55 (3m30s ago)   178m
strawberry                          httpd-example-69b9dff887-vvlhc      0/1     Pending            0                177m
```

## Find all pods with a restartCount > 4:
```bash
oc get pods --all-namespaces | awk '$5 > 4'
```
Example Output:
```text
NAMESPACE              NAME                             READY   STATUS             RESTARTS          AGE
openshift-monitoring   alertmanager-main-0              5/6     CrashLoopBackOff   217 (2m13s ago)   19d
openshift-monitoring   thanos-querier-9984c799c-5bs7z   6/6     Running            61                19d
strawberry             httpd-example-5855477686-592mb   0/1     CrashLoopBackOff   109 (4m58s ago)   6h5m
```






Check for Events in OpenShift Namespaces
oc get events --sort-by='{.lastTimestamp}' | grep -Ev ‘Normal’
Check Resource Usages
oc adm top pod
oc adm top node
Check Alerts on the Clusters
https://access.redhat.com/solutions/4250221
oc rsh -n openshift-monitoring alertmanager-main-0 amtool alert query --alertmanager.url http://localhost:9093
oc get podnetworkconnectivitychecks -n openshift-network-diagnostics

Check resource usage overall > n-1 of CPU or Memory
Check for any pod that has restarts greater than X(5?)
Check for any probe failures
Check uptime on nodes (anything greater than 90 days is a red flag)


## Get all api resources
```
oc get --raw="/"
```


## Fix expired certificates
1. Ensure `kubeconfig-noingress` is copied to the node.
```
rsync kubeconfig-noingress sno:~/
```

2. Setup `KUBECONFIG`
```
$ ssh sno
$ export KUBECONFIG=kubeconfig-noingress
```

3. Approve all pending csrs.
```
$ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```

4. Wait ~5-10 minutes for the node to finish booting. You can monitor the starting pods with the following:
```
$ sudo crictl ps | wc -l
```