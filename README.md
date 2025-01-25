### updating the server
dnf update

update k3s: curl -sfL https://get.k3s.io | sh -

update awx: change operator version in kustomization.yaml
./kustomize build . | kubectl apply -f -

### replacing the SSL certificate after expiration

place the cert + CA in tls.crt

place the key in nano tls.key

Put the base64 encoded version in the awx-tls.yaml at the respective places

cat tls.crt | base64

cat tls.key | base64


reapply the config:

./kustomize build . | kubectl apply -f -

### basic preparations for deploying awx

Install AlmaLinux 9, git.

Clone this repository into /root/... and adjust parameters accordingly.

```bash
curl -sfL https://get.k3s.io | sh -
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
kubectl create namespace awx
./kustomize build . | kubectl apply -f -

kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```

### Prepare Traefik

To enable redirection, you need to deploy a middleware with [redirectScheme](https://doc.traefik.io/traefik/v2.0/middlewares/redirectscheme/).

Since this can be referenced from other namespaces, you will create it in the `default` namespace for ease of sharing.

```bash
cat <<EOF > middleware.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
spec:
  redirectScheme:
    scheme: https
    permanent: true
EOF

kubectl -n default apply -f middleware.yaml
kubectl -n default get middleware
```

#### Patch your AWX using Kustomize

In this repository, Kustomize was used to deploy AWX. If you still have the files you used for your first deployment, it is easy to use them again to modify AWX.

Add these two lines to your `awx.yaml`,

```yaml
spec:
  ...
  ingress_annotations: |     👈👈👈
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd     👈👈👈
```

then invoke `apply` again. Once the command has been invoked, then AWX Operator will start to modify related resources. Note that the AWX Pod will be recreated, so AWX will be temporarily disabled.

```bash
$ kubectl apply -k base
namespace/awx unchanged
secret/awx-admin-password unchanged
secret/awx-postgres-configuration unchanged
secret/awx-secret-tls configured
persistentvolume/awx-postgres-13-volume unchanged
persistentvolume/awx-projects-volume unchanged
persistentvolumeclaim/awx-projects-claim unchanged
awx.awx.ansible.com/awx configured     👈👈👈
```

Once this completed, the logs of `deployments/awx-operator-controller-manager` end with:

```txt
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager --tail=100
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----
PLAY RECAP *********************************************************************
localhost                  : ok=54   changed=0    unreachable=0    failed=0    skipped=37   rescued=0    ignored=0
```

You can confirm that the annotations will be added to the Ingress resource.

```bash
$ kubectl -n awx get ingress awx-ingress -o=jsonpath='{.metadata.annotations}' | jq
{
  ...
  "traefik.ingress.kubernetes.io/router.middlewares": "default-redirect@kubernetescrd"
}
```

Now the redirection should be working. Go to `http://awx.example.com/` or the hostname you specified and make sure you are redirected to `https://`.

### AWX bugfixes
applied to awx.yaml https://github.com/ansible/awx/issues/14693
