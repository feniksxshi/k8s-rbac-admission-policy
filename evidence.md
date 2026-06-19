## 1. Kubernetes RBAC

### 1.1 Access model

![RBAC assignment requirements](assets/image.png)

The expected authorization matrix is shown below.

![Expected RBAC authorization results](assets/image-3.png)

### 1.2 Authorization test

Authorization was validated with user impersonation through `kubectl auth can-i`:

```bash
kubectl auth can-i create deployments.apps -n demo --as=alice
kubectl auth can-i create deployments.apps -n kube-system --as=alice
kubectl auth can-i get pods -A --as=bob
kubectl auth can-i delete nodes --as=carol
```

Observed results:

- Alice can create deployments in `demo`.
- Alice cannot create deployments in `kube-system`.
- Bob can read pods across all namespaces.
- Carol cannot delete cluster nodes.

![Observed RBAC authorization results](assets/image-2.png)

## 2. OPA Gatekeeper admission controls

### 2.1 Policy scope

The Gatekeeper configuration evaluates these workload properties:

- Disallowed image tag: `latest`
- Required CPU and memory limits
- Non-root execution
- Disabled host networking

![Gatekeeper policy requirements](assets/image-4.png)

![Installed Gatekeeper resources](assets/image-6.png)

### 2.2 Application manifest review

The application rollout was reviewed before admission testing:

- `hostNetwork` is not configured and therefore defaults to `false`.
- The container image uses a pinned version rather than `latest`.
- CPU and memory limits are configured.
- A container security context was added to require non-root execution.

![Pinned application image](assets/image-7.png)

![Application resource limits](assets/image-8.png)

![Non-root application security context](assets/image-9.png)

The relevant Gatekeeper constraints were also adjusted to target the required workload resources.

![Gatekeeper constraint match configuration](assets/image-10.png)

### 2.3 Admission test: disallowed `latest` tag

```bash
kubectl set image -f tmp/test-pod.yaml \
  test=busybox:latest --local -o yaml |
kubectl apply --dry-run=server -f -
```

Result: Gatekeeper detected the `latest` tag and emitted a warning. The resource remained admissible because this constraint uses `enforcementAction: warn`.

![Gatekeeper warning for the latest image tag](assets/image-11.png)

### 2.4 Admission test: missing resource limits

```bash
kubectl patch --local -f tmp/test-pod.yaml \
  --type=json \
  -p='[{"op":"remove","path":"/spec/containers/0/resources/limits"}]' \
  -o yaml |
kubectl apply --dry-run=server -f -
```

Result: Gatekeeper detected the missing CPU and memory limits and emitted a warning. The resource remained admissible because this constraint uses `enforcementAction: warn`.

![Gatekeeper warning for missing resource limits](assets/image-12.png)

### 2.5 Admission test: root execution

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

Result: Gatekeeper detected `runAsUser: 0` and emitted a warning. The resource remained admissible because this constraint uses `enforcementAction: warn`.

![Gatekeeper warning for root execution](assets/image-13.png)

### 2.6 Admission test: host networking

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

Result: Gatekeeper denied the request because `hostNetwork: true` violates the enforced policy.

![Gatekeeper denial for host networking](assets/image-14.png)

### 2.7 Admission test: compliant pod

```bash
kubectl apply --dry-run=server -f tmp/test-pod.yaml
```

The compliant pod uses a pinned image, defines resource limits, runs as a non-root user, and does not enable host networking. It was admitted without a Gatekeeper violation.

![Compliant pod server-side dry-run](assets/image-15.png)

![No Gatekeeper violations for the compliant pod](assets/image-16.png)

## 3. Custom deployment naming policy

The custom policy is implemented in:

- `gatekeeper/templates/template-naming-convention.yaml`
- `gatekeeper/constraints/enforce-naming-convention.yaml`

It requires deployments in the `demo` namespace to start with either `api-` or `web-`.

The following test uses the invalid name `gatekeeper-test`:

```bash
kubectl -n demo create deployment gatekeeper-test \
  --image=busybox:1.36.1 --dry-run=server -o yaml
```

Result: the deployment was denied because its name did not have an allowed prefix.

![Custom deployment naming policy denial](assets/image-17.png)

## 4. External Secrets Operator and AWS Secrets Manager

### 4.1 AWS credentials and secret synchronization
AWS credentials were provided to the local Minikube environment for ESO testing
![AWS credentials configuration for Minikube](assets/image-19.png)
![AWS credentials secret in Kubernetes](assets/image-20.png)
![External Secrets Operator synchronization status](assets/image-21.png)

### 4.2 Secret rotation
The source secret value was changed in AWS Secrets Manager
![Secret value updated in AWS Secrets Manager](assets/image-23.png)

ESO subsequently synchronized the new value into Kubernetes
![External secret after rotation](assets/image-24.png)
![Updated Kubernetes secret value](assets/image-25.png)

### 4.3 Rotation without a pod restart, auto-sync within 60 seconds
![Application pods before secret rotation](assets/image-26.png)
![Secret synchronization test](assets/image-30.png)
![Rotated mounted-secret value](assets/image-32.png)
![Application pods after secret rotation](assets/image-27.png)
The evidence confirms that:

- Pod names did not change
- The restart count remained zero
- The pods remained in the `Running` state

### 4.4 Repository secret-safety check
The repository was checked to ensure that credentials and private secret material were not committed
![Repository credential exposure check](assets/image-33.png)

## 5. Trivy and Cosign software-supply-chain controls
### 5.1 HIGH or CRITICAL vulnerability failure

A test image containing a HIGH or CRITICAL vulnerability caused the GitHub Actions job to fail
![Failed Trivy CI run](assets/image-38.png)

The workflow stopped before the Cosign signing step, so a vulnerable image was not promoted as trusted
![Cosign signing skipped after Trivy failure](assets/image-37.png)

### 5.2 Unsigned image rejection

A new image digest was created without adding a trusted signature
![Unsigned image digest](assets/image-39.png)

Cosign verification confirmed that no trusted signature existed for that digest
![Cosign verification failure for unsigned image](assets/image-40.png)

The Sigstore admission webhook then denied the image during Kubernetes server-side dry-run
![Kubernetes rejection of unsigned image](assets/image-41.png)

### 6.4 Trusted signed image admission

A successful CI run produced and signed an image digest. Local Cosign verification confirmed that the claims and signature were valid against `signing/cosign.pub`.

![Successful CI image-signing run](assets/image-44.png)

![Signed CI image digest](assets/image-45.png)

![Cosign verification of the trusted signature](assets/image-46.png)

For compatibility with the deployed policy-controller, the signature was also published using Cosign's legacy signature-tag layout.

```bash
docker manifest inspect \
  "${IMAGE}:sha256-${DIGEST#sha256:}.sig" >/dev/null &&
echo "PASS: legacy signature exists"
```

![Legacy Cosign signature exists](assets/image-48.png)

The signed digest was then submitted to Kubernetes admission:

```bash
SIGNED_IMAGE="${IMAGE}@${DIGEST}"

kubectl set image -f tmp/test-pod.yaml \
  test="$SIGNED_IMAGE" --local -o yaml |
kubectl apply --dry-run=server -f -
```

Result: the policy-controller verified the trusted signature and admitted the pod.

![Trusted signed image admitted by Kubernetes](assets/image-49.png)
