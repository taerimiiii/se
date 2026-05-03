# 요구사항 정의서: 미니 드라이브 (Mini Drive)

## 1. 개요 (Introduction)

### 1.1 문서의 목적
본 문서는 안전한 사내 협업을 위한 경량 클라우드 파일 공유 시스템인 '미니 드라이브(Mini Drive)'를 구축하기 위해 필요한 사용자의 요구사항과 개발 관련 요구사항을 명확히 정의하는 것을 목적으로 한다. 본 문서는 프로젝트에 참여하는 모든 이해관계자(기획자, 개발자, 관리자 등) 간의 원활한 의사소통 및 시스템 설계, 구현, 테스트의 명확한 기준이 된다.

### 1.2 프로젝트 개요
미니 드라이브(Mini Drive)는 조직 내 여러 팀이 문서를 중앙 집중식으로 보관하고 안전하게 공유할 수 있도록 지원하는 웹 기반의 클라우드 파일 관리 시스템이다. 복잡한 엔터프라이즈 기능 대신 5대 핵심 기능(파일/폴더 관리, 권한 기반 공유, 다중 필터 검색, 버전 관리, 보안/계정 관리)에 집중하여 직관적인 UI를 제공한다. 또한, 'Lean & Fast Architecting' 전략을 통해 서버 리소스를 최소화하고 저비용 고효율로 안정적인 운영이 가능한 환경을 제공한다.

### 1.3 관련 문서, 용어 및 약어
* **낙관적 락 (Optimized Locking):** 데이터 갱신 시 충돌이 발생하지 않을 것이라고 가정하고, 마지막에 충돌 여부를 확인하여 처리하는 동시성 제어 방식
* **Quota:** 시스템 내에서 개별 사용자 또는 그룹에게 할당된 데이터 최대 저장 공간 한도
* **TCO (Total Cost of Ownership):** 소프트웨어의 개발 및 도입부터 운영, 유지보수, 폐기까지 발생하는 총 소유 비용
* **Redis:** 파일 검색 속도 향상을 위한 메타데이터 캐싱 목적으로 활용되는 고성능 인메모리 데이터 구조 저장소

---

## 2. 시스템 개요
### 2.1 소프트웨어 컨텍스트(Context)
### 2.1.1 Actor Table

| **Actor Role** | **Description** |
| --- | --- |
| **관리자** | 이 시스템을 관리, 제공하는 사용자를 말한다. 사용자 계정을 생성 및 삭제하고, 사용자별 저장 공간(Quota)을 할당한다.   |
| **사용자** | 이 시스템을 사용하는 사용자를 말한다. 조직 내 일반 직원으로서 파일의 업로드/다운로드, 검색, 공유, 버전 관리 기능을 이용한다.   |
| **시스템** | 이 시스템을 이용하는 사용자와 관리자에게 파일 저장 및 권한 검증, 메타데이터 캐싱(Redis), 파일 버전 이력 보관 기능을 제공하는 주체를 말한다.   |

### 2.1.2 UseCase Diagram
![UseCase Diagram 이미지](../image/UseCase%20Diagram.png "미니 드라이브에 맞춰 작성한 유스케이스 다이어그램입니다.")

### 2.1.3 UseCase Description

#### Use Case 1: 로그인을 한다.

**Use Case Name :** 로그인을 한다.
**ID :** U_01
**Importance Level:** High
**Primary Actor :** 사용자, 관리자
**Use Case Type:** Detail, essential
**Brief Description :** 이 Use-Case는 사용자 및 관리자가 미니 드라이브 시스템에 접근하기 위해 이메일 기반으로 인증을 수행하는 Use Case를 표현한다.**Stakeholders and Interests**

- 사용자 및 관리자 : 자신의 권한과 할당된 Quota 내에서 시스템을 안전하게 사용하기 위해 인증받기를 원한다.
- 스폰서(경영진) : 인가된 내부 직원만 시스템에 접근하여 보안 리스크가 차단되기를 원한다.

**Trigger :** 사용자는 미니 드라이브 메인 화면에서 로그인 버튼을 누른다.

**Relationships**

- **Association :** 사용자, 관리자, 시스템
- **Include :**
- **Extend :**
- **Generalization :**

