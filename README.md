# Privacy Preserving Machine Learning Demo using Tensorflow 

This demo is based on the [Graphene Tensorflow Demo](https://github.com/oscarlab/graphene), using Graphene to run a Tensorflow Model Server in an SGX enclave and Marblerun to take care of attestation and secret provisioning.

## How it works
![marblerun-tensorflow](illustration.svg)

1. The TensorFlow Aministrator uploads the Marblerun manifest to the Marblerun Coordinator, defining parameters under which programs are allowed to run
2. The Coordinator performs attestation on the TensorFlow Serving application running on SGX using Graphene, and provides the application with secrets defined by the Marblerun manifest.
3. The Administrator encrypts a pre-trained model and uplodas it to Kubernetes file storage
4. The TensorFlow application reads and decrypts the model using a key provided by the coordinator
5. A Client performs attestation of the Marblerun Coordinator and receives a certificate for connections to TensorFlow Serving
6. The Client requests a prediction on some input data
7. TensorFlow sends its response

## Install Dependencies

To run the python scripts we need python3 and some extra libraries. Make sure pip is up to date and run:
```bash
pip3 install -r ./client/requirements.txt
pipe install grpcio~=1.34.0
``` 

## Running as a Container

1. Create a mapping of machine B's IP adress (the machine you plan to run the docker image on) to the Tensorflow Serving domain name (127.0.0.1 if you are running on just one machine)
    ```bash
    machineB_ip_addr=XX.XX.XX.XX
    echo "${machineB_ip_addr} grpc.tf-serving.service.com" >> /etc/hosts
    ```

1. Start Marblerun
    ```bash
    EDG_COORDINATOR_DNS_NAMES=grpc.tf-serving.service.com erthost ${marblerun_dir}/build/coordinator-enclave.signed
    export MARBLERUN=grpc.tf-serving.service.com:4433
    ```

1. Download and convert the model
    ```bash
    ./tools/download_model.sh
    models_abs_dir=`pwd -P`
    python3 ./tools/model_graph_to_saved_model.py --import_path ${models_abs_dir}/models/resnet50-v15-fp32/resnet50-v15-fp32.pb --export_dir ${models_abs_dir}/models/resnet50-v15-fp32 --model_version 1 --inputs input --outputs predict
    mkdir plain
    mv models/resnet50-v15-fp32/1/saved_model.pb plain/
    ```

1. Use Graphenes `pf_crypt` to generate a key and encrypt the model.
    ```bash
    pf_crypt gen-key --wrap-key model_key
    pf_crypt encrypt --input plain/saved_model.pb --output models/resnet50-v15-fp32/1/saved_model.pb --wrap-key model_key
    ```

1. Set the model key in Marbleruns manifest
    ```bash
    cat tf-server-manifest.json | sed "s|YOUR_KEY_HERE|$(hexdump -ve '1/1 "%02x"' model_key)|g" > manifest.json
    ```

1. Upload the manifest
    ```bash
    marblerun manifest set manifest.json $MARBLERUN
    ```

1. Start the Tensorflow Model Server
    ```bash
    ./tools/run_tf_image.sh
    ```

1. Get Marbleruns intermediate certificate to connect to the model server
    ```bash
    marblerun certificate intermediate $MARBLERUN -o tensorflow.crt
    ```

1. Test the model server using the gRPC client
    ```bash
    python3 ./client/resnet_client_grpc.py --url grpc.tf-serving.service.com:8500 --crt tensorflow.crt --batch 1 --cnum 1 --loop 10
    ```


## Running on Kubernetes

Make sure your cluster supports SGX and out-of-process attestation. You can follow [the guide by Microsoft](https://docs.microsoft.com/en-us/azure/confidential-computing/confidential-nodes-out-of-proc-attestation) to create a AKS cluster with all the needed resources.

1. Start the Marblerun coordinator
    ```bash
    marblerun install --domain="grpc.tf-serving.service.com\,localhost"
    ```

1. Wait for Marblerun to setup
    ```bash
    marblerun check
    ```

1. Port-forward the client API service to localhost
    ```bash
    kubectl -n marblerun port-forward svc/coordinator-client-api 4433:4433 --address localhost >/dev/null &
    export MARBLERUN=localhost:4433
    ```

1. Download and convert the model
    ```bash
    ./tools/download_model.sh
    models_abs_dir=`pwd -P`
    python3 ./tools/model_graph_to_saved_model.py --import_path ${models_abs_dir}/models/resnet50-v15-fp32/resnet50-v15-fp32.pb --export_dir ${models_abs_dir}/models/resnet50-v15-fp32 --model_version 1 --inputs input --outputs predict
    mkdir plain
    mv models/resnet50-v15-fp32/1/saved_model.pb plain/
    ```

1. Use Graphenes `pf_crypt` to generate a key and encrypt the model.
    ```bash
    pf_crypt gen-key --wrap-key model_key
    pf_crypt encrypt --input plain/saved_model.pb --output models/resnet50-v15-fp32/1/saved_model.pb --wrap-key model_key
    ```

1. Set the model key in Marbleruns manifest
    ```bash
    cat tf-server-manifest.json | sed "s|YOUR_KEY_HERE|$(hexdump -ve '1/1 "%02x"' model_key)|g" > manifest.json
    ```

1. Upload the manifest:
    ```bash
    marblerun manifest set manifest.json $MARBLERUN
    ```

1. Create and add the tensorflow namespace to Marblerun
    ```bash
    kubectl create namespace tensorflow
    marblerun namespace add tensorflow
    ```

1. Start the Tensorflow Model Server
    ```bash
    helm install -f ./kubernetes/values.yaml tensorflow-demo ./kubernetes -n tensorflow
    ```

1. Upload the model to Kubernetes
    ```bash
    kubectl cp ./models/resnet50-v15-fp32/1/saved_model.pb tensorflow/tf-server-xxxxxxxxx-xxxxx:/graphene/Examples/tensorflow-marblerun/models/resnet50-v15-fp32/1/saved_model.pb
    ```

1. Get Marblerun's certificate
    ```bash
    marblerun certificate intermediate $MARBLERUN -o tensorflow.crt
    ```

1. Create mapping of the Tensorflow Model Server IP to it's domain name
    * First get the IP Adress:
        ```bash
        tf_ip_addr=`kubectl get svc -n tensorflow -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}'`
        ```
    * Set the mapping in `/etc/hosts`:
        ```bash
        echo "${tf_ip_addr} grpc.tf-serving.service.com" >> /etc/hosts
        ```

1. Submit a request using encrypted traffic over gRPC
    ```bash
    python3 ./client/resnet_client_grpc.py --url grpc.tf-serving.service.com:8500 --crt ./tensorflow.crt --batch 1 --cnum 1 --loop 10
    ```

### Cleaning up

1. Remove tensorflow from the cluster
    ```bash
    helm uninstall tensorflow-demo -n tensorflow
    kubectl delete namespace tensorflow
    ```

1. Uninstall Marblerun
    ```bash
    marblerun uninstall
    ```

## Building the Docker Image

*Prerequisite*: Graphene is set up and example applications are working correctly with SGX.

1. Assuming you have built Graphene in `/graphene` copy everything from `./graphene-files` into `/graphene/Examples/tensorflow-marblerun`
    ```bash
    mkdir /graphene/Examples/tensorflow-marblerun
    cp ./graphene-files/* /graphene/Examples/tensorflow-marblerun
    ```

1. Next we can build the Docker image:
    ```bash
    docker buildx build --tag ghcr.io/edgelesssys/tensorflow-graphene-marble:latest -f tensorflow.dockerfile /graphene
    ```