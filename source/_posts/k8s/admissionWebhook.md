---
title: admissionWebhook
author: falser101
date: 2023-11-21
category:
  - k8s
tag:
  - admissionWebhook
---

[原文链接](https://cloudnative.to/blog/mutating-admission-webhook/#:~:text=Admission%20webhook%20%E6%98%AF%E4%B8%80%E7%A7%8D%E7%94%A8%E4%BA%8E%E6%8E%A5%E6%94%B6%E5%87%86%E5%85%A5%E8%AF%B7%E6%B1%82%E5%B9%B6%E5%AF%B9%E5%85%B6%E8%BF%9B%E8%A1%8C%E5%A4%84%E7%90%86%E7%9A%84%20HTTP%20%E5%9B%9E%E8%B0%83%E6%9C%BA%E5%88%B6%E3%80%82%20%E5%8F%AF%E4%BB%A5%E5%AE%9A%E4%B9%89%E4%B8%A4%E7%A7%8D%E7%B1%BB%E5%9E%8B%E7%9A%84%20admission%20webhook%EF%BC%8C%E5%8D%B3,%E3%80%82%20Mutating%20admission%20webhook%20%E4%BC%9A%E5%85%88%E8%A2%AB%E8%B0%83%E7%94%A8%E3%80%82%20%E5%AE%83%E4%BB%AC%E5%8F%AF%E4%BB%A5%E6%9B%B4%E6%94%B9%E5%8F%91%E9%80%81%E5%88%B0%20API%20%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%9A%84%E5%AF%B9%E8%B1%A1%E4%BB%A5%E6%89%A7%E8%A1%8C%E8%87%AA%E5%AE%9A%E4%B9%89%E7%9A%84%E8%AE%BE%E7%BD%AE%E9%BB%98%E8%AE%A4%E5%80%BC%E6%93%8D%E4%BD%9C%E3%80%82)
## 什么是准入 Webhook？

准入 Webhook 是一种用于接收准入请求并对其进行处理的 HTTP 回调机制。 可以定义两种类型的准入 Webhook， 即验证性质的准入 Webhook 和变更性质的准入 Webhook。 变更性质的准入 Webhook 会先被调用。它们可以修改发送到 API 服务器的对象以执行自定义的设置默认值操作。

在完成了所有对象修改并且 API 服务器也验证了所传入的对象之后， 验证性质的 Webhook 会被调用，并通过拒绝请求的方式来强制实施自定义的策略。

Kubernetes 中的准入控制 webhook 的执行流程如下：
1. 当用户尝试创建、修改或删除 Kubernetes 资源时，API 服务器会将请求发送到准入控制器。
2. 准入控制器会将请求发送到 webhook。
3. webhook 会对请求进行处理，并返回一个响应。
4. API 服务器会根据 webhook 的响应来处理请求。

准入控制 webhook 可以分为两种类型：

- 变更准入控制器（MutatingAdmissionWebhook）：在资源存储之前对资源进行修改。

- 验证准入控制器（ValidatingAdmissionWebhook）：在资源存储之前对资源进行验证。

webhook返回不同响应
- Webhook 允许请求的最简单响应示例：
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0="
  }
}
```
当允许请求时，mutating准入 Webhook 也可以选择修改传入的对象。 这是通过在响应中使用 patch 和 patchType 字段来完成的。 当前唯一支持的 patchType 是 JSONPatch，对于 patchType: JSONPatch，patch 字段包含一个以 base64 编码的 JSON patch 操作数组

例如：设置deployment的 spec.replicas 的单个补丁操作将是 [{"op": "add", "path": "/spec/replicas", "value": 3}]，如果以 Base64 形式编码，结果将是 W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0=

- Webhook 禁止请求的最简单响应示例：
```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "<value from request.uid>",
    "allowed": false,
    "status": {// 可自定义
      "code": 403,
      "message": "You cannot do this because it is Tuesday and your name starts with A"
    }
  }
}
```
当拒绝请求时，Webhook 可以使用 status 字段自定义 http 响应码和返回给用户的消息。 有关状态类型的详细信息，请参见 API 文档。 禁止请求的响应示例，它定制了向用户显示的 HTTP 状态码和消息

## 配置准入WebHook
你可以通过 `ValidatingWebhookConfiguration` 或者 `MutatingWebhookConfiguration` 动态配置哪些资源要被哪些准入 Webhook 处理。

以下是一个 `MutatingWebhookConfiguration` 示例，`ValidatingWebhookConfiguration Webhook` 配置与此类似。 有关每个配置字段的详细信息，请参阅 Webhook 配置部分。
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  objectSelector:
    matchLabels:
      hello: "true"
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"
  clientConfig:
    service:
      namespace: "example-namespace"
      name: "example-service"
      path: /mutate
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

当一个 API 服务器收到与 rules 相匹配的请求时， 该 API 服务器将按照 clientConfig 中指定的方式向 Webhook 发送一个 admissionReview 请求。

创建 Webhook 配置后，系统将花费几秒钟使新配置生效。

接下来演示如何开发一个自定义Webhook API并使用

### 开发Go API 服务端
代码来自[Github仓库](https://github.com/didil/k8s-hello-mutating-webhook)
```golang
func (app *App) HandleMutate(w http.ResponseWriter, r *http.Request) {
	admissionReview := &admissionv1.AdmissionReview{}

	// read the AdmissionReview from the request json body
	err := readJSON(r, admissionReview)
	if err != nil {
		app.HandleError(w, r, err)
		return
	}

	// unmarshal the pod from the AdmissionRequest
	pod := &corev1.Pod{}
	if err := json.Unmarshal(admissionReview.Request.Object.Raw, pod); err != nil {
		app.HandleError(w, r, fmt.Errorf("unmarshal to pod: %v", err))
		return
	}

	// add the volume to the pod
	pod.Spec.Volumes = append(pod.Spec.Volumes, corev1.Volume{
		Name: "hello-volume",
		VolumeSource: corev1.VolumeSource{
			ConfigMap: &corev1.ConfigMapVolumeSource{
				LocalObjectReference: corev1.LocalObjectReference{
					Name: "hello-configmap",
				},
			},
		},
	})

	// add volume mount to all containers in the pod
	for i := 0; i < len(pod.Spec.Containers); i++ {
		pod.Spec.Containers[i].VolumeMounts = append(pod.Spec.Containers[i].VolumeMounts, corev1.VolumeMount{
			Name:      "hello-volume",
			MountPath: "/etc/config",
		})
	}

	containersBytes, err := json.Marshal(&pod.Spec.Containers)
	if err != nil {
		app.HandleError(w, r, fmt.Errorf("marshall containers: %v", err))
		return
	}

	volumesBytes, err := json.Marshal(&pod.Spec.Volumes)
	if err != nil {
		app.HandleError(w, r, fmt.Errorf("marshall volumes: %v", err))
		return
	}

	// build json patch
	patch := []JSONPatchEntry{
		{
			OP:    "add",
			Path:  "/metadata/labels/hello-added",
			Value: []byte(`"OK"`),
		},
		{
			OP:    "replace",
			Path:  "/spec/containers",
			Value: containersBytes,
		},
		{
			OP:    "replace",
			Path:  "/spec/volumes",
			Value: volumesBytes,
		},
	}

	patchBytes, err := json.Marshal(&patch)
	if err != nil {
		app.HandleError(w, r, fmt.Errorf("marshall jsonpatch: %v", err))
		return
	}

	patchType := admissionv1.PatchTypeJSONPatch

	// build admission response
	admissionResponse := &admissionv1.AdmissionResponse{
		UID:       admissionReview.Request.UID,
		Allowed:   true,
		Patch:     patchBytes,
		PatchType: &patchType,
	}

	respAdmissionReview := &admissionv1.AdmissionReview{
		TypeMeta: metav1.TypeMeta{
			Kind:       "AdmissionReview",
			APIVersion: "admission.k8s.io/v1",
		},
		Response: admissionResponse,
	}

	jsonOk(w, &respAdmissionReview)
}
```
上述代码主要做了以下事情：
1. 将来自 Http 请求中的 AdmissionReview json 输入反序列化。
2. 读取 Pod 的 spec 信息。
3. 将 hello-configmap 作为数据源，添加 hello-volume 卷到 Pod。
4. 挂载卷至 Pod 容器中。
5. 以 JSON PATCH 的形式记录变更信息，包括卷的变更，卷挂载信息的变更。顺道为容器添加一个“hello-added=true”的标签。
6. 构建 json 格式的响应结果，结果中包含了这次请求中的被修改的部分。
### 创建k8s资源
在webhook项目下执行
```shell
$ kubectl apply -k k8s/deployment
# 日志
configmap/hello-configmap created
service/hello-webhook-service created
mutatingwebhookconfiguration.admissionregistration.k8s.io/hello-webhook.leclouddev.com created

