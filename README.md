# Objective

To upgrade a worker node manually

In this POC, we are upgrading a cluster + worker node from v1.19 to v1.20

# WARNING

This is not a recommended practice. Be sure to test that everything works before applying to production

The recommended approach are documented here
1. https://docs.aws.amazon.com/eks/latest/userguide/update-workers.html

Proceed at your own risk

# Pre-requisite

1. Spin up eks cluster (v1.19) using `eksctl create -f eksctl/cluster.yaml`

# Steps taken

1. Update control plane to 1.20 - `eksctl upgrade cluster --name <my-cluster> --approve`

```bash
> k version --short                                                                        leesebas@147dda1b330c
Client Version: v1.23.1
Server Version: v1.20.15-eks-14c7a48
WARNING: version difference between client (1.23) and server (1.20) exceeds the supported minor version skew of +/-1
---------------------------------------------------------------------------------------------------------------------------------------------------------------------
> k get nodes                                                                              leesebas@147dda1b330c
NAME                                               STATUS   ROLES    AGE     VERSION
ip-<IPADDRESS>.ap-southeast-1.compute.internal   Ready    <none>   3h32m   v1.19.15-eks-9c63c4
ip-<IPADDRESS>.ap-southeast-1.compute.internal    Ready    <none>   3h32m   v1.19.15-eks-9c63c4
```

2. Setup env variable - https://github.com/awslabs/amazon-eks-ami/blob/master/eks-worker-al2.json

```bash
export BINARY_BUCKET_NAME=amazon-eks
export BINARY_BUCKET_REGION=us-west-2
# Refer to https://github.com/awslabs/amazon-eks-ami/issues/524
# use aws s3 ls s3://amazon-eks
export KUBERNETES_VERSION="1.20.10"
# aws s3 ls s3://amazon-eks/$KUBERNETES_VERSION/
export KUBERNETES_BUILD_DATE="2021-10-12"
# aws s3 ls s3://amazon-eks/$KUBERNETES_VERSION/$KUBERNETES_BUILD_DATE/bin/
export PLATFORM=linux
# aws s3 ls s3://amazon-eks/$KUBERNETES_VERSION/$KUBERNETES_BUILD_DATE/bin/$PLATFORM/
export ARCH=amd64
export CNI_PLUGIN_VERSION="v0.8.6"
```

3. Run upgrade

