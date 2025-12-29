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

### 내부 블로그 참조
- **실제 블로그 링크 사용**: `https://blog.imprun.dev/{번호}` 형식
- 블로그 번호는 [README.md](../README.md) 테이블에서 확인

```markdown
### 관련 블로그
- [Serena MCP: AI 코딩 어시스턴트를 위한 시맨틱 코드 분석 도구](https://blog.imprun.dev/29)
```

### 문서 간 링크 (docs 폴더 내)
- **같은 디렉토리 내**: 상대 경로 사용 `./filename.md`
- **다른 디렉토리**: 상대 경로 사용 `../../path/filename.md`

### 외부 링크
- 공식 문서, GitHub 등 외부 URL은 절대 경로 사용

### 예시

```markdown
## 참고 자료

### 공식 문서
- [Kubernetes Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

### 관련 블로그
- [MongoDB가 죽었다: 150GB 디스크가 있는데 왜?](https://blog.imprun.dev/64)
```
