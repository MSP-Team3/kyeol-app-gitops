# Cluster Autoscaler (CA) 테스트 가이드

## 🎯 테스트 목표

```
[ 강한 부하 발생 ]
     ↓
[ HPA: Pod 2개 → 10개 증가 ]
     ↓
[ Node 자원 부족 (CPU/Memory 한계) ]
     ↓
[ Pod Pending 상태 발생 ]
     ↓
[ Cluster Autoscaler 판단 ]
     ↓
[ 새로운 Node 추가 ]
     ↓
[ Pending Pod 스케줄링 완료 ]
```

## 📋 전제 조건

### 1. Cluster Autoscaler 설치 확인

```bash
# CA 확인
kubectl get deployment cluster-autoscaler -n kube-system

# CA 로그 확인
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=50
```

### 2. Node Group 확인 (AWS EKS 기준)

```bash
# Node 현황
kubectl get nodes

# Node 리소스 확인
kubectl top nodes
```

### 3. HPA 리소스 증가 (더 빠른 Pod 증가)

현재 HPA 설정을 더 공격적으로 수정:

```yaml
# hpa-storefront.yaml 수정
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: storefront-hpa
  namespace: kyeol
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: storefront
  minReplicas: 2
  maxReplicas: 20  # 10 → 20 (더 많은 Pod)
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50  # 70% → 50% (더 빠른 트리거)
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # 30초 → 0초 (즉시)
      policies:
        - type: Percent
          value: 200  # 100% → 200% (더 빠른 증가)
          periodSeconds: 10  # 15초 → 10초
        - type: Pods
          value: 5  # 2개 → 5개 (한 번에 더 많이)
          periodSeconds: 10
      selectPolicy: Max
```

## 🚀 Step-by-Step 테스트

### Step 1: 리소스 요청량 증가 (Pod가 더 많은 자원 필요)

```yaml
# deployment-storefront.yaml 수정
resources:
  requests:
    cpu: 500m      # 100m → 500m (더 많은 CPU 요청)
    memory: 512Mi  # 128Mi → 512Mi (더 많은 메모리 요청)
  limits:
    cpu: 1000m     # 500m → 1000m
    memory: 1Gi    # 512Mi → 1Gi
```

이렇게 하면 Node 당 들어갈 수 있는 Pod 수가 줄어들어 빨리 Node 한계에 도달합니다.

### Step 2: GitOps 적용

```bash
cd D:\4th_Parkminwook\WORKSPACE\kyeol-project\kyeol-app-gitops

# 변경 사항 커밋
git add apps/saleor/base/
git commit -m "feat: Increase HPA limits and Pod resources for CA test

- HPA maxReplicas: 10 → 20
- HPA CPU threshold: 70% → 50%
- HPA scaleUp: more aggressive (200%, 5 pods)
- Pod CPU request: 100m → 500m
- Pod Memory request: 128Mi → 512Mi"

git push origin main

# ArgoCD 동기화
argocd app sync saleor
```

### Step 3: 모니터링 준비 (5개 터미널)

#### Terminal 1: HPA 상태
```bash
watch -n 1 'kubectl get hpa storefront-hpa -n kyeol'
```

#### Terminal 2: Pod 상태 (Pending 확인)
```bash
watch -n 1 'kubectl get pods -n kyeol -l app=storefront -o wide'
```

#### Terminal 3: Node 상태
```bash
watch -n 2 'kubectl get nodes'
```

#### Terminal 4: Node 리소스
```bash
watch -n 2 'kubectl top nodes'
```

#### Terminal 5: CA 로그
```bash
kubectl logs -n kube-system -l app=cluster-autoscaler -f
```

### Step 4: 초강력 부하 테스트

#### 방법 1: k6 (매우 강한 부하)

`loadtest-ca-aggressive.js`:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 200 },   // 빠르게 200 VUs
    { duration: '3m', target: 500 },   // 500 VUs (매우 강함)
    { duration: '2m', target: 1000 },  // 1000 VUs (최대 부하)
    { duration: '3m', target: 500 },
    { duration: '1m', target: 0 },
  ],
};

const BASE_URL = 'https://origin-dev.kyeol.click';