```bash
sudo mkdir -p /etc/kubernetes/manifests
sudo mkdir -p /var/lib/kubernetes
sudo mkdir -p /var/lib/kubelet
sudo mkdir -p /opt/cni/bin

echo "Downloading binaries from: s3://$BINARY_BUCKET_NAME"
S3_DOMAIN="amazonaws.com"
if [ "$BINARY_BUCKET_REGION" = "cn-north-1" ] || [ "$BINARY_BUCKET_REGION" = "cn-northwest-1" ]; then
    S3_DOMAIN="amazonaws.com.cn"
elif [ "$BINARY_BUCKET_REGION" = "us-iso-east-1" ]; then
    S3_DOMAIN="c2s.ic.gov"
elif [ "$BINARY_BUCKET_REGION" = "us-isob-east-1" ]; then
    S3_DOMAIN="sc2s.sgov.gov"
fi
S3_URL_BASE="https://$BINARY_BUCKET_NAME.s3.$BINARY_BUCKET_REGION.$S3_DOMAIN/$KUBERNETES_VERSION/$KUBERNETES_BUILD_DATE/bin/linux/$ARCH"
S3_PATH="s3://$BINARY_BUCKET_NAME/$KUBERNETES_VERSION/$KUBERNETES_BUILD_DATE/bin/linux/$ARCH"

BINARIES=(
    kubelet
    aws-iam-authenticator
)
for binary in ${BINARIES[*]} ; do
    if [[ -n "$AWS_ACCESS_KEY_ID" ]]; then
        echo "AWS cli present - using it to copy binaries from s3."
        aws s3 cp --region $BINARY_BUCKET_REGION $S3_PATH/$binary .
        aws s3 cp --region $BINARY_BUCKET_REGION $S3_PATH/$binary.sha256 .
    else
        echo "AWS cli missing - using wget to fetch binaries from s3. Note: This won't work for private bucket."
        sudo wget $S3_URL_BASE/$binary
        sudo wget $S3_URL_BASE/$binary.sha256
    fi
    sudo sha256sum -c $binary.sha256
    sudo chmod +x $binary
    sudo mv $binary /usr/bin/
done

# Since CNI 0.7.0, all releases are done in the plugins repo.
CNI_PLUGIN_FILENAME="cni-plugins-linux-${ARCH}-${CNI_PLUGIN_VERSION}"

if [ "$PULL_CNI_FROM_GITHUB" = "true" ]; then
    echo "Downloading CNI plugins from Github"
    wget "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGIN_VERSION}/${CNI_PLUGIN_FILENAME}.tgz"
    wget "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGIN_VERSION}/${CNI_PLUGIN_FILENAME}.tgz.sha512"
    sudo sha512sum -c "${CNI_PLUGIN_FILENAME}.tgz.sha512"
    rm "${CNI_PLUGIN_FILENAME}.tgz.sha512"
else
    if [[ -n "$AWS_ACCESS_KEY_ID" ]]; then
        echo "AWS cli present - using it to copy binaries from s3."
        aws s3 cp --region $BINARY_BUCKET_REGION $S3_PATH/${CNI_PLUGIN_FILENAME}.tgz .
        aws s3 cp --region $BINARY_BUCKET_REGION $S3_PATH/${CNI_PLUGIN_FILENAME}.tgz.sha256 .
    else
        echo "AWS cli missing - using wget to fetch cni binaries from s3. Note: This won't work for private bucket."
        sudo wget "$S3_URL_BASE/${CNI_PLUGIN_FILENAME}.tgz"
        sudo wget "$S3_URL_BASE/${CNI_PLUGIN_FILENAME}.tgz.sha256"
    fi
    sudo sha256sum -c "${CNI_PLUGIN_FILENAME}.tgz.sha256"
fi
sudo tar -xvf "${CNI_PLUGIN_FILENAME}.tgz" -C /opt/cni/bin
rm "${CNI_PLUGIN_FILENAME}.tgz"

sudo rm ./*.sha256

sudo mkdir -p /etc/kubernetes/kubelet
sudo mkdir -p /etc/systemd/system/kubelet.service.d
sudo mv $TEMPLATE_DIR/kubelet-kubeconfig /var/lib/kubelet/kubeconfig
sudo chown root:root /var/lib/kubelet/kubeconfig

sudo mv $TEMPLATE_DIR/kubelet.service /etc/systemd/system/kubelet.service
sudo chown root:root /etc/systemd/system/kubelet.service
sudo mv $TEMPLATE_DIR/kubelet-config.json /etc/kubernetes/kubelet/kubelet-config.json
sudo chown root:root /etc/kubernetes/kubelet/kubelet-config.json


sudo systemctl daemon-reload
# Disable the kubelet until the proper dropins have been configured
sudo systemctl disable kubelet

```


Output

```bash
> k get nodes -o wide

NAME                                               STATUS   ROLES    AGE     VERSION               INTERNAL-IP     EXTERNAL-IP      OS-IMAGE         KERNEL-VERSION                 CONTAINER-RUNTIME
ip-<IPADDRESS>.ap-southeast-1.compute.internal   Ready    <none>   3h49m   v1.20.10-eks-3bcdcd   <IPADDRESS>   <IPADDRESS>    Amazon Linux 2   5.4.188-104.359.amzn2.x86_64   docker://20.10.13
ip-<IPADDRESS>.ap-southeast-1.compute.internal    Ready    <none>   3h49m   v1.19.15-eks-9c63c4   <IPADDRESS>    <IPADDRESS>   Amazon Linux 2   5.4.188-104.359.amzn2.x86_64   docker://20.10.13


```


# References
1. AWS EKS AMI setup for kubernetes (https://github.com/awslabs/amazon-eks-ami/blob/master/scripts/install-worker.sh#L197)
