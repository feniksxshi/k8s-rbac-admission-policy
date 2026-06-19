# RBAC + Policy Enforcement for K8s Cluster
## RBAC
![alt text](image.png)

Test: 
![alt text](image-3.png)
![alt text](image-2.png)

## Policy Enforcement
![alt text](image-4.png)
![alt text](image-6.png)

Reviewing the current `rollout.yaml`
- No `hostNetwork` -> Not configured, defaults to `false`
- No `latest` image -> Uses `v0.0.1-0b93af4`
	![alt text](image-7.png)
- CPU/memory limits -> Both are configured 
	![alt text](image-8.png)
- Must run non-root  -> No securityContext; image defaults to root ❌ \
	Fix: 
	![alt text](image-9.png)
	Also modify this part in 4 constraints files:
	![alt text](image-10.png)

Test: Use Kubernetes server-side dry-run. It triggers Gatekeeper admission without creating a real Pod. 
1. Reject `:latest`
	```bash
	kubectl set image -f /tmp/test-pod.yaml \
	  test=busybox:latest --local -o yaml |
	kubectl apply --dry-run=server -f -
	```
	![alt text](image-11.png)

2. Reject missing resource limits
	```bash
	kubectl patch --local -f tmp/test-pod.yaml \
	  --type=json \
	  -p='[{"op":"remove","path":"/spec/containers/0/resources/limits"}]' \
	  -o yaml |
	kubectl apply --dry-run=server -f -
	```
	![alt text](image-12.png)

3. Reject runAsUser: 0
	```bash
	kubectl patch --local -f tmp/test-pod.yaml \
	  --type=json \
	  -p='[
	    {
	      "op": "replace",
	      "path": "/spec/containers/0/securityContext/runAsUser",
	      "value": 0
	    }
	  ]' \
	  -o yaml |
	kubectl apply --dry-run=server -f -
	```
	![alt text](image-13.png)

3. Reject host network
	```bash
	kubectl patch --local -f tmp/test-pod.yaml \
	  --type=json \
	  -p='[
	    {
	      "op": "add",
	      "path": "/spec/hostNetwork",
	      "value": true
	    }
	  ]' \
	  -o yaml |
	kubectl apply --dry-run=server -f -
	```
	![alt text](image-14.png)

4. Valid Pod (version pinned + resource limits + non-root user)
	```bash
	kubectl apply --dry-run=server -f tmp/test-pod.yaml
	```
	![alt text](image-15.png)
	Produce no Gatekeeper violations.
	![alt text](image-16.png)

## Custom ConstraintTemplate
Docs: https://open-policy-agent.github.io/gatekeeper/website/docs/howto/

Code: 
- gatekeeper/constraints/enforce-naming-convention.yaml
- gatekeeper/templates/template-naming-convention.yaml

Test the Deployment policy:
```bash
kubectl -n demo create deployment gatekeeper-test \
  --image=busybox:1.36.1 --dry-run=server -o yaml
```
![alt text](image-17.png)

# ESO + Trivy + Cosign
> ESO: External Secrets Operator

Create AWS Crendentials in Minikube
![alt text](image-19.png)
![alt text](image-20.png)

![alt text](image-21.png)

## TC07–TC09: test rotate quan trọng nhất
Đổi value trong AWS Secrets Manager Console:
![alt text](image-23.png)

Sau khi đổi value mới: 
![alt text](image-24.png)
![alt text](image-25.png)

Kiểm tra pod không restart: \
Before
![alt text](image-26.png)
After
![alt text](image-27.png)
- Tên pods không đổi
- RESTARTS = 0
- POD vẫn running

Kiểm tra file mount:
![alt text](image-28.png)
---
Test
![alt text](image-30.png)
![alt text](image-32.png)

Kiểm tra repo không lộ credentials
![alt text](image-33.png)

## Trivy + Cosign
```
build/push → Trivy scan → scan pass → Cosign sign
```
![alt text](image-34.png)
- private key -> không commit
- signing/cosign.pub  -> public - được commit
--- 
Latest commit on origin/main
```bash
git fetch origin main
git rev-parse --short origin/main
```

Commit currently compared by Argo CD
```bash
kubectl -n argocd get application root \
  -o jsonpath='{.status.sync.revision}{"\n"}'
```

Force a refresh:
```bash
kubectl -n argocd annotate application root \
  argocd.argoproj.io/refresh=hard --overwrite

kubectl -n argocd get application root -w
```
![alt text](image-5.png)