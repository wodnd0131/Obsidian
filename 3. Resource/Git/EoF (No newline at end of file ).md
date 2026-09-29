#Java/Idiom #Status/완료 

### 뭔가요?

파일의 마지막 줄에 개행문자(`\n`)가 없는 상태예요.

```
// ❌ 개행 없음
public class Lotto {
    ...
}   ← 여기서 파일 끝 (줄바꿈 없음)

// ✅ 개행 있음
public class Lotto {
    ...
}
↵  ← 마지막에 \n 존재
```

---

### 왜 문제가 되나요?

**POSIX 표준** 때문이에요. POSIX에서 텍스트 파일의 줄(line)은 `\n`으로 끝나야 한다고 정의해요. 개행이 없으면 엄밀히 말해 마지막 줄이 "줄"로 인식되지 않아요.

**Git에서 diff가 지저분해져요**

```bash
# 개행 없는 파일을 수정하면
-}
\ No newline at end of file
+}
\ No newline at end of file

# 내용은 안 바뀌었는데 diff에 노이즈가 생김
```

**협업 시 불필요한 변경이 생겨요**

```bash
# A가 개행 추가하면
# 코드 변경 없이 파일이 수정된 것으로 잡힘
# → 코드 리뷰에서 불필요한 diff 발생
```

---

### 해결 방법

**IntelliJ 설정**

```
Settings
→ Editor
→ General
→ "Ensure every saved file ends with a line break" 체크
```

**EditorConfig로 팀 전체 통일**

```ini
# .editorconfig
root = true

[*]
insert_final_newline = true  # 모든 파일에 개행 강제
```

프로젝트 루트에 `.editorconfig` 파일 하나로 팀원 전체 에디터 설정을 통일할 수 있어요.

---

### 사소해 보이지만 중요한 이유

```
작은 컨벤션을 지키는 습관
        ↓
코드 리뷰에서 본질적인 내용에 집중 가능
        ↓
불필요한 diff 노이즈 제거
        ↓
협업 품질 향상
```

스터디에서 이런 작은 것들을 짚어주는 게 중요한 이유는, 실무에서 PR 올렸을 때 **개행 하나 때문에 리뷰어의 집중력이 흐트러지는 걸 방지**하기 위해서예요.