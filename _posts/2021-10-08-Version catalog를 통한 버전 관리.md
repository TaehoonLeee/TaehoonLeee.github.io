---
categories: [Android, Version Sharing]
tags: [gradle, version catalog]
---

안녕하세요. 오늘은 version catalog를 통해 라이브러리 버전 관리 하는 법을 알아보겠습니다.

흔히들 사용하시는 버전 관리 방법은 아래와 같습니다.
> 1. ext를 통한 버전 관리
2. buildSrc를 통한 버전 관리
3. composite build를 통한 버전 관리

하지만, 이번에 gradle이 7.0으로 업그레이드 되면서 추가된 version catalog를 통해 더욱 간편하게 버전 관리를 할 수 있게 됐습니다.

먼저, 프로젝트 최상단 폴더에 있는 **settings.gradle.kts**에 다음과 같은 코드를 추가해줍니다.

> Gradle 버전이 7.4가 release 됨에 따라 version catalog 기능이 stable 해졌습니다.
7.4 이상의 버전을 사용하시는 분들은 아래와 같은 코드를 추가할 필요가 없습니다.
{: .prompt-info }

```kotlin
seetings.gradle.kts

enableFeaturePreview("VERSION_CATALOGS")

dependencyResolutionManagement {
    versionCatalogs {
        create("deps") {
            from(files("deps.version.toml"))
        }
    }
}
```
{: .nolineno }

여기서, ```create("deps")``` 에서 파라미터를 어떤 이름을 넣어주시든 상관 없습니다. 맘에 드시는 걸로 넣어주시면 됩니다.
ex) deps, libs, ...

그리고, 버전을 명시해 줄 파일을 만듭니다. 여기서 만든 파일은 ```files("deps.version.toml")```에서 쓰입니다.

> [파일을 루트 디렉토리의 gradle 폴더에 만들어주면 settings.gradle에 파일을 명시해줄 필요가 없습니다.](https://docs.gradle.org/current/userguide/platforms.html#sub:conventional-dependencies-toml)
{: .prompt-info }

```java
deps.version.toml

[versions]

kotlin = "1.5.31"
androidGradle = "7.0.2"
compose = "1.1.0-alpha03"

[libraries]

kotlin-plugin = { module = "org.jetbrains.kotlin:kotlin-gradle-plugin", versions.ref = "kotlin" }

android-gradle = { group = "com.android.tools.build", name = "gradle", version.ref = "androidGradle" }

androidx-compose-foundation = { module = "androidx.compose.foundation:foundation", version.ref = "compose" }
androidx-compose-ui = { module = "androidx.compose.ui:ui", version.ref = "compose" }
androidx-compose-material = { module = "androidx.compose.material:material", version.ref = "compose" }
androidx-compose-uiTooling = { module = "androidx.compose.ui:ui-tooling", version.ref = "compose" }

[bundles]

androidx-compose = [ "androidx-compose-foundation", "androidx-compose-ui", "androidx-compose-material", "androidx-compose-uiTooling" ]
```
{: .nolineno }

저는 위와 같이 구성해봤습니다. 

보시는 바와 같이 파일은 세 가지 영역으로 나뉩니다. [versions], [libraries], [bundles]

versions와 libraries는 보시면 딱 감이 오실텐데 bundle은 여러 라이브러리를 묶어서 관리를 해줄 수 있는 거라 생각하시면 됩니다.

그럼 이제, 각 모듈의 **build.gradle.kts**로 가서 각 라이브러리를 추가해주면 끝입니다!! buildSrc보다 훨씬 간편하죠?

먼저, 프로젝트 최상단에 있는 build.gradle.kts로 가서 플러그인을 설정해줍니다.

> catalog를 사용할 때 7.4 이상의 버전을 사용하시는 분들은 catalog를 직접 설정해주시면 에러가 발생하게 됩니다. 따라서 아래와 같이 다르게 설정해주시면 되겠습니다.
{: .prompt-warning }

```kotlin
build.gradle.kts

// legacy plugins
// gradle version >= 7.4

dependencies {
    classpath(deps.kotlin.plugin)
    classpath(deps.android.gradle)
}


// gradle version < 7.4

dependencies {
    val deps = project.extensions.getByType<VersionCatalogsExtension>().named("deps") as org.gradle.accessors.dm.LibrariesForDeps
    
    classpath(deps.kotlin.plugin)
    classpath(deps.android.gradle)
}
```
{: .nolineno }


alias를 통해 플러그인을 설정하는 방법도 있습니다. 버전에 따라 위처럼 deps 변수를 직접 할당해주시면 됩니다.
```kotlin
// new plugins
@Suppress("DSL_SCOPE_VIOLATION")
plugins {
    alias(deps.plugins.android.application) apply false
    alias(deps.plugins.kotlin) apply false
}
```
{: .nolineno }

다음으로, 각 모듈에 있는 build.gradle.kts로 가서 라이브러리를 설정해줍니다.

```kotlin
module/build.gradle.kts

dependencies {

    // bundle을 통해 한꺼번에 관리
    implementation(deps.bundles.androidx.compose)
    
    // 각 라이브러리별로 관리
    implementation(deps.androidx.compose.foundation)
    implementation(deps.androidx.compose.ui)
    implementation(deps.androidx.compose.material)
    implementation(deps.androidx.compose.uiTooling)
}
```
{: .nolineno }

프로젝트 최상단에 있는 build.gradle.kts에서 플러그인 설정할 때와는 달리 deps 변수를 설정해주지 않아도 자동으로 인식합니다.

라이브러리 버전이 필요할 때는 다음과 같이 하면 됩니다.
```kotlin
deps.versions.compose.get()

ex)
android {
    compileSdk = 31
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = deps.versions.compose.get()
    }
}
```
{: .nolineno }

끄읕!