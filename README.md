# Java Save Formatter 0.2.0

프로젝트의 **실제 Gradle Spotless Java 설정**을 가져와 상주 Java 프로세스로 포맷합니다.
`shortenFullyQualifiedTypes()` 같은 단계와 Palantir 버전을 도구에 고정하지 않습니다.
Gradle이 평가한 단계 순서, 옵션, 라이브러리, 대상 파일과 `targetExclude`를 사용합니다.

## 동작

1. 처음 Java 파일을 포맷할 때 프로젝트의 Gradle wrapper로 설정을 평가합니다.
2. 각 프로젝트의 `spotlessJava` 포매터와 라이브러리 경로를 IDE 캐시 폴더로 내보냅니다.
3. 저장된 설정이 같으면 Gradle을 다시 실행하지 않고 상주 엔진을 재사용합니다.
4. 설정 변경을 감지하면 이전 엔진을 종료하고 새 설정을 가져와 다시 시작합니다.
5. 설정을 가져오지 못하면 오류와 로그 경로를 표시합니다. 이전 규칙으로 조용히 포맷하지 않습니다.

설정 확인은 **Java 포맷 요청 시** 수행합니다. 빌드 파일을 저장하자마자 백그라운드에서 Gradle을
실행하는 방식은 아닙니다. `build.gradle` 변경을 저장한 뒤 다음 Java 저장/포맷에 새 규칙이 적용됩니다.
아직 저장하지 않은 IDE 편집 내용은 Gradle에서 읽을 수 없습니다.
첫 요청과 설정 변경 직후에는 Gradle 평가 비용이 있고, 평상시에는 파일 변경 확인과 엔진 처리만 합니다.

## 변경 감지 범위

파일 내용의 SHA-256을 비교하므로 같은 크기와 수정 시각으로 바뀌어도 감지합니다.
다음 경로의 추가·수정·삭제를 확인합니다.

- Gradle 루트와 하위 프로젝트의 `*.gradle`, `*.gradle.kts`, `gradle.properties`
- `gradle/` 파일: 버전 카탈로그, wrapper, 검증 메타데이터 등
- `buildSrc/`, `build-logic/` 파일: 빌드 로직 소스와 리소스
- `*.toml`, `.editorconfig`, `.gitattributes`, wrapper 실행 스크립트
- Gradle이 알려준 외부 프로젝트/포함 빌드의 파일
- Gradle 사용자 홈의 `gradle.properties`, `init.gradle(.kts)`, `init.d/`
- 설정 화면의 **추가 변경 감지 경로**에 지정한 파일/폴더

`.git`, `.idea`, `.gradle`, `build`, `out`, `node_modules`, `.venv` 디렉터리는 제외합니다.
일반 Java 소스 수정은 설정 재평가를 유발하지 않습니다. 다만 아직 설정 내보내기에 포함되지 않은
새 Java 파일은 대상 여부를 확인하려고 한 번 재평가합니다. 제외 파일은 이후 반복 재평가하지 않습니다.

외부 `apply from`, 라이선스 헤더 파일, 외부 XML 등 기본 범위 밖의 입력은 추가 변경 감지 경로에
등록하세요. 한 줄에 하나씩 적고, 상대 경로는 Gradle 루트 기준입니다.
환경 변수, 원격 설정, 임의의 파일 읽기 등 모든 Gradle 입력을 자동 추적하지는 않습니다.
파일로 감지할 수 없는 변경은 **다음 포맷 시 Spotless 설정 다시 불러오기** 버튼으로 갱신합니다.
별도 CLI `-P` 인자나 IDE Gradle 실행 구성의 추가 인자를 자동으로 가져오지는 않습니다.

## 설치·업데이트

빌드/클래스 연결 확인 대상: **IntelliJ IDEA 2026.2.3, IU-262.10968.63**.
지원 범위는 262 계열로 제한했습니다. 실제 IDE 설치와 저장 동작 확인은 사용자가 수행합니다.

1. **Settings → Plugins → 톱니바퀴 → Install Plugin from Disk…**에서
   `build/distributions/java-save-formatter-0.2.0.zip`을 선택합니다. 0.1.0이 설치되어 있으면 같은 ID로 업데이트합니다.
   IDE가 요구하면 재시작합니다.
2. **Settings → Tools → Java Save Formatter**에서 프로젝트 사용을 켭니다.
3. JDK 21 홈 경로를 지정합니다. 이 Mac에서는
   `/Library/Java/JavaVirtualMachines/amazon-corretto-21.jdk/Contents/Home`입니다.
   비우면 프로젝트 SDK를 사용합니다. 선택한 JDK로 wrapper와 상주 엔진을 모두 실행합니다.
4. **Gradle 루트**는 보통 비워둡니다. 포맷 대상 Java 파일에서 가장 가까운 wrapper를 탐색합니다.
   하위 프로젝트에 별도 wrapper가 있지만 상위 빌드를 사용해야 한다면 상위 루트를 명시하세요.
5. 기존 **Run spotless**와 프로젝트의 **Enable palantir-java-format**을 끕니다.
6. **Tools → Actions on Save → Reformat code**를 켜고 **Java / Whole file**을 선택합니다.
   Java의 Optimize imports, Rearrange code, Code cleanup은 꺼서 같은 저장에 규칙이 겹치지 않게 합니다.
7. Java 파일을 저장합니다. 첫 실행에는 `Spotless 설정 불러오는 중` 알림이 표시됩니다.

설치 스크립트가 IDE 설정이나 업무 저장소를 수정하지 않습니다. 플러그인 설정은 사용자가 Apply할 때
IDE workspace 설정으로 저장됩니다. 설정 내보내기는 프로젝트 wrapper를 실행하므로 일반 Gradle과
동일하게 캐시와 프로젝트 `.gradle` 등에 빌드 메타데이터를 만들 수 있습니다.
내보내기 작업은 `spotlessApply`나 컴파일을 요청하지 않고, 포맷 대상 소스를 덮어쓰지 않습니다.

## 직접 확인

현재 프로젝트 설정이 다음과 같다고 가정합니다.

```groovy
spotless {
    java {
        shortenFullyQualifiedTypes()
        removeUnusedImports()
        palantirJavaFormat('2.97.0')
    }
}
```

Java 소스 폴더에 임시 클래스를 만들고 저장합니다.

```java
import java.util.Set;
class FormatterSmoke{java.util.List<String> values;}
```

`Set` import가 제거되고 `List` import가 추가되며 Palantir 스타일로 정리되어야 합니다.
이어서 `shortenFullyQualifiedTypes()`를 제거하고 **build.gradle을 저장**한 뒤,
Java 파일을 위 입력으로 되돌리고 다시 저장합니다. 새 설정을 가져온 후에는
`java.util.List`가 그대로 남아야 합니다. 테스트가 끝나면 프로젝트 설정을 원래대로 돌립니다.

버전 변경, 단계 추가·삭제와 순서 변경 모두 Gradle 평가 결과를 따릅니다.
이름 충돌 해결이나 static import 생성 등 원래 설정이 하지 않는 변환을 추가하지 않습니다.

## 구현 범위와 제한

- Spotless **Java** 형식만 대상으로 합니다. Kotlin, JSON 등 다른 형식은 아직 처리하지 않습니다.
- 다중 프로젝트는 각 `spotlessJava`의 대상 파일에 맞는 규칙을 선택합니다.
- 기본 Java 단계와 직렬화 가능한 포매터 설정을 사용합니다. Gradle 프로젝트 객체를 캡처한 사용자
  정의 단계처럼 상주 프로세스로 내보낼 수 없는 설정은 실패로 알립니다.
- `ratchetFrom`은 지원하지 않습니다. 해당 설정을 감지하면 내보내기를 중단합니다.
- 검증된 기준은 Gradle 8.14.3, Spotless 플러그인 8.10.2, Palantir 2.97.0/2.96.0입니다.
  Spotless의 내부 내보내기 API가 바뀌면 어댑터 수정이 필요할 수 있습니다.
- 프로젝트 Java 소스·테스트 소스에 적용합니다. 선택 영역 포맷과 입력 중 들여쓰기는 이 엔진이 처리하지 않습니다.
- 문서가 바뀌거나 작업이 취소되면 변경 번호와 내용을 검사해 이전 결과를 버립니다.
  최종 반영은 IntelliJ 비동기 포맷 API의 변경 확인도 거칩니다.