export default function () {
  // 여러 페이지 동시 요청으로 CPU 부하 증가
  http.batch([
    ['GET', `${BASE_URL}/`],
    ['GET', `${BASE_URL}/default-channel/products`],
    ['GET', `${BASE_URL}/default-channel/search`],
  ]);

  sleep(0.5);  // 짧은 대기로 요청 빈도 증가
}
```

실행:
```bash
k6 run loadtest-ca-aggressive.js
```

#### 방법 2: hey (더 간단)

```bash
# 500 동시 연결, 10분간 지속
hey -z 10m -c 500 https://origin-dev.kyeol.click/
```

#### 방법 3: 다중 AB (여러 개 동시 실행)

```bash
# 5개의 AB를 동시에 실행
for i in {1..5}; do
  ab -n 100000 -c 200 https://origin-dev.kyeol.click/ &
done
```

### Step 5: 관찰 포인트

#### 1단계: HPA 작동 (1-2분)
```bash
# Pod 증가 확인
kubectl get pods -n kyeol -l app=storefront

# 기대: 2 → 4 → 8 → 12 → 16 → 20 Pods
```

#### 2단계: Node 자원 한계 도달 (3-5분)
```bash
# Node CPU/Memory 사용률 확인
kubectl top nodes

# 기대: CPU/Memory 80-90% 사용
```

#### 3단계: Pod Pending 발생 (5-7분)
```bash
# Pending Pod 확인
kubectl get pods -n kyeol -l app=storefront | grep Pending

# 상세 확인
kubectl describe pod <pending-pod-name> -n kyeol

# 기대 메시지:
# "0/3 nodes are available: 3 Insufficient cpu."
# 또는
# "0/3 nodes are available: 3 Insufficient memory."
```

#### 4단계: CA 판단 (7-8분)
```bash
# CA 로그에서 확인
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100

# 기대 로그:
# "Scale-up: group <node-group> size set to 4"
```

#### 5단계: Node 증가 (8-10분)
```bash
# 새로운 Node 확인
kubectl get nodes -w

# 기대: Node 3개 → 4개 → 5개
```

#### 6단계: Pending 해소 (10-12분)
```bash
# 모든 Pod Running 확인
kubectl get pods -n kyeol -l app=storefront

# 기대: Pending 0개, Running 20개
```

## 📊 상세 모니터링 명령어

### 한 번에 모든 정보 확인

```bash
# 종합 상태 스크립트
watch -n 2 '
echo "=== HPA Status ==="
kubectl get hpa storefront-hpa -n kyeol

echo ""
echo "=== Pod Status ==="
kubectl get pods -n kyeol -l app=storefront | head -n 25

echo ""
echo "=== Pod Count ==="
echo "Running: $(kubectl get pods -n kyeol -l app=storefront --field-selector=status.phase=Running --no-headers | wc -l)"
echo "Pending: $(kubectl get pods -n kyeol -l app=storefront --field-selector=status.phase=Pending --no-headers | wc -l)"

echo ""
echo "=== Node Status ==="
kubectl get nodes

echo ""
echo "=== Node Resources ==="
kubectl top nodes
'
```

### Pending Pod 상세 분석

```bash
# Pending Pod 이유 확인
for pod in $(kubectl get pods -n kyeol -l app=storefront --field-selector=status.phase=Pending -o name); do
  echo "=== $pod ==="
  kubectl describe $pod -n kyeol | grep -A 10 "Events:"
  echo ""
done
```

### CA 상태 확인

```bash
# CA ConfigMap 확인
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# CA 최근 이벤트
kubectl get events -n kube-system --field-selector involvedObject.name=cluster-autoscaler --sort-by='.lastTimestamp'
```

## 🔥 더 빠른 CA 트리거 방법

### 방법 1: 수동으로 대량 Pod 생성

```bash
# Deployment replicas 강제 증가
kubectl scale deployment storefront -n kyeol --replicas=30

# 결과: 즉시 30개 Pod 생성 시도 → Node 부족 → CA 작동
```

### 방법 2: 임시 Stress Pod 추가 생성

```yaml
# stress-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-test
  namespace: kyeol
spec:
  replicas: 10
  selector:
    matchLabels:
      app: stress-test
  template:
    metadata:
      labels:
        app: stress-test
    spec:
      containers:
      - name: stress
        image: polinux/stress
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        command: ["stress"]
        args: ["--cpu", "2", "--timeout", "600s"]
```

적용:
```bash
kubectl apply -f stress-test.yaml

# Node 부족 즉시 발생 → CA 작동

