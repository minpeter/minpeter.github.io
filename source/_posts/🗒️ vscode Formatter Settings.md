---
title: "🗒️ vscode Formatter Settings"
description: "Prettier만으로 해결되지 않는 vscode Formatter 설정을 도와줄 가이드"
date: 2022-03-27T10:27:57.379Z
tags: ["Formatter","vscode"]
---
vscode를 사용할 때 Formatter는 상당히 중요하다.
보통 구글링 해보면 Prettier와 Beautify를 추천한다.

나 같은 경우에는 Prettier를 선택하여 사용하였는데, 가끔 동작하지 않는 언어가 있었다.
예를 들면 rust나 go, java같은 경우에는 prettier가 동작하지 않았다.

이런 경우에는 해당 언어는 별도의 포맷터를 사용하도록 설정하면 되는데,
보통 해당 언어의 확장기능에 포함되있는 경우가 많다.

아래의 settings.json 파일은 내 vscode formatter 설정이다.

```json
"editor.formatOnSave": true,
"[rust]": {
  "editor.defaultFormatter": "matklad.rust-analyzer"
},
"[go]": {
  "editor.defaultFormatter": "golang.go"
},
"[java]": {
  "editor.defaultFormatter": "redhat.java"
},
"editor.defaultFormatter": "esbenp.prettier-vscode",
```

다음 설정을 적용하면 적어도 rust, go, java와 prettier가 지원하는 언어는 문제 없을꺼다.