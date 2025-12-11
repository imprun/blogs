# 이미지 및 링크 규칙

## 이미지 사용 규칙

### 외부 이미지 링크
- **공식 문서 이미지**: 공식 사이트의 이미지는 직접 링크하여 사용 가능
- **출처 표기**: 이미지 아래에 출처를 명시

```markdown
![이미지 설명](https://official-site.com/path/to/image.jpg)
*출처: [공식 문서 이름](https://official-site.com/docs)*
```

### 주의사항
- 저작권이 있는 이미지는 사용하지 않음
- 공식 문서, 오픈소스 프로젝트의 이미지만 링크
- 이미지 URL이 변경될 수 있으므로 중요한 내용은 텍스트로도 설명

## 링크 규칙

### 문서 간 링크
- **같은 디렉토리 내**: 상대 경로 사용 `./filename.md`
- **다른 디렉토리**: 상대 경로 사용 `../../path/filename.md`
- **외부 블로그 참조**: 필요시 절대 URL 사용

### 예시

```markdown
## 참고 자료

### 관련 문서
- [Kubernetes Ephemeral Storage 문제 해결 가이드](./kubernetes-ephemeral-storage-troubleshooting-guide.md) - 상세 기술 문서
- [Oracle Cloud 준비 및 설정](../../10/25/01-preparation-oracle-cloud.md) - Block Volume 설정 가이드

### 공식 문서
- [Kubernetes Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
```
