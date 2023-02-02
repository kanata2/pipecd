- Start Date: 2022-01-18
- Target Version: 0.41.3

# Summary

Support install PipeCD control plane on other platform which is not k8s

# Motivation

Now we can deploy the control plane to kubernetes cluster, but some developers that would like to introduce PipeCD can not prepare kubernetes environments. We want to support install PipeCD control plane on platforms other than kubernetes.

# Detailed design

## 1. Control Plane on docker-compose
    ### Note:
    - Developers can deploy control plane on a single machine.
    - Developers do not have to prepare a datastore and a filestore by themselves or they can easily use a database or a filestore on local machine.
    ### Production Architecture
    - Port-forward from docker-compose to your machine to enable external client access to pipecd-gateway container.
    ![image](assets/control-plane-on-docker-compose.jpg)
    
    
## 2. Control Plane on managed container services (ex. ECS)
    ### Note:
    - Pipecd give the abstruct architecture for ECS.
        - Give a sample repositpry that describes architecure by using Terraform.
    - They can easily use the managed database or storage system on cloud as datastore and filestore.
    
    ### Production Architecture on ECS
    ![image](assets/control-plane-on-aws.jpg)
    
    Note:
    - Devide a pipecd-server and a pipecd-ops to different services because they have the same port and have different authorization.
    - Pay attention to brocking public access to s3.
        - Add IAM role to ECS to access S3.
    - Make two type of target group for both of HTTP and GRPC. Use HTTPS for ALB listners because GRPC listner needs HTTPS.
    - Set up before containers start to run so that pipecd can use config files and encryption key. You can choose from 3 options as bellow.
        - Use Secrets Manager as config store.
        ```
        # get configuration files from priavete s3 bucket via aws-cli
        echo $CONTROL_PLANE_CONFIG >> control_plane_config.yaml
        # replace datastore endpoint (if you use terraform, you can replace by using variable)
        sed -i -e s/pipecd-mysql/${var.db_instance_address}/ control-plane-config.yaml
        # You must set encryption key to environment value
        echo $ENCRYPTION_KEY >> encryption-key
        # replace cache endpoint (if you use terraform, you can replace by using variable)
        pipecd server \
        --insecure-cookie=true \
        --cache-address=${var.redis_host}:6379 \
        --config-file=control-plane-config.yaml \
        --enable-grpc-reflection=false \
        --encryption-key-file=encryption-key \
        --log-encoding=humanize --metrics=true;
        ```
        - Use Parameter Store as config store (Attention: Envoy config has over 4096 charactors, so you can not use Parameter Store as eonvy config file store.)
        - Use S3 as config store. (Attention: install aws-cli to the pipecd-server container to get configuration files from priavete s3 bucket)
        ```
        # get configuration files from priavete s3 bucket via aws-cli
        apk add aws-cli
        aws s3 cp s3://namba-pipecd-control-plane-config/control-plane-config.yaml ./
        ```

# Alternatives

Control Plane without container image
    - This alternative supports deploy control plane as a binary such as Piped.
    - It is stressful for developers to set up networking by themselves.

2. Abolish envoy and use ALB instead of envoy for access distribution
    - We must move configuration for envoy to ALB.
    - We must make the same number of target groups as pipecd-server has.
# Unresolved questions

None
