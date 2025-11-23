# Monaco Editor "TextModel got disposed" 에러 완벽 해결 가이드

**작성일:** 2025년 10월 27일
**카테고리:** React, Monaco Editor, 디버깅
**난이도:** 중급

---

## TL;DR

- **문제**: `TextModel got disposed before DiffEditorWidget model got reset` 에러 발생
- **원인**: @monaco-editor/react의 DiffEditor가 props 변경 시 모델을 재생성하면서 dispose 충돌 발생
- **해결**: `keepCurrentOriginalModel={true}` + `keepCurrentModifiedModel={true}` props 추가 (단 2줄!)
- **결과**: 복잡한 cleanup 로직 없이 깔끔하게 해결, 코드 45% 감소

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. 웹 콘솔에서 **함수 배포 히스토리 비교** 기능을 구현하면서 Monaco Editor의 DiffEditor를 사용했는데, 다음과 같은 에러가 지속적으로 발생했습니다:

```
Uncaught Error: TextModel got disposed before DiffEditorWidget model got reset
```

이 에러는:
- ❌ **Dialog(모달)를 닫을 때마다** 발생
- ❌ **파일이나 버전을 변경할 때** 발생
- ❌ **사용자 경험을 크게 해침** (콘솔 에러 폭탄)
- ❌ **메모리 누수 가능성** 존재

---

## 문제 상황: 언제, 왜 발생하나?

### 🔍 에러 발생 시나리오

```tsx
// FunctionHistoryModal.tsx
function FunctionHistoryModal({ open, onOpenChange }) {
  const [selectedFile, setSelectedFile] = useState<string | null>(null)

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DiffEditor
          original={previousVersion?.files[selectedFile] || ''}
          modified={currentVersion?.files[selectedFile] || ''}
          language="typescript"
          theme="vs-dark"
        />
      </DialogContent>
    </Dialog>
  )
}
```

**에러 발생 타이밍**:
1. 사용자가 파일을 선택 → `selectedFile` 변경 → DiffEditor props 변경 → **에러!**
2. 사용자가 다른 버전 선택 → `previousVersion` / `currentVersion` 변경 → **에러!**
3. 사용자가 모달 닫기 → DiffEditor unmount → **에러!**

---

## 원인 분석: Monaco Editor의 내부 동작

### 📚 Monaco Editor의 Model 관리

Monaco Editor는 내부적으로 `TextModel`을 사용하여 코드를 관리합니다:

```
DiffEditor
  ├─ OriginalEditor
  │   └─ TextModel (original)
  └─ ModifiedEditor
      └─ TextModel (modified)
```

**문제의 핵심**:
- `@monaco-editor/react`의 DiffEditor는 **props가 변경되면 기존 모델을 dispose하고 새 모델을 생성**합니다
- 하지만 dispose 순서가 비동기적으로 처리되면서 **race condition** 발생:
  ```
  1. TextModel.dispose() 시작
  2. DiffEditorWidget이 아직 모델을 참조 중
  3. TextModel이 먼저 dispose 완료
  4. DiffEditorWidget이 dispose된 모델 접근 시도 → 💥 에러!
  ```

### 🔬 실제 에러 스택 트레이스

```javascript
Error: TextModel got disposed before DiffEditorWidget model got reset
    at lR.value (monaco-editor.js:388:50203)
    at P._deliver (monaco-editor.js:7:2719)
    at tl.dispose (monaco-editor.js:243:68815)
    at commitHookEffectListUnmount (react-dom.js:9449:160)
```

React의 unmount phase에서 Monaco Editor의 내부 dispose 메커니즘과 충돌하는 것을 확인할 수 있습니다.

---

## 시도한 해결 방법들

### ❌ 시도 1: 수동 cleanup (실패)

```tsx
useEffect(() => {
  return () => {
    if (editorRef.current) {
      const editor = editorRef.current
      const originalModel = editor.getOriginalEditor().getModel()
      const modifiedModel = editor.getModifiedEditor().getModel()

      // 모델 분리
      editor.getOriginalEditor().setModel(null)
      editor.getModifiedEditor().setModel(null)

      // Editor dispose
      editor.dispose()

      // 모델 dispose
      originalModel?.dispose()
      modifiedModel?.dispose()
    }
  }
}, [])
```

**결과**: ❌ **여전히 에러 발생**
- dispose 순서를 보장해도 @monaco-editor/react 내부에서 이미 dispose 시도

---

### ❌ 시도 2: setValue()로 수동 갱신 (실패)

```tsx
useEffect(() => {
  if (!editorRef.current) return

  const originalModel = editor.getOriginalEditor().getModel()
  const modifiedModel = editor.getModifiedEditor().getModel()

  // props 변경 시 setValue()로 내용만 갱신 (모델 재생성 방지)
  if (originalModel) originalModel.setValue(original)
  if (modifiedModel) modifiedModel.setValue(modified)
}, [original, modified])
```

**결과**: ❌ **여전히 에러 발생**
- DiffEditor가 props 변경을 감지하면 setValue() 이전에 모델 재생성 시도

---

### ❌ 시도 3: Props 고정 + 수동 갱신 (실패)

```tsx
// 초기값을 ref로 저장하여 DiffEditor props는 절대 변경하지 않음
const initialOriginalRef = useRef(original)
const initialModifiedRef = useRef(modified)

<DiffEditor
  original={initialOriginalRef.current}  // 고정!
  modified={initialModifiedRef.current}  // 고정!
/>

// useEffect에서 setValue()로만 갱신
useEffect(() => {
  updateEditorContent(original, modified)
}, [original, modified])
```

**결과**: ❌ **여전히 에러 발생**
- 150줄의 복잡한 코드, 여전히 unmount 시 충돌

