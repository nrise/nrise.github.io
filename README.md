# 모씨 개발 수첩
## 개요
github.io 는 static html 을 github 에 올리면 자동으로 호스팅을 해 주는 서비스이다.

자세한 내용은 [여기](http://github.io/)를 참고한다.

## 이용 방법
nrise.github.io 를 클론 받은 후에 직접 html 을 수정해서 push 해도 되긴 하지만
그 원시적인 방법으로 작업을 할 거면 이거 안썼지...다음과 같은 절차로 이용하면 된다.

1\. nrise.github.io 를 클론 받는다.
```bash
$ git clone git@github.com:nrise/nrise.github.io.git
```
2\. [jekyll](http://jekyllrb.com/) 을 설치한다.
jekyll 은 static page 를 생성해 주는 ruby 기반의 툴이다.
```bash
$ sudo gem install jekyll
```
3\. 클론 받은 디렉토리로 이동하여 jekyll 서버를 켜서 로컬에서 확인해 본다.
```bash
$ jekyll serve
```
위와 같이 입력하면 4000 번 포트로 서버가 열린다. http://localhost:4000 으로 접속하면 된다.

## 게시물 작성 방법
1\. 클론 받은 저장소의 _posts 디렉토리에 파일을 생성한다. 파일 생성 규칙은 다음과 같다.
* YYYY-MM-DD-TITLE.md 파일 형태여야 한다.
* TITLE 은 가급적 영어로 작성하도록 한다.
 * 예) 2015-01-26-hello-moci.md
 * 파일의 첫 부분엔 Front Matter 내용이 들어가야 한다. jekyll 이 해석하기 위한 기본 정보들이다. 다른 것들은 몰라도 layout 과 title 은 반드시 들어가야 한다. [이곳](https://raw.githubusercontent.com/nrise/nrise.github.io/master/_posts/2015-01-26-hello.md) 을 참고하라.

2\. 해당 마크다운 파일을 에디터로 작성한다. [Sublime Text 2](http://www.sublimetext.com/2) 나 웹 기반의 [stackedit.io](https://stackedit.io/) 를 추천한다. 마크다운 문법은 [이곳](https://guides.github.com/features/mastering-markdown/) 을 참고한다.

3\. 작성이 완료되면 저장 후 로컬에서 결과물을 확인한다.
```bash
$ jekyll serve
```
4\. 수정사항을 커밋하고 푸시하면 몇 초 내외로 사이트에 반영된다.

## 참고
* [jekyll](http://jekyllrb.com/), [jekyll 한글문서](http://jekyllrb-ko.github.io/)
* [markdown](http://nolboo.github.io/blog/2014/04/15/how-to-use-markdown/)