$ kubectl apply -k k8s/other
# 日志
configmap/hello-configmap created
service/hello-webhook-service created
mutatingwebhookconfiguration.admissionregistration.k8s.io/hello-webhook.leclouddev.com created
```
### 生成TLS证书
Webhook API 服务器需要通过 TLS 方式通信。如果想将其部署至 Kubernetes 集群内，我们还需要证书。原作者的仓库使用的kubectl版本较低，原作者是通过job生成pod来生成相关证书的，我们可以直接执行脚本来生成，脚本内容如下：
```sh
#!/usr/bin/env sh

set -e

usage() {
  cat <<EOF
Generate certificate suitable for use with any Kubernetes Mutating Webhook.
This script uses k8s' CertificateSigningRequest API to a generate a
certificate signed by k8s CA suitable for use with any Kubernetes Mutating Webhook service pod.
This requires permissions to create and approve CSR. See
https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster for
detailed explantion and additional instructions.
The server key/cert k8s CA cert are stored in a k8s secret.
usage: ${0} [OPTIONS]
The following flags are required.
    --service          Service name of webhook.
    --webhook          Webhook config name.
    --namespace        Namespace where webhook service and secret reside.
    --secret           Secret name for CA certificate and server certificate/key pair.
The following flags are optional.
    --webhook-kind     Webhook kind, either MutatingWebhookConfiguration or
                       ValidatingWebhookConfiguration (defaults to MutatingWebhookConfiguration)
EOF
  exit 1
}

