# Kubernetes Cluster Setup with Kops on AWS

This guide provides a step-by-step process to create a Kubernetes cluster on AWS using Kops. 

## Prerequisites

- An AWS account
- Basic knowledge of AWS and Kubernetes
- AWS CLI configured with appropriate permissions
- Domain managed in Route 53

## Steps

### 1. Launch an EC2 Instance

Launch an EC2 instance with the `t2.micro` instance type. You can either create a new instance or use an existing one. Ensure you have SSH access to the instance.

### 2. Install Kops

SSH into your EC2 and install Kops. Follow the installation instructions from the official Kops documentation [here](https://kops.sigs.k8s.io/getting_started/install/).

### 3. Update the System

Update the system packages.

```sh
sudo apt update -y
```

### 4. Create IAM Role and Attach it to EC2

Create an IAM role with necessary permissions and programmatic access. Attaching **AdministratorAccess** is recommended for simplicity. Attach the IAM role to your EC2 instance.

### 5. Create an S3 Bucket

Create an S3 bucket which will be used by Kops to store cluster state information.

### 6. Create a Hosted Zone in Route53

Create a hosted zone in Route53. If you already have a hosted zone, you can skip this step.

### 7. Install kubectl

Install kubectl by following the instructions [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

### 8. Create the Cluster

Run the following command to create your cluster. Update the placeholders with your specific values. Also change the clsuter name that end with your hosted zone name. Example. kubernetes.myhosteszone.com, kubernetes.mydomain.com, etc... 

```sh
kops create cluster --name=k8s-cluster.example.com --state=s3://<your-s3-bucket> --zones=<your-availability-zone> --node-count=1 --node-size=t2.medium --control-plane-size=t2.medium --dns-zone=<your-domain>
```

### 9. Update and Apply the Cluster Configuration

The previous command generates a preview of all the resources Kops will create. If the preview looks good, run the following command to create the resources. (Same like terraform plan(8th step) and terraform apply(9th step))

```sh
kops update cluster --name=k8s-cluster.example.com--yes --admin --state=s3://<your-s3-bucket>
```
This process may take 5-10 minutes.

### 10. Validate the Cluster

Validate that the cluster is ready.

```sh
kops validate cluster --name=k8s-cluster.example.com --state=s3://<your-s3-bucket>
```

Here is the example of expected above command output:
![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/adc2d923-f7f0-44f2-a916-7f44a41bedad)

### 11. Deploy Your Application

Once the cluster is ready, follow these steps to deploy your application:

1. Clone the repository using the following command:
    ```sh
    git clone https://github.com/Divya4242/k8s-aws-kops-ingress.git
    ```

2. Change directory to the cloned repository:
    ```sh
    cd k8s-aws-kops-ingress
    ```

3. Apply the backend deployment configuration:
    ```sh
    kubectl apply -f backend-deployment.yaml
    ```

4. Apply the backend service configuration:
    ```sh
    kubectl apply -f backend-service.yaml
    ```

5. Apply the frontend deployment configuration:
    ```sh
    kubectl apply -f frontend-deployment.yaml
    ```

6. Apply the frontend service configuration:
    ```sh
    kubectl apply -f frontend-service.yaml
    ```
Here is the example of expected above commands output:
![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/07098fdd-dcba-4a45-8595-03b8bff8a296)

To check deployment status of the application, open Chrome and navigate to the following URLs:

    • <ec2-public-ip>:30006 # for the frontend
    • <ec2-public-ip>:30007 # for the backend
###

Until the 12th step, we've successfully configured a Kops cluster, deployed Kubernetes resources, and established Kubernetes services.

### 13. Add NGINX Ingress Controller

To expose your services using NGINX Ingress Controller, follow these steps:

1. Download the NGINX Ingress Controller manifest:
    ```sh
    wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
    ```

2. Edit the `deploy.yaml` file and update the following configurations:
   - Replace `XXX.XXX.XXX/XX` with your VPC CIDR in use for the Kubernetes cluster under `proxy-real-ip-cidr`.
   - Replace `arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX` with your ACM certificate ID.

3. Deploy the NGINX Ingress Controller:
    ```sh
    kubectl apply -f deploy.yaml
    ```

4. Verify that the Ingress Controller is deployed successfully by checking the Kubernetes namespace:
    ```sh
    kubectl get ns
    ```

You should see a namespace named `ingress-nginx`.

### 14. Apply the NGINX Ingress Resource File
   The **NGINX Ingress Controller** is the runtime component that actively manages traffic, while the **NGINX Ingress Resource File** is a configuration artifact that defines the desired behavior for that traffic within the Kubernetes ecosystem. (same like we deploy nginx web server(nginx ingress controller) for serving content to user and reverse proxy and deployment.conf(nginx ingress resource file) in sites availiable that define the servername, listning port, etc.)
```sh
    kubectl apply -f ingress-resource.yaml
```
### 15. Obtain Load Balancer DNS
To find the DNS name of the Load Balancer:

• Run the following command and look for the EXTERNAL-IP of the ingress-nginx-controller service:   
```sh
    kubectl get svc -n ingress-nginx
```

 • Alternatively, you can find the Load Balancer DNS name in the AWS Management Console under EC2 -> Load Balancers.

![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/466978b1-5e99-488d-bf6f-35ee0468eafb)

### 16. Configure DNS in Route 53
Add a CNAME record in Route 53 to point to the Load Balancer DNS name:

1. Go to Route 53 and select your hosted zone.
2. Add a new record:
   
    • Name: The record name specified in your ingress-resource.yaml file.

    • Type: CNAME.

    • Value: The DNS name of the AWS Load Balancer.

![image](https://github.com/Divya4242/Kops-Kubernetes/assets/113757574/23899d31-b70e-4634-9b5b-3a0efb14932b)

Cheers! 🎉 You can now access your deployed website.

Open your browser and navigate to your domain or the appropriate service URL. Your application should be live and accessible.


### 17. Destroy a Kops Cluster
To completely remove a Kops-managed Kubernetes cluster, execute the following command:

```sh
kops delete cluster k8s-cluster.example.com --state=s3://<your-s3-bucket> --yes
```

## Conclusion

By following these steps, you will have set up a Kubernetes cluster on AWS with Kops, deployed your application, and configured NGINX Ingress for external access via Route 53.

## Troubleshooting
If you encounter an error such as: 
> [!NOTE]
> error: You must be logged in to the server (the server has asked for the client to provide credentials)

Run the following command to refresh your Kubernetes configuration:
```sh
kops export kubecfg --admin --state=s3://<your-s3-bucket>
```
