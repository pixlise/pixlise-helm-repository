# Deploy the Helm chart

To deploy the helm chart in a Kubernetes environment use something along the lines of:

```console
helm --kubeconfig /etc/rancher/k3s/k3s.yaml upgrade --install pixlise . -f localhelmvalues.yml
```

Here are some suggested local overrides:

```yaml
awsConfig:
  aws_access_key_id: "<aws_access_key>"
  aws_secret_access_key: "<aws_secret>"
  aws_region: "us-east-1"
  aws_default_region: "us-east-1"
  aws_regional_endpoint: "regional"

ingress:
  enabled: false

resources:
  limits:
    cpu: "1"
  requests:
    cpu: "0.5"

imagePullSecrets:
  - name: api-auth

args:
  - "-configLocation"
  - "s3://devstack-<bucket>/PixliseConfig/config.json"
  - "-kubernetesLocation"
  - "internal"
  - "-quantExecutor"
  - "kubernetes"

ui:
  apihost: http://localhost:30001
  image:
    tag: "1.1.63-ALPHA"
  service:
    type: NodePort
    nodePort: 30000
api:
  service:
    type: NodePort
    nodePort: 30001
```

To deploy into AWS here are the recommended AWS overrides:

```yaml
awsConfig:
  aws_access_key_id: "<aws_access_key>"
  aws_secret_access_key: "<aws_secret_key>"
  aws_region: "us-east-1"
  aws_default_region: "us-east-1"
  aws_regional_endpoint: "regional"

ingress:
  enabled: true
  style: alb
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/certificate-arn: <cert-arn>
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'

resources:
  limits:
    cpu: "1"
  requests:
    cpu: "0.5"

imagePullSecrets:
  - name: api-auth

args:
  - "-configLocation"
  - "s3://devstack-<bucket>/PixliseConfig/config.json"
  - "-kubernetesLocation"
  - "internal"
  - "-quantExecutor"
  - "kubernetes"
```


## Image Pull Secret

The easiest way to create the image pull secret for the private docker repo is by running the following command:

```console
   kubectl --namespace <target namespace> create secret docker-registry api-auth --docker-server=registry.gitlab.com --docker-username=<gitlab username> --docker-password=<password/api key> --docker-email=<email>
```

## Deploy ALB

To deploy the ALB in your EKS Cluster, follow the steps in the AWS Docs: [Installing the AWS Load Balancer Controller add-on - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

In summary, the following commands should do it:

```
export LBC_VERSION="v2.4.1"
export LBC_CHART_VERSION="1.4.1"

eksctl utils associate-iam-oidc-provider --region us-east-2 --cluster pixlise-cluster --approve
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/$\{LBC_VERSION\}/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl create iamserviceaccount --region us-east-2 --cluster pixlise-cluster --namespace kube-system --name aws-load-balancer-controller --attach-policy-arn arn:aws:iam::963058736014:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

helm upgrade -i aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=pixlise-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set image.tag="${LBC_VERSION}" \
    --version="${LBC_CHART_VERSION}" \
    --set region="us-east-2" --set vpcId="vpc-04b9ed0a8fe6340b7"
```
