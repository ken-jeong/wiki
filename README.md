# Archive

Quartz 4 기반 [개인 학습·기술 노트](https://ken-jeong.github.io/wiki/).

![노트 그래프 뷰](assets/graph-view.png)

## 로컬 실행

Node.js 22.22.0 이상, npm 10.9.2 이상이 필요합니다.

`content/`에 노트·홈(`index.md`)을 추가하고 실행한다.

```bash
npm ci
npx quartz build --serve
```

배포용 파일은 `npx quartz build`로 `public/`에 생성합니다.

## 노트 동기화

`copy.sh`의 `SRC`를 Obsidian vault 경로로 설정하고 실행하면 `draft: false`인 Markdown 문서를 가져옵니다.

```bash
./copy.sh
```

> 기존 `content/` 내용은 `.gitkeep`을 제외하고 삭제됩니다. 이미지 등 첨부 파일은 별도로 복사해야 합니다.

## 주요 파일

- `content/`: 공개할 Markdown 노트
- `quartz.config.ts`: 사이트 정보, 테마, 플러그인 설정
- `quartz.layout.ts`: 화면 레이아웃
- `quartz/styles/custom.scss`: 사용자 정의 스타일

## 배포

`v4` push 또는 Actions의 `Deploy to GitHub Pages` 수동 실행으로 배포한다.
Settings → Pages → Source는 `GitHub Actions`, `quartz.config.ts`의 `baseUrl`은 `ken-jeong.github.io/wiki`.

## 라이선스

Quartz 소스 코드는 [MIT License](LICENSE.txt)를 따릅니다. 노트 재배포에는 별도 허가가 필요합니다.
