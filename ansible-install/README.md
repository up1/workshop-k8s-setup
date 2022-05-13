## Deploy kubernetes cluster with [Ansible](https://docs.ansible.com/)

### Step 1 :: Create 3 VM instances on Google Cloud Compute Engine
* kube_server for masters = 1 VM
* kube_agents for nodes/worker = 2 VM

โดยกำหนด ip ของแต่ละ node ไว้ในไฟล์ ansible-hosts.txt

```
[all]
34.143.254.158
34.142.197.246
34.143.227.170

[kube_server]
34.143.254.158

[kube_agents]
34.142.197.246
34.143.227.170
```

### Step 2 :: ตรวจสอบว่าสามารถเข้าใช้งานทุก ๆ node ผ่าน ssh ได้หรือไม่
```
$ansible -i ansible-hosts.txt all -u somkiat -m ping

34.142.197.246 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
34.143.254.158 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
34.143.227.170 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

ถ้าไม่ได้ให้ทำการนำ public key ของแต่ละคน
ไปไว้ที่ไฟล์ ~/.ssh/authorized_keys ในแต่ละ node

```
$sudo vi ~/.ssh/authorized_keys
```

แต่ถ้าต้องการให้อยู่แบบถาวรใน Google Cloud ให้ทำการเพิ่มใน metadata ของ แต่ละ node ประกอบไปด้วยค่าดังนี้
* key = username
* value = public key ของ user


### Step 3 ::  ติดตั้ง kubernetes dependendcies
กำหนดในไฟล์ ansible-install-kubernetes-dependencies.yml

```
$ansible-playbook -i ansible-hosts.txt ansible-install-kubernetes-dependencies.yml


PLAY [all] *******************************************************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [34.143.254.158]
ok: [34.142.197.246]
ok: [34.143.227.170]

TASK [Install packages that allow apt to be used over HTTPS] *****************************************

```

### Step 4 ::  สร้าง kubernetes cluster บน master node
```
$ansible-playbook -i ansible-hosts.txt ansible-init-cluster.yml
```

### Step 5 ::  ทำการ join worker nodes เข้าไปยัง cluster

Get join worker command
```
$ansible-playbook -i ansible-hosts.txt ansible-get-join-command.yml
```

ทำการ join เข้าไปยัง cluster
```
$ansible-playbook -i ansible-hosts.txt ansible-join-workers.yml
```

### Step 6 :: ทำการตรวจสอบ kubernetes cluster
โดยเข้าไปยัง master node
```
$kubectl get nodes

NAME         STATUS   ROLES                  AGE     VERSION
somkiat-01   Ready    control-plane,master   8m40s   v1.23.6
somkiat-02   Ready    <none>                 52s     v1.23.6
somkiat-03   Ready    <none>                 51s     v1.23.6
```

### Step 7 :: Access ไปยัง kubernetes dashboard
โดยเข้าไปยัง master node

สร้างไฟล์  sa.yml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

สร้างไฟล์ clusterrole.yml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

ทำการ run คำสั่งต่อไปนี้
```
$kubectl apply -f sa.yml
$kubectl apply -f clusterrole.yml

$kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml

$kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
```

ให้ทำการ copy token ไว้ใช้งานต่อไป !!

เข้าใช้งานด้วยคำสั่ง
```
$ssh -L 8001:127.0.0.1:8001 somkiat@34.143.254.158
$kubectl proxy
```

จากนั้นเข้าใช้งานผ่าน browser
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

`Ready to Kubernetes workshop ...`

