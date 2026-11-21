# kubeadm 실습 환경 재현

이 디렉터리는 kubeadm 실습 클러스터의 워커 조인, Helm 배포, 내부 API 부하 검증을 다룬다. EC2·네트워크·IAM·FIS는 [workspace-v2/terraform의 k8s-pilot 구성](https://github.com/workspace-v2/terraform/tree/main/k8s-pilot)에서 관리한다.

| 역할 | 구성 |
| --- | --- |
| Control plane | EC2 1대, kubeadm, Flannel, MySQL·Redis, 업로드 NFS, k6 |
| 워커 | EC2 2대, 워커별 앱 파드 한 개, 로컬 containerd 이미지 |
| 장애 실습 | 워커 두 대 중 한 대만 재부팅하는 AWS FIS 템플릿 |
| 부하 실습 | 클러스터 내부 브랜드 읽기 API에 5 RPS로 30초 실행 |

## 1. 워커 조인

AWS 인프라를 준비한 뒤 Terraform 출력에서 `instance_ids`와 `private_ips`를 확인한다. AWS CLI와 Session Manager 플러그인이 있는 터미널에서 각 인스턴스에 접속한다.

```bash
aws --profile personal-project --region ap-northeast-2 ssm start-session --target INSTANCE_ID
```

Control plane에서 `sudo cloud-init status --wait`가 끝난 뒤 `sudo kubeadm token create --ttl 10m --print-join-command`를 실행한다. 각 워커에서도 cloud-init 완료를 확인하고, 받은 명령을 `sudo`로 실행하면서 `--node-name k8s-pilot-worker-a` 또는 `--node-name k8s-pilot-worker-b`를 붙인다.

Control plane의 관리자 셸에서 두 노드가 준비되면 역할을 지정하고 조인에 사용한 토큰을 폐기한다.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl wait --for=condition=Ready node/k8s-pilot-worker-a node/k8s-pilot-worker-b --timeout=180s
kubectl label node k8s-pilot-worker-a k8s-pilot-worker-b node-role.kubernetes.io/worker= k8s-pilot/worker=true
kubeadm token delete TOKEN_ID
```

## 2. 플랫폼과 이미지 준비

Control plane에서 이 저장소를 체크아웃한 뒤, 각 워커의 프라이빗 IP를 사용해 공유 저장소를 준비한다. Control plane에 Java 21과 Buildah, Helm 3가 필요하다.

```bash
apt-get update
apt-get install -y git openjdk-21-jdk-headless buildah
bash lab/scripts/configure-storage.sh WORKER_A_PRIVATE_IP WORKER_B_PRIVATE_IP
bash lab/scripts/build-image.sh
```

각 워커의 관리자 셸에서 이미지를 가져온다. `CONTROL_PLANE_IP`에는 Terraform 출력의 control plane 프라이빗 IP를 넣는다.

```bash
share=$(mktemp -d)
mount -t nfs4 -o ro,nfsvers=4.1 CONTROL_PLANE_IP:/ "$share"
ctr -n k8s.io images import --platform linux/amd64 "$share/community-service-25aebbc.tar"
umount "$share"
rmdir "$share"
```

Control plane에서 Traefik과 관측 도구를 설치한다. 이 실습 차트의 앱은 아래에서 Helm으로 직접 설치한다.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd argo/argo-cd --version 10.2.1 -n argocd --create-namespace \
  -f argocd/values/argocd-values.yaml --wait --timeout 8m
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
if ! kubectl -n monitoring get secret grafana-admin-secret >/dev/null 2>&1; then
  secret_file=$(mktemp)
  openssl rand -hex 24 | tr -d '\n' > "$secret_file"
  kubectl -n monitoring create secret generic grafana-admin-secret \
    --from-literal=admin-user=admin --from-file=admin-password="$secret_file"
  rm -f "$secret_file"
fi
for app in argocd traefik prometheus grafana loki vector; do
  kubectl apply -f "argocd/applications/$app.yaml"
done
```

## 3. 애플리케이션 배포

Control plane에서 네임스페이스와 Secret을 준비한다. 기존 Secret이 있으면 그대로 사용한다.

```bash
kubectl create namespace community-service --dry-run=client -o yaml | kubectl apply -f -
if ! kubectl -n community-service get secret community-service-credentials >/dev/null 2>&1; then
  secret_file=$(mktemp)
  for key in DB_PASSWORD MYSQL_ROOT_PASSWORD REDIS_PASSWORD JWT_SECRET; do
    printf '%s=%s\n' "$key" "$(openssl rand -hex 32)"
  done > "$secret_file"
  kubectl -n community-service create secret generic community-service-credentials --from-env-file="$secret_file"
  rm -f "$secret_file"
fi
helm upgrade --install community-service lab/chart -n community-service \
  --set-string storage.nfsServer=CONTROL_PLANE_IP --wait --timeout 8m
kubectl -n community-service get pods -o wide
curl -fsS -H 'Host: api.pattie.cloud' http://CONTROL_PLANE_IP:32080/v1/brands
```

실습 DB를 외부 MySQL로 구성할 때는 DB 접속을 준비하고 Secret에 그 계정의 비밀번호를 설정한 뒤, 최초 설치에 `--set mysql.enabled=false --set-string mysql.host=MYSQL_HOST`를 추가한다. 기존 데이터를 옮기는 경우 먼저 백업과 복원을 별도로 진행한다.

## 4. 실습 실행

```bash
job="k6-lab-$(date +%s)"
kubectl -n community-service create job "$job" --from=cronjob/community-service-k6
kubectl -n community-service logs "job/$job" -f
```

워커 재부팅은 [FIS 실행 절차](https://github.com/workspace-v2/terraform/blob/main/k8s-pilot/README.md#fis-워커-재부팅)를 따른다.
