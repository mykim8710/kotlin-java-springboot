## 1. Gradle Kotlin DSL

- Gradle은 빌드 스크립트를 작성할때 기본적으로 Groovy 언어를 사용해 작성한다
- 익숙하지 않은 Groovy 대신 Kotlin 기반으로 빌드 스크립트를 작성할 수 있다. 이를 `Gradle Kotlin DSL` 이라고 한다
- Kotlin DSL로 Gradle 빌드 스크립트를 작성하면 IntelliJ, Android Studio와 같은 Jetbrains IDEA에서 코틀린 관련 지원을 받을 수 있다(자동완성, 컴파일 오류 체크 등)
- Kotlin DSL로 작성한 빌드 스크립트는 `.kts` 확장자를 가진다 KTS는 `Kotlin Script`의 약자이다

- groovy로 작성된 build.gradle
```
plugins {
    id 'org.springframework.boot' version '2.7.0'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'org.jetbrains.kotlin.jvm' version "1.6.21"
}

group = 'com.fastcampus'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}

compileKotlin {
    kotlinOptions {
        jvmTarget = "1.8"
    }
}
```

- kotlin으로 작성된 build.gradle.kts
```
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id ("org.springframework.boot") version "2.7.0"
    id ("io.spring.dependency-management") version "1.0.11.RELEASE"
    id ("org.jetbrains.kotlin.jvm") version "1.6.21"
}

group = "com.mykim"
version = "0.0.1-SNAPSHOT"
java.sourceCompatibility = JavaVersion.VERSION_17

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.h2database:h2")
    testImplementation ("org.springframework.boot:spring-boot-starter-test")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf("-Xjsr305=strict")
        jvmTarget = "17"
    }
}
```
- Kotlin DSL로 작성된 빌드 스크립트는 순수 Groovy로 작성한 빌드 스크립트보단 조금 느린 건 사실이지만 점점 개선되고 있음



## 2. Spring 플러그인
- 코틀린의 클래스는 기본적으로 `final` 즉, 상속이 불가능한 클래스이다
- 상속을 열어뒀을때 발생하는 부작용으로 인해 코틀린은 상속이 꼭 필요한 경우에만 적용하도록 `open` 키워드를 사용해 상속을 허용하도록 지원한다
- 문제는 스프링은 기본적으로 `CGLIB 프록시` 를 사용해 애노테이션이 붙은 클래스에 대한 프록시 객체를 생성하는데 CGLIB 프록시는 대상 객체를 상속하여 프록시 객체를 생성한다
- 코틀린의 클래스는 기본적으로 final 이기 때문에 상속이 불가능해 프록시 객체를 생성할 수 없다

- 아래 코드는 컴파일 에러가 발생하므로 open 키워드를 붙여야 한다
```
package com.mykim.kotlinjavasrping

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
// class KotlinJavaSpringApplication // 컴파일 에러 
open class KotlinJavaSpringApplication

fun main(args: Array<String>) {
    runApplication<KotlinJavaSpringApplication>()
}
```

- 매번 open 키워드를 붙이는 건 불편하므로 코틀린은  `All-open` 컴파일러 플러그인을 제공한다
- build.gradle.kts에 all-open 플러그인 추가
```
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id ("org.springframework.boot") version "2.7.0"
    id ("io.spring.dependency-management") version "1.0.11.RELEASE"
    id ("org.jetbrains.kotlin.jvm") version "1.6.21"
    id ("org.jetbrains.kotlin.plugin.allopen") version "1.6.21" // 추가
}

// 추가
allOpen {
    annotations("org.springframework.boot.autoconfigure.SpringBootApplication")
}
```

- `@Transactional` 애노테이션을 추가하는 경우는 ?
- open을 붙이지 않아서 마찬가지로 컴파일 에러가 발생한다
```
@Transactional
class Service
```

- 마찬가지로 allOpen 플러그인에 추가해야한다
```
allOpen {
    annotations(
        "org.springframework.boot.autoconfigure.SpringBootApplication",
        "org.springframework.transaction.annotation.Transactional"
    )
}
```

- 이처럼 매번 문제가 생길때마다 allopen에 추가하기 어려우므로 allopen 플러그인을 래핑한 kotlin-spring 플러그인을 사용하면 매우 간편해진다
```
plugins {
    id("org.springframework.boot") version "2.7.0"
    id("io.spring.dependency-management") version "1.0.11.RELEASE"
    id("org.jetbrains.kotlin.jvm") version "1.6.21"
    kotlin("plugin.spring") version "1.6.21"
}
```
- kotlin-spring 플러그인은 스프링에서 CGLIB 프록시를 사용하는 모든 애노테이션에 대해 자동으로 open 처리를 해준다
    - @Component
    - @Transactional
    - @Configuration, @Controller, @Service, @Repository 등은 내부에 @Component 애노테이션을 메타-애노테이션으로 가지고 있다


## 3. JPA 플러그인
- JPA에서 엔티티 클래스를 생성하려면 매개 변수가 없는 기본 생성자가 필요하다
- 실제로 예시와 같이 엔티티를 작성하면 컴파일 에러가 발생한다
```
@Entity
@Table
class Person(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long?,

    @Column
    var name: String,

    @Column
    var age: Int,
)
```

- 코틀린은 매개 변수가 없는 기본 생성자를 자동으로 만들어주는 no-arg 컴파일러 플러그인을 제공한다
```
plugins {
    id("org.springframework.boot") version "2.7.0"
    id("io.spring.dependency-management") version "1.0.11.RELEASE"
    id("org.jetbrains.kotlin.jvm") version "1.6.21"
    kotlin("plugin.spring") version "1.6.21"
    kotlin("plugin.noarg") version "1.6.21"
}

noArg {
    annotation("javax.persistence.Entity")
}
```

- JPA를 쓸 경우 Spring 플러그인과 마찬가지로 `kotlin-jpa` 플러그인을 제공한다
- JPA 플러그인을 쓰면 @Entity, @Embeddable, @MappedSuperclass에 대한 기본 생성자를 자동으로 생성해준다
```
plugins {
    id("org.springframework.boot") version "2.7.0"
    id("io.spring.dependency-management") version "1.0.11.RELEASE"
    id("org.jetbrains.kotlin.jvm") version "1.6.21"
    kotlin("plugin.spring") version "1.6.21"
    kotlin("plugin.jpa") version "1.6.21"
}
```

