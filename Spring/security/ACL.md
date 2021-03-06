# ACL

스프링 시큐리티는 ACL을 설정하는 전용 모듈을 지원합니다. ACL에는 도메인 객체와 연결하는 ID를 비록해서 여러 개의 ACE(접근 통제 엔티티)가 들어 있습니다. ACE는 다음 두 가지 핵심 요소로 구성됩니다.

## 퍼미션(Permission, 인가 받은 권한)
ACE 퍼미션은 각 비트 값을 특성 퍼미션을 의미하는 비트 마스크 입니다. `BasePermission` 클래스는 다섯 가지 기본 퍼미션을 갖고 있습니다.

| 권한              | 비트  | 정수  |
|-----------------|-----|-----|
| READE           | 0   | 1   |
| WRITE           | 1   | 2   |
| CREATE          | 2   | 4   |
| DELETE          | 3   | 8   |
| ADMINPERMISSION | 4   | 16  |

사용하지 않은 비트를 통해서 임의로 퍼미션을 지정할 수 있습니다.

## 보안 식별자(SID, Security IDentity)

<p algin ="cneter">
    <img src ="/assets/acl-sid.png">
</p>
각 ACE는 특정 SID에 대한 퍼미션을 가집니다. SID 주체는 (PrincipalSid)일 수도 있고 퍼미션과 관련된 권한(GrantedAuthoritySid)일 수도 있습니다. 스프링 시큐리티에는 ACL 객체 모델의 정의 뿐만 아니라 이 모델을 읽고 관리하는 API도 정의도어있습니다. 또 이 API를 구현한 고성능 JDBC 구현체까지 제공합니다. 아울러 ACL을 더욱쉽게 사용할 수 있도록 접근 통제결정 거수기나 JSP 태그 같은 편의 기능도 마련되어 있어서 애플리케이션 다른 보안 장치들과 일관된 방향으로 사용할 수 있습니다.

## ACL 데이터 구조
* Security Identity - SID
* Domain Object
* ACL Entry

Domain Object는 두 개의 엔티티로 구성됩니다.
* Class - 엔티티의 실제 Java 클래스
* Object Identity - 보안을 설정하려는 엔티티의 기본 식별자입니다.

ACL Entry
* 주체가 도메인 개체에 대해 가지는 실제 사용 권한을 나타냅니다.
* 기본적으로 읽기, 쓰기, 생성, 삭제, 관리자입니다. 정수 비트 마스크로 표시되므로 32 비트가 사용 가능합니다 (5 사용됨).
* 32 비트의 사용 가능 비트가 있고 5를 사용하기 때문에 필요한 경우 새로운 맞춤형 액세스 유형을 추가 할 수 있습니다.


## Code

* ACL 서비스를 설정하는 방법과 엔티티의 ACL 퍼미션을 다루는 방법을 차례로 설명
* ACL 퍼미션을 이용해서 엔티티에 보안 접근을 하는 보안 표현식의 사용법 설명


### ACL 서비스 설정하기

