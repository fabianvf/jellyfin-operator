# jellyfin-operator
[![Build Status](https://travis-ci.org/fabianvf/jellyfin-operator.svg?branch=master)](https://travis-ci.org/fabianvf/jellyfin-operator)

This operator deploys the [jellyfin](https://jellyfin.readthedocs.io) media server into Kubernetes or OpenShift.

## Deploying the Operator
To deploy it, ensure your kubeconfig is properly set up (ie, that you can use `kubectl` or `oc` to connect to it), 
ensure you have `cluster-admin` permissions (you will need this to create CustomResourceDefinitions),
install [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) and
the [openshift python client](https://github.com/openshift/openshift-restclient-python/#installation), and run

```bash
ansible-playbook deploy.yml
```

If you'd rather deploy it by hand, you can instead run:

```bash
kubectl apply -f deploy/crds/apps_v1alpha1_jellyfin_crd.yaml
kubectl apply -f deploy/role.yaml
kubectl apply -f deploy/role_binding.yaml
kubectl apply -f deploy/service_account.yaml
kubectl apply -f deploy/operator.yaml
```

## Deploying the Jellyfin server
Once this is done, your operator will be running. In order to deploy an instance of jellyfin into your cluster,
all you need to do is create an instance of the `apps.fabianism.us/v1alpha1` `Jellyfin` resource. A basic example
Jellyfin definition exists in `deploy/crds/apps_v1alpha1_jellyfin_cr.yaml`. You can create it with

```bash
kubectl apply -f deploy/crds/apps_v1alpha1_jellyfin_cr.yaml
```

This will kick off the deployment of the Jellyfin server. To monitor the status of the deployment, run

```bash
kubectl describe jellyfin
```

and monitor the status field. After some amount of time, a field called `URL` should appear in the status,
this is the URL that the operator has generated for your server. You should be able to access the jellyfin
web interface and configure it from there.


### Deployment Options

The example deployment will only give you a very basic, ephemeral jellyfin. In order to get a real deployment,
you will at least need to add storage options. The following options are supported in the Jellyfin spec:

| name | description | default | required |
|-|-|-|-|
| image | The jellyfin image to deploy | docker.io/jellyfin/jellyfin:latest | no |
| volumes | The volumes to create/mount into your container. See [VolumeSpec](#volumespec) for more detail. | | yes |
| nodePort | If running on Kubernetes, the port on the host that Jellyfin should listen on. | 30001 | no |
| host | If running openshift, the host that Jellyfin will be accessible from | `http://jellyfin-{namespace}.{openshift subdomain}` | no |


#### VolumeSpec

The `volumes` field is a map of key:value pairs, where each key is a volume name and the value is a map of key:value pairs defining
how the volume is to be created/mounted/accessed.

For configuration persistence, only one volume is required, the `config` volume. It must be mounted to `/config` with read-write permissions.

All volumes accept the following options:

| name | description | default | required |
|-|-|-|-|
| type | The type of volume. Choices are [`PersistentVolumeClaim`, `EmptyDir`] | | yes |
| mountPath | The full path to mount in the container. | The name of the volume (ie, `config` -> `/config`). | no |
| claimName | The name of the `PersistentVolumeClaim` to use. | The name of the volume. | no |
| size | The size of the volume to be created |  | When `type` is `PersistentVolumeClaim` and `create` is `true`. |
| create | Whether the `PersistentVolumeClaim` be created if it does not exist. | no | no |
| storageClassName | The name of the storage class to set for a `PersistentVolumeClaim` | The default storageClass for the cluster. | no |
| accessModes | The list of accessModes for a volume. Options are [`ReadWriteOnce`, `ReadWriteMany`, `ReadOnlyMany`] | [`ReadWriteOnce`] | no |
| readOnly | Whether the volume should be mounted read only | false | no |

#### Examples

Below are a few examples of some more complex Jellyfins

##### Using existing PersistentVolumeClaims
```yaml
apiVersion: apps.fabianism.us/v1alpha1
kind: Jellyfin
metadata:
  name: existing-persistent-jellyfin
spec:
  image: docker.io/jellyfin/jellyfin:latest
  volumes:
    config:
      type: PersistentVolumeClaim
      claimName: jellyfin-config
      create: no
    media:
      type: PersistentVolumeClaim
      claimName: jellyfin-media
      create: no
      mountPath: /data/media
```

##### Using more than one media volume

```yaml
apiVersion: apps.fabianism.us/v1alpha1
kind: Jellyfin
metadata:
  name: existing-persistent-jellyfin
spec:
  image: docker.io/jellyfin/jellyfin:latest
  volumes:
    # This is an existing claim
    config:
      type: PersistentVolumeClaim
      claimName: jellyfin-config
      create: no
    # But these media claims are being created
    tv:
      type: PersistentVolumeClaim
      claimName: jellyfin-tv
      create: yes
      mountPath: /data/media/tv
    movies:
      type: PersistentVolumeClaim
      claimName: jellyfin-movies
      create: yes
      mountPath: /data/media/movies
```
