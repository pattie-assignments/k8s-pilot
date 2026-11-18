# K8s Pilot

Kubernetes 기반 애플리케이션 배포 환경을 구축하고, **GitHub Actions와 Argo CD를 활용한 GitOps 배포 자동화 및 통합 모니터링 환경**을 구성한 개인 프로젝트입니다.

**기간:** 2026.07 ~ 2026.08  
**기술:** AWS, Kubernetes, Helm, Argo CD, GitHub Actions, Docker, Prometheus, Grafana, Loki, Vector

## 1. Architecture

**요청 흐름:** `Route 53 → ALB → Nginx → Traefik → Kubernetes Service → Pod`

<p align="center">
  <img src="https://github.com/user-attachments/assets/5f1c2dbc-b1ed-447f-9d56-bd2e7e08d297" alt="Architecture" width="100%" />
</p>

## 2. GitOps 기반 배포 자동화

**배포 흐름**

`Code Push → CI → 이미지 빌드·GHCR Push → GitOps Repository 갱신 → Argo CD Sync → Kubernetes 배포`

| 구성 요소 | 역할 |
|---|---|
| GitHub Actions | 애플리케이션 빌드·테스트 및 이미지 발행 |
| GHCR | Docker 이미지 저장 및 버전 관리 |
| GitOps Repository | Helm values에 배포할 이미지 태그 기록 |
| Argo CD | Git 변경사항 감지 및 Kubernetes 배포 동기화 |
| Helm | Kubernetes 리소스 템플릿 및 설정 관리 |

이미지 발행은 별도 Workflow에서 실행되며, 발행 성공 이후 GitOps 저장소 갱신 및 Argo CD 자동 동기화가 이어지도록 구성했습니다.

## 3. 주요 설계

- **CI/CD 책임 분리:** GitHub Actions는 이미지 빌드·발행을 담당하고, Argo CD는 Kubernetes 배포 상태를 관리하도록 구성
- **App of Apps:** Root Application으로 백엔드와 플랫폼 컴포넌트의 배포 선언을 통합 관리
- **모니터링:** Prometheus·Grafana로 메트릭 조회 환경을 구성하고, Vector·Loki로 Kubernetes 로그를 중앙 수집하도록 구성
