# Java Save Formatter: 제작 배경과 동작 원리

기준: 0.2.0 구현 · 2026-10-08

Java Save Formatter는 **프로젝트의 Spotless 규칙에 따라 현재 편집 중인 Java 파일을 포맷하는 IntelliJ 플러그인**이다. 포매터를 실행한 Java 프로세스를 종료하지 않고 계속 재사용해, 저장할 때마다 작업을 준비하는 시간을 줄인다. 이렇게 프로세스를 계속 실행해 두는 방식을 이 문서에서는 “상주”라고 부른다.

설치와 설정 방법은 [README](../README.md)를 참고한다. 이 문서는 도구를 만든 이유와 동작 방식, 실행에 필요한 기술을 설명한다.

## 1. 왜 만들었나

IntelliJ에서 저장한 코드가 Gradle의 `spotlessApply`로 정리한 코드와 다르게 나오는 것이 시작이었다. 프로젝트에는 다음과 같은 설정이 있었다.

```groovy
spotless {
    java {
        shortenFullyQualifiedTypes()
        removeUnusedImports()
        palantirJavaFormat('2.97.0')
    }
}
```

이 설정은 세 가지 작업을 순서대로 수행한다.

| 단계 | 역할 | 예시 |
| --- | --- | --- |
| `shortenFullyQualifiedTypes()` | 가능한 경우 패키지 경로를 포함한 타입 이름을 줄이고 import를 추가한다. | `java.util.List<String>` → `List<String>` 및 import 추가 |
| `removeUnusedImports()` | 사용하지 않는 import를 제거한다. | 사용하지 않는 `import java.util.Set;` 삭제 |
| `palantirJavaFormat(...)` | Palantir 규칙으로 코드 배치를 정리한다. | 들여쓰기, 공백, 줄바꿈 등 |

IntelliJ의 Palantir 플러그인은 Palantir 포맷을 적용하지만, 위 설정의 타입 이름 축약이나 미사용 import 제거까지 모두 실행해 주지는 않는다. 같은 Palantir를 쓰더라도 함께 실행하는 단계나 버전, 옵션에 따라 결과가 달라질 수 있다. 저장할 때도 같은 결과를 얻기 위해 Gradle Spotless를 연결했지만, 이번에는 저장할 때마다 기다리는 시간이 생겼다.