**Normal Flow of Events :**

1. 사용자는 이메일과 비밀번호를 입력한다.
2. 사용자는 로그인 버튼을 누른다.
3. 시스템은 입력된 데이터가 DB의 계정 정보와 일치하는지 확인하고 전송 구간을 TLS 1.3으로 암호화한다.
4. 시스템은 인증이 성공한 경우 사용자의 대시보드(루트 폴더) 화면으로 넘어간다.

**Subflows :** 없음

**Alternate / Exceptional Flows :**

3.a1 : 이메일 형식에 오류가 있거나 공란일 경우 시스템은 "올바른 이메일을 입력하세요"라는 메시지를 화면에 출력한다.

3.a2 : 비밀번호가 일치하지 않을 경우 시스템은 "로그인 정보가 일치하지 않습니다"라는 실패 이유를 화면에 출력한다.

#### Use Case 2: 파일 및 폴더를 관리한다.

**Use Case Name :** 파일 및 폴더를 관리한다.
**ID :** U_02
**Importance Level:** High
**Primary Actor :** 사용자
**Use Case Type:** Detail, essential
**Brief Description :** 이 Use-Case는 사용자가 다양한 포맷의 파일을 업로드/다운로드하고 계층형 폴더를 관리하는 Use Case를 표현한다.**Stakeholders and Interests**

- 사용자 : 복잡한 기능 없이 직관적인 3-Click 이내의 UI를 통해 100MB 이하의 파일을 지연 없이 업로드 및 정리하길 원한다.

**Trigger :** 사용자는 대시보드에서 '파일 업로드' 또는 '폴더 생성' 버튼을 누른다.

**Relationships**

- **Association :** 사용자, 시스템
- **Include :** [접근 권한을 검증한다]
- **Extend :**
- **Generalization :**

**Normal Flow of Events :**

1. 사용자는 업로드할 파일을 선택하거나 생성할 폴더명을 입력한다.
2. 사용자는 확인(업로드) 버튼을 누른다.
3. 시스템은 사용자의 남은 저장 공간(Quota)이 충분한지 확인한다.
4. 시스템은 낙관적 락(Optimized Locking)을 적용하여 파일 메타데이터를 갱신하고 스토리지에 파일을 저장한다.
5. 시스템은 Redis 인메모리 캐시를 갱신하여 검색 인덱스에 반영한다.
6. 시스템은 작업 완료 메시지를 화면에 출력한다.

**Subflows :**
4.a 파일 삭제 시: 시스템은 파일을 즉시 삭제하지 않고 휴지통으로 이동시켜 일정 기간 복구가 가능하도록 상태를 변경한다.

**Alternate / Exceptional Flows :**
3.a1 : 사용자의 할당된 저장 공간(Quota)을 초과한 경우, 시스템은 "저장 공간이 부족합니다"라는 에러 메시지를 출력하고 작업을 취소한다.
4.a1 : 네트워크 오류 등으로 업로드가 지연될 경우, 시스템은 자동 재시도 메커니즘을 3회 수행한다.

#### Use Case 3: 파일을 공유한다.

**Use Case Name :** 파일을 공유한다.
**ID :** U_03
**Importance Level:** High
**Primary Actor :** 사용자
**Use Case Type:** Detail, essential
**Brief Description :** 이 Use-Case는 조직 내 다른 사용자나 외부 협력업체에게 파일 접근 권한을 부여하고 공유 링크를 생성하는 Use Case를 표현한다.**Stakeholders and Interests**

- 사용자 : 업무 효율성을 높이기 위해 타 부서 팀원과 안전하고 쉽게 문서를 공유하길 원한다.
- 스폰서 : 외부 공유 시 만료 기한이 강제되어 기업의 데이터 유출 리스크 비용이 사전에 차단되기를 원한다.

**Trigger :** 사용자는 특정 파일이나 폴더를 우클릭하여 '공유' 버튼을 누른다.

**Relationships**

- **Association :** 사용자, 시스템
- **Include :** [접근 권한을 검증한다]
- **Extend :** [외부 다운로드 링크를 생성한다]
- **Generalization :**

**Normal Flow of Events :**