# 테스트 후 삭제
kubectl delete -f stress-test.yaml
```

## 📈 예상 타임라인

| 시간 | 이벤트 | 확인 방법 |
|------|--------|-----------|
| 0:00 | 부하 시작 | k6 실행 |
| 0:30 | HPA 트리거 (CPU 50% 초과) | `kubectl get hpa` |
| 1:00 | Pod 2 → 6 증가 | `kubectl get pods` |
| 2:00 | Pod 6 → 12 증가 | `kubectl get pods` |
| 3:00 | Pod 12 → 18 증가 | `kubectl get pods` |
| 4:00 | Node CPU/Memory 90% 도달 | `kubectl top nodes` |
| 5:00 | **Pod Pending 발생** | `kubectl get pods \| grep Pending` |
| 6:00 | **CA 판단 시작** | CA 로그 |
| 7:00 | **Node 추가 시작** | `kubectl get nodes` |
| 9:00 | **새 Node Ready** | `kubectl get nodes` |
| 10:00 | **Pending Pod Running** | `kubectl get pods` |
| 12:00 | 부하 종료 | k6 완료 |
| 17:00 | Pod 감소 시작 (5분 안정화) | `kubectl get pods` |
| 25:00 | Pod 2개로 복귀 | `kubectl get pods` |
| 35:00 | Node 감소 시작 (CA scale-down) | `kubectl get nodes` |

## 🐛 트러블슈팅

### 문제 1: Pod가 Pending 안 됨

**원인**: Node에 여유 자원이 많음

**해결**:
```bash
# 1. Pod 리소스 요청 더 증가
resources:
  requests:
    cpu: 1000m      # 500m → 1000m
    memory: 1Gi     # 512Mi → 1Gi

# 2. 또는 HPA maxReplicas 더 증가
maxReplicas: 50  # 20 → 50
```

### 문제 2: CA가 작동 안 함

```bash
# CA 설치 확인
kubectl get deployment cluster-autoscaler -n kube-system

# CA 설정 확인
kubectl describe deployment cluster-autoscaler -n kube-system

# AWS Node Group 확인 (EKS)
aws autoscaling describe-auto-scaling-groups --region ap-northeast-2

# CA가 Node Group 인식하는지 확인
kubectl logs -n kube-system -l app=cluster-autoscaler | grep "Discovering"
```

### 문제 3: Node 추가가 너무 느림

**원인**: AWS EC2 인스턴스 시작 시간 (2-5분)

**해결**: 기다리기 또는 Warm Pool 사용

### 문제 4: Pending Pod가 계속 Pending

```bash
# Pod 이벤트 확인
kubectl describe pod <pending-pod> -n kyeol

# 일반적인 원인:
# 1. Node의 실제 가용 리소스 부족
# 2. Node에 Taint 있음
# 3. Pod에 NodeSelector/Affinity 있음
# 4. CA가 비활성화됨
```

## 📊 성공 기준

테스트 성공으로 판단하는 기준:

- ✅ HPA로 Pod가 2 → 20개 증가
- ✅ Node CPU/Memory 90% 이상 도달
- ✅ Pod Pending 상태 발생
- ✅ CA 로그에 "Scale-up" 메시지
- ✅ Node 개수 증가 (예: 3 → 5)
- ✅ Pending Pod가 Running으로 전환
- ✅ 부하 종료 후 Pod/Node 감소

## 🎓 즉시 실행 명령어

```bash
# 1. HPA/Deployment 수정 (GitOps)
cd kyeol-app-gitops
# (위 Step 1, 2 내용 반영 후)
git add . && git commit -m "feat: Aggressive HPA/Pod for CA test" && git push
argocd app sync saleor

# 2. 모니터링 시작 (5개 터미널)
# Terminal 1
watch -n 1 'kubectl get hpa storefront-hpa -n kyeol'

# Terminal 2
watch -n 1 'kubectl get pods -n kyeol -l app=storefront'

# Terminal 3
watch -n 2 'kubectl get nodes'

# Terminal 4
watch -n 2 'kubectl top nodes'

# Terminal 5
kubectl logs -n kube-system -l app=cluster-autoscaler -f

# 3. 강력한 부하 시작
hey -z 10m -c 500 https://origin-dev.kyeol.click/

# 또는 수동 스케일
kubectl scale deployment storefront -n kyeol --replicas=30
```

---

**목표**: Node 자원 소진 → Pod Pending → CA Node 추가 확인
**예상 시간**: 10-15분
