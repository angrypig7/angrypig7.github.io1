---
tag: 네트워크, 뻘글, 일상
date: 2023-08-22
---

다른일을 하다가 갑자기 대학교 모바일 인증 앱의 관련 API를 분석해보고 싶어서 진행한 프로젝트인데, 흥미로운 내용인 것 같아 기록으로 남겨봅니다..!

사실 개념은 비교적 간단한데에 비해, 과정은 조금 번거로웠습니다.

### TLDR: 3줄 요약

- 프록서 서버 만들어서 중간에서 캡쳐하면 되나, TLS암호화된 HTTPS는 해독불가
- HTTPS도 해독하려면 프록시 서버에서 생성한 CA인증서를 휴대폰에 설치한 후,
- 안드로이드 앱의 `targetSdkVersion`을 23 이하로 강제로 내리면 앱이 바보가 되어서 아무 인증서나 주는대로 믿음, MITM 공격 수행 가능


---
# 안드로이드 앱에서 네트워크 활동 분석하기
###### 부재: MITM 공격

웹 환경이라면 Chrome DevTools 등의 분석툴로 간편하게 모든 네트워크 트래픽을 분석할 수 있지만, 모바일 앱의 트래픽의 경우 조금 손이 많이 갑니다.

기본적인 개념은 이러합니다.

1. VPN/프록시 서버를 만들어서 데이터를 모두 우회시킨다
2. 구축한 프록시 서버에서 모든 데이터를 캡쳐하여 저장한다
3. 분석한다

즉, 프록시서버에서 오고가는 패킷을 *무단으로* 모니터링 하는 것입니다.

## Steps
- 안드로이드 앱에 PCAPdroid 어플리케이션 설치 및 시키는대로 권한 할당
- 아래 화면에서 Target app 설정: 휴대폰의 모든 네트워크를 기록하는 것이 아니라, 필요한 앱만 불러올 수 있습니다
- ‘Ready’ 버튼을 누르면 그때부터 실시간으로 네트워크 캡쳐가 시작되며, `pcap` 파일로 추출도 가능합니다

![Untitled|400](PCAPdroid%20main.png)

## 실행 결과
| 모바일 학생증 실행 | 데이터 캡쳐 화면 | connection list |
| --- | --- | --- |
| ![Untitled](모바일학생증.png) | ![Untitled](데이터%20캡쳐.png) | ![Untitled](connection%20list.png) |

캡쳐 시작 후 모바일 학생증 앱을 실행하면, 이처럼 7.3KB의 데이터를 캡쳐한 것을 확인할 수 있습니다.
그리고 세번째 화면에서 보이듯이 DNS lookup 요청과, HTTPS 통신 요청 패킷 2개의 connection이 이뤄졌던 것을 확인할 수 있습니다.
(packet 단위가 아니라 connection 단위로 정리해서 보여줍니다)

캡쳐 데이터는 `pcap`파일로 추출이 가능하며 PC에서 WireShark등의 툴로 분석도 가능합니다.

## 하지만..!
모바일 환경에서는 한가지 큰 문제가 있습니다.

Chrome DevTools의 경우 Chrome 내부에서 데이터를 불러오는 것이기에 HTTPS 데이터도 해독된 상태이나, 중간자인 프록시 서버에서는 HTTPS연결처럼 TLS 암호화된 데이터를 해독하지 못합니다.

![Untitled|500](conn%20list.png)

자세히 보면 plain-text로 주고받는 DNS데이터는 자물쇠가 풀려있으나 TLS암호화가 이뤄진 HTTPS통신은 해독을 하지 못하였다는 표시로 자물쇠가 걸려 있음을 확인할 수 있습니다.

| DNS payload data | HTTPS payload data |
| --- | --- |
| ![Untitled](DNS%20payload.png) | ![Untitled](HTTPS%20payload.png) |

평문으로 전송되어 해독가능한 좌측의 DNS 요청과 달리 우측의 HTTPS 연결은 암호화된 상태로, 해독이 불가능합니다.

###### 해독이 불가능한 이유
클라이언트(앱)과 서버(실제서버)사이에서 암호화가 이뤄졌기에, 중간에서 엿듣기만 하는 분석 앱은 이를 해독하지 못합니다.


---
# 그래서 어떻게 하느냐: MITM
암호화를 2번 진행하면 됩니다.

클라이언트와 프록시 서버 사이에서 임의로(무단으로) 한번, 그리고 프록시 서버와 실제 서버 사이에서 다시 한번.

이때 클라이언트가 자신이 실제 서버와 암호화를 하고 있다고 **착각하게 만드는** 것이 핵심입니다. **(MITM 공격)**

## CA 인증서 설치
그러기 위해서는 프록시 서버에서 사용할 CA인증서(TLS 암호화에 사용됨)가 위조되지 않은 것이라고 주장할 수 있도록 미리 휴대폰에 설치를 해 두어야 합니다.

- ‘PCAPdroid mitm’ 애드온 어플리케이션을 설치하고 시키는대로 절차를 밟습니다.
- 이때, CA인증서를 파일로 추출해주는데, 이를 안드로이드 보안 정책상 바로 설치는 불가하고, 휴대폰 설정에 들어가서 수동으로 파일을 선택하고 설치해주면 됩니다. (그랬던 것 같습니다)
- 'Decryption Rules'에 트래픽을 캡쳐할 앱을 목록에 추가해줍니다.
- 이후 다시 앱에서 캡쳐를 진행합니다.

![Untitled|500](proxy%20cert%20error.png)

###### 하지만, 정상적인 암호화가 이뤄지지 않았음을 앱이 감지하였습니다
안드로이드 어플리케이션도 바보가 아닌지라 아무 인증서나 주는대로 받아먹지 않습니다.

공인기관에서 서명한 인증서가 아닌, 프록시 서버가 마음대로 서명하고 만들어낸 인증서기에 이처럼 `The client does not trust the proxy’s certificate` 에러를 뱉습니다.

#### 인증서를 신뢰하도록 만들기