```sql
CREATE TABLE ACL_SID(
    ID         BIGINT        NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    SID        VARCHAR(100)  NOT NULL,
    PRINCIPAL  SMALLINT      NOT NULL,
    PRIMARY KEY (ID),
    UNIQUE (SID, PRINCIPAL)
);

CREATE TABLE ACL_CLASS(
    ID     BIGINT        NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    CLASS  VARCHAR(100)  NOT NULL,
    PRIMARY KEY (ID),
    UNIQUE (CLASS)
);

CREATE TABLE ACL_OBJECT_IDENTITY(
    ID                  BIGINT    NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    OBJECT_ID_CLASS     BIGINT    NOT NULL,
    OBJECT_ID_IDENTITY  BIGINT    NOT NULL,
    PARENT_OBJECT       BIGINT,
    OWNER_SID           BIGINT,
    ENTRIES_INHERITING  SMALLINT  NOT NULL,
    PRIMARY KEY (ID),
    UNIQUE (OBJECT_ID_CLASS, OBJECT_ID_IDENTITY),
    FOREIGN KEY (PARENT_OBJECT)   REFERENCES ACL_OBJECT_IDENTITY,
    FOREIGN KEY (OBJECT_ID_CLASS) REFERENCES ACL_CLASS,
    FOREIGN KEY (OWNER_SID)       REFERENCES ACL_SID
);

CREATE TABLE ACL_ENTRY(
    ID                  BIGINT    NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    ACL_OBJECT_IDENTITY BIGINT    NOT NULL,
    ACE_ORDER           INT       NOT NULL,
    SID                 BIGINT    NOT NULL,
    MASK                INTEGER   NOT NULL,
    GRANTING            SMALLINT  NOT NULL,
    AUDIT_SUCCESS       SMALLINT  NOT NULL,
    AUDIT_FAILURE       SMALLINT  NOT NULL,
    PRIMARY KEY (ID),
    UNIQUE (ACL_OBJECT_IDENTITY, ACE_ORDER),
    FOREIGN KEY (ACL_OBJECT_IDENTITY) REFERENCES ACL_OBJECT_IDENTITY,
    FOREIGN KEY (SID)                 REFERENCES ACL_SID
);
```
* 스프링 JDBC로 RDBMS에 접속해서 ACL 데이터 저장/조회 하는 기능을 기본적 지원합니다.
* 스프링 시큐리티에는 테이블에 지정된 ACL 데이터에 액세스할 수 있는 고성능 JDBC 구현체 및 API가 준비되어 있음
* ACL 개수는 상당히 많아 질수 있어 스프링 시큐리티는 ACL 객체를캐시하는 기능을 지원함


```java
@Configuration
public class TodoAclConfig {

    private final DataSource dataSource;

    public TodoAclConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AclEntryVoter aclEntryVoter(AclService aclService) {
        return new AclEntryVoter(aclService, "ACL_MESSAGE_DELETE", new Permission[]{BasePermission.ADMINISTRATION, BasePermission.DELETE});
    }

    @Bean
    public EhCacheCacheManager ehCacheManagerFactoryBean() {
        return new EhCacheCacheManager();
    }

    @Bean
    public AuditLogger auditLogger() {
        return new ConsoleAuditLogger();
    }

    @Bean
    public PermissionGrantingStrategy permissionGrantingStrategy() {
        return new DefaultPermissionGrantingStrategy(auditLogger());
    }

    @Bean
    public AclAuthorizationStrategy aclAuthorizationStrategy() {
        return new AclAuthorizationStrategyImpl(new SimpleGrantedAuthority("ADMIN"));
    }

    @Bean
    public AclCache aclCache(CacheManager cacheManager) {
        return new SpringCacheBasedAclCache(cacheManager.getCache("aclCache"), permissionGrantingStrategy(), aclAuthorizationStrategy());
    }

    @Bean
    public LookupStrategy lookupStrategy(AclCache aclCache) {
        return new BasicLookupStrategy(this.dataSource, aclCache, aclAuthorizationStrategy(), permissionGrantingStrategy());
    }

    @Bean
    public AclService aclService(LookupStrategy lookupStrategy, AclCache aclCache) {
        return new JdbcMutableAclService(this.dataSource, lookupStrategy, aclCache);
    }

    @Bean
    public AclPermissionEvaluator permissionEvaluator(AclService aclService) {
        return new AclPermissionEvaluator(aclService);
    }
}
```
* 스프링 시큐리티에서 ACL 서비스 작업은 `AclService`, `MutableAclService` 두 인터페이스로 정의합니다.
* `AclService`는 읽기 작업을, 그 하위 인터페이스는 `MutableAclService`는 나머지 ACL 작업들(생성, 수정, 삭제)를 각각 기술합니다.
* 그냥 ACL 읽기만 할 경우 `JdbcAclService` 같은 `AclService` 구현체를, 그 외에는 `JdbcMutableAclService`같은 MutableAclService 구현체를 각각 골라쓰면 됩니다.
* 예제에서는 ADMIN 권한을 지닌 유저만 ACL 소유권 ACE 검사 세부 등 여러가지 ACL/ACE 상세 정보를 수정할 수 있습니다.
* `PermissionGrantingStrategy형` 생성자 인수는 자신이 가지고 이쓴ㄴ Permission 값으로 주어진 SID에 ACL 약세스를 허용할지 결정합니다.
* `JdbcMutableAclService`에는 ACL 데이터를 RDBMS에서 관리할 때 필요한 표준 SQL문이 들어 있지만 모든 DB제품이 호환되는건 아닙니다.(아파티 더비)

