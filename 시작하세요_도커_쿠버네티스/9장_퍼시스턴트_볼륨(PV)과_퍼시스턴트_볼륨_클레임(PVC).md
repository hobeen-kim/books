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
```

