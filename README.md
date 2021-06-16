Helpful Resources:

[Repository](https://github.com/pangeo-data/terraform-deploy) and [blog post](https://medium.com/pangeo/terraform-jupyterhub-aws-34f2b725f4fd) that has a Terraform config that creates a JupyterHub cluster on AWS
  

Issues I’m having: 

 - Labelling and tainting worker groups (not node groups)
 - deleting VPN subnets when `terraform destroy` is run. Open issue [here](https://github.com/hashicorp/terraform-provider-aws/issues/9495) 
	 - Try to delete manually in [console](https://aws.amazon.com/premiumsupport/knowledge-center/troubleshoot-dependency-error-delete-vpc/), need to delete load balancer as well to delete subnets
 - Enabling cluster autoscaling through Terraform. The above blog post has a config that has autoscaling, but I had trouble getting it to work.

Common errors:  
  
1. [https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1234](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1234)  
  
2. If you are having issues with https, try having 
````
proxy:
  https:
    enabled: false
````  

in the YAML file.

Support for managed node group taints was added to the EKS Terraform module within the last few [weeks](https://github.com/terraform-aws-modules/terraform-aws-eks/pull/1424), but managed node groups can’t scale to 0, so a worker group config similar to [this](https://github.com/pangeo-data/terraform-deploy) repo might work better.  

Forked version of the blog post repo with updated code: https://github.com/adhil0/terraform-deploy. Check [aws-examples](https://github.com/adhil0/terraform-deploy/tree/master/aws-examples)/**minimal-deployment-tutorial**/ for the code.
  
 ------------
 How to manually build an autoscaling EKS Cluster for JupyterHub:
 
[Autoscaler Steps for your Reference](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)

 1.  

    eksctl create cluster \
    --name <my-cluster> \
    --version <1.18> \
    --region <us-west-2> \
    --nodegroup-name <linux-nodes> \
    --nodes <3> \
    --nodes-min <1> \
    --nodes-max <4> \
    --with-oidc \
    --ssh-access \
    --ssh-public-key <name-of-ec2-keypair> \
    --managed \
    --asg-access
    
2.

    kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin  —-user=system:anonymous

3.
Policy name is in the IAM Role for node group (eksctl-autoscaling-test-cluster-n-NodeInstanceRole-...). For me it was called AmazonEKSClusterAutoscalerPolicy.

    eksctl create iamserviceaccount \
      --cluster=<my-cluster> \
      --namespace=kube-system \
      --name=cluster-autoscaler \
      --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/<AmazonEKSClusterAutoscalerPolicy> \
      --override-existing-serviceaccounts \
      --approve

4.

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

5.

    kubectl annotate serviceaccount cluster-autoscaler \
      -n kube-system \
      eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/<AmazonEKSClusterAutoscalerRole>

Step 6 may not be necessary. Eksctl might take care of this on your behalf.

6.

    kubectl patch deployment cluster-autoscaler \
      -n kube-system \
      -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": “false"}}}}}'

For step 7, use step 4 [here](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html#ca-deploy)

7.

    kubectl -n kube-system edit deployment.apps/cluster-autoscaler

8.

    kubectl set image deployment cluster-autoscaler \
      -n kube-system \
      cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v<1.19.n>
      
Step 9 verifies that the autoscaler is working
9.

    kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

10. Add labels and taints. `kubectl label nodes <your-node-name> hub.jupyter.org/node-purpose=user` `kubectl taint nodes <your-node-name> hub.jupyter.org/dedicated=user:NoSchedule`

11.

    helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
    helm repo update
12.

    # Suggested values: advanced users of Kubernetes and Helm should feel
    # free to use different values.
    RELEASE=jhub
    NAMESPACE=jhub
    
    helm upgrade --cleanup-on-fail \
      --install $RELEASE jupyterhub/jupyterhub \
      --namespace $NAMESPACE \
      --create-namespace \
      --version=0.9.0 \
      --values config.yaml
13.

    kubectl get service --namespace jhub
    

   
14.

    eksctl delete cluster —name <my-cluster>



