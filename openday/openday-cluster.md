# Changes in ansible book

## Adding nodes ips in hosts.ini

```yaml
[master]
192.168.17.11 

[node]
192.168.17.12
192.168.17.13
```

## Group vars (debian user and k8s version)

```yaml
k3s_version: v1.26.3+k3s1 # changed from v1.22.3+k3s1
ansible_user: pi # changed from debian
```

## Fixing cgroup error

After running the playbook, k3s service was failing to start, to debug:
```bash
ssh pi@raspberrypi-1
journalctl -xe
```
Error was:  
```bash
Apr 13 11:36:38 raspberrypi-1 k3s[3158]: time="2023-04-13T11:36:38+01:00" level=fatal msg="failed to find memory cgroup (v2)"
```

Fix was found here: https://github.com/k3s-io/k3s-ansible/pull/185 
Explanation: cgroup enabling step was already in the playbook but it was not executed due to a failure to parse raspbian version 11 (bullseye)

changes were made according to the suggested PR in [main.yml](roles/raspberrypi/tasks/main.yml) and the file [Raspbian-11.yml](roles/raspberrypi/tasks/prereq/Raspbian-11.yml) was also added.

# Post runbook steps

## Getting the kubeconfig 

```bash
scp pi@rasoberrypi-1:~/.kube/config ~/.kube/config
```

## Installing the k8s dashboard

Following this [guide](https://docs.k3s.io/installation/kube-dashboard):

```bash
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```

Admin user was defined under [dashboard.admin-user.yml](dashboard-user/dashboard.admin-user.yml) and the corresponding role in [dasboard.admin-user-role.yml](dashboard-user/dashboard.admin-user.yml)

```bash
# creating the user/role

$ kubectl create -f dashboard-user/

serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

For easier access, the dashboard service was change from `ClusterIP` to `NodePort`. 
```bash
kubectl --namespace kubernetes-dashboard patch svc kubernetes-dashboard -p '{"spec": {"type": "NodePort"}}' 
```

## Accessing the dashboard
1. Connect to the PiCluster Wifi network (for password: see the sticker on the router)
2. Ensure you have access to the cluster via kubectl 
3. Take note of the port via the command:
```bash
$ kubectl -n kubernetes-dashboard get svc kubernetes-dashboard -o wide 

kubernetes-dashboard   NodePort   10.43.104.241   <none>        443:31105/TCP   37m   k8s-app=kubernetes-dashboard
```

4. Using the port you noted from step 3, the dashboard would be accessible throught `https://raspberrypi-1:31105/` 

5. To login you will need a dashboard admin user token, you can get it by running:

```bash
kubectl -n kubernetes-dashboard create token admin-user 
```
