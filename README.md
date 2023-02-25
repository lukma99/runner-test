# runner-test

##

Guide: https://github.com/actions/actions-runner-controller/blob/master/docs/quickstart.md

Run the following steps:
```shell
k3d cluster create github-runner-test

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml

helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

# Generate new token here with repo scope and copy the generated string: https://github.com/settings/tokens/new

helm upgrade --install --namespace actions-runner-system --create-namespace\
  --set=authSecret.create=true\
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE"\
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```


Apply the following file: `kubectl apply -f runnerdeployment.yaml`
```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runnerdeploy
spec:
  replicas: 1
  template:
    spec:
      repository: USER/REPONAME # Replace with your repository
```



Apply the following file and commands: 
```
kubectl apply -f ci-cd-role.yaml # yaml from below
kubectl create serviceaccount ci-cd
kubectl create rolebinding ci-cd --role ci-cd --serviceaccount=default:ci-cd
```

```yaml
# This file includes a minimal Role which can deploy this application.
# Specifically, the following operations were tested:
# * helm install --wait
# * helm upgrade --wait
# * helm list
# * helm uninstall
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-cd
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
    - configmaps
    - pods
    - secrets
    - services
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
    - apps
  resources:
    - pods
    - deployments
    - replicasets  # needed for 'helm upgrade --wait'
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
    - autoscaling
  resources:
    - horizontalpodautoscalers
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
    - networking.k8s.io
  resources:
    - ingresses
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
    - monitoring.coreos.com
  resources:
    - prometheusrules
    - servicemonitors
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
    - networking.k8s.io
  resources:
    - networkpolicies
  verbs:
    - create
    - delete
    - get
    - list
    - patch
    - update
    - watch
- apiGroups:
  - cert-manager.io
  resources:
  - certificates
  verbs:
  - get
  - list
  - watch
```


Run the following script as a whole (Guide [here](https://docs.armory.io/continuous-deployment/armory-admin/manual-service-account/):
```shell
SERVICE_ACCOUNT_NAME=ci-cd
CONTEXT=$(kubectl config current-context)
NAMESPACE=default

NEW_CONTEXT=ci-cd-context
KUBECONFIG_FILE="kubeconfig-ci-cd"


SECRET_NAME=$(kubectl get serviceaccount ${SERVICE_ACCOUNT_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.secrets[0].name}')
TOKEN_DATA=$(kubectl get secret ${SECRET_NAME} \
  --context ${CONTEXT} \
  --namespace ${NAMESPACE} \
  -o jsonpath='{.data.token}')

TOKEN=$(echo ${TOKEN_DATA} | base64 -d)

# Create dedicated kubeconfig
# Create a full copy
kubectl config view --raw > ${KUBECONFIG_FILE}.full.tmp
# Switch working context to correct context
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp config use-context ${CONTEXT}
# Minify
kubectl --kubeconfig ${KUBECONFIG_FILE}.full.tmp \
  config view --flatten --minify > ${KUBECONFIG_FILE}.tmp
# Rename context
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  rename-context ${CONTEXT} ${NEW_CONTEXT}
# Create token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-credentials ${CONTEXT}-${NAMESPACE}-token-user \
  --token ${TOKEN}
# Set context to use token user
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --user ${CONTEXT}-${NAMESPACE}-token-user
# Set context to correct namespace
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  set-context ${NEW_CONTEXT} --namespace ${NAMESPACE}
# Flatten/minify kubeconfig
kubectl config --kubeconfig ${KUBECONFIG_FILE}.tmp \
  view --flatten --minify > ${KUBECONFIG_FILE}
# Remove tmp
rm ${KUBECONFIG_FILE}.full.tmp
rm ${KUBECONFIG_FILE}.tmp
```

Run the following command:
```shell
base64 kubeconfig-ci-cd > data.b64
```
Copy the contents of `data.b64` to `Repository secrets` to https://github.com/USERNAME/REPONAME/settings/secrets/actions and call it `KUBECONFIG_BASE_64`.
