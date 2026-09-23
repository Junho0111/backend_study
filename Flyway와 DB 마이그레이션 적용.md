# Flyway와 DB 마이그레이션 적용기

프로젝트를 진행하면서 데이터베이스 스키마를 변경해야하는 일이 생각보다 빈번하였습니다. 그리고 그때마다 실수로 코드에만 반영을 하고 DB에는 적용하지 않으므로써 생기는 문제가 있었습니다.
<br>
때문에 이를 해결하고자 변경 이력을 체계적으로 관리하면서 최신의 상태를 유지해주는 **Flyway**에 대해 정리하고자 작성하였습니다.

<br>

## 1. 마이그레이션과 Flyway

### 마이그레이션이란?
일반적으로 아는 개발 쪽에서의 마이그레이션이라고 하면 php -> Java 또는 MySQL -> PostgreSQL이라고 생각할것입니다.

### 데이터베이스 마이그레이션이란?
하지만 데이터베이스 마이그레이션은 데이터베이스의 스키마(구조)나 데이터를 변경하고 이를 지속적으로 관리하며 추적할 수 있도록 하는 과정을 의미합니다. 
즉 Git과 같은 버전 관리 시스템의 개념입니다. 데이터베이스의 구조 변경 이력(CREATE, ALTER, DROP 등)도 버전으로 관리하여 팀원들 간의 환경을 맞추고, 운영 환경에 안전하게 배포하기 위해 사용됩니다.

### 대표적인 DB 마이그레이션 도구
- **Flyway**: SQL 기반으로 동작하며, 직관적이고 설정이 매우 단순합니다.
- **Liquibase**: XML, YAML, JSON 등 다양한 형식으로 정의할 수 있어 DBMS에 덜 종속적이지만 학습 곡선이 있습니다.
> 스프링 부트 환경에서는 SQL 작성의 직관성과 단순성 때문에 Flyway를 많이 사용합니다.

<br>

## 2. Flyway는 언제, 왜 사용할까?

이해하기 쉽게 다음과 같은 시나리오를 상상해봅시다.
당신은 백엔드 개발 팀에 새롭게 합류한 신규 입사자입니다. 
```java
> A 팀원: "신입님, 프로젝트 세팅하고 서버 한 번 띄워보세요."

> 당신: (Git 클론 후 프로젝트를 실행한다) "서버 띄웠는데 에러가 나요. `Table 'member' doesn't exist` 라고 하네요."

> B 팀원: "아, 제가 어제 회원 테이블 컬럼 수정했는데 공유를 깜빡했네요. 슬랙에 쿼리 올려드릴 테니 로컬 DB에서 실행해주세요~"

> 당신: (슬랙에서 쿼리를 복사해 실행) "네, 이번엔 `column 'nickname' cannot be null` 이라는데요?"

> C 팀원: "그거 제가 닉네임 Not Null로 바꿨어요. 단톡방 공지사항에 SQL 스크립트 모아뒀으니, 그거 전부 순서대로 다 실행해보세요~."
```

<br>

다음과 같은 문제가 발생합니다.
- 팀원들마다 로컬 DB 스키마 상태가 달라져 에러가 발생합니다.
- 배포를 할 때, 운영 서버 DB에 어떤 쿼리를 실행해야 하는지 누군가는 수동으로 관리해야 합니다. <br>(실수로 누락하면 서비스 장애 발생)
- 누가, 언제, 왜 데이터베이스 구조를 변경했는지 추적하기가 어렵습니다.

**Flyway**는 바로 이러한 문제를 해결해줍니다. 
<br>
데이터베이스의 버전을 애플리케이션 코드처럼 관리하게 해주어, 애플리케이션이 실행될 때 자동으로 필요한 SQL 스크립트(마이그레이션)를 실행해 모든 환경의 DB 상태를 항상 동일하게 맞춰줍니다.

<br>

### Flyway 동작 원리와 문법
Flyway는 프로젝트 내에 지정된 경로(기본값: `resources/db/migration`)에 있는 `.sql` 파일들을 읽어 데이터베이스에 적용합니다.
> 설정을 통해 운영, 개발, 로컬 등 환경별 세팅이 가능함.

데이터베이스에는 `flyway_schema_history`라는 메타데이터 테이블이 자동으로 생성됩니다.
이 테이블을 통해 현재 DB가 몇 버전까지 스크립트를 실행했는지 기억하고, 서버가 켜질 때 적용되지 않은 다음 버전의 스크립트들만 순차적으로 실행합니다.

**네이밍 규칙**:
파일명은 반드시 규칙을 지켜야 인식합니다.
- `V1__init.sql`
- `V2__add_column.sql`
- **V (Prefix)**: Versioned Migration을 의미합니다. (반드시 대문자 V)
- **1, 2 (Version)**: 버전 번호. 주로 숫자나 날짜를 사용합니다.
- **__ (Separator)**: 언더바 2개로 버전과 설명을 구분합니다.
- **설명 (Description)**: 변경 내용을 적습니다.
- **.sql (Suffix)**: 확장자.

<br>

## 3. Flyway 적용

### 1) 의존성 추가 (build.gradle)
Spring Boot 환경에서는 `flyway-core`와 사용하는 DB에 맞는 의존성을 추가해주면 됩니다.
예시 프로젝트는 MySQL을 사용하고 있으므로 관련 의존성을 아래와 같이 포함합니다.
```gradle
dependencies {
    // ... 생략 ...
    
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-mysql'
    runtimeOnly 'com.mysql:mysql-connector-j'
}
```

<br>

### 2) 설정 (application-local.yaml)
application-local.yaml 설정 파일에 Flyway 활성화 여부와 마이그레이션 스크립트를 찾을 경로를 지정합니다.
<br>
(경로 지정에서 아실수있듯 경로는 원하는데로 변경이 가능합니다.)
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

<br>


### 3) 마이그레이션 스크립트 작성
지정한 경로(`src/main/resources/db/migration`) 하위에 규칙에 맞춰 스크립트를 작성해둡니다. 예시 프로젝트의 경우 다음과 같이 구성되어 있습니다.

- `V1__init.sql`
- `V2__add_area_code_to_region.sql`
- `V3__alter_poi_closed_days_length.sql`
- `V4__add_source_id_to_poi.sql`

예를 들어, 가장 처음 베이스가 되는 `V1__init.sql`의 내용 중 `region`과 `member` 테이블의 DDL은 다음과 같습니다.
> ENGINE=InnoDB DEFAULT CHARSET=utf8mb4; 이 부분은 무시하셔도됨.

```sql
CREATE TABLE region (
    id        BIGINT       NOT NULL AUTO_INCREMENT,
    parent_id BIGINT,
    name      VARCHAR(50)  NOT NULL,
    depth     INT          NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE member (
    id          BIGINT       NOT NULL AUTO_INCREMENT,
    email       VARCHAR(255) NOT NULL,
    password    VARCHAR(255),
    nickname    VARCHAR(50)  NOT NULL,
    provider    VARCHAR(20)  NOT NULL,
    provider_id VARCHAR(255),
    role        VARCHAR(20)  NOT NULL DEFAULT 'USER',
    created_at  DATETIME(6)  NOT NULL,
    updated_at  DATETIME(6)  NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_member_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

이렇게 파일을 만들어두고 스프링 부트 서버를 기동하면, Flyway가 자동으로 `V1`부터 `V4`까지 순서대로 DB에 반영해주며 덕분에 팀원 누구든 Git에서 코드를 Pull 받고 서버를 켜기만 하면, 모두가 동일한 최신 상태의 데이터베이스 환경을 유지할 수 있습니다. 

<br>

