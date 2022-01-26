In the meantime, how to configure nvidia gpu in kubernetes was very tricky. To provision a simple GPU pod, we had to build essential components such as drivers, container runtimes, device plugins, and monitoring. Recently, functions such as NVIDIA GPU Feature Discovery have been added to reduce the hassle for GPU configuration, but the system engineer’s thirst was not quenched because the NVIDIA Driver and Runtime had to be configured as it is. And since this part is difficult for conventional system engineers to access, a lot of focus has been placed on engineers with experience in GPU configuration.

<img title="" src="/assets/img/infra/2022-01-25-22-59-55-image.png" alt="" data-align="left">

###### Fig1 [https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes](https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes/)

Complex configuration to use GPUs in Kubernetes was all solved with the advent of NVIDIA GPU Operator. If you look at the NVIDIA GPU Operator-related content provided by the NVIDIA blog (Fig1), you can see that Driver, Runtime, Device Plugin, and Monitoring are all configured as containers. As a result, you no longer need to install the NVIDIA Driver in your OS. (It reminds me of the days when I stayed up all night to install the GPU Driver in the OS.)

![](/assets/img/infra/2022-01-25-23-01-24-image.png)

###### Fig2 https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes/

Additionally, a feature called MIG was introduced on the NVIDIA A100 GPU. In the meantime, GPUs were limited to a function called Virtual GPU to use parallel processing functions like Hyper Threading of Intel CPUs. This was a function to use the GPU memory in parallel after fragmenting it, and to use it, an additional license fee was required, and it was also very difficult to manage because the GPU driver had to be installed in the Hypervisor and Guest OS. Basically, since the memory is fragmented and used, there is a problem that the size of a model that can be trained in Machine Learning or Deep Learning is reduced, so it was used limitedly in ML/DL and used a lot in Virtual Desktop Infrastructure.

![](/assets/img/infra/2022-01-25-23-02-10-image.png)

###### Fig3 [Using GPUs with Virtual Machines on vSphere – Part 3: Installing the NVIDIA Virtual GPU Technology - Virtualize Applications](https://blogs.vmware.com/apps/2018/09/using-gpus-with-virtual-machines-on-vsphere-part-3-installing-the-nvidia-grid-technology.html)

The MIG function introduced in the A100 GPU partially solves the above problem by supporting virtualization on the GPU. One GPU can be divided into up to 7 instances, and since the GPU layer provides virtualization functions, the hypervisor part can be reduced, so there are management and performance gains, but the fragmentation of VRAM based on the instance is a disappointing part. This is also an unavoidable part of the GPU architecture.

![](/assets/img/infra/2022-01-25-23-03-32-image.png)

###### Fig4 [NVIDIA Multi-Instance GPU User Guide :: NVIDIA Tesla Documentation](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

## Prerequisite

Now, I will explain how to configure NVIDIA GPU Operator. First, let’s check the essential requirements to configure NVIDIA GPU Operator.

- Kubernetes v1.19+

- NVIDIA GPU Operator v1.9

- Linux distributions  
  Ubuntu 18.04.z, 20.04.z LTS, Red Hat Enterprise Linux CoreOS (RHCOS) for use with OpenShift 4.8 and 4.9, **CentOS 7**

***If you look at the above requirements, you can see that CentOS 8 version is not supported. Rocky Linux is not supported as it is a version built based on the CentOS 8 version. I suffered a lot because I couldn’t check this part beforehand. Fortunately, I found a Ubi (Universal Base Image) image and it worked out for me.***

## Install NVIDIA GPU Operator

Now that the requirements have been confirmed, let’s install the GPU Operator. The easiest way to install is to use helm.

### Install helm

If you don’t have helm installed on your working node, make sure to install helm first. After the helm installation is complete, add the NVIDIA helm repository.

```console
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
   && chmod 700 get_helm.sh \
   && ./get_helm.sh
$ helm repo add nvidia https://nvidia.github.io/gpu-operator \
   && helm repo update
```

### Install NVIDIA GPU Operator simply

GPU Operator in general environment can be installed simply with the following command. However, as described in the prerequisite, **Rocky Linux and the MIG option of the A100 GPU** are selected for the basic OS, so we need to proceed with more detailed settings.

```console
$ helm install --wait \
     -n gpu-operator-resources --create-namespace \
     gpu-operator \
     nvidia/gpu-operator
```

### Specific option needs for MIG and Rocky Linux

When installing gpu operator using helm, the following additional settings are required to use MIG and Rocky linux. As you can see from the options below, all versions are ubi8. ubi stands for Universal Base Image, and for more information, please refer to the [Redhat official website](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image).

- mig.strategy=mixed
  (You can select mixed to use the A100 GPU’s MIG and single to disable it.)

- driver.repository=**homelab.net**/nvcr.io/nvidia
  (If you search for the nvidia gpu driver in the [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/driver/tags), the tag includes all os types. This tag is automatically selected when the pod is launched, and it can find the CentOS 8 version, but not Rocky Linux. So, you need to make it look for the image in the Private Repository.)

- driver.version=450.156.00
  (Select the GPU Driver to use. Depending on the CUDA version to be used, the GPU driver can be selected, and [**if version 470 is used, additional memory that is discarded depending on the instance selection can be used**](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#a100-profiles). Related links can be found on the NVIDIA MIG manual homepage.)

- driver.manager.version=v0.2.0-ubi8

- toolkit.version=1.7.2-ubi8

- dcgm.version=2.3.1-ubi8

- operator.version=v1.9.0-ubi8

- operator.initContainer.version=cuda:11.2.1-base-ubi8

- validator.version=v1.9.0-ubi8

※ You can change the docker image tag as follows.

```console
$ docker pull nvcr.io/nvidia/driver:450.156.00-centos8
$ docker tag \
    nvcr.io/nvidia/driver:450.156.00-centos8 \
    homelab.net/nvcr.io/nvidia/driver:450.156.00-rocky8
$ docker push homelab.net/nvcr.io/nvidia/driver:450.156.00-rocky8
```

## Install NVIDIA GPU Operator for A100 MIG and Rocky

All ready to use MIG and Rocky Linux on the A100 GPU. Try redeploying it using helm.

```console
$ helm install --wait \
      -n gpu-operator-resources --create-namespace \
      --version v1.9.0 \
      --set mig.strategy=mixed \
      --set driver.repository=homelab.net/nvcr.io/nvidia \
      --set driver.version=450.156.00 \
      --set driver.manager.version=v0.2.0-ubi8 \
      --set toolkit.version=1.7.2-ubi8 \
      --set dcgm.version=2.3.1-ubi8 \
      --set operator.version=v1.9.0-ubi8 \
      --set operator.initContainer.version=cuda:11.2.1-base-ubi8 \
      --set validator.version=v1.9.0-ubi8 \
      gpu-operator \
      nvidia/gpu-operator
```

Don’t forget it. **f you use MIG’s mixed mode, node labels for instance slicing must be set in advance**. Use the command below to proceed with the node label.

```console
$ kubectl get node \
    --selector='!node-role.kubernetes.io/master' \
    -o name | xargs -I {} kubectl label {} nvidia.com/mig.config=all-balanced \
        --overwrite
```

## Results

When all settings are completed, you can check the settings below.

```console
$ kubectl -n gpu-operator-resources get pods
NAME                                                        READY   STATUS      RESTARTS    AGE
gpu-feature-discovery-28wq6                                 1/1     Running     0           1h
gpu-operator-854d7c5c770lzsxc                               1/1     Running     0           1h
gpu-operator-node-feature-discovery-master-868f8cb84c-hvp9q 1/1     Running     0           1h
gpu-operator-node-feature-discovery-worker-26d46            1/1     Running     0           1h
nvidia-container-toolkit-daemonset-2d8xs                    1/1     Running     0           1h
nvidia-cuda-validator-27fdv                                 0/1     Completed   0           1h
nvidia-dcgm-4lvvf                                           1/1     Running     0           1h
nvidia-dcgm-exporter-28rn7                                  1/1     Running     0           1h
nvidia-device-plugin-daemonset-2gp6q                        1/1     Running     0           1h
nvidia-driver-daemonset-4bkgg                               1/1     Running     0           1h
nvidia-mig-manager-2txtt                                    1/1     Running     0           1h
nvidia-operator-validator-2jx7t                             1/1     Running     0           1h
```

# Conclusion

You have seen how to configure to deploy gpu pods in kubernetes using NVIDIA gpu operator. Using the gpu operator makes it very easy to change the nvidia driver and cuda driver versions in the future because you do not have to configure the OS-dependent nvidia driver and container runtime separately. This can be a huge advantage as there are more nodes and GPU types to manage.

I hope it will be helpful to those who are looking for related content. And if you have saved a lot of time with this content, please donate a cup of coffee. (Please help me to write while eating ice americano at a local cafe.)



![](/Users/awslife/Blogs/awslife.github.io/assets/7e6a7443ca4b279326519f6012b002ebff25eb3c.png)



[buymeacoffee](https://buymeacoffee.com/7ov2xm5)

# References

[NVIDIA GPU Operator: Simplifying GPU Management in Kubernetes | NVIDIA Developer Blog](https://developer.nvidia.com/blog/nvidia-gpu-operator-simplifying-gpu-management-in-kubernetes/)

[Kubernetes provides access to special hardware resources such as NVIDIA GPUs, NICs, Infiniband adapters and other…](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)

[Dependency errors when installing older versions of nvidia-container-runtime on Debian-based systems](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/platform-support.html#linux-distributions)

[This edition of the user guide describes the Multi-Instance GPU feature of the NVIDIA® Ampere architectre.](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