---

## ✅ 최종 해결: keepCurrentModel Props

### 🎯 공식 Props 발견!

@monaco-editor/react 문서를 자세히 읽다가, **공식적으로 제공하는 props**를 발견했습니다:

```tsx
<DiffEditor
  original={original}
  modified={modified}
  keepCurrentOriginalModel={true}   // ← 핵심!
  keepCurrentModifiedModel={true}   // ← 핵심!
/>
```

**이 props의 역할**:
- `keepCurrentOriginalModel={true}`: props 변경 시 기존 original 모델 유지, setValue()로 내용만 갱신
- `keepCurrentModifiedModel={true}`: props 변경 시 기존 modified 모델 유지, setValue()로 내용만 갱신
- **모델 재생성 완전 방지** → dispose 충돌 근본적으로 해결

---

### 📝 최종 구현 코드

```tsx
/**
 * Code Diff Viewer Component
 *
 * keepCurrentOriginalModel, keepCurrentModifiedModel props로
 * props 변경 시 모델 재생성 방지
 */
import { useEffect, useRef, memo } from "react"
import { DiffEditor } from "@monaco-editor/react"
import type { editor } from "monaco-editor"

interface CodeDiffViewerProps {
  original: string
  modified: string
  language: string
  visible: boolean
}

export const CodeDiffViewer = memo(function CodeDiffViewer({
  original,
  modified,
  language,
  visible,
}: CodeDiffViewerProps) {
  const editorRef = useRef<editor.IStandaloneDiffEditor | null>(null)

  const handleEditorDidMount = (editor: editor.IStandaloneDiffEditor) => {
    editorRef.current = editor
  }

  useEffect(() => {
    return () => {
      if (editorRef.current) {
        try {
          editorRef.current.dispose()
        } catch (error) {
          console.debug('DiffEditor cleanup:', error)
        }
        editorRef.current = null
      }
    }
  }, [])

  if (!visible) {
    return null
  }

  return (
    <div className="flex-1 border rounded-lg overflow-hidden">
      <DiffEditor
        height="100%"
        language={language}
        original={original}
        modified={modified}
        theme="vs-dark"
        onMount={handleEditorDidMount}
        keepCurrentOriginalModel={true}    // ← 핵심 해결!
        keepCurrentModifiedModel={true}    // ← 핵심 해결!
        options={{
          readOnly: true,
          renderSideBySide: true,
          minimap: { enabled: false },
          fontSize: 13,
          automaticLayout: true,
        }}
      />
    </div>
  )
})
```

**코드 개선 효과**:
- ✅ **150줄 → 85줄** (45% 감소)
- ✅ **복잡한 cleanup 로직 제거**
- ✅ **공식 지원 기능 사용** (유지보수성 향상)
- ✅ **에러 완전 해결**

---

## 결과 및 검증

### 🧪 테스트 시나리오

```
1. 함수 히스토리 모달 열기
2. 다양한 버전 빠르게 전환 (20회)
3. 다양한 파일 빠르게 전환 (20회)
4. 모달 닫기
5. 1-4를 20회 반복
```

### ✅ Before vs After

| 항목 | Before | After |
|------|--------|-------|
| 콘솔 에러 | ❌ 매번 발생 | ✅ 완전히 사라짐 |
| 코드 라인 수 | 150줄 | 85줄 (-45%) |
| 복잡도 | 높음 (ref, useEffect 다수) | 낮음 (props 2개 추가) |
| 메모리 누수 | ⚠️ 가능성 존재 | ✅ 없음 |
| 유지보수성 | 낮음 (수동 관리) | 높음 (공식 API) |

---

## 핵심 교훈

### 💡 문제 해결 과정

1. **복잡한 해결책부터 시도하지 말 것**
   - 150줄의 수동 cleanup 코드 작성
   - 실제로는 props 2개로 해결 가능

2. **공식 문서를 끝까지 읽을 것**
   - `keepCurrentOriginalModel` props는 문서 하단에 있었음
   - 초기에 발견했다면 몇 시간 절약 가능

3. **라이브러리 내부 동작 이해의 중요성**
   - Monaco Editor의 모델 관리 메커니즘 이해
   - React lifecycle과의 상호작용 파악

### 🎯 일반화된 해결 전략

Monaco Editor와 같은 복잡한 외부 라이브러리 사용 시:

1. **공식 문서 정독**
   - 특히 "Advanced Usage", "Performance", "Troubleshooting" 섹션

2. **Props 탐색**
   - TypeScript 타입 정의 파일 확인
   - JSDoc 주석 읽기

3. **GitHub Issues 검색**
   - 동일한 문제를 겪은 사람들의 해결책
   - Maintainer의 권장 방법

4. **최소한의 코드로 시작**
   - 복잡한 해결책은 마지막 수단
   - 공식 API가 있는지 먼저 확인

---

## 참고 자료

- [@monaco-editor/react GitHub](https://github.com/suren-atoyan/monaco-react)
- [Monaco Editor API](https://microsoft.github.io/monaco-editor/api/index.html)
- [React Performance Optimization](https://react.dev/learn/render-and-commit)

---

## 마치며

Monaco Editor의 "TextModel got disposed" 에러는 많은 개발자들이 겪는 문제입니다. 복잡한 수동 관리 대신, **공식적으로 제공하는 props를 활용**하면 간단하게 해결할 수 있습니다.

**핵심은 단 2줄입니다**:
```tsx
keepCurrentOriginalModel={true}
keepCurrentModifiedModel={true}
```

이 글이 같은 문제로 고민하는 개발자들에게 도움이 되길 바랍니다! 🚀

---

**imprun.dev 팀**
*Kubernetes 기반 서버리스 Cloud Function 플랫폼*