1. 사용자는 공유할 대상(조직 내 직원 이메일)을 입력한다.
2. 사용자는 대상자에게 부여할 권한(조회, 수정, 댓글) 중 하나를 세분화하여 선택한다.
3. 사용자는 공유하기 버튼을 누른다.
4. 시스템은 해당 파일의 ACL(Access Control List) 메타데이터를 업데이트한다.
5. 시스템은 대상자에게 공유 알림 이메일을 전송한다.

**Subflows :**
2.a 외부 협력업체 공유 시: 사용자는 대상을 입력하는 대신 '링크 복사'를 선택한다. 시스템은 자동으로 최대 7일의 만료 기간이 설정된 다운로드 전용 암호화 링크를 생성하여 화면에 출력한다.

**Alternate / Exceptional Flows :**

1.a1 : 존재하지 않는 사내 이메일을 입력한 경우 시스템은 "해당 사용자를 찾을 수 없습니다"라고 출력한다.


#### Use Case 4: 파일 버전을 복원한다.

**Use Case Name :** 파일 버전을 복원한다.
**ID :** U_04
**Importance Level:** High
**Primary Actor :** 사용자
**Use Case Type:** Detail, essential
**Brief Description :** 이 Use-Case는 동일한 파일이 수정되었을 때 시스템이 자동으로 보관한 이전 이력을 조회하고, 특정 과거 버전으로 되돌리거나 다운로드하는 Use Case를 표현한다.**Stakeholders and Interests**

- 사용자 : 실수로 덮어씌운 파일을 복원하여 재작업 비용(시간 손실)을 최소화하기를 원한다.

**Trigger :** 사용자는 특정 파일의 '버전 기록 보기' 버튼을 누른다.

**Relationships**

- **Association :** 사용자, 시스템
- **Include :**
- **Extend :**
- **Generalization :**

**Normal Flow of Events :**

1. 사용자는 버전 이력 목록에서 원하는 과거 버전을 선택한다.
2. 사용자는 '이 버전으로 복원' 버튼을 누른다.
3. 시스템은 낙관적 락(Optimized Locking)을 확인하여 현재 다른 사용자가 편집 중이지 않은지 검증한다.
4. 시스템은 선택된 과거 버전의 데이터를 현재 활성 파일로 덮어쓰고, 복원되었다는 새로운 버전 로그를 추가한다.
5. 시스템은 복원이 완료되었음을 화면에 표시한다.

**Subflows :** 없음

**Alternate / Exceptional Flows :**
3.a1 : 동시 수정을 시도하여 낙관적 락 충돌이 발생한 경우, 시스템은 "현재 다른 사용자가 파일을 업데이트 중입니다. 잠시 후 다시 시도해 주세요"라는 메시지를 출력한다.

---

## 3. 요구사항
요구사항은 고유 식별자, 그룹(분류), 단문 형태의 상세 내용, 우선순위(High/Medium/Low)로 체계화하여 기술한다.

### 3.1 정적 분석
![요구사항 정적 분석 이미지](../image/요구사항%20정적%20분석%20다이어그램.png "미니 드라이브에 맞춘 요구사항 정적 분석 다이어그램입니다.")

### 3.2 기능적 요구사항

| 식별자(ID) | 그룹(분류) | 요구사항 상세 내용 | 우선순위 |
| :--- | :--- | :--- | :--- |
| **F-001** | 파일 관리 | 사용자는 다양한 포맷의 파일을 업로드한다. | High |
| **F-002** | 파일 관리 | 사용자는 업로드된 파일을 다운로드한다. | High |
| **F-003** | 폴더 관리 | 사용자는 계층형 폴더를 생성하고 이동한다. | High |
| **F-004** | 폴더 관리 | 시스템은 삭제된 파일 및 폴더를 일정 기간 휴지통에 보관한다. | Medium |
| **F-005** | 공유 관리 | 사용자는 다른 사용자에게 파일의 조회, 수정, 댓글 권한을 세분화하여 부여한다. | High |
| **F-006** | 공유 관리 | 사용자는 외부 협력업체를 위한 만료 기간이 설정된 다운로드 링크를 생성한다. | High |
| **F-007** | 검색 관리 | 사용자는 파일명, 업로드 날짜, 사용자, 파일 유형을 조합하여 파일을 검색한다. | Medium |
| **F-008** | 버전 관리 | 시스템은 동일 파일 수정 시 이전 버전을 자동으로 보관한다. | High |
| **F-009** | 버전 관리 | 사용자는 보관된 특정 과거 버전으로 파일을 복원한다. | High |
| **F-010** | 계정 관리 | 시스템은 이메일과 비밀번호 기반으로 로그인 인증을 수행한다. | High |
| **F-011** | 계정 관리 | 관리자는 새로운 사용자 계정을 생성하고 삭제한다. | High |
| **F-012** | 계정 관리 | 관리자는 사용자별로 저장 공간(Quota)을 할당한다. | Medium |

### 3.3 비기능적 요구사항

| 식별자(ID) | 그룹(분류) | 요구사항 상세 내용 | 우선순위 |
| :--- | :--- | :--- | :--- |
| **N-001** | 성능 | 시스템은 100MB 이하 파일 기준 응답 지연 시간을 3초 이내로 유지한다. | High |
| **N-002** | 성능 | 시스템은 메타데이터 중심의 인메모리 캐싱(Redis)을 활용하여 데이터베이스 부하를 최소화한다. | Medium |
| **N-003** | 보안 | 시스템은 모든 데이터 전송 구간에 TLS 1.3 암호화를 적용한다. | High |
| **N-004** | 보안 | 시스템은 외부 공유 링크의 만료 기간을 최대 7일로 강제 설정한다. | High |
| **N-005** | 운영 | 시스템은 연간 99.9% 이상의 가동률(Availability)을 보장한다. | High |
| **N-006** | 운영 | 시스템은 파일 작업 오류 시 자동 재시도 메커니즘을 수행한다. | Medium |

### 3.4 인터페이스 요구사항

| 식별자(ID) | 그룹(분류) | 요구사항 상세 내용 | 우선순위 |
| :--- | :--- | :--- | :--- |
| **I-001** | UI/UX | 시스템은 사용자가 3-Click 이내에 원하는 목적지에 도달하는 인터페이스를 제공한다. | High |

### 3.5 CRC 카드.
#### 1. 사용자
- Class Name: 사용자 ID: 01 Type: Concrete, Domain
- Description: 시스템을 통해 파일을 업로드, 검색, 공유하는 일반 직원을 나타낸다.
- Associated Use Case: U_01, U_02, U_03, U_04
- Responsibilities
    - 파일 업로드 요청() : void
    - 파일 공유 요청() : void
    - 버전 복원 요청() : void
- Collaborators
    - 파일
    - 공유 권한
    - 버전 이력
- Attributes
    - 이메일 : String
    - 이름 : String
    - 할당용량 : Double
    - 사용용량 : Double
- Relationships
    - Aggregation (has-parts): 폴더, 파일
    - Other Associations: 인증 및 보안, 시스템 DB

#### 2. 관리자
- Class Name: 관리자 ID: 02 Type: Concrete, Domain
- Description: 미니 드라이브 시스템의 사용자 계정과 저장 공간(Quota)을 관리하는 사용자를 나타낸다.
- Associated Use Case: U_01
- Responsibilities
    - 사용자 계정 생성() : void
    - 할당 용량 설정() : void
- Collaborators
    - 사용자
    - 시스템 DB
- Attributes
    - 관리자 사번 : Integer
- Relationships
    - Generalization (a-kind-of): 사용자
    - Other Associations: 시스템 DB

#### 3. 인증 및 보안
- Class Name: 인증 및 보안 ID: 03 Type: Concrete, Controller
- Description: 시스템의 로그인 인증 및 전송 구간 암호화(TLS 1.3) 처리를 담당한다.
- Associated Use Case: U_01, U_02, U_03
- Responsibilities
    - 로그인 검증() : void
    - 로그아웃 처리() : void
    - 접근 권한 확인() : bool
- Collaborators
    - 사용자
    - 시스템 DB
- Attributes
    - 세션 ID : String
    - 토큰 : String
    - TLS 적용 여부 : Boolean
- Relationships
    - Other Associations: 사용자, 시스템 DB