[android dev network configurations 문서](https://developer.android.com/training/articles/security-config) 참고

임의로 추가한 인증서를 탐지하는 방법은 크게 2가지가 있습니다.

- **Certificate Pinning**: 앱을 빌드하는 시점에서 신뢰할 인증서를 화이트리스트 처리한 후 이들만 신뢰하는 것. 인증서 갱신 가능성 때문에 권장되지 않으며, 엄격한 사내용/내부용 서비스가 아닌 이상 잘 사용되지 않음
- **Trust Store**: 사용자가 임의로 설치한 인증서는 기본적으로 ‘신뢰하지 않음’ 처리를 하는 보안 설정입니다. 개발자가 임의로 허용처리를 해줘야 하며, 안드로이드 7.0(SDK 24~)부터는 기본 설정으로 탑재가 됩니다.

결론적으로 임의로 설치한 인증서는 Trust Store 기법에 의해 막혔으나, 해결책도 원리는 매우 간단합니다.

**위 탐지 기법은 안드로이드 7.0과 함께나온 Android SDK Version 24 이후에서부터 디폴트로 탑재되어 있기에, 앱에서 지원하는 Android SDK 버전(`targetSdkVersion`)을 23 이하로 설정해버리면 위 기능은 기본으로 동작하지 않습니다.**


---
# 앱 수정, SDK버전 강제로 하향조정

절차는 아래와 같습니다

- 안드로이드 어플리케이션을 `apk` 파일로 추출
- PC에 자바 JDK 및 [apktools](https://apktool.org/) 설치 후 아래 명령 실행
- `apktool [APK_FILE]`: apk file을 reverse engineering 하여 파일 추출

![Untitled](attachments/Untitled%208.png)

- `apktool.yml` 파일에서 `targetSdkVersion` 항목을 설정된 31에서 23 이하로 수정, 저장
	![Untitled](attachments/Untitled%209.png)
- `apktool build [OUTPUT_FOLDER]`: 추출한 파일들을 기반으로 build(`apk` 파일로 추출됨)
    - 하지만 아직 인증서로 서명되지 않았기에 휴대폰에서는 설치를 거부합니다
    - 네트워크에서 사용되는 CA인증서와 다른 개념으로, `apk` 파일 자체의 무결성을 인증하기 위한 용도
- `keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000`: 임의로 인증서 생성
- `jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore MobileID-rebuild.apk alias_name`: 임의로 만든 인증서로 `apk` 파일 서명, 완성됨
- 기존 앱 제거 후 새로 설치
    - 기존 앱에서 사용된 원본 인증서가 아니기에 제거 후 설치해야함

## apk 파일 인증서 부가설명

구글 아저씨들은 저희같은 사람들이 마음대로 앱을 수정~~*훼손*~~ 하도록 냅두지 않았습니다.

그렇기에 원작자는 apk파일에 서명을 하는데, 저희는 원본 인증서가 없기에 임의로 생성한 인증서로 서명을 하였고, 이제 인증서가 달라짐을 감지한 휴대폰에서는 수정한 앱의 설치를 거부하게 됩니다.

또한 임의로 서명한 인증서를 신뢰하기 위해서 플레이 스토어의 Play Protect 설정을 꺼야 할 수도 있습니다.


---
# 빠바바밤!

이제 `targetSdkVersion`이 23으로 낮아진 *~~바보가 된~~* 앱을 설치하였습니다.

PCAPdroid 앱에서 다시 네트워크 캡쳐를 진행하면 앱은 프록시 서버가 생성한 CA인증서를 신뢰하며, 모든 데이터를 평문으로 확인 가능함을 알 수 있습니다.

| decrypted connections | decrypted HTTPS payload |
| --- | --- |
| ![Untitled](decrypted%20conn%20list.png) | ![Untitled](decrypted%20HTTPS%20payload%20data.png) |


이제 `pcap`파일 및 해독을 위한 `sslkeylogfile` 파일의 정상적인 추출이 가능합니다.

`pcap`파일에는 암호화된 네트워크 데이터, 그리고 `sslkeylogfile`에는 이를 해독 가능한 패킷별 키가 아래 예시와 같이 리스트로 저장되어 있습니다.

```
SERVER_HANDSHAKE_TRAFFIC_SECRET 17a992f4104a498b42fc7c5b9a8d01a9f52d3fdce8ae39b59f470f399fdff8d9 2a2fa8da8db113616195b9972c9621d1a9cab2f39ccad2e27d72a246f49ce75497c242bec28e79460b9ec66057ae1c05
CLIENT_HANDSHAKE_TRAFFIC_SECRET 17a992f4104a498b42fc7c5b9a8d01a9f52d3fdce8ae39b59f470f399fdff8d9 99b995da118ada9787713323863c5b5a2e3e50d6004a22ae130e3f4f4dbcb2a9e8e68dffb243eee7c14cd3da9ee3fda5
EXPORTER_SECRET 17a992f4104a498b42fc7c5b9a8d01a9f52d3fdce8ae39b59f470f399fdff8d9 ec105f5940c333cf1958be90d7274c0fb3feb96b9fffb9d0851fc468a0d81fbae5bdf718df4119554e7fe834df7808c8
SERVER_TRAFFIC_SECRET_0 17a992f4104a498b42fc7c5b9a8d01a9f52d3fdce8ae39b59f470f399fdff8d9 04ade585f47aea1f0f6f67902e637ecc1e58a0c0e034cf114117b78f5ae946652e167ee1985441fc6b83f8746ec52c8f
SERVER_HANDSHAKE_TRAFFIC_SECRET 2f562f0109554ec0732f1af1c1e49370ee2007ffd2796c8efc71867122646e2d b75943b8a12a8e29626dcd0c0bb6b818ded875c8404e8c0f61ac27cab9e72b875143b48b7aefb4c431d46cbcfab06647
CLIENT_HANDSHAKE_TRAFFIC_SECRET 2f562f0109554ec0732f1af1c1e49370ee2007ffd2796c8efc71867122646e2d bb31d5a2acbd07f8c7671c515e18c505cd33a59e833554b9bc37bb3279ad0f23585cb8259c1dbb4f823fd8af88df8d74
...(생략)
```

PC에서 WireShark 등의 프로그램에 두 파일을 함께 불러오면 보다 정밀한 분석이 가능해집니다.

![Untitled](attachments/Untitled%2012.png)


p.s. 사실 이렇게 PCAPdroid 앱으로 하는것보다 Burp Suite같은 트래픽 분석용 프록시 서버 PC에서 세팅하는게 훨씬 편합니다