### 도메인 객체에 대한 ACL 관리하기
백엔드 서비스와 DAO에는 의존성 주입을 이용해서 앞서 정의한 ACL 서비스를 이용하여도 도메인 객체용 ACL을 관리해야합니다. 가령 스케줄 관리 앱에서는 할 일을 등록/삭제할 때마다 각각 ACL/생성 삭제해야합니다.

```java
@Service
@Transactional
class TodoServiceImpl implements TodoService {

    private final TodoRepository todoRepository;
    private final MutableAclService mutableAclService;

    TodoServiceImpl(TodoRepository todoRepository, MutableAclService mutableAclService) {
        this.todoRepository = todoRepository;
        this.mutableAclService = mutableAclService;
    }

    @Override
    @PreAuthorize("hasAuthority('USER')")
    public void save(Todo todo) {

        this.todoRepository.save(todo);

        ObjectIdentity oid = new ObjectIdentityImpl(Todo.class, todo.getId());
        MutableAcl acl = mutableAclService.createAcl(oid);
        acl.insertAce(0, READ, new PrincipalSid(todo.getOwner()), true);
        acl.insertAce(1, WRITE, new PrincipalSid(todo.getOwner()), true);
        acl.insertAce(2, DELETE, new PrincipalSid(todo.getOwner()), true);

        acl.insertAce(3, READ, new GrantedAuthoritySid("ADMIN"), true);
        acl.insertAce(4, WRITE, new GrantedAuthoritySid("ADMIN"), true);
        acl.insertAce(5, DELETE, new GrantedAuthoritySid("ADMIN"), true);

    }
}
```
유저가 할 일등 등록하면 할일 ID와 ACL 객체의 ID를 이용해 ACL을 생성하고 반대로 할일을 삭제하면 해당 ACL도 함께 삭제합니다. 새로 등록한 할 일에 대해서는 다음 ACE를 ACL에 산입합니다.

* 할 일 등록자는 할일을 READE, WRITE, DELETE를 할 수 있습니다.
* ADMIN 권한 유저도 할일을 READE, WRITE, DELETE 할 수 있습니다.
  
### 표현식을 이용해 접근 통제 결정하기

```java
@Service
@Transactional
class TodoServiceImpl implements TodoService {

    ...

    @Override
    @PreAuthorize("hasAuthority('USER')")
    @PostFilter("hasAnyAuthority('ADMIN') or hasPermission(filterObject, 'read')")
    public List<Todo> listTodos() {
        return todoRepository.findAll();
    }

    @Override
    @PreAuthorize("hasPermission(#id, 'com.apress.springrecipes.board.Todo', 'write')")
    public void complete(long id) {
        Todo todo = findById(id);
        todo.setCompleted(true);
        todoRepository.save(todo);
    }

    @Override
    @PreAuthorize("hasPermission(#id, 'com.apress.springrecipes.board.Todo', 'delete')")
    public void remove(long id) {
        todoRepository.remove(id);

        ObjectIdentity oid = new ObjectIdentityImpl(Todo.class, id);
        mutableAclService.deleteAcl(oid, false);
    }

    @Override
    @PostFilter("hasPermission(filterObject, 'read')")
    public Todo findById(long id) {
        return todoRepository.findOne(id);
    }
}
```
* 도메인 객체마다 ACL이 부착되어 있으니 이 객체에 속한 메서드마다 접그 텅제 결정을 내릴 수 있습니다.
* 유저가 할 일을 삭제하려고 하면 ACL을 보고 그 유저가 정말 삭제할 권한이 있는지 체크할 수 있습니다,
* `@PreAuthorize / @PreFilter, @PostAuthorize / @PostFilter` 에노테이션을 이용하면 해당 리소스에 대한 사용 권한이 있는지 체크할 수 있습니다.
* `@EnableGobalMethodSecurity(prePostEnalbe = true)` 설정을 해햐 위의 어노테이션을 사용할 수 있습니다.
* `@PreAuthorize`는 유저가 메서드를 수행할 퍼미션을 갖고 있는지 체크합니다.