Spotless에는 IDE가 지정한 파일만 처리하는 기능인 IDE hook이 있다. `-PspotlessIdeHook=<현재 파일 경로>`를 전달하면 프로젝트 전체가 아닌 해당 파일만 포맷한다. 다만 파일이 하나여도 Gradle이 요청을 받아 작업을 준비하고 실행하는 과정은 필요하다. [Spotless IDE hook 문서](https://github.com/diffplug/spotless/blob/main/plugin-gradle/IDE_HOOK.md)

Gradle은 Daemon이라는 백그라운드 프로세스를 재사용하므로, 매번 Java를 새로 실행하지는 않는다. 그래도 `spotlessApply`를 호출할 때마다 Gradle에 작업을 요청하는 과정은 거쳐야 한다. Java Save Formatter는 처음에 Gradle에서 포맷 설정을 가져온 뒤, **설정이 바뀌기 전까지는 실행 중인 포매터에 현재 파일을 바로 전달한다.** 매번 Gradle에 작업을 요청할 필요가 없어, 평소 저장할 때 기다리는 시간이 줄어든다. [Gradle Daemon 문서](https://docs.gradle.org/current/userguide/gradle_daemon.html)

## 2. Node, language server, LSP에서 얻은 아이디어

아이디어는 “VS Code에서 ESLint나 Prettier가 빠르게 반응하는 것처럼, Java 포매터도 계속 실행해 두면 되지 않을까?”라는 질문에서 시작했다.

이 아이디어를 이해하려면 Node.js, language server, LSP가 각각 무엇인지 먼저 구분할 필요가 있다.

| 개념 | 의미 | 이 도구와의 관계 |
| --- | --- | --- |
| Node.js | JavaScript 프로그램을 실행하는 환경 | 프로그램을 계속 실행해 두는 방식에 참고했다. 이 도구를 실행하는 데 Node.js는 필요하지 않다. |
| Language server | 편집기에 진단, 자동 완성, 포맷 같은 언어 기능을 제공하는 프로그램 | 편집기는 요청을 보내고 별도 프로그램이 처리하는 구조를 참고했다. |
| LSP | 편집기와 언어 서버가 요청·응답을 주고받는 표준 프로토콜 | 현재 구현에는 사용하지 않는다. 통신 규칙을 정하는 역할이다. |

LSP는 편집기와 서버가 어떤 형식으로 요청과 응답을 주고받을지 정한다. 메시지는 JSON-RPC를 기반으로 한다. 서버를 계속 실행해 두거나 이전에 읽은 설정을 재사용해 속도를 높이는 일은 각 서버에서 구현한다. [LSP 명세](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)

VS Code의 ESLint 확장은 클라이언트와 서버로 나뉘어 동작한다. Prettier 확장은 Prettier 라이브러리를 불러와 사용하며, 이 라이브러리는 `format()` 함수와 읽어 둔 설정을 재사용하는 캐시를 제공한다. 두 확장의 구조가 같지는 않지만, 여기에서 **필요한 프로그램과 라이브러리를 준비해 두고 현재 문서의 처리만 요청한다**는 아이디어를 얻었다. [VS Code ESLint](https://github.com/microsoft/vscode-eslint), [VS Code Prettier](https://github.com/prettier/prettier-vscode), [Prettier API](https://prettier.io/docs/api)

### 왜 Node 서버 대신 Java 프로세스를 사용했나

Spotless와 Palantir Java Format을 실행하려면 JVM이 필요하다. Node로 요청을 받는 서버를 만들어도, 실제 포맷을 맡을 Java 프로그램은 따로 있어야 한다.

- Node가 요청마다 Java를 새로 실행하면 JVM 시작과 라이브러리 초기화가 반복된다.
- Node와 Java를 모두 상주시킬 수도 있지만, 현재 요구사항에서는 중간 프로세스와 통신 단계가 하나 더 생긴다.
- IntelliJ 플러그인이 상주 Java 프로세스에 직접 요청하면 기존 라이브러리를 사용하면서 구조를 단순하게 유지할 수 있다.

그래서 **IntelliJ 플러그인이 상주 Java worker에 직접 요청하는 구조**를 선택했다. worker는 포맷 요청을 기다렸다가 처리하는 별도 Java 프로세스다. 플러그인과 worker는 표준 입력·출력을 통해 데이터를 주고받으므로 HTTP 서버나 네트워크 포트가 필요하지 않다. Spring Boot나 LSP도 사용하지 않는다.

## 3. 구현이 발전한 과정

처음에는 정해진 세 단계와 버전으로 상주 포매터를 만들어 속도와 결과를 검증했다. 이것이 0.1.0 방식이다. 하지만 프로젝트의 `build.gradle`을 바꾸어도 고정된 엔진 설정은 함께 바뀌지 않는 문제가 있었다.

0.2.0부터는 **Gradle이 프로젝트 설정을 읽고 구성한 포매터를 가져온다.** 따라서 프로젝트에서 Palantir 버전이나 단계 순서를 바꾸면 도구에도 반영된다. 옵션과 라이브러리, 적용할 파일과 제외할 파일도 함께 가져온다.

Gradle 설정에는 변수나 외부 스크립트, 다른 플러그인의 로직이 포함될 수 있다. `build.gradle`의 일부 문장만 읽어서는 최종 설정을 알기 어렵다. 그래서 프로젝트에 포함된 Gradle 실행 도구인 wrapper로 설정을 직접 읽고 구성하게 했다. 이 과정을 이후 설명에서는 “설정 평가”라고 부른다.

## 4. 전체 구조

```text
IntelliJ: 저장 / Reformat Code
  │ 현재 Java 문서의 경로와 내용
  ▼
Java Save Formatter 플러그인
  ├─ 첫 요청이거나 설정이 바뀜
  │    └─ 프로젝트 Gradle wrapper + export.gradle
  │         └─ 평가된 Spotless 설정·라이브러리 경로·대상 파일 내보내기
  │              └─ 새 Java worker 준비
  │
  └─ 설정이 같음
       └─ 기존 Java worker 재사용
            └─ 해당 파일의 Spotless 단계 실행
                 └─ 결과 텍스트 반환 → IntelliJ 문서에 반영
```

worker는 IntelliJ 프로젝트 안에서 **Gradle 빌드별로** 관리한다. 각 빌드를 실행하는 기준 폴더가 Gradle 루트다. 하나의 IntelliJ 프로젝트에 독립된 Gradle 빌드가 여러 개 있으면 worker도 여러 개 실행될 수 있다.

### 최초 요청 또는 설정 변경 후

1. 플러그인이 사용할 JDK와 Gradle 루트를 결정한다. Gradle 루트를 지정하지 않았다면 Java 파일에서 상위 폴더로 올라가며 wrapper를 찾는다.
2. 프로젝트 wrapper에 도구의 `export.gradle` 초기화 스크립트를 전달한다.
3. `__javaSaveFormatterExport` 작업이 활성화된 각 `spotlessJava` 태스크에서 실제 설정을 읽는다.
4. Spotless `Formatter` 객체를 나중에 복원할 수 있는 파일로 저장한다. 이를 직렬화라고 한다. 라이브러리 경로와 파일별 적용 규칙도 함께 저장한다.
5. worker가 해당 스냅샷을 읽고 포맷 요청을 처리한다.

이렇게 한 번 가져온 설정을 묶어서 저장한 것을 스냅샷이라고 부른다. 스냅샷에는 다음 파일이 들어 있다.

| 파일 | 내용 |
| --- | --- |
| `snapshot.properties` | 포매터 목록, 단계 이름, 파일별 적용 규칙, 감지할 경로 |
| `formatter-N/formatter.ser` | 직렬화한 Spotless `Formatter` 객체 |
| `formatter-N/classpath.txt` | 필요한 클래스와 라이브러리를 로드할 경로 |
| `gradle-export.log`, `worker.log` | 설정 내보내기와 worker 오류 로그 |

스냅샷은 IntelliJ의 캐시 폴더 아래 `java-save-formatter/build-*/snapshot-*`에 저장한다. 설정을 가져오는 스크립트는 Gradle 실행 시 전달하므로 프로젝트의 빌드 파일을 고칠 필요가 없다. 이때는 설정을 내보내는 작업만 요청하며, `spotlessApply`나 애플리케이션 컴파일은 요청하지 않는다. Gradle 실행에 필요한 캐시와 빌드 정보는 생성될 수 있다.

### 설정이 같은 평상시 저장

1. IntelliJ 비동기 포맷 API가 현재 문서의 경로·텍스트·변경 번호를 가져온다.
2. 관련 설정 파일의 내용으로 SHA-256 해시를 계산하고, 이전 값과 비교해 변경 여부를 확인한다.
3. 기존 worker에 파일 경로와 문서 내용을 보낸다. 이 요청에서 Gradle 태스크는 실행하지 않는다.
4. worker가 파일에 해당하는 Spotless 단계를 순서대로 실행한다.
5. 설정이나 문서가 처리 도중 달라지지 않았는지 확인한 후 결과를 IntelliJ에 반환한다.

worker가 돌려주는 것은 요청받은 문서의 포맷 결과다. 이 결과를 편집기에 반영하고 파일로 저장하는 일은 IntelliJ가 담당한다.

## 5. 실제로 사용한 기술

### IntelliJ 비동기 포맷 API

`AsyncDocumentFormattingService`로 IntelliJ의 Reformat Code와 Actions on Save에 연결했다. 포맷 작업은 비동기로 처리하고, 결과가 돌아왔을 때 문서 변경 번호와 내용을 검사한다. 사용자가 이미 수정한 문서에 오래된 결과를 적용하지 않기 위한 처리다. [IntelliJ 외부 포매터 API](https://plugins.jetbrains.com/docs/intellij/code-formatting.html#external-code-formatter)

### Java 프로세스와 표준 입출력

`ProcessBuilder`로 Java worker를 실행하고 `DataInputStream`·`DataOutputStream`으로 통신한다. 데이터는 직접 정한 이진 형식으로 주고받는다. 요청에는 파일 경로와 UTF-8 문서 내용을 넣고, 응답에는 성공 여부와 결과 텍스트를 넣는다. 시작 시에는 `JSF2` 식별값을 확인한다.

한 worker는 요청을 하나씩 순서대로 처리한다. worker가 예기치 않게 종료되거나 처리 시간을 초과하면, 다음 요청에서 새 worker를 시작한다. 프로젝트 종료나 플러그인 설정 적용 시에는 기존 worker를 종료한다.

### 직렬화, 클래스 로더, 리플렉션

Gradle에서 파일로 저장한 Spotless 객체는 worker가 읽어 복원한다. 이때 `URLClassLoader`가 프로젝트의 라이브러리를 불러온다. 필요한 Spotless 클래스와 메서드는 실행 중에 찾아 호출하는데, 이 방법을 리플렉션이라고 한다.

이 구조 덕분에 플러그인 ZIP에 Spotless나 Palantir의 특정 버전을 포함할 필요가 없다. 대신 프로젝트가 사용하는 라이브러리와 설정을 불러온다. 읽어 들이는 스냅샷도 해당 프로젝트에서 직접 내보낸 로컬 파일로 한정한다.

Spotless의 `DirtyState`로 포맷 결과를 확인하고, 반복 적용해도 결과가 계속 달라지는 문제가 없는지 검사한다. 지정한 인코딩으로 표현할 수 없는 문자가 있으면 오류로 처리한다.

IntelliJ에 결과를 전달할 때는 줄바꿈을 LF로 맞춘다. 실제 파일을 저장할 때의 인코딩과 줄바꿈은 IntelliJ 설정을 따르므로, 저장 설정이 다른 환경에서는 파일의 바이트까지 같다고 보장할 수 없다.

## 6. 무엇에 종속되어 있나

**실행 구조는 IntelliJ·JDK·Gradle·Spotless에 의존한다. Palantir는 프로젝트의 Spotless 설정에서 선택하는 포맷 엔진이다.** 아래 표에는 각 기술의 역할과 테스트에 사용한 버전을 정리했다.

| 기술 | 역할 | 검증 기준 / 공급 위치 |
| --- | --- | --- |
| IntelliJ Platform 및 Java 플러그인 API | 저장·포맷 요청, 문서 반영, 설정 화면, 프로젝트 수명 관리 | IDEA 2026.2.3 / 262.10968.63. 현재 플러그인 호환 범위는 262 계열 |
| JDK / JVM | 플러그인 코드의 Java 기반, Gradle wrapper와 상주 worker 실행 | Java 21 대상. 사용자 지정 JDK 또는 프로젝트 SDK |
| Gradle wrapper | 프로젝트의 실제 빌드 설정 평가와 라이브러리 해석 | Gradle 8.14.3 검증. 설정 가져오기에는 대상 프로젝트 wrapper 사용 |
| Spotless Gradle 플러그인 | `spotless { java { ... } }`를 평가하고 대상·제외·단계 설정 제공 | 8.10.2 검증. 대상 프로젝트에서 공급 |
| Spotless 라이브러리 | `Formatter`, `FormatterStep`, `DirtyState`로 단계 실행과 결과 처리 | 위 검증 환경의 `spotless-lib` 4.10.2 |
| Palantir Java Format | Palantir 스타일의 Java 코드 포맷 | 2.97.0 및 2.96.0으로 변경 후 재적용 검증. 프로젝트가 해당 단계를 설정할 때 사용 |
| JavaParser | 예시 설정의 `shortenFullyQualifiedTypes()` 구현에 사용 | 위 Spotless 라이브러리 기준 3.27.1 |
| Google Java Format | 예시 설정의 기본 `removeUnusedImports()` 구현에 사용 | 위 Spotless 라이브러리와 JDK 21 기준 1.30.0 |
| Java 표준 라이브러리 | 프로세스, 입출력, 파일 탐색, SHA-256, 직렬화, 클래스 로딩 | JDK 제공 |

Spotless는 여러 변환 단계를 묶어 실행하는 역할이고, Palantir는 그중 코드 포맷을 담당하는 엔진이다. Palantir로 코드 스타일을 정리하더라도, 앞의 미사용 import 제거 단계에서는 Google Java Format의 기능이 사용될 수 있다. 전체 코드 스타일을 Google 스타일로 바꾼다는 뜻은 아니다.

JavaParser와 Google Java Format 버전·역할은 검증 환경의 Spotless 소스에서 확인했다. 설정이나 Spotless 버전이 바뀌면 달라질 수 있다. [ShortenFullyQualifiedTypesStep](https://github.com/diffplug/spotless/blob/lib/4.10.2/lib/src/main/java/com/diffplug/spotless/java/ShortenFullyQualifiedTypesStep.java), [RemoveUnusedImportsStep](https://github.com/diffplug/spotless/blob/lib/4.10.2/lib/src/main/java/com/diffplug/spotless/java/RemoveUnusedImportsStep.java), [GoogleJavaFormatStep](https://github.com/diffplug/spotless/blob/lib/4.10.2/lib/src/main/java/com/diffplug/spotless/java/GoogleJavaFormatStep.java)

각 라이브러리가 실행에 필요로 하는 다른 라이브러리도 프로젝트 설정에 따라 함께 준비된다. Node.js, ESLint, Prettier는 제작 아이디어의 배경이며 설치나 실행에 필요하지 않다. 도구 자체의 빌드 의존성은 [build.gradle](../build.gradle), 실제 포맷 라이브러리는 프로젝트에서 내보낸 `classpath.txt`와 단계 설정으로 구분해서 확인할 수 있다.

### Spotless를 쓰지 않는 프로젝트라면

현재 도구는 Gradle Spotless 설정이 있는 프로젝트를 전제로 한다. Spotless가 없으면 가져올 Java 포맷 규칙도 없으므로, 첫 포맷 요청에서 설정 가져오기가 실패한다. 이 경우 문서를 수정하지 않고 IDE에 오류와 로그 경로를 표시한다. 기본 규칙을 임의로 정하거나 Spotless를 자동으로 설치하지는 않는다.

처음 사용하는 사람에게는 **“JDK와 Gradle wrapper를 준비하고, 프로젝트의 빌드 설정에 Spotless와 Java 포맷 규칙을 추가한 뒤 Java Save Formatter를 설치하세요”**라고 안내하면 된다. IntelliJ용 Spotless·Palantir 플러그인은 필요하지 않다. Palantir 라이브러리는 빌드 설정에 버전을 지정하면 Gradle이 내려받는다.

Spotless가 있어도 Palantir 같은 코드 포맷 단계를 설정하지 않았다면 도구가 대신 선택해 주지 않는다. 프로젝트에 등록된 단계만 실행한다. 실제로 복사해 사용할 수 있는 Gradle 설정과 준비 항목은 [README의 처음 사용하는 프로젝트의 준비 사항](../README.md#처음-사용하는-프로젝트의-준비-사항)에 정리했다.

## 7. build.gradle 변경은 어떻게 반영하나

빌드 파일을 바꾸고 저장하면 **다음 Java 저장이나 포맷 요청에서 변경을 확인한다.** 빌드 파일을 저장한 순간에는 Gradle을 실행하지 않는다.

`*.gradle`, `*.gradle.kts`, `gradle.properties`, `gradle/`, `buildSrc/`, `build-logic/`, 버전 카탈로그, `.editorconfig`, `.gitattributes`, Gradle 사용자 초기화 파일 등을 내용 기반으로 확인한다. 외부 프로젝트·포함 빌드 경로와 사용자가 추가한 감지 경로도 반영한다. 상세한 범위와 제외 디렉터리는 [README의 변경 감지 범위](../README.md#변경-감지-범위)에 정리되어 있다.

설정 내용이 달라지면 기존 worker를 종료하고 Gradle로 설정을 다시 가져온다. `palantirJavaFormat('2.97.0')`을 다른 버전으로 바꾸거나, 단계를 추가·삭제·재배치한 경우도 같은 경로를 거친다. 가져오기에 실패하면 이전 설정으로 조용히 포맷하지 않고 오류를 알린다.

새 Java 파일은 이전에 가져온 대상 목록에 없으므로, 설정을 다시 가져와 포맷 대상인지 확인한다. 제외할 파일로 확인되면 그 결과를 기억해 두었다가 설정이 바뀔 때 다시 확인한다.

임의의 외부 파일이나 환경 변수, 원격 설정 등 모든 Gradle 입력을 자동 추적하지는 않는다. 외부 파일은 추가 감지 경로에 등록하고, 그 외에는 수동 다시 불러오기를 사용한다. Gradle은 디스크의 설정을 읽으므로 빌드 파일의 편집 내용도 먼저 저장해야 한다.

## 8. 빨라진 이유와 측정 범위

첫 요청이 끝나면 worker와 포매터는 다음 요청을 기다린다. 설정이 같고 worker가 정상적으로 실행 중이라면, 이후 저장에서는 다음 준비 과정을 건너뛴다.

- Gradle에 작업을 요청하고 실행 준비하기
- 포맷을 맡을 Java 프로세스 시작하기
- 포매터 객체와 라이브러리 불러오기

소스 파싱과 각 포맷 단계의 계산은 여전히 필요하다. 설정 변경 감지를 위한 파일 탐색·해시 계산과 프로세스 통신 비용도 남는다. 따라서 파일이 크거나 감지 경로가 많으면 처리 시간이 늘 수 있다.

0.2.0은 별도의 Gradle 테스트 프로젝트에 Java 파일 복사본을 넣어 측정했다. 아래 시간에는 설정 변경 확인과 worker의 포맷 처리가 포함된다. IntelliJ 화면에서 저장 버튼을 누른 뒤 완료될 때까지 걸리는 시간을 측정한 값은 아니다.

| 항목 | 측정 결과 |
| --- | ---: |
| Gradle 설정 가져오기를 포함한 첫 요청 | 1,390 ms |
| 상주 상태: 57줄 파일 중앙값 | 11.3 ms |
| 상주 상태: 246줄 파일 중앙값 | 28.5 ms |
| 상주 상태: 1,509줄 파일 중앙값 | 146.4 ms |
| 전체 42회 요청의 Gradle 설정 가져오기 횟수 | 1회 |

42번의 포맷 결과는 모두 기준으로 삼은 Gradle 결과와 같았다. 처리 시간, 특히 첫 요청에 걸리는 시간은 프로젝트 설정과 라이브러리 캐시 상태에 따라 달라질 수 있다. 측정·검증 기록은 로컬 빌드 산출물 `build/verification-0.2.0.json`과 `build/dynamic-benchmark.txt`에 있다.

worker를 계속 실행해 두는 동안에는 메모리도 사용한다. 현재 설정인 `-Xmx256m`은 Java 객체를 저장하는 힙 메모리의 최대 크기를 뜻한다. 항상 256 MiB를 사용하는 것은 아니며, JVM이 쓰는 다른 메모리까지 포함한 전체 사용량은 이보다 클 수 있다. 별도로 검토한 메모리 절약 설정은 현재 구현에 적용하지 않았다.

## 9. 지원 범위와 제한

현재는 프로젝트의 Java 소스·테스트 소스에 대한 전체 파일 포맷을 지원한다. Kotlin·JSON 등 다른 Spotless 형식이나 선택 영역 포맷, 자동 완성·진단 같은 언어 서버 기능은 구현하지 않았다.

Spotless의 `stepsInternalRoundtrip` 등 내부 API와 포매터 직렬화에 의존하므로 모든 Spotless 버전에 자동 호환되는 것은 아니다. 사용자가 만든 단계가 Gradle 객체를 참조하는 등 파일로 저장해 다른 프로세스에서 복원하기 어려운 설정은 가져오기에 실패할 수 있다. Git 기준점 이후의 변경만 처리하는 `ratchetFrom`도 지원하지 않는다. 이후 버전에서 내부 구조가 바뀌면 설정 내보내기 코드를 수정해야 할 수 있다.

이 도구는 편집 중인 파일을 빠르게 정리하는 역할을 맡는다. 프로젝트 전체가 규칙을 따르는지 확인할 때는 기존처럼 CI나 빌드에서 `spotlessCheck`를 실행한다.

## 10. 코드를 읽는 순서

| 파일 | 확인할 내용 |
| --- | --- |
| [JavaFormattingService.java](../src/idea/java/dev/local/javaformatter/idea/JavaFormattingService.java) | IntelliJ 포맷 요청과 문서 결과 반영 |
| [ProjectEngine.java](../src/idea/java/dev/local/javaformatter/idea/ProjectEngine.java) | Gradle 루트별 엔진, JDK·캐시 경로, 종료 처리 |
| [GradleFormatter.java](../src/main/java/dev/local/javaformatter/GradleFormatter.java) | 설정 해시, Gradle 실행, 재설정 조건 |
| [export.gradle](../src/main/resources/export.gradle) | 실제 Spotless 설정·라이브러리·대상 파일 내보내기 |
| [WorkerClient.java](../src/main/java/dev/local/javaformatter/WorkerClient.java) | 상주 프로세스 실행, 요청·응답, 시간제한 |
| [Worker.java](../src/main/java/dev/local/javaformatter/Worker.java) | 설정 복원, 라이브러리 로드, 단계 실행 |
| [FormatterSelfTest.java](../src/test/java/dev/local/javaformatter/FormatterSelfTest.java) | 실제 Gradle 결과 일치와 설정 변경 검증 |
| [ProcessSelfTest.java](../src/test/java/dev/local/javaformatter/ProcessSelfTest.java) | 재사용, 오류 복구, 동시 요청, 프로세스 종료 검증 |
