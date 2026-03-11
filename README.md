# HiveWiki Gitops

HiveWiki GitOps 레포지토리는 HiveWiki 프로젝트의 Kubernetes 배포 구성을 관리하는 저장소입니다.
ArgoCD를 사용한 GitOps 방식으로 애플리케이션 배포와 클러스터 구성을 관리합니다.

이 레포지토리에는 다음과 같은 리소스가 포함됩니다.

- Kubernetes manifests
- ArgoCD applications
- Helm charts 및 values
- Cilium 설정
- 기타 클러스터 구성

---

## 개발 환경

이 프로젝트는 다음 도구들을 사용합니다.

- Kubernetes
- ArgoCD
- Helm
- Cilium
- pre-commit
- gitleaks
- commitizen

---

## 시작하기

### 1. 필수 도구 설치

`pre-commit`은 Python 기반 도구이므로 먼저 설치합니다.

```bash
pip install pre-commit
```

### 2. Repository Clone

```bash
git clone <repository-url>
cd hivewiki-gitops
```

### 3. pre-commit hook 설치

```bash
pre-commit install --hook-type pre-commit --hook-type commit-msg
```

이 프로젝트는 pre-commit hooks를 사용합니다.  
코드가 자동으로 수정되면 커밋이 중단될 수 있으며, 수정된 파일을 확인한 뒤 다시 add하고 커밋하면 됩니다.

Commit message는 **영어로 작성해야 하며**, Conventional Commits 규칙을 따릅니다.  
자세한 규칙은 이 [문서](https://commitizen-tools.github.io/commitizen/tutorials/writing_commits/)를 참고하세요.