while [ $# -gt 0 ]; do
  case ${1} in
      --service)
          service="$2"
          shift
          ;;
      --webhook)
          webhook="$2"
          shift
          ;;
      --secret)
          secret="$2"
          shift
          ;;
      --namespace)
          namespace="$2"
          shift
          ;;
      --webhook-kind)
          kind="$2"
          shift
          ;;
      *)
          usage
          ;;
  esac
  shift
done

[ -z "${service}" ] && echo "ERROR: --service flag is required" && exit 1
[ -z "${webhook}" ] && echo "ERROR: --webhook flag is required" && exit 1
[ -z "${secret}" ] && echo "ERROR: --secret flag is required" && exit 1
[ -z "${namespace}" ] && echo "ERROR: --namespace flag is required" && exit 1

fullServiceDomain="${service}.${namespace}.svc"

# THE CN has a limit of 64 characters. We could remove the namespace and svc
# and rely on the Subject Alternative Name (SAN), but there is a bug in EKS
# that discards the SAN when signing the certificates.
#
# https://github.com/awslabs/amazon-eks-ami/issues/341
if [ ${#fullServiceDomain} -gt 64 ] ; then
  echo "ERROR: common name exceeds the 64 character limit: ${fullServiceDomain}"
  exit 1
fi

if [ ! -x "$(command -v openssl)" ]; then
  echo "ERROR: openssl not found"
  exit 1
fi

csrName=${service}.${namespace}
tmpdir=$(mktemp -d)
echo "creating certs in tmpdir ${tmpdir} "

cat <<EOF >> "${tmpdir}/csr.conf"
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${service}
DNS.2 = ${service}.${namespace}
DNS.3 = ${fullServiceDomain}
DNS.4 = ${fullServiceDomain}.cluster.local
EOF
echo "/CN=${fullServiceDomain}"
openssl genrsa -out "${tmpdir}/server-key.pem" 2048
#openssl req -new -key "${tmpdir}/server-key.pem" -subj "/CN=${fullServiceDomain}" -out "${tmpdir}/server.csr" -config "${tmpdir}/csr.conf"
openssl req -new -key "${tmpdir}/server-key.pem" -subj "/CN=system:node:${fullServiceDomain};/O=system:nodes" -out "${tmpdir}/server.csr" -config "${tmpdir}/csr.conf"
set +e
# clean-up any previously created CSR for our service. Ignore errors if not present.
if kubectl delete csr "${csrName}"; then
    echo "WARN: Previous CSR was found and removed."
fi
set -e

# create server cert/key CSR and send it to k8s api
cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${csrName}
spec:
  #signerName: kubernetes.io/kube-apiserver-client
  signerName: kubernetes.io/kubelet-serving
  groups:
  - system:authenticated
  request: $(base64 < "${tmpdir}/server.csr" | tr -d '\n')
  usages:
  - server auth
  - digital signature
  - key encipherment
EOF

set +e
# verify CSR has been created
while true; do
  if kubectl get csr "${csrName}"; then
      echo "CertificateSigningRequest create succsee"
      break
  fi
done
set -e

# approve and fetch the signed certificate . !! not working with k8s 1.19.1, running the command separately outside of the container / node
set +e
while true; do
  if kubectl certificate approve "${csrName}"; then
     echo "${csrName} certificate approve"
     break
  fi
done

set -e

set +e
# verify certificate has been signed
i=1
while [ "$i" -ne 10 ]
do
  serverCert=$(kubectl get csr "${csrName}" -o jsonpath='{.status.certificate}')
  if [ "${serverCert}" != '' ]; then
      break
  fi
  sleep 5
  i=$((i + 1))
done

set -e
if [ "${serverCert}" = '' ]; then
  echo "ERROR: After approving csr ${csrName}, the signed certificate did not appear on the resource. Giving up after 10 attempts." >&2
  exit 1
fi

echo "${serverCert}" | openssl base64 -d -A -out "${tmpdir}/server-cert.pem"

# create the secret with CA cert and server cert/key
kubectl create secret tls "${secret}" \
      --key="${tmpdir}/server-key.pem" \
      --cert="${tmpdir}/server-cert.pem" \
      --dry-run -o yaml |
  kubectl -n "${namespace}" apply -f -

#caBundle=$(base64 < /run/secrets/kubernetes.io/serviceaccount/ca.crt  | tr -d '\n')
caBundle=$(cat ${tmpdir}/server-cert.pem)
set +e
# Patch the webhook adding the caBundle. It uses an `add` operation to avoid errors in OpenShift because it doesn't set
# a default value of empty string like Kubernetes. Instead, it doesn't create the caBundle key.
# As the webhook is not created yet (the process should be done manually right after this job is created),
# the job will not end until the webhook is patched.
while true; do
  echo "INFO: Trying to patch webhook adding the caBundle."
  if kubectl patch "${kind:-mutatingwebhookconfiguration}" "${webhook}" --type='json' -p "[{'op': 'add', 'path': '/webhooks/0/clientConfig/caBundle', 'value':'${serverCert}'}]"; then
      break
  fi
  echo "INFO: webhook not patched. Retrying in 5s..."
  sleep 5
done
```

执行脚本生成证书：
- --service：对应webhook api服务的service name
- --webhook：我们创建的MutatingWebhookConfiguration的webhooks的name
- --secret：我们的webhook api服务pod所需要挂载的secret的名称
- --namespace：命名空间

```shell
./generate_certificate.sh  --service hello-webhook-service --webhook hello-webhook.leclouddev.com --secret hello-tls-secret --namespace default
```

```shell
# 输出日志
creating certs in tmpdir /var/folders/m6/11gz2m9x1m11lkts0h83x8800000gn/T/tmp.ebrOVBA0 
/CN=hello-webhook-service.default.svc
Error from server (NotFound): certificatesigningrequests.certificates.k8s.io "hello-webhook-service.default" not found
certificatesigningrequest.certificates.k8s.io/hello-webhook-service.default created
NAME                            AGE   SIGNERNAME                      REQUESTOR          REQUESTEDDURATION   CONDITION
hello-webhook-service.default   0s    kubernetes.io/kubelet-serving   kubernetes-admin   <none>              Pending
CertificateSigningRequest create succsee
certificatesigningrequest.certificates.k8s.io/hello-webhook-service.default approved
hello-webhook-service.default certificate approve
W1122 10:40:41.297220   55111 helpers.go:692] --dry-run is deprecated and can be replaced with --dry-run=client.
secret/hello-tls-secret configured
INFO: Trying to patch webhook adding the caBundle.
mutatingwebhookconfiguration.admissionregistration.k8s.io/hello-webhook.leclouddev.com patched
```

### 创建一个带hello=true标签的pod
```shell
kubectl run busybox-1 --image=busybox  --restart=Never -l=app=busybox,hello=true -- sleep 3600
```

查看webhook api的pod日志
```shell
kubectl logs hello-webhook-deployment-5957bbb8bf-mkrzv
```

输出日志
```shell
Listening on port 8000
2023/11/22 02:46:16 [hello-webhook-deployment-5957bbb8bf-mkrzv/JbYpz7vK1i-000001] "POST https://hello-webhook-service.default.svc:443/mutate?timeout=10s HTTP/1.1" from 10.244.235.192:46335 - 200 1393B in 1.844861ms
```

查看busybox-1的volume,可以看到cm已经挂载进去了
```yaml
  volumes:
  - configMap:
      defaultMode: 420
      name: hello-configmap
    name: hello-volume
```

### 创建一个不带hello=true标签的则不会挂载
```shell
kubectl run busybox-2 --image=busybox --restart=Never -l=app=busybox -- sleep 3600
```
查看busybox-2的yaml并没有对应的volume