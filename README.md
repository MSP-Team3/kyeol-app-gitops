# KYEOL Application GitOps

> Saleor 애플리케이션 Kubernetes 매니페스트 레포지토리
> ArgoCD를 통한 GitOps 기반 자동 배포

---

## 개요

이 레포지토리는 KYEOL 프로젝트의 Saleor 애플리케이션 컴포넌트들을 Kubernetes 클러스터에 배포하기 위한 Kustomize 매니페스트와 ArgoCD Application 정의를 포함합니다.

### 관리 대상 애플리케이션

- **Saleor** (Storefront): Next.js 기반 고객용 프론트엔드
- **Saleor API** (Core): Django 기반 GraphQL API 서버
- **Saleor Dashboard**: React 기반 관리자 대시보드

---

## 디렉토리 구조

```
kyeol-app-gitops/
├── apps/                           # 애플리케이션 매니페스트
│   ├── saleor/                     # Storefront (고객용 UI)
│   │   ├── base/                   # 공통 매니페스트
│   │   │   ├── deployment-storefront.yaml
│   │   │   ├── service-storefront.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── namespace.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/               # 환경별 오버레이
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/        # DEV 환경 패치
│   │       │       ├── env-patch.yaml
│   │       │       ├── ingress-patch.yaml
│   │       │       └── replicas-patch.yaml
│   │       ├── stage/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/        # STAGE 환경 패치
│   │       └── prod/
│   │           ├── kustomization.yaml
│   │           └── patches/        # PROD 환경 패치
│   │               ├── hpa-patch.yaml
│   │               └── resources-patch.yaml
│   ├── saleor-api/                 # Saleor Core (API 서버)
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── ingress.yaml
│   │   │   ├── serviceaccount.yaml  # IRSA용 ServiceAccount
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── env-dev.properties   # DEV 환경변수
│   │       │   ├── deployment-patch.yaml
│   │       │   ├── ingress-patch.yaml
│   │       │   └── serviceaccount-patch.yaml
│   │       ├── stage/
│   │       │   ├── env-stage.properties
│   │       │   └── ...
│   │       └── prod/
│   │           ├── env-prod.properties
│   │           └── patches/
│   └── saleor-dashboard/           # 관리자 대시보드
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           ├── stage/
│           └── prod/
└── argocd/                         # ArgoCD Application 정의
    └── applications/
        ├── saleor-dev.yaml         # Storefront DEV
        ├── saleor-stage.yaml
        ├── saleor-prod.yaml
        ├── saleor-api-dev.yaml     # API DEV
        ├── saleor-api-stage.yaml
        ├── saleor-api-prod.yaml
        ├── dashboard-dev.yaml      # Dashboard DEV
        ├── dashboard-stage.yaml
        └── dashboard-prod.yaml
```

---

## 주요 개념

### Kustomize Base + Overlays 패턴

- **Base**: 모든 환경에서 공통으로 사용되는 기본 매니페스트
- **Overlays**: 환경별 설정 차이를 패치로 적용

```yaml
# base/kustomization.yaml - 공통 리소스
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml

# overlays/dev/kustomization.yaml - DEV 환경 커스터마이제이션
bases:
  - ../../base
patchesStrategicMerge:
  - patches/env-patch.yaml
  - patches/replicas-patch.yaml
```

### ArgoCD Application

각 애플리케이션은 ArgoCD Application CRD로 정의되어 자동 동기화됩니다.

```yaml
# argocd/applications/saleor-api-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: saleor-api-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/MSP-Team3/kyeol-app-gitops.git
    targetRevision: main
    path: apps/saleor-api/overlays/dev
  destination:
    server: https://DEV-CLUSTER-URL
    namespace: saleor
  syncPolicy:
    automated:
      prune: true       # 삭제된 리소스 자동 제거
      selfHeal: true    # 수동 변경 시 자동 복구
```

---

## 환경별 구성

### DEV 환경
- **Replicas**: 1
- **Resources**: 최소 사양 (CPU 200m, Memory 256Mi)
- **Ingress**: dev.kyeol.com
- **Auto Scaling**: 비활성화

### STAGE 환경
- **Replicas**: 2
- **Resources**: 중간 사양 (CPU 500m, Memory 512Mi)
- **Ingress**: stage.kyeol.com
- **Auto Scaling**: HPA 활성화 (2-4 Pods)

### PROD 환경
- **Replicas**: 3
- **Resources**: 고사양 (CPU 1000m, Memory 1Gi)
- **Ingress**: kyeol.com
- **Auto Scaling**: HPA 활성화 (3-10 Pods)

---

## 배포 워크플로우

### GitOps 자동 배포

```
코드 변경 → Git Push → ArgoCD 감지 → Sync → 클러스터 배포
```

1. **개발자**: 코드 수정 및 Git Push
2. **ArgoCD**: 레포지토리 변경 감지 (Polling 또는 Webhook)
3. **ArgoCD**: 자동 Sync 실행 (`syncPolicy.automated`)
4. **Kubernetes**: 리소스 생성/업데이트

### 수동 배포

ArgoCD CLI 또는 UI를 통해 수동으로 동기화할 수도 있습니다.

```bash
# ArgoCD CLI를 통한 수동 Sync
argocd app sync saleor-api-dev

# 특정 Revision으로 Rollback
argocd app rollback saleor-api-dev <REVISION>
```

---

## 주요 작업

### 1. 새 환경변수 추가

Saleor API 환경변수 추가 예시:

```bash
# 1. overlays/dev/env-dev.properties 파일 수정
cat >> apps/saleor-api/overlays/dev/env-dev.properties <<EOF
NEW_FEATURE_FLAG=true
EOF

# 2. Git Commit & Push
git add apps/saleor-api/overlays/dev/env-dev.properties
git commit -m "feat: Add NEW_FEATURE_FLAG to dev"
git push origin main

# 3. ArgoCD가 자동으로 변경사항 감지 및 배포
```

### 2. Replica 수 변경

```yaml
# overlays/prod/patches/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: saleor-api
spec:
  replicas: 5  # 3 → 5로 증가
```

```bash
git add overlays/prod/patches/replicas-patch.yaml
git commit -m "scale: Increase saleor-api replicas to 5 in prod"
git push origin main
```

### 3. 이미지 태그 변경

```yaml
# overlays/prod/kustomization.yaml
images:
  - name: saleor-api
    newName: 827913617839.dkr.ecr.ap-northeast-2.amazonaws.com/kyeol-prod-saleor-api
    newTag: v1.2.3  # 이미지 태그 변경
```

### 4. Ingress 도메인 변경

```yaml
# overlays/prod/patches/ingress-patch.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: saleor-api-ingress
spec:
  rules:
  - host: api.kyeol.com  # 도메인 변경
```

---

## ECR 레포지토리 매핑

| 환경 | Storefront ECR | Dashboard ECR | API ECR |
|:----:|---------------|--------------|---------|
| DEV | kyeol-storefront | kyeol-dashboard | kyeol-saleor-api |
| STAGE | kyeol-stage-storefront | kyeol-stage-dashboard | kyeol-stage-saleor-api |
| PROD | kyeol-prod-storefront | kyeol-prod-dashboard | kyeol-prod-saleor-api |

---

## 트러블슈팅

### ArgoCD Sync 실패

**증상**: ArgoCD UI에서 "OutOfSync" 상태

**확인**:
```bash
# Application 상태 확인
argocd app get saleor-api-dev

# Sync 로그 확인
argocd app logs saleor-api-dev
```

**일반적인 원인**:
- YAML 문법 오류
- Kustomization 빌드 실패
- 클러스터 권한 부족

**해결**:
```bash
# 로컬에서 Kustomize 빌드 테스트
kubectl kustomize apps/saleor-api/overlays/dev

# 문법 검증
kubectl apply --dry-run=client -k apps/saleor-api/overlays/dev
```

### Pod ImagePullBackOff

**증상**: Pod가 `ImagePullBackOff` 상태

**확인**:
```bash
kubectl describe pod <pod-name> -n saleor
```

**원인**:
- ECR 이미지 태그 없음
- IRSA 권한 부족 (ECR Pull 권한)

**해결**:
```bash
# ECR 이미지 확인
aws ecr describe-images --repository-name kyeol-saleor-api --region ap-northeast-2

# ServiceAccount IRSA 확인
kubectl describe sa saleor-api -n saleor | grep eks.amazonaws.com/role-arn
```

### 환경변수가 적용되지 않음

**증상**: ConfigMap 변경했지만 Pod에 반영 안 됨

**원인**: Kubernetes는 ConfigMap 변경 시 자동으로 Pod를 재시작하지 않음

**해결**:
```bash
# Pod 재시작
kubectl rollout restart deployment/saleor-api -n saleor

# 또는 ArgoCD에서 Hard Refresh
argocd app sync saleor-api-dev --force
```

---

## 모범 사례

### 1. GitOps 원칙 준수

kubectl apply로 직접 수정하지 말고, 항상 Git을 통해 변경합니다.

```bash
# ❌ 잘못된 방법
kubectl edit deployment saleor-api -n saleor

# ✅ 올바른 방법
# 1. Git에서 매니페스트 수정
# 2. Commit & Push
# 3. ArgoCD 자동 동기화 대기
```

### 2. 환경별 격리

각 환경은 독립적인 EKS 클러스터에 배포합니다.

- DEV → kyeol-dev-eks
- STAGE → kyeol-stage-eks
- PROD → kyeol-prod-eks

### 3. Rollback 전략

문제 발생 시 즉시 이전 버전으로 롤백합니다.

```bash
# ArgoCD History 확인
argocd app history saleor-api-prod

# 특정 Revision으로 Rollback
argocd app rollback saleor-api-prod <REVISION>
```

### 4. Secret 관리

민감 정보는 Kubernetes Secret으로 관리하며, Git에 평문 저장 금지합니다.

```bash
# Secret 생성 (Git에 저장하지 않음)
kubectl create secret generic saleor-db-secret \
  --from-literal=DATABASE_URL=postgres://... \
  -n saleor

# 또는 AWS Secrets Manager + External Secrets Operator 사용
```

---

## 배포 전 체크리스트

- [ ] YAML 문법 검증 완료
- [ ] Kustomize 빌드 성공 확인
- [ ] 이미지 태그가 ECR에 존재하는지 확인
- [ ] 환경변수 값 검증
- [ ] STAGE 환경에서 충분히 테스트
- [ ] Rollback 계획 수립
- [ ] 주요 이해관계자 공지

---

## 다른 레포지토리와의 관계

| 레포지토리 | 관계 |
|-----------|------|
| kyeol-infra-terraform | 이 레포 실행 전 EKS 클러스터 생성 필요 |
| kyeol-platform-gitops | 이 레포 실행 전 ALB Controller 설치 필요 |
| kyeol-storefront-org | Storefront 이미지 소스 (ECR로 Push) |
| saleor | Saleor API 이미지 소스 (ECR로 Push) |
| kyeol-dashboard | Dashboard 이미지 소스 (ECR로 Push) |

---

## 관련 문서

- **인프라 구성**: [kyeol-docs/runbook/runbook-infra.md](../kyeol-docs/runbook/runbook-infra.md)
- **플랫폼 운영**: [kyeol-docs/runbook/runbook-platform.md](../kyeol-docs/runbook/runbook-platform.md)
- **애플리케이션 운영**: [kyeol-docs/runbook/runbook-ops.md](../kyeol-docs/runbook/runbook-ops.md)
- **장애 대응**: [kyeol-docs/troubleshooting.md](../kyeol-docs/troubleshooting.md)

---

**마지막 업데이트**: 2026-01-21
**레포지토리**: https://github.com/MSP-Team3/kyeol-app-gitops
**ArgoCD URL**: https://argocd.kyeol.com (MGMT 클러스터)
