# deploy-gitlab-on-kubernetes-cluster-offline

How to deploy Gitlab into an isolated kubernetes cluster.

## Steps
1. Download into your laptop official docker images from internet.
2. Export images into .tar files, and copy them into Master server.
3. Create namespace "gitlab"
4. Load, Tag and Push images into local docker registry.
5. Create directories on NFS server to store permanent data.
6. Create and Claim persistent volumes on NFS server, then create Services and Deploy images.
7. Access into Gitlab (register).



## 1. Download into your laptop official docker images from internet
To do that, docker must be installed first (https://docs.docker.com/install/).

Pull docker images:
```
  $ docker pull gitlab/gitlab-ce
  $ docker pull postgres:11.0-alpine
  $ docker pull redis:5.0.0-alpine
```

Verify docker images:
```
  $ docker images
  REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
  gitlab/gitlab-ce    latest              1ddc5d2b7c9b        2 days ago          1.56GB
  redis               5.0.0-alpine        05635ee9e1c7        2 weeks ago         40.8MB
  postgres            11.0-alpine         47510d0ec468        2 weeks ago         71.6MB
```

**IMPORTANT**:
Image's versions change oftently. To check latest available, go to their official web sites:

- postgres image (use alpine image edition, which is much lighter than postgres default one)
https://hub.docker.com/_/postgres/

- redis image (use alpine image edition, which is much lighter than redis default one)
https://hub.docker.com/_/redis/

- gitlab-ce image (use community edition)
https://hub.docker.com/r/gitlab/



## 2. Export docker images into tar files, and copy them into Master server
Export docker images to make them portable:
```
  $ docker save postgres  >  postgres-alpine.tar
  $ docker save redis  >  redis-alpine.tar
  $ docker save gitlab/gitlab-ce  >  gitlab-ce.tar
```

Verify .tar files:
```
  $ ls -lh /Users/coco/Docker
  -rw-r--r--  1 coco  staff   1.5G Nov  7 17:29 gitlab-ce.tar
  -rw-r--r--  1 coco  staff    72M Nov  7 17:29 postgres-alpine.tar
  -rw-r--r--  1 coco  staff    40M Nov  7 17:28 redis-alpine.tar
```

Then copy these 3 .tar files into "Master" server (ie: under /tmp)
```
  $ ssh root@icp01-master-1
  $ ls -lrt /tmp
  total 1698844
  -rw-r--r-- 1 root root 1621800448 Nov  7 17:55 gitlab-ce.tar
  -rw-r--r-- 1 root root   75747328 Nov  7 17:55 postgres-alpine.tar
  -rw-r--r-- 1 root root   42062336 Nov  7 17:55 redis-alpine.tar
```


## 3. Create namespace "gitlab"
On Master server run:
```
  $ ssh root@icp01-master-1
  $ cd /tmp
  $ cat gitlab-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: gitlab
  $ kubectl create -f ./gitlab-namespace.yaml
  $ kubectl get namespaces | grep gitlab
```


## 4. Load, Tag and Push images into "Master" local docker registry
Login into Kubernetes cluster:
```
  $ ssh root@icp01-master-1
  $ docker login mycluster.icp:8500
  admin / admin
```

Load docker images locally ("Master" server):
```
  $ cd /tmp
  $ docker load -i gitlab-ce.tar
  $ docker load -i postgres-alpine.tar
  $ docker load -i redis-alpine.tar
```

Verify loaded images in "Master":
```
  $ docker images | grep gitlab
    gitlab/gitlab-ce                    latest           1ddc5d2b7c9b        2 days ago          1.56GB
  $ docker images | grep postgres
    postgres                            11.0-alpine      47510d0ec468        2 weeks ago         71.6MB
  $ docker images | grep redis
    redis                               5.0.0-alpine     05635ee9e1c7        2 weeks ago         40.8MB
```

Tag docker image version (optional, but recommended):
```
  $ docker tag gitlab/gitlab-ce:latest mycluster.icp:8500/gitlab/gitlab-ce:11.4.5-ce.0
  $ docker tag postgres:11.0-alpine mycluster.icp:8500/gitlab/postgres:11.0-alpine
  $ docker tag redis:5.0.0-alpine mycluster.icp:8500/gitlab/redis:5.0.0-alpine
```
  ie: docker tag imagename:tagname <cluster_CA_domain>:8500/namespacename/imagename:tagname

Verify tags (filtering by "gitlab" namespace):
```
  $ docker images | grep gitlab
    mycluster.icp:8500/gitlab/gitlab-ce     11.4.5-ce.0      1ddc5d2b7c9b        2 days ago          1.56GB
    mycluster.icp:8500/gitlab/redis         5.0.0-alpine     05635ee9e1c7        2 weeks ago         40.8MB
    mycluster.icp:8500/gitlab/postgres      11.0-alpine      47510d0ec468        2 weeks ago         71.6MB
```

Push docker images to local registry:
```
  $ docker push mycluster.icp:8500/gitlab/gitlab-ce:11.4.5-ce.0
  $ docker push mycluster.icp:8500/gitlab/postgres:11.0-alpine
  $ docker push mycluster.icp:8500/gitlab/redis:5.0.0-alpine
```

Verification:
If using ICP, at this step, we should see under ICP console these 3 new images:

  Login into ICP Console https://10.71.9.10:8443
  admin / admin

	   Container Images:
		   - gitlab/gitlab-ce
		   - gitlab/postgres
		   - gitlab/redis

	   Namespaces:
    	 - gitlab



## 5. Create directories on NFS server to store permanent data
```
  $ ssh root@icp01-nfs-1
  $ mkdir /export/gitlab
  $ mkdir /export/gitlab/postgres-pv
  $ mkdir /export/gitlab/redis-pv
  $ mkdir /export/gitlab/gitlab-pv
```


## 6. Create and Claim persistent volumes on NFS server, then create Services and Deploy images
IMPORTANT:
Yes, this can be done through a Helm template, or even merging actions like persistent's volumes creation.
I just run "yaml's" one by one for learning purposes.

Before anything, copy and extract "yamls.tar" file into "Master" server.
```
  $ cp "yamls.tar" root@10.71.9.10:/tmp ("Master" server)

  $ ssh root@icp01-master-1
  $ cd /tmp
  $ tar -xvf yamls.tar
  $ cd /tmp/yamls
```

Create "postgres-pv" persistent volume (on NFS server)
```
  $ kubectl create -f kubernetes/postgres-pv.yaml
```

Claim persistent volume, create service and deploy "postgres" image
```
  $ kubectl create -f postgres.yaml
```

Create "redis-pv" persistent volume (on NFS server)
```
  $ kubectl create -f redis-pv.yaml
```

Claim persistent volume, create service and deploy "redis" image
```
  $ kubectl create -f redis.yaml
```

Create "gitlab-pv" persistent volume (on NFS server)
```
  $ kubectl create -f gitlab-pv.yaml
```

Claim persistent volume, create service and deploy "gitlab" image
```
  $ kubectl create -f gitlab.yaml
```

Verification:
```
  $ kubectl get nodes
```

  Access into NFS server and check new files were created under:
```
    - /export/gitlab/postgres-pv
    - /export/gitlab/redis-pv
    - /export/gitlab/gitlab-pv
```


## 7. Access into Gitlab (register)

https://10.71.9.10:30008/gitLab
