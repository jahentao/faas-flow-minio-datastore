# faasflow-minio-datastore
A **[faasflow](https://github.com/s8sg/faasflow)** datastore implementation that uses minio DB to store data  
which can also be used with s3 bucket

## Getting Stated

### Deploy Minio
* Generate secrets for Minio
```bash
SECRET_KEY=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)
ACCESS_KEY=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)
```
#### Deploy in Kubernets
* Store the secrets in Kubernetes
```bash
kubectl create secret generic -n openfaas-fn \
 s3-secret-key --from-literal s3-secret-key="$SECRET_KEY"
kubectl create secret generic -n openfaas-fn \
 s3-access-key --from-literal s3-access-key="$ACCESS_KEY"
```
* Install Minio with helm
```bash
helm install --name cloud --namespace openfaas \
   --set accessKey=$ACCESS_KEY,secretKey=$SECRET_KEY,replicas=1,persistence.enabled=false,service.port=9000,service.type=NodePort \
  stable/minio
```
The name value should be `cloud-minio.openfaas.svc.cluster.local`  
   
Enter the value of the DNS above into `s3_url` in `gateway_config.yml` adding the port at the end: `cloud-minio-svc.openfaas.svc.cluster.local:9000`

#### Deploy in Swarm
* Store the secrets
```bash
echo -n "$SECRET_KEY" | docker secret create s3-secret-key -
echo -n "$ACCESS_KEY" | docker secret create s3-access-key -
```
* Deploy Minio
```bash
docker service rm minio

docker service create --constraint="node.role==manager" \
 --name minio \
 --detach=true --network func_functions \
 --secret s3-access-key \
 --secret s3-secret-key \
 --env MINIO_SECRET_KEY_FILE=s3-secret-key \
 --env MINIO_ACCESS_KEY_FILE=s3-access-key \
minio/minio:latest server /export
```
> Note: For debugging and testing. You can expose the port of Minio with docker service update minio --publish-add 9000:9000, but this is not recommended on the public internet.


#### Deploy Minio with Nomad and OpenFaaS

if `MINIO_SECRET_KEY_FILE` and `MINIO_ACCESS_KEY_FILE` not configured
```
Detected default credentials 'minioadmin:minioadmin', please change the credentials immediately using 'MINIO_ACCESS_KEY' and 'MINIO_SECRET_KEY'
```

* Create secrets with OpenFaaS

```bash
echo -n "$SECRET_KEY" | faas-cli secret create s3-secret-key -g http://$IP_ADDRESS:8080
echo -n "$ACCESS_KEY" | faas-cli secret create s3-access-key -g http://$IP_ADDRESS:8080
```

* Deploy Minio with Nomad

```
nomad run minio.hcl
```

minio.hcl for example

```hcl
job "minio" {
  datacenters = ["dc1"]

  type = "service"

  constraint {
    attribute = "${attr.cpu.arch}"
    operator  = "="
    value     = "amd64"
  }

  group "faas-datastore" {
    count = 1

    restart {
      attempts = 10
      interval = "5m"
      delay    = "25s"
      mode     = "delay"
    }

    task "minio" {
      driver = "docker"

      config {
        image = "minio/minio"

        port_map {
          minio = 9000
        }

        args = [
          "server",
          "/data",
        ]

        volumes = [
          "/root/minio/data:/data",
        ]
      }

      // TODO: use Vault for secret management
//       template {
//         destination   = "secrets/s3-secret-key"
//         data = <<EOH
// minioadmin
// EOH
//       }
//       template {
//         destination   = "secrets/s3-access-key"
//         data = <<EOH
// minioadmin
// EOH
//       }

      template {
        env = true
        destination   = "secrets/minio.env"

        data = <<EOH
MINIO_SECRET_KEY=minioadmin
MINIO_ACCESS_KEY=minioadmin
EOH
      }

      resources {
        cpu    = 500 # 500 MHz
        memory = 128 # 128MB

        network {
          mbits = 10

          port "minio" {
            static = 9000
          }
        }
      }

      service {
        port = "minio"
        name = "faas-datastore"
        tags = ["faas-datastore"]
      }
    }
  }
}
```

### Use Minio dataStore in `faasflow`
* Set the `stack.yml` with the necessary environments
```yaml
      s3_url: "minio:9000"
      s3_tls: false
    secrets:
      - s3-secret-key
      - s3-access-key
```
* Use the `faasflowMinioDataStore` as a DataStore on `handler.go`
```go
minioDataStore "github.com/s8sg/faas-flow-minio-datastore"

func DefineDataStore() (faasflow.DataStore, error) {

       // initialize minio DataStore
       miniods, err := minioDataStore.InitFromEnv()
       if err != nil {
               return nil, err
       }

       return miniods, nil
}
```
