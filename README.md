# 🐳 Docker Image Optimization

> Docker 이미지 최적화 기법을 단계별로 적용하고, 각 기법이 이미지 크기와 보안에 미치는 영향을 실험한 프로젝트입니다.

<br>

## 👩🏻‍💻 팀원 소개

| <img src="https://avatars.githubusercontent.com/u/178015712?v=4" width="150" height="150"/> | <img src="https://avatars.githubusercontent.com/u/107402806?v=4" width="150" height="150"/> |
| :-----------------------------------------------------------------------------------------: | :-----------------------------------------------------------------------------------------: |
| 권순재<br/>[@Soooonnn](https://github.com/Soooonnn) | 김소연<br/>[@thdus](https://github.com/thdus) |

<br />

## 📌 목차

1. [실험 개요](#실험-개요)
2. [실험 환경](#실험-환경)
3. [최적화 기법](#최적화-기법)
   - [Step 1. 최적화 없음 (Baseline)](#step-1-최적화-없음-baseline)
   - [Step 2. 경량 베이스 이미지 (Alpine)](#step-2-경량-베이스-이미지-alpine)
   - [Step 3. Multi-Stage Build](#step-3-multi-stage-build)
   - [Step 4. Multi-Stage + Alpine (최종)](#step-4-multi-stage--alpine-최종)
4. [실험 결과](#실험-결과)
5. [결론](#결론)

<br>

## 🌸 실험 개요

도커 이미지를 아무 최적화 없이 빌드하면 불필요한 빌드 도구, 소스코드, 캐시 등이 그대로 포함되어 이미지 크기가 비대해지고 보안 취약점에 노출될 위험이 높아집니다.

본 프로젝트에서는 아래 두 가지 최적화 기법을 단계적으로 적용하고 그 효과를 측정했습니다.

- **경량 베이스 이미지 선택** — 시작점 자체를 작게
- **Multi-Stage Build** — 빌드 도구를 최종 이미지에서 완전히 제거

<br>

## 🌳실험 환경

| 항목 | 내용 |
|------|------|
| OS | Ubuntu 24.04 |
| Java | 17 |
| Framework | Spring Boot 4.0.4 |
| Build Tool | Gradle 8.14 |
| Docker | Docker Engine |
| 취약점 스캐너 | Trivy |

<br>

## ✨ 최적화 기법

### Step 1. 최적화 없음 (Baseline)

빌드 도구인 `gradle:8.14-jdk17` 이미지 위에서 빌드하고, 그 이미지를 그대로 런타임으로 사용합니다.
빌드에 사용된 JDK, Gradle, 각종 라이브러리가 최종 이미지에 전부 포함됩니다.

```dockerfile
FROM gradle:8.14-jdk17

WORKDIR /app

COPY . .

RUN gradle bootJar --no-daemon -x test

ENTRYPOINT ["java", "-jar", "build/libs/*.jar"]
```

<br>

### Step 2. 경량 베이스 이미지

### 1) Alpine

베이스 이미지를 `eclipse-temurin:17-jre-alpine`으로 교체합니다.
Alpine Linux는 약 5MB의 초경량 배포판으로, 불필요한 패키지가 없어 이미지 크기와 보안 취약점이 동시에 감소합니다.

> 빌드 방식은 동일하고 런타임 이미지만 교체한 케이스입니다.

```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY *.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Alpine 적용 효과**
- 불필요한 OS 패키지 제거
- 패키지가 줄어든 만큼 CVE 취약점 감소
- 이미지 크기 감소

### 2) Slim

베이스 이미지를 `eclipse-temurin:17-jre-jammy`로 교체합니다.

Slim 계열 이미지는 Ubuntu 기반으로, 기본 이미지보다 불필요한 패키지를 일부 제거하여 **안정성과 경량화를 동시에 확보**할 수 있습니다.

> 빌드 방식은 동일하고 런타임 이미지만 교체한 케이스입니다.
> 

```dockerfile
FROM eclipse-temurin:17-jre-jammy

WORKDIR /app

COPY *.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Slim 적용 효과**

- 기본 JDK 이미지 대비 불필요한 패키지 감소
- Alpine 대비 높은 호환성과 안정성 확보
- 이미지 크기 감소 (단, Alpine/Distroless 대비 제한적)
- OS 패키지가 남아 있어 취약점 감소 효과는 제한적

  베이스 이미지를 `gcr.io/distroless/java17`로 교체합니다.
  
### 3) Distroless
Distroless 이미지는 shell, package manager 등 OS 구성 요소를 제거한 **최소 실행 환경 이미지**로, 보안 측면에서 가장 강력한 선택입니다.

> 빌드 단계에서 생성된 jar만 포함하여 실행 환경을 최소화한 케이스입니다.
> 

```dockerfile
FROM gcr.io/distroless/java17

WORKDIR /app

COPY *.jar app.jar

EXPOSE8080

ENTRYPOINT ["app.jar"]
```

**Distroless 적용 효과**

- OS 레이어 제거 → 공격 표면 최소화
- CVE 취약점 수 대폭 감소
- 이미지 크기 최소화
- shell 미포함 → 디버깅 어려움 (운영 편의성 감소)
<br>

### Step 3. Multi-Stage Build

Dockerfile을 **빌드 스테이지**와 **런타임 스테이지**로 분리합니다.
빌드 스테이지에서 JAR를 생성하고, 런타임 스테이지에는 JAR 파일만 복사합니다.
이렇게 하면 컴파일러, Gradle, 소스코드 등 빌드에만 필요한 모든 것이 최종 이미지에서 제거됩니다.

```dockerfile
# ================================
# 빌드 스테이지
# ================================
FROM gradle:8.14-jdk17 AS builder

WORKDIR /app

COPY build.gradle settings.gradle ./
COPY src ./src

RUN gradle bootJar --no-daemon -x test

# ================================
# 런타임 스테이지
# ================================
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Multi-Stage Build 효과**
- 빌드 도구(JDK, Gradle) 최종 이미지에서 완전 제거
- 소스코드가 이미지에 포함되지 않아 코드 노출 방지
- 공격 표면(Attack Surface) 감소 → 보안 취약점 감소

<br>

### Step 4. Multi-Stage + Alpine (최종)

Multi-Stage Build와 Alpine 이미지를 결합한 최종 최적화입니다.
런타임 스테이지에 `eclipse-temurin:17-jre-alpine`을 사용하여 두 기법의 효과를 동시에 적용합니다.

```dockerfile
# ================================
# 빌드 스테이지
# ================================
FROM gradle:8.14-jdk17 AS builder

WORKDIR /app

COPY build.gradle settings.gradle ./
COPY src ./src

RUN gradle bootJar --no-daemon -x test

# ================================
# 런타임 스테이지 (Alpine)
# ================================
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**최종 조합 효과**
- Multi-Stage: 빌드 도구 제거
- Alpine: 경량 OS로 패키지 최소화
- 두 기법 결합으로 이미지 크기 및 보안 취약점 최소화

<br>

## 🧪 실험 방법

### 이미지 크기 측정

```dockerfile
docker images
```

### 보안 취약점 분석

Trivy를 사용하여 Docker 이미지의 보안 취약점을 분석했습니다.

```
sudo apt update
sudo apt install-ywget apt-transport-https gnupg lsb-release

wget-qO- https://aquasecurity.github.io/trivy-repo/deb/public.key |sudo apt-key add-

echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main |sudotee /etc/apt/sources.list.d/trivy.list

sudo apt update
sudo apt install trivy
```

각 이미지에 대해 동일한 방식으로 취약점을 측정했습니다.

```
trivy image app:before
trivy image app:alpine
trivy image app:multi
trivy image app:multi-alpine
trivy image app:slim
trivy image app:distroless
```

CVE 취약점 수를 기준으로 보안 수준을 비교했습니다.
## 👀 실험 결과

### 이미지 크기 비교

| 방법 | 베이스 이미지 | 이미지 크기 | 감소율 |
|------|-------------|-----------|--------|
| Baseline | `gradle:8.14-jdk17` | 1.37GB | - |
| Alpine | `eclipse-temurin:17-jre-alpine` | 548MB | 60.0% |
| Slim | `eclipse-temurin:17-jre-jammy` | 668MB | 51.2% |
| Distroless | `gcr.io/distroless/java17` | 269MB | 80.3% |
| Multi-Stage | `eclipse-temurin:17-jre` | 411MB | 70.0% |
| Multi-Stage + Alpine | `eclipse-temurin:17-jre-alpine` | 293MB | 78.6% |

### 보안 취약점 비교 (Trivy)

| 방법 | Vulnerabilities |
|------|------|
| Baseline | 59 |
| Alpine | 8 | 
| Slim | 84 |
| Distroless | 21 | 
| Multi-Stage | 29 |
| Multi-Stage + Alpine | 8 | 

<br>

## 🙆‍♂️ 결론

| 최적화 목적 | 권장 기법 |
|------------|---------|
| 이미지 크기 최소화 | Multi-Stage + Alpine |
| 보안 취약점 감소 | Multi-Stage + Alpine |
| CI/CD 빌드 속도 향상 | 레이어 캐시 최적화 |
| 둘 다 | Multi-Stage + Alpine |

단순히 베이스 이미지를 교체하거나 Multi-Stage Build를 적용하는 것만으로도 이미지 크기와 보안 취약점을 크게 줄일 수 있습니다.
두 기법을 조합하면 각각 적용했을 때보다 훨씬 큰 효과를 얻을 수 있으며, 실서비스 환경에서도 충분히 적용 가능한 수준의 결과를 확인했습니다.
