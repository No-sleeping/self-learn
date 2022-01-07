### 1.1   建ns、secret、config
```
kubectl create ns kubemark

kubectl create configmap node-configmap -n kubemark --from-literal=content.type="member2"

kubectl create secret generic kubeconfig --type=Opaque --namespace=kubemark --from-file=kubelet.kubeconfig=/root/.kube/config --from-file=kubeproxy.kubeconfig=/root/.kube/config
```

### 1.2 建 hollownode
- ps：kubemark-press: "true" 这个label 只会打到pod上
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: hollow-node
  namespace: kubemark
spec:
  replicas: 3
  selector:
      name: hollow-node
  template:
    metadata:
      labels:
        name: hollow-node
        kubemark-press: "true"
    spec:
      initContainers:
      - name: init-inotify-limit
        image: registry.vivo.xyz/library/busybox:v1.0
        command: ['sysctl', '-w', 'fs.inotify.max_user_instances=200']
        securityContext:
          privileged: true
      volumes:
      - name: kubeconfig-volume
        secret:
          secretName: kubeconfig
      - name: logs-volume
        hostPath:
          path: /var/log
      containers:
      - name: hollow-kubelet
        image: registry.vivo.xyz/library/kubemark:v1.0
        ports:
        - containerPort: 4194
        - containerPort: 10250
        - containerPort: 10255
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        command:
        - /kubemark
        args:
        - --morph=kubelet
        - --name=$(NODE_NAME)
        - --kubeconfig=/kubeconfig/kubelet.kubeconfig
        - --alsologtostderr
        - --v=2
        volumeMounts:
        - name: kubeconfig-volume
          mountPath: /kubeconfig
          readOnly: true
        - name: logs-volume
          mountPath: /var/log
        resources:
          requests:
            cpu: 20m
            memory: 50M
        securityContext:
          privileged: true
      - name: hollow-proxy
        image: registry.vivo.xyz/library/kubemark:v1.0
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        command:
        - /kubemark
        args:
        - --morph=proxy
        - --name=$(NODE_NAME)
        - --use-real-proxier=false
        - --kubeconfig=/kubeconfig/kubeproxy.kubeconfig
        - --alsologtostderr
        - --v=2
        volumeMounts:
        - name: kubeconfig-volume
          mountPath: /kubeconfig
          readOnly: true
        - name: logs-volume
          mountPath: /var/log
        resources:
          requests:
            cpu: 20m
            memory: 50M
      tolerations:
      - effect: NoExecute
        key: node.kubernetes.io/unreachable
        operator: Exists
      - effect: NoExecute
        key: node.kubernetes.io/not-ready
        operator: Exists
```

### 1.3 给所有hollownode 打lable

```
$ cat lablenode.sh
#!/bin/bash
echo "kubectl get node |grep hollow-node |grep -w Ready |awk '{print$1}'> nodelist.txt"
kubectl get node |grep hollow-node |grep -w Ready |awk '{print$1}'> nodelist.txt
while read line
do
echo $line
kubectl label nodes $line kubemark-press=true --overwrite
done < nodelist.txt
```



参考：https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scalability/kubemark-setup-guide.md
