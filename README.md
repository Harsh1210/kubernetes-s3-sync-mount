# s3-sync

S3 Sync sidecar container for airflow DAGs

## Parameters

| name       | description                             |
| ---------- | --------------------------------------- |
| AWS_BUCKET | S3 bucket name. ex) my-bucket           |
| KEY_PATH   | S3 key name. ex) dags                   |
| DEST_PATH  | destination path. ex) /opt/airflow/dags |
| INTERVAL   | sync process interval seconds. ex) 10   |

The agenda is to create a side car to mount S3 as volume on pod by utilising rsync. 

 

Pre-requisites: 

EKS cluster 

AWS user with S3FullAccess and access to EKS cluster 

Service account with S3 access (https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html) 

 

Step – 1: CREATING S3 BUCKET 

Create directory s3-mount 

Create a file main.tf - this will create an S3 bucket and push a file called test.py in it. 

```json
terraform { 
    required_providers { 
        aws = { 
            source = "hashicorp/aws" 
            version = "~> 3.0" 
        } 
    } 
} 
provider "aws" { 
    region = "us-east-1" 
   access_key = "my-access-key" 
   secret_key = "my-secret-key" 
} 
resource "aws_s3_bucket" "S3Bucket" { 
    bucket = "empower-s3-mount" 
}  
resource "local_file" "foo" { 
  content  = "foo!" 
  filename = "./test.py" 
} 
resource "aws_s3_bucket_object" "example" { 
  bucket = " empower-s3-mount" 
  key = "test.py" 
  source = "./test.py" 
  depends_on = [local_file.foo] 
} 
```

Run following commands: 

terraform init 
terraform apply 

This will create S3 bucket. 

Step – 2: CREATE SCRIPT TO FOR SIDECAR CONTAINER 

Create directory s3-mount 

Create a file called run.sh and add following code: 

```Shell
#!/bin/sh -e 
s3get () { 
	echo "sync from s3" 
	mkdir -p /opt/airflow/dags 
	result=$(aws s3 sync --exact-timestamps s3://empower-s3-mount/ /opt/airflow/dags --exclude "*.pyc" --delete 2>&1) 
	result_code=$? 
	if [[ $result_code != 0 ]]; then 
		echo "s3 sync failed" 
		continue 
	fi 
	echo "finished downloading from s3" 
} 
s3get 
while true; do 
	s3get 
	sleep 1 
done 
```
Step – 3: CREATE DOCKERFILE FOR SIDECAR 

Create Dockerfile with following content 

```dockerfile
FROM python:3.8-alpine 
RUN pip install -U pip awscli && \ 
apk add --no-cache --update rsync 
COPY . /src 
WORKDIR /src 
USER root 
CMD ["/bin/sh","/src/run.sh"] 
```

Step – 4: BUILD DOCKER IMAGE FOR SIDE CAR 
```
docker build –t <docker-registry>:01 . 
```
Step – 5: PUSH TO DOCKER REGISTRY 
```
docker push <docker-registry>:01 
```
Step – 6: CREATE DEPLOYMENT TO RUN THE SIDE CAR AND TEST 

Create deployment.yaml for nginx deployment along with the side car. Replace name of service account 
```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  annotations: 
    deployment.kubernetes.io/revision: "1" 
  name: nginx 
spec: 
  progressDeadlineSeconds: 600 
  replicas: 1 
  revisionHistoryLimit: 10 
  selector: 
    matchLabels: 
      app: nginx 
  strategy: 
    type: Recreate 
  template: 
    metadata: 
      creationTimestamp: null 
      labels: 
        app: nginx 
    spec: 
      containers: 
      - image: nginx 
        imagePullPolicy: Always 
        name: nginx 
        ports: 
        - containerPort: 80 
          protocol: TCP 
        resources: {} 
        terminationMessagePath: /dev/termination-log 
        terminationMessagePolicy: File 
        volumeMounts: 
        - name: external 
          mountPath: /opt/airflow/dags 
      - image: 660061364911.dkr.ecr.ap-south-1.amazonaws.com/s3-sync-sidecar:01  
        imagePullPolicy: Always 
        name: s3-sidecar 
          #ports: 
          #- containerPort: 80 
          #protocol: TCP 
          #resources: {} 
          #terminationMessagePath: /dev/termination-log 
          #terminationMessagePolicy: File 
        volumeMounts: 
        - name: external 
          mountPath: /opt/airflow/dags 
      volumes: 
      - name: external 
        persistentVolumeClaim: 
          claimName: ebs-claim  
      ServiceAccount: <Name of service account> 
      dnsPolicy: ClusterFirst 
      restartPolicy: Always 
      schedulerName: default-scheduler 
      securityContext: {} 
      terminationGracePeriodSeconds: 30 
```
 

Run the following command: 
```
kubectl apply –f deoployment.yaml 
kubectl get po 
NAME                             READY   STATUS    RESTARTS   AGE 
nginx-6bb994d559-t4rsj   2/2     Running   0          6h15m 

[ec2-user@ip-10-221-42-103 nginx]$ kubectl exec -it nginx-6bb994d559-t4rsj -c nginx bash 
root@nginx-6bb994d559-t4rsj:/# cd /opt/airflow/dags/ 
root@nginx-6bb994d559-t4rsj:/opt/airflow/dags# ls 
SendMail.java  lost+found  sa.yaml  test.py 
```
Since, test.py is present here and in the bucket therefore the mount is working fine. 