- 문법 오류와 인코딩 불가능한 문자는 원문을 유지하고 오류를 알립니다.
- 설정 내보내기 제한 120초, 파일 포맷 제한 15초, UTF-8 요청/응답 상한 16 MiB, JVM 힙 상한 256 MiB입니다.
- 포매터 단계 실행에 실제 파일 경로를 전달합니다. 기본 엔진은 문서 내용을 반환하며 소스 파일을 직접 쓰지 않습니다.
  사용자가 설정한 사용자 정의 단계 자체의 외부 동작까지 제한하지는 않습니다.
- 포매터의 인코딩과 줄바꿈 정책을 사용한 뒤 IDE에는 LF 텍스트를 반환합니다. 실제 디스크 저장은 IDE가 담당합니다.
- 프로세스가 죽거나 시간 초과가 나면 다음 요청에 다시 시작합니다. 프로젝트 종료/설정 변경/사용 중지 시 종료합니다.
- HTTP 포트나 LSP 서버는 사용하지 않습니다.

설정과 로그는 IntelliJ system/cache 경로의 `java-save-formatter/build-*/snapshot-*`에 있습니다.
`gradle-export.log`, `worker.log`, `snapshot.properties`에서 실패 원인과 실제 단계/대상을 확인할 수 있습니다.
캐시에는 이전 내보내기 로그도 남습니다. 필요하면 IDE를 종료한 뒤 이 도구의 캐시 폴더만 삭제할 수 있습니다.

## 빌드와 검증

```sh
JAVA_HOME=/Library/Java/JavaVirtualMachines/amazon-corretto-21.jdk/Contents/Home \
  ./gradlew selfTest pluginZip
```

다른 IDE SDK 위치는 `-PideaHome=/path/to/idea/Contents`로 지정합니다.
플러그인 컴파일은 설치된 IDE의 `javac`를 사용하며, SDK/JBR을 ZIP에 포함하지 않습니다.
프로덕션 엔진에는 Spotless/Palantir 버전 의존성을 묶지 않습니다. 해당 프로젝트에서 가져옵니다.

`selfTest`는 별도 임시 Gradle 프로젝트를 만들어 다음을 검증합니다.

- 실제 Gradle Spotless 출력과 일치, 설정이 같을 때 Gradle 재실행 없이 엔진 재사용
- 단계 제거, `gradle.properties`, 적용 스크립트, `settings.gradle`, 추가 감지 파일 변경 반영
- Palantir 2.97.0 → 2.96.0 버전 변경 후 실제 Gradle 결과 일치
- `targetExclude`, 새 파일, 다중 프로젝트의 서로 다른 규칙
- 잘못된 Gradle 설정에서 이전 규칙 적용 차단, 설정 복구 후 재개
- 문자 인코딩 보존, 문법 오류 복구, 동시 요청, 강제 종료 후 재시작, 시간제한, 종료 처리

설치된 실제 IDE의 저장/UI 동작은 이 테스트에 포함하지 않습니다.
선택적 벤치마크는 기존 Gradle 기준 자료와 원본 복사본으로 실행합니다.

```sh
python3 scripts/verify-benchmark.py /path/to/spotless-benchmark /path/to/jdk21
```

이 벤치마크는 이전 측정의 세 단계(Spotless 8.10.2 / Palantir 2.97.0)를 재현하는 독립 fixture입니다.
프로덕션 설정을 고정하는 코드가 아닙니다. 변경 감지까지 포함한 전체 요청 경로를 측정하며 업무 소스를 수정하지 않습니다.

## 사용 중지

Settings의 사용 체크를 해제하고 Apply합니다. 기존 방식으로 돌아가려면 `Run spotless`를 다시 켜고
이 도구를 위해 켰던 `Reformat code`를 끕니다. 플러그인은 Plugins 화면에서 수동 제거합니다.

## 참고

- [Gradle 초기화 스크립트](https://docs.gradle.org/current/userguide/init_scripts.html)
- [Spotless IDE hook과 대상 파일 처리](https://github.com/diffplug/spotless/blob/main/plugin-gradle/IDE_HOOK.md)
- [IntelliJ 외부 포매터 API](https://plugins.jetbrains.com/docs/intellij/code-formatting.html#external-code-formatter)
- [IntelliJ 비동기 결과 처리 계약](https://github.com/JetBrains/intellij-community/blob/master/platform/code-style-api/src/com/intellij/formatting/service/AsyncFormattingRequest.java)

Gradle wrapper는 Gradle의 Apache 2.0 라이선스를 따릅니다. 포매터 라이브러리는 프로젝트 Gradle 캐시의 원본을 사용합니다.
