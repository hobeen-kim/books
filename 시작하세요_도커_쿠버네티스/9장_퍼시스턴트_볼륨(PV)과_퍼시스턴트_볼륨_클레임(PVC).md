![image-20240504191030509](images/9장_퍼시스턴트_볼륨(PV)과_퍼시스턴트_볼륨_클레임(PVC)/image-20240504191030509.png)

  데이터를 저장하고 상태를 유지하기 위해서 (stateful) 어느 노드에서도 접근해 사용할 수 있는 퍼시스턴트 볼륨(Persistent Volume) 을 사용할 수 있다.

**기타 데이터 공유**

- 워커 노드의 로컬 디렉토리를 볼륨으로 사용 : hostPath
  - 포드가 다른 노드로 옮겨갈 경우 사용 불가
  - 모니터링 툴을 모든 워커 노드에 배포할 때 사용할 수 있음
- 포드 내의 컨테이너 간 임시 데이터 공유 : emptyDir

# 네트워크 볼륨

![image-20240504191654566](images/9장_퍼시스턴트_볼륨(PV)과_퍼시스턴트_볼륨_클레임(PVC)/image-20240504191654566.png)

 네트워크 볼륨은 많은 종류가 있음

**NFS 를 네트워크 볼륨으로 사용하기**

  NFS 기능을 간단히 사용하기 위한 임시 NFS 서버

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
```

```
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
spec:
  ports:
  - nmae: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    role: nfs-server
```

```
$ kubectl apply -f nfs-deployment.yaml
$ kubectl apply -f nfs-service.yaml
```

  아래는 만든 NFS 서버의 볼륨을 포드에서 마운트해 데이터를 영속적으로 저장함

```
apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
  - name: nfs-mount-container
    image: busybox
    args: [ "tail", "-f", "/dev/null" ]
    volumeMounts:
    - name: nfs-volume
      mountPath: /mnt
  volumes:
  - name: nfs-volume
    nfs:
      path: /
      server: {NFS_SERVICE_IP}
```

# PV, PVC 를 이용한 볼륨 관리

  위 NFS 예시는 반드시 NFS 서버가 있어야 함. 하지만 이렇게 되면 볼륨과 애플리케이션 정의가 서로 밀접하게 연관되어 있어 분리할 수 없게 된다. PV, PVC 를 사용하면 볼륨을 추상화해서 YAML 파일 작성 시에 네트워크 볼륨이 NFS 인지, AWS 의 EBS 인지 상관없이 볼륨을 사용할 수 있게 된다.

![image-20240505200815738](images/9장_퍼시스턴트_볼륨(PV)과_퍼시스턴트_볼륨_클레임(PVC)/image-20240505200815738.png)

**AWS 에서 EBS 를 퍼시스턴트 볼륨으로 사용하기**

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: <VOLUME_ID>
```

<VOLUME_ID> 에 EBS 볼륨 id 를 넣어서 만듦.

- 퍼시스턴트 볼륨 만들기: `kubectl apply -f ebs-pv.yaml`
- 모든 퍼시스턴트 볼륨 확인 : `kubectl get pv`

  그리고 아래와 같이 애플리케이션 배포하는 입장에서 퍼시스턴트 볼륨 크레임과 포드를 함께 생성

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  storageClassName: ""
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-mount-container
spec:
  containers:
  - name: ebs-mount-container
    image: busybox
    args: [ "tail", "-f", "/dev/null" ]
    volumeMounts:
    - name: ebs-volume
      mountPath: /mnt
  volumes:
  - name: ebs-volume
    persistentVolumeClaim:
			claimName: my-ebs-pvc
```

- pvc 에 있는 조건 리스트에 만족하는 pv 만 바인드됨
  - ![image-20240505220844714](images/9장_퍼시스턴트_볼륨(PV)과_퍼시스턴트_볼륨_클레임(PVC)/image-20240505220844714.png)