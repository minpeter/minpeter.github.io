---
title: 백엔드에서 client의 IP를 로깅하는 방법
date: 2023-02-25 03:03:18 +0900
tags: ["http", " backend", " reverse proxy"]
categories: ""
---

안녕 백엔드 프로그래머 여러분

아마도 여러분들은 직접 만든 백엔드가 하나 쯤은 있을거다.

뭐 나의 경우에는 [https://api.tempfile.ml/](https://api.tempfile.ml/) 이 있었는데 한가지 아쉬운 점이 있었다.

바로 user access logging.. 접속하는 유저들의 기록을 남기는 작업이 부족했다.

사실 꼭 필요하지 않을 수도 있는데 뭐 필요하다고 생각해보자

여러가지 케이스가 있을 수 있다.

우선 클라이언트와 서버가 바로 통신하는 경우

매우 간단하게 처리할 수 있다.

바로 RemoteAddr 필드를 파싱해서 사용하면 된다.

![스타벅스 아이피다.. 괴롭히지 말자.](/images/client-ip/Untitled.png)

스타벅스 아이피다.. 괴롭히지 말자.

두번째 경우다.

서버와 중간에 프록시 서버가 존재하는 경우이다.

이 경우에는 새롭게 알아야되는 것이 있는데 X-Forwarded-For 헤더이다.

해당 헤더는 프록시를 지날때 유실되는 client IP를 프록시 뒤에 서비스까지 전달하기 위해 만든 사실상의 표준 헤더이다.

다음 스크린샷은 whoami 서비스가 2개의 리버스 프록시 뒤에서 실행되고 있는 경우이다.

![Untitled](/images/client-ip/Untitled%201.png)

자세히보면 원본 클라이언트 IP를 넣을 수 있는 부분이 2가지 존재하는데 클라우드플레어에서 추가한 `Cf-Connecting-Ip` 필드와 `X-Forwarded-For` 필드의 첫번째 값이다.

우선 `Cf-Connecting-Ip` 의 경우 클라우드플레어를 사용하는 경우에만 해당되므로 제외하고 아래의 `X-Forwarded-For` 헤더를 유심히 살펴보자.

첫번째 값은 분명 client ip 이다. 그럼 다음 값을 무엇일까?

바로 클라우드플레어 프록시 서버의 아이피이고 그 다음 위에선 client ip 였던 RemoteAddr의 경우 클라우드플레어 다음 리버스프록시인 traefik의 아이피이다.

아이피는 다르지만 다음과 같은 구조를 지닌다.

![Untitled](/images/client-ip/Untitled%202.png)

프록시를 하나 지날때 기존의 RemoteAddr 부분을 X-Forwarded-For 헤더에 append하고 자신의 IP가 RemoteAddr에 들어가는 형식이다.

따라서 “대부분의 경우”에 다음 로직은 정상적으로 클라이언트의 IP를 가져오는데 성공한다.

```go
forward := r.Header.Get("X-Forwarded-For")
  var ip string
  var err error
  if forward != "" {
    // With proxy
    ip = strings.Split(forward, ",")[0]
  } else {
    // Without proxy
    ip, _, err = net.SplitHostPort(r.RemoteAddr)
    if err != nil {
      http.Error(w, "Error parsing remote address ["+r.RemoteAddr+"]", http.StatusInternalServerError)
      return
    }
  }
```

근데 위에 로직에는 심각한 오류가 한가지 존재한다.

만약 위에서 넣은 IP로 접근제어를 한다고 생각해보자.

서버가 프록시가 있는 상태이든 없는 상태이든 r.Header라는 값은 “유저에게서” 전송되게 되어있다.

또한 `X-Forwarded-For` 헤더는 말 그대로 HTTP 헤더이다.

헤더는 간단하게 조작 가능하다.

만약 `X-Forwarded-For` 헤더를 처음 전송할 때 “추가” 한 상태로 전송하게 된다면 다음과 같은 형태로 전송된다.

X-Forwarded-For: <변조하여 추가한 IP A>, <클라이언트의 실제 IP B>, <proxy 1 C>

물론 경우에 따라서 proxy C가 없을 수도 있고, 클라이언트의 실제 IP가 RemoteAddr에 있을 수도 있지만 상관 없다.

위에 논리대로면 XFF 헤헤더가 설정되어 있다. → XFF 헤더 첫번째 값을 읽는다. 이기 때문에 유저가 변조하여 추가한 값이 실제 IP로 인식되게 된다.

이건 문제다.

따라서 이걸 처리할 별도의 로직이 필요한데 …..

대부분의 프로덕션 프로그램에서 이 문제를 해결하는 방법을 찾아냈다.

사실 이 기능과 관련된 어떤 설정도 허용하지 않는다는 원래 목표랑은 약간 달라지긴 했지만..

아무튼 다음과 같이 해결 가능하다.

새로운 설정 값 FORWORDED를 생성한다.

이 설정값은 숫자로 각 숫자는 다음을 나타낸다.

| 숫자 | 의미                                                                         |
| ---- | ---------------------------------------------------------------------------- |
| 0    | 리버스프록시가 없으며 RemoteAddr를 사용합니다.                               |
| 1    | 서비스 앞에 1개의 리버스 프록시가 있으며 XFF 헤더의 -1 번째 값을 사용합니다. |
| ..   | ..                                                                           |
| 3    | 서비스 앞에 3개의 리버스 프록시가 있으며 XFF 헤더의 -3 번째 값을 사용합니다. |
| ..   | ..                                                                           |

이렇게 한다면 당장 사용자가 헤헤더 값을 변조하여 보낸 경우에도 설정값을 초과하는 요청에 대해서는 그냥 무시 하기에 별 영양을 받지 않게 된다.

근데 이 방법은 앞서 말했듯 완벽하지 않다.

왜냐하면 뭔가 설정값이 추가되었기 때문에 모든 환경에서 적응하여 서비스하는 것이 불가능하다.

다음날이다.

어쩌다 보니 burp suite를 이용해서 내가 제작한 사이트를 공격하고 코드를 다시 검증하고 자칫 그냥 넘어갈 수 있는 부분을 점검하기까지 했다.

이 문제는 어떻게 알려져 있을까?

다음과 같은 자료들을 찾을 수 있었는데 그 중 흥미로운 몇가지 자료다.

![Untitled](/images/client-ip/Untitled%203.png)

[https://blog.ircmaxell.com/2012/11/anatomy-of-attack-how-i-hacked.html](https://blog.ircmaxell.com/2012/11/anatomy-of-attack-how-i-hacked.html)

한마디로 글쓴이가 SSH 터널로 인해 우발적으로 XFF 헤헤더에 127.0.0.1 주소가 들어가게 되었는데 이를 스택오버플로우가 “관리자”의 접근으로 인식해 권한 상승이 일어났다는 이야기이다.

또 이런 자료도 찾을 수 있었다.

[https://www.acunetix.com/vulnerabilities/web/x-forwarded-for-http-header-security-bypass/](https://www.acunetix.com/vulnerabilities/web/x-forwarded-for-http-header-security-bypass/)

XFF HTTP header security bypass, 분류번호 CWE-289

CTFd 처럼 프록시 홉을 제한하여 방지하는 로직이 있는 경우도 있었지만 무려 스택오버플로우에서 권한 상승이 일어나기도 하는 등 상당히 재밌는 기법이였다.

재미있으니 조금 더 나아가 다른 서비스들에서 IP 로깅 방법을 공격하고 분석해보자.

먼저 [ifconfig.me](http://ifconfig.me) 이다.

평소에 즐겨쓰는 서비스이기에 선택해보았다.

![Untitled](/images/client-ip/Untitled%204.png)

다음과 같이 proxy option를 선택하여 intercept를 진행했다.

![Untitled](/images/client-ip/Untitled%205.png)

아무런 변조를 하지 않았을 떄 응답이다.

![Untitled](/images/client-ip/Untitled%206.png)

요청 헤더에 강제로 값을 넣어보았다.

![Untitled](/images/client-ip/Untitled%207.png)

분명 더 값은 조작되었다.

그렇다면 ifconfig.me는 어떤 방식으로 client ip를 확인할까?

github에서 repo를 찾았다.

[GitHub - pmarques/ifconfig.me: Simple HTTP application for demos and tests](https://github.com/pmarques/ifconfig.me)

인줄 알았으나 다른 서비스였다는 이야기..

실제로 로컬에서 실행시키고 “아 이건 아니다 싶었다.”

심지어 여기에선 해당 우회에 대한 대비를 하지 않았으며 사실 할 필요가 있지는.. 그냥 보여주는거니까 ~

사실 이외에도 다양한 래포가 “A IP echo server inspired by [http://ifconfig.me](http://ifconfig.me/)” 라는 타이틀 하에 제작되어 있었는데 몇가지 래포의 IP 획득 로직을 가져와보았다.

```go
ip := this.GetString("ip", this.Ctx.Input.IP())
```

여기서 this 변수는 beego 라이브러리의 라우터 파라미터였다.

재밌는 걸 찾았다.

[Security issue: Trusted Reverse Proxy and X-Forwarded-\* headers · Issue #4589 · beego/beego](https://github.com/beego/beego/issues/4589)

지금 내가 해결하고자 하는 문제의 대한 이슈인데.. 그냥 닫혔다.

이렇다는건 대부분의 beego로 작성된 golang 백엔드, 그 중 이전에 언급한 스택오버플로우같은 로직을 사용하는 경우 간단하게 해킹할 수 있다는 이야기가 된다.

```go
// IP returns request client ip.
// if in proxy, return first proxy id.
// if error, return RemoteAddr.
func (input *BeegoInput) IP() string {
	return (*context.BeegoInput)(input).IP()
}
```

현제 beego의 develop 브랜치의 코드 구현 및 주석을 발쵀해 왔다.

음.. 2번에 주석의 뜻을 짐작해보면 “우린 그냥 무지성으로 XXF 더 첫 번째 값 뺴올꺼임 ㅅㄱ ㅎ”

![Untitled](/images/client-ip/Untitled%208.png)

다음은 저 `(*context.BeegoInput)(input).IP()` 구현체다.

[https://github.com/beego/beego/blob/031c0fc8af57ea1a18e21fd5a7a8e6f23c26bbea/server/web/context/input.go](https://github.com/beego/beego/blob/031c0fc8af57ea1a18e21fd5a7a8e6f23c26bbea/server/web/context/input.go) 중 일부코드이다.

```go
// IP returns request client ip.
// if in proxy, return first proxy id.
// if error, return RemoteAddr.
func (input *BeegoInput) IP() string {
	ips := input.Proxy()
	if len(ips) > 0 && ips[0] != "" {
		rip, _, err := net.SplitHostPort(ips[0])
		if err != nil {
			rip = ips[0]
		}
		return rip
	}
	if ip, _, err := net.SplitHostPort(input.Context.Request.RemoteAddr); err == nil {
		return ip
	}
	return input.Context.Request.RemoteAddr
}
```

이거 뭔가 구현이 이상하다…

마지막 희망으로 `input.Proxy()` 구현을 보자

```go
// Proxy returns proxy client ips slice.
func (input *BeegoInput) Proxy() []string {
	if ips := input.Header("X-Forwarded-For"); ips != "" {
		return strings.Split(ips, ",")
	}
	return []string{}
}
```

까약…

이게 맞나 싶은 구현이다.

일단 여기까지 봤으면 조용히 방금 이슈를 열 수 도 있지만… 일단은 beego 내부에서 IP() 구현을 어떻게 사용하는지 검색해보자.

server/web/error.go

```go
"AppError":      fmt.Sprintf("%s:%v", BConfig.AppName, err),
"RequestMethod": ctx.Input.Method(),
"RequestURL":    ctx.Input.URI(),
"RemoteAddr":    ctx.Input.IP(),
"Stack":         stack,
"BeegoVersion":  beego.VERSION,
"GoVersion":     runtime.Version(),
```

이건 일단 RemoteAddr가 아니다. 이러면 에러로그를 속일 수 있게 된다.

server/web/server.go

```go
record := &logs.AccessLogRecord{
RemoteAddr:     ctx.Input.IP(),
RequestTime:    requestTime,
RequestMethod:  r.Method,
Request:        fmt.Sprintf("%s %s %s", r.Method, r.RequestURI, r.Proto),
```

여기서도 마찬가지이다.

server/web/router.go

```go
if p.cfg.RunMode == DEV && !p.cfg.Log.AccessLogs {
		match := map[bool]string{true: "match", false: "nomatch"}
		devInfo := fmt.Sprintf("|%15s|%s %3d %s|%13s|%8s|%s %-7s %s %-3s",
			ctx.Input.IP(),
			logs.ColorByStatus(statusCode), statusCode, logs.ResetColor(),
			timeDur.String(),
			match[findRouter],
```

여기는 dev 모드에서 엑세스로깅을 문제있는 구현체로 하게 된다.

뭐 beego에서 엑세스로깅을 믿지 않는걸로…

다음은 fiber, echo를 알아보겠다.

[🚀 Trusted Reverse Proxy and get c.Hostname() value from X-Forwarded-Host and etc · Issue #1300 · gofiber/fiber](https://github.com/gofiber/fiber/issues/1300)

누가 TrustedProxy 기능을 추가했다

[IP Address | Echo - High performance, minimalist Go web framework](https://echo.labstack.com/guide/ip-address/#case-1-with-no-proxy)

개발자가 알잘딱 해버렸따.

## 결론

이게 내 결론이다.

범용적으로 배포할 백엔드에서 처음 개발자가 별도의 설정 파일 없이 “신뢰할 수 있는” 클라이언트의 IP를 얻는다는건 힘들다.

하지만 몇가지 선택 가능한 옵션이 존재하는데

1. RemoteAddr 만 믿고 백엔드를 프록시 없이 배포하는 경우
2. XFF 헤더 첫번째 값을 사용하나 신뢰할 수 없는 상태로 사용
3. 서버가 배포될 환경의 프록시 갯수를 환경변수로 설정하여 신뢰할 수 있는 프록시 갯수를 설정
   해당 갯수 만큼만 XFF 헤더를 읽도록 설정
4. 신뢰할 수 있는 프록시의 IP 대역대를 설정해두고 가장 가까운 신뢰할 수 없는 XFF헤헤더의 IP를 사용

여기서 3번과 4번은 설정 파일이 필요하나 설정만 잘한다면 언제나 신뢰할 수 있는 IP를 가지게 되고,

1번은 신뢰는 가능하나 환경에 따라 클라이언트 IP가 아닌 경우가 (프록시가 존재하는 경우) 생길 수 있다.

2번은 범용성은 높으나 손쉽게 해킹할 수 있다.

또 이외에 해결방법도 존재한다.

클라우드플레어를 사용한다는 가정하에 Cf-Con… 헤더를 사용하거나 서버 앞단에 있는 프록시에서 X-Real-IP 값을 설정하고 그 걸 신뢰하는 방법등 다양한 방법이 있다.

언제나 선택이 필요하고 또 백엔드 프로그램을 개발할 때나 사용하게 될때 해당 프로그램이 어떠헥 개발되었는지 분석하고 신뢰할 수 있는 방식인지 검증하여야겠다.

그래서 저는 어떻게 개발할거냐면요…

1. XFF 헤더를 사용
2. 무조건 신뢰가 아닌 로컬주소와 클라우드플어의 IP를 신뢰
3. 추가 프록시 옵션으로 RemoteAddr를 사용
4. 추가 옵션 신뢰 가능한 프록시 주소를 추가 할 수 있도록 설정
5. 추가 옵션 프록시 홉 수로만 설정하여 신뢰

근데 끝은 아니다.

그래도 WAF나 알 수 없는 원인으로 인해 XFF 더가 강제 변경되어 고정되거나 할 수도 있기 때문에 케바케다..

뭐 내가 개발할 떄는 열심히 참고해야

---

Q. 전에 한번 관제 인프라 점검 당시 XFF 헤더가 실종되는 이상한 일이 있었었죠?

그때 원인이 뭐였나요?

A. 바보같이 오케스트레이션을 한답시고 설정해둔 swarm에서 overlay 네트워크 설정 덕분에 네트워크가 이상해진 탓이였습니다.

확실히 순련된 기술자가 없으면 기술도입을 미뤄야된다고 느낀 대목이였죠.

Q. ipLogger 프로젝트를 최종적으로 진행하셨던데, 어떤걸 느끼셨나요?

A. 생각보다 많은 프로젝트에서 XFF 헤더를 믿고 있는 것 같고 그 구현이 불안정한 경우도 있었습니다.

물론 ipLogger 프로젝트가 완벽하다는 것은 아니지만 제작하며 로깅 기술과 클라이언트의 진짜 IP를 얻게되는 기술면에서는 확실히 공부가 되었습니다.
