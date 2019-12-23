# 엔라이즈 엔지니어링 블로그
엔라이즈 엔지니어링 블로그는 github page 를 통해 서비스 되며,
hugo 를 사용하여 페이지를 관리합니다.

# 글 쓰기 전 준비
## hugo 설치
자세한 내용은 [hugo](https://gohugo.io/getting-started/installing/) 설치를 참고하세요.

macOS 사용자의 경우 brew 로 hugo 를 설치합니다.

```bash
$ brew install hugo
```

## 저장소 클론
github 저장소를 클론 받습니다.

```bash
$ git clone git@github.com:nrise/nrise.github.io.git
```

## 서버 실행
hugo 서버를 실행합니다.

```bash
$ cd nrise.github.io
$ hugo server -D
```

이후 브라우저에서 127.0.0.1:1313 으로 접속하여 사이트가 로딩되면 글쓰기를 시작할 수 있습니다.

# 글 쓰기
nrise.github.io 저장소 하위 다음 디렉토리에 파일을 작성하면 됩니다.

```bash
$ cd nrise.github.io
$ hugo new posts/my-first-post.md  # my-first-post.md 파일이 생성됩니다.
```

파일 생성 시 명명 규칙은 다음과 같습니다.

* 영어로 작성합니다.
* 단어 단위로 띄어쓰도록 합니다.
* 단어 사이는 - 문자를 사용합니다.

생성된 파일은 markdown 문법으로 게시물을 작성하면 됩니다.

# 글 발행
작성한 글을 발행하기 위해서는 먼저 작성한 글의 상단 메타 정보에 draft 를 false 로 변경해야 합니다.
변경 후 hugo 로 페이지를 새로 빌드 후 github 에 push 하면 게시물 발행이 완료됩니다.

```bash
$ hugo  # 게시물 빌드
$ git commit -a -m "commit message"
$ git push
```