### 4. 파일
- Class Name: 파일 ID: 04 Type: Concrete, Domain
- Description: 시스템에 업로드되어 보관 및 관리되는 개별 파일 객체를 나타낸다.
- Associated Use Case: U_02, U_03, U_04
- Responsibilities
    - 파일 저장() : void
    - 파일 다운로드() : void
    - 메타데이터 갱신() : void
- Collaborators
    - 사용자
    - 검색 캐시
    - 시스템 DB
- Attributes
    - 파일 ID : Integer
    - 파일명 : String
    - 확장자 : String
    - 파일 크기 : Double
    - 등록일시 : Date
- Relationships
    - Aggregation (has-parts): 버전 이력, 공유 권한
    - Other Associations: 폴더, 검색 캐시, 시스템 DB

#### 5. 폴더
- Class Name: 폴더 ID: 05 Type: Concrete, Domain
- Description: 파일을 계층적으로 묶어서 분류하고 관리하는 폴더 객체를 나타낸다.
- Associated Use Case: U_02
- Responsibilities
    - 폴더 생성() : void
    - 폴더 이동() : void
    - 폴더명 변경() : void
- Collaborators
    - 파일
    - 시스템 DB
- Attributes
    - 폴더 ID : Integer
    - 폴더명 : String
    - 상위 폴더 ID : Integer
- Relationships
    - Aggregation (has-parts): 파일
    - Other Associations: 사용자, 시스템 DB

#### 6. 버전 이력
- Class Name: 버전 이력 ID: 06 Type: Concrete, Domain
- Description: 동일 파일 수정 시 낙관적 락을 기반으로 변경된 과거 이력을 보관하고 관리한다.
- Associated Use Case: U_04
- Responsibilities
    - 이전 버전 조회() : void
    - 동시 수정 충돌 검증() : bool
    - 특정 버전 복원() : void
- Collaborators
    - 파일
    - 시스템 DB
- Attributes
    - 버전 ID : Integer
    - 파일 ID : Integer
    - 버전 번호 : Integer
    - 변경 일시 : Date
    - 수정자 ID : String
- Relationships
    - Other Associations: 파일, 시스템 DB

#### 7. 공유 권한
- Class Name: 공유 권한 ID: 07 Type: Concrete, Domain
- Description: 특정 파일이나 폴더에 대해 조직 내 사용자에게 부여된 세분화된 접근 권한을 정의한다.
- Associated Use Case: U_03
- Responsibilities
    - 읽기/수정 권한 부여() : void
    - 접근 권한 조회() : void
- Collaborators
    - 파일
    - 사용자
    - 시스템 DB
- Attributes
    - 공유 ID : Integer
    - 대상자 이메일 : String
    - 권한 유형 : String
- Relationships
    - Other Associations: 파일, 사용자, 시스템 DB

#### 8. 외부 링크
- Class Name: 외부 링크 ID: 08 Type: Concrete, Domain
- Description: 외부 협력업체와의 파일 공유를 위해 생성되는 만료 기한이 설정된 다운로드 전용 링크를 나타낸다.
- Associated Use Case: U_03
- Responsibilities
    - 외부 링크 생성() : String
    - 링크 유효기간 검증() : bool
    - 만료 처리() : void
- Collaborators
    - 파일
    - 시스템 DB
- Attributes
    - 링크 ID : Integer
    - 암호화된 URL : String
    - 만료 일자 : Date
- Relationships
    - Other Associations: 파일, 시스템 DB

#### 9. 검색 캐시
- Class Name: 검색 캐시 ID: 09 Type: Concrete, Infrastructure
- Description: 빠른 다중 필터 검색을 위해 파일 및 폴더의 메타데이터를 인메모리(Redis)에 보관한다.
- Associated Use Case: U_02
- Responsibilities
    - 다중 필터 검색() : void
    - 캐시 데이터 저장() : void
    - 메타데이터 캐시 갱신() : void
- Collaborators
    - 파일
    - 시스템 DB
- Attributes
    - 캐시 Key : String
    - 메타데이터 : String
- Relationships
    - Other Associations: 파일, 시스템 DB

#### 10. 시스템 DB
- Class Name: 시스템 DB ID: 10 Type: Concrete, Infrastructure
- Description: 미니 드라이브의 사용자, 파일, 권한, 버전 등의 데이터를 영구적으로 저장하는 데이터베이스를 나타낸다.
- Associated Use Case: U_01, U_02, U_03, U_04
- Responsibilities
    - 데이터 저장() : void
    - 데이터 수정() : void
    - 데이터 삭제() : void
- Collaborators
    - 모든 Domain 클래스
- Attributes
    - DB URL : String
    - 연결 상태 : Boolean
    - 커넥션 풀 사이즈 : Integer
- Relationships
    - Other Associations: 사용자, 파일, 폴더, 버전 이력, 공유 권한, 외부 링크

---

## 4. 품질 요구사항

### 4.1 신뢰성 (Reliability)
* 시스템 가동률(Availability)은 연간 99.9% 이상을 유지해야 한다.
* 네트워크 혹은 시스템 일시 오류 시 작업 내역 손실을 방지하기 위한 자동 재시도 기능을 제공해야 한다.

### 4.2 사용성 (Usability)
* 신규 사용자가 별도의 매뉴얼이나 교육 없이 3분 이내에 주요 기능(파일 업/다운로드, 공유 등)을 숙달할 수 있도록 직관적으로 설계되어야 한다. (학습 비용 제로화)

### 4.3 성능 효율성 (Performance Efficiency)
* 사용자의 대기 시간을 최소화하기 위해 100MB 이하 파일의 업로드/다운로드 및 조회 응답 지연 시간을 3초 이내로 유지해야 한다.

### 4.4 유지보수성 및 가독성 (Maintainability & Readability)
* 향후 시스템 확장을 고려하여 결합도를 낮춘 모듈형 아키텍처(Modular Monolith)로 구성되어야 한다.
* 인수인계 및 코드 리뷰 비용을 최소화하기 위해 코드 중복 비율을 5% 미만으로 통제해야 한다.

### 4.5 검증 가능성 (Testability)
* 안정적인 배포와 운영 중 장애 발생 리스크를 낮추기 위해, 유닛 테스트 커버리지를 80% 이상 확보해야 한다.

### 4.6 비용 효율성 (Cost Effectiveness)
* 클라우드 등 인프라 자원 사용량을 모니터링하여 월간 운영 예산 상한선의 90% 내에서 구동되도록 서버 리소스를 최적화해야 한다.

---

## 5. 개발 요구사항

### 5.1 하드웨어 환경
* **개발 및 테스트 서버:** 라즈베리파이 4 Model B (1.5GHz 쿼드 코어 64비트 CPU, RAM 4GB, MicroSD 64GB)
* **로컬 개발 PC:** Intel i5 11세대 이상 CPU, RAM 16GB, SSD 512GB 이상 사양의 노트북

### 5.2 소프트웨어 및 개발 환경
* **운영체제 (OS):** Windows 11 (개발용), Ubuntu 22.04 LTS 또는 Raspbian (서버 배포용)
* **백엔드/데이터베이스:** Spring Boot 기반 아키텍처, Redis (인메모리 메타데이터 캐싱)
* **버전 관리 및 협업 도구:** Git, GitHub (소스 코드 및 산출물 버전 관리)

---

## 6. 기타 고려 사항
* **개발 프로세스:** 짧은 주기의 피드백과 요구사항 변경 수용을 위해 Agile의 SCRUM 방법론을 채택하며, 2~4주 단위의 Sprint를 운영한다.
* **아키텍처 전략:** 동시성 제어를 위해 무거운 실시간 동시 편집 엔진 대신 '낙관적 락(Optimized Locking)'을 도입하여 서버 리소스 소모를 절감하는 'Lean & Fast Architecting'을 지향한다.

---

## 7. 참고 문헌 및 부록

* **참고 문헌:**
    * 최신 소프트웨어 공학 (요구사항 도출 강의 교재 자료를 참고하였습니다.)
    * Spring Boot & Redis 가이드 문서
* **관련 부록:**
    * Project I: 프로젝트 정의서 (시스템 정의서)
    * Project II: 대상 시스템 품질 요소 추정하기
    * [김태림]프로젝트관리계획서_260425_Doc-001