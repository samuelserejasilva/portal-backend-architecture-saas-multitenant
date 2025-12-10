# 🧩 Guia de Desenvolvimento — Portal Auditoria 2.0 (Modulith)

**Última atualização:** 2025-10-29 08:30
**Contexto:** Spring Boot 3.5.7 • Java 21 • Spring Modulith 1.3.2 • MariaDB 12.0.2

> 📖 **Sobre este documento:**
> Este é o guia completo de desenvolvimento do projeto, consolidando boas práticas, padrões arquiteturais, convenções de código e histórico de alterações técnicas.

---

## 📚 **GUIA DE BOAS PRÁTICAS E PADRÕES DO PROJETO**

### 🚫 **O QUE NÃO FAZER**

#### **1. Dependências entre Módulos**

```java
// ❌ ERRADO - Dependência circular ou invertida
@ApplicationModule(allowedDependencies = {"content"})  // Mídia NÃO deve depender de Content
package com.auditoria.portalweb.modules.midia;

// ❌ ERRADO - Acessar repositório de outro módulo diretamente
@Service
class EmpresaService {
    @Autowired
    private MediaRepository mediaRepository;  // VIOLA fronteira modular!
}

// ❌ ERRADO - Nome incompleto de módulo
@ApplicationModule(allowedDependencies = {"midia::domain"})  // Falta o prefixo "modules."
```

#### **2. Spring Beans - Tipos de Retorno**

```java
// ❌ ERRADO - Retornar interface em vez de classe concreta
@Bean
public PasswordEncoder passwordEncoder() {  // ⚠️ Warning: Ensure concrete bean type
    return new BCryptPasswordEncoder();
}

@Bean
public SecurityFilterChain authFilterChain(HttpSecurity http) {  // ⚠️ Warning
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {  // ⚠️ Warning
    return source;
}
```

#### **3. APIs Deprecadas**

```java
// ❌ ERRADO - JJWT API antiga (deprecada desde 0.12.x)
import io.jsonwebtoken.SignatureAlgorithm;  // Deprecated!

return Jwts.builder()
    .signWith(key, SignatureAlgorithm.HS256);  // Método deprecado

// ❌ ERRADO - Spring Data JPA API antiga (deprecada desde 3.5.0)
Specification<AuditEvent> spec = Specification.where(pathSpec)  // Deprecated!
    .and(emailSpec)
    .and(statusSpec);

// ❌ ERRADO - Java Stream API antiga
.collect(Collectors.toList())  // Verboso, use .toList() em Java 16+
```

#### **4. Configuração de Testes**

```java
// ❌ ERRADO - Falta configuração JWT nos testes
@SpringBootTest
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb"
    // FALTANDO: app.auth.jwt.secret, etc. - vai falhar!
})

// ❌ ERRADO - JWT secret muito curto
"app.auth.jwt.secret=secret123"  // Precisa mínimo 32 caracteres para HS256!
```

#### **5. Comentários e Documentação**

```java
// ❌ ERRADO - Comentários óbvios ou redundantes
// Retorna uma lista de empresas
public List<Empresa> listarEmpresas() { ... }  // Comentário inútil

// ❌ ERRADO - Código comentado (usar controle de versão)
// @Autowired
// private OldService oldService;

// ❌ ERRADO - TODOs sem contexto
// TODO: fix this  // O que precisa ser corrigido?
```

---

### ✅ **O QUE FAZER - PADRÕES CORRETOS**

#### **1. Configuração de Módulos Spring Modulith**

```java
// ✅ CORRETO - Módulo autossuficiente (infraestrutura)
@ApplicationModule
package com.auditoria.portalweb.modules.midia;

// ✅ CORRETO - Dependências explícitas e unidirecionais
@ApplicationModule(allowedDependencies = {
    "shared",                    // Módulo compartilhado
    "shared::mapper",            // Named interface
    "shared::dto",               // Named interface
    "modules.midia::domain"      // Nome COMPLETO do módulo
})
package com.auditoria.portalweb.modules.corporate;

// ✅ CORRETO - Expor interface pública do módulo
@NamedInterface("domain")
package com.auditoria.portalweb.modules.midia.domain;

@NamedInterface("api")
package com.auditoria.portalweb.modules.corporate.api;

@NamedInterface("spi")  // Service Provider Interface
package com.auditoria.portalweb.modules.corporate.spi;
```

#### **2. Spring Beans - Tipos Concretos**

```java
// ✅ CORRETO - Retornar classe concreta (AOT/Native compatibility)
@Configuration
public class UsersConfig {
    @Bean
    public BCryptPasswordEncoder passwordEncoder() {  // ✅ Tipo concreto
        return new BCryptPasswordEncoder();
    }
}

@Configuration
public class AuthSecurityConfig {
    @Bean
    DefaultSecurityFilterChain authFilterChain(HttpSecurity http) throws Exception {  // ✅ Tipo concreto
        http.cors(c -> {})
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth ->
                auth.requestMatchers(PUBLIC_URLS).permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();  // ✅ Retorna DefaultSecurityFilterChain
    }
}

@Configuration
public class CorsConfig {
    @Bean
    public UrlBasedCorsConfigurationSource corsConfigurationSource() {  // ✅ Tipo concreto
        CorsConfiguration config = new CorsConfiguration();
        // ... configuração
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

**Por quê tipos concretos?**

- ✅ Compatibilidade com Spring AOT (Ahead-of-Time compilation)
- ✅ Compatibilidade com GraalVM Native Image
- ✅ Melhor inferência de tipos pelo compilador
- ✅ Mais eficiente em tempo de execução

#### **3. APIs Modernas - JJWT 0.12.x**

```java
// ✅ CORRETO - JJWT API moderna (0.12.x)
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;

public class JwtTokenProvider {
    private final SecretKey key;

    public JwtTokenProvider(@Value("${app.auth.jwt.secret}") String secret) {
        byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
        this.key = Keys.hmacShaKeyFor(keyBytes);  // ✅ Gera SecretKey
    }

    public String newAccessToken(Long userId, String email, String role) {
        Instant now = Instant.now();
        return Jwts.builder()
            .issuer(issuer)
            .subject(email)
            .claim("uid", userId)
            .claim("role", role)
            .issuedAt(Date.from(now))
            .expiration(Date.from(now.plusSeconds(accessTtlSec)))
            .signWith(key)  // ✅ Algoritmo inferido automaticamente do tipo SecretKey
            .compact();
    }
}
```

**Por quê?**

- ✅ `signWith(key)` infere automaticamente o algoritmo (HS256, HS384, HS512)
- ✅ API type-safe e moderna
- ✅ Sem enums deprecados

#### **4. APIs Modernas - Spring Data JPA 3.5+**

```java
// ✅ CORRETO - Specification.allOf() / anyOf()
import org.springframework.data.jpa.domain.Specification;

Specification<AuditEvent> spec = Specification.allOf(  // ✅ API moderna
    AuditSpecs.pathLike(path),
    AuditSpecs.actorEmailLike(actorEmail),
    AuditSpecs.statusEq(status),
    AuditSpecs.createdBetween(from, to)
);

Page<AuditEvent> page = repository.findAll(spec, pageable);
```

**Por quê?**

- ✅ Mais explícito: `allOf()` = AND, `anyOf()` = OR
- ✅ `.where()` está deprecado desde Spring Data JPA 3.5.0
- ✅ Melhor legibilidade do código

#### **5. Java Moderno - Streams**

```java
// ✅ CORRETO - Java 16+ Stream.toList()
List<String> origins = Arrays.stream(allowedOrigins.split(","))
    .map(String::trim)
    .filter(s -> !s.isEmpty())
    .toList();  // ✅ Conciso e moderno

// ❌ EVITE (verboso)
.collect(Collectors.toList())  // Ainda funciona, mas verboso
```

#### **6. Relacionamentos Cross-Module (EntityManager)**

```java
// ✅ CORRETO - Usar EntityManager.getReference() para entidades de outros módulos
@Service
public class EmpresaServiceImpl implements EmpresaService {
    @PersistenceContext
    private EntityManager em;

    @Transactional
    public EmpresaDTO save(EmpresaSaveDTO dto) {
        Empresa empresa = new Empresa();
        // ... preencher campos

        if (dto.logoMediaId() != null) {
            // ✅ Cria proxy JPA sem carregar entidade do outro módulo
            Media logoMedia = em.getReference(Media.class, dto.logoMediaId());
            empresa.setLogoMedia(logoMedia);
        }

        return mapper.toDto(repository.save(empresa));
    }
}
```

**Por quê?**

- ✅ Respeita fronteiras modulares (não precisa do MediaRepository)
- ✅ Performance: não carrega entidade desnecessariamente
- ✅ JPA cria proxy lazy que só carrega se acessado

#### **7. Configuração de Testes**

```java
// ✅ CORRETO - TestPropertySource completo
@SpringBootTest(classes = {PortalwebApplication.class, TestConfig.class})
@TestPropertySource(properties = {
    // H2 Database
    "spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE",
    "spring.datasource.username=sa",
    "spring.datasource.password=",
    "spring.datasource.driver-class-name=org.h2.Driver",
    "spring.jpa.hibernate.ddl-auto=create-drop",
    "spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect",

    // JWT Configuration (MÍNIMO 32 caracteres para HS256!)
    "app.auth.jwt.secret=TEST_SECRET_KEY_FOR_TESTING_PURPOSES_MIN_32_CHARS_REQUIRED",
    "app.auth.jwt.issuer=portalweb-test",
    "app.auth.jwt.access-ttl-sec=900",
    "app.auth.jwt.refresh-ttl-sec=1209600",

    // Logging
    "logging.level.org.hibernate=WARN",
    "logging.level.org.springframework=WARN"
})
class PortalwebApplicationTests {
    @Test
    void contextLoads() {
        // Context loading test
    }
}
```

**Regras JWT Secret:**

- ✅ HS256: mínimo **32 caracteres** (256 bits)
- ✅ HS384: mínimo **48 caracteres** (384 bits)
- ✅ HS512: mínimo **64 caracteres** (512 bits)

#### **8. Comentários Úteis**

```java
// ✅ CORRETO - Comentários que explicam "POR QUÊ", não "O QUÊ"

/**
 * Usa EntityManager.getReference() em vez de MediaRepository.findById()
 * para respeitar fronteiras modulares do Spring Modulith.
 * O proxy JPA só carrega a entidade se necessário (lazy loading).
 */
Media logoMedia = em.getReference(Media.class, dto.logoMediaId());

/**
 * IMPORTANTE: O secret JWT deve ter no mínimo 32 caracteres para HS256.
 * Valores menores causam IllegalArgumentException em tempo de execução.
 */
@Value("${app.auth.jwt.secret}")
private String jwtSecret;

/**
 * TODO(SAMUEL-2025-10-29): Implementar cache Redis para tokens revogados
 * Atualmente usando banco de dados, mas com alto volume pode gerar gargalo.
 * Estimativa: 2 dias de trabalho.
 */
```

#### **9. Anotações Spring Modulith**

```java
// ✅ Módulo simples (sem dependências)
@ApplicationModule
package com.auditoria.portalweb.modules.midia;

// ✅ Módulo com dependências explícitas
@ApplicationModule(allowedDependencies = {
    "shared",
    "shared::mapper",
    "shared::dto",
    "modules.midia::domain"
})
package com.auditoria.portalweb.modules.corporate;

// ✅ Expor interface pública
@NamedInterface("domain")  // Entidades JPA
package com.auditoria.portalweb.modules.midia.domain;

@NamedInterface("api")  // DTOs e Services públicos
package com.auditoria.portalweb.modules.users.api;

@NamedInterface("spi")  // Service Provider Interface
package com.auditoria.portalweb.modules.corporate.spi;
```

#### **10. Convenções de Nome**

```java
// ✅ CORRETO - Sufixos consistentes
@Service
public class EmpresaServiceImpl implements EmpresaService { }  // Implementação

@RestController
@RequestMapping("/api/empresas")
public class EmpresaController { }  // Controller sempre singular

@RestController
@RequestMapping("/api/admin/empresas")
public class AdminEmpresaController { }  // Admin prefix para recursos administrativos

@Repository
public interface EmpresaRepository extends JpaRepository<Empresa, Long> { }  // Repository

@Mapper(config = MapStructConfig.class)
public interface EmpresaMapper { }  // Mapper

public record EmpresaSaveDTO(...) { }  // DTO com sufixo descritivo (Save/Update/Response)
```

---

### 🛡️ **VALIDAÇÃO E TESTES**

#### **Executar Testes de Arquitetura**

```bash
# Validar fronteiras modulares
mvn test -Dtest=ModulithArchitectureTests

# Executar todos os testes
mvn clean test

# Gerar documentação Modulith (automático nos testes)
# Saída: target/spring-modulith-docs/
mvn clean verify
```

#### **Testes Obrigatórios**

1. **ModulithArchitectureTests** - Valida arquitetura modular
2. **PortalwebApplicationTests** - Valida contexto Spring
3. **Testes unitários** - Para cada service/controller

---

## 📝 **CONVENÇÕES DE CÓDIGO E NOMENCLATURA**

### 🏷️ **Nomenclatura Padrão**

#### **Módulos e Pacotes**

```java
// ✅ CORRETO - Módulos sempre no plural
com.auditoria.portalweb.modules.corporate
com.auditoria.portalweb.modules.content
com.auditoria.portalweb.modules.users

// ✅ CORRETO - Entidades sempre no singular
public class Empresa { }
public class Post { }
public class Usuario { }
```

#### **Classes de Serviço**

```java
// ✅ CORRETO - Interface + Implementação
public interface EmpresaService { }

@Service
class EmpresaServiceImpl implements EmpresaService { }
```

#### **Controllers**

```java
// ✅ CORRETO - Sufixo Controller
@RestController
@RequestMapping("/api/empresas")
public class EmpresaController { }

@RestController
@RequestMapping("/api/admin/empresas")
public class AdminEmpresaController { }
```

#### **DTOs**

```java
// ✅ CORRETO - Sufixos descritivos
public record EmpresaDTO(...) { }           // Read/Response
public record EmpresaCreateDTO(...) { }     // Create
public record EmpresaUpdateDTO(...) { }     // Update/Patch
public record EmpresaFilterDTO(...) { }     // Filtros de busca
```

### 📦 **Estrutura de Pacotes por Módulo**

```txt
modules/{nome}/
├── package-info.java          # @ApplicationModule
├── domain/                    # Entidades JPA (PRIVADO)
│   └── {Entity}.java
├── repository/                # Spring Data JPA (PRIVADO)
│   └── {Entity}Repository.java
├── service/                   # Interfaces de serviço (PRIVADO)
│   └── {Entity}Service.java
├── internal/                  # Implementações (PACKAGE-PRIVATE)
│   └── {Entity}ServiceImpl.java
├── api/                       # API REST interna do módulo
│   ├── dto/                   # DTOs para API REST
│   │   └── {Entity}DTO.java
│   └── mapper/                # MapStruct mappers
│       └── {Entity}Mapper.java
├── web/                       # Controllers REST
│   └── {Entity}Controller.java
├── spi/                       # ✅ INTERFACE PÚBLICA (outros módulos)
│   ├── package-info.java      # @NamedInterface("spi")
│   ├── {Module}Api.java       # Contratos públicos
│   └── dto/                   # DTOs públicos
│       ├── package-info.java  # @NamedInterface
│       └── {Entity}PublicDTO.java
└── domain/                    # ✅ EXPOSIÇÃO DE ENTIDADES (quando necessário)
    └── package-info.java      # @NamedInterface("domain")
```

### 🎯 **Regras de Camadas**

#### **1. Fluxo HTTP Obrigatório**

```java
// ✅ CORRETO
Browser → Controller → Service → Repository → Database

// ❌ ERRADO - Controller NÃO pode acessar Repository direto
@RestController
class EmpresaController {
    @Autowired
    private EmpresaRepository repository;  // ❌ PROIBIDO!
}
```

#### **2. DTOs Obrigatórios em APIs REST**

```java
// ✅ CORRETO - Controller retorna DTO
@GetMapping("/{id}")
public ResponseEntity<EmpresaDTO> getById(@PathVariable Long id) {
    return ResponseEntity.ok(empresaService.findById(id));
}

// ❌ ERRADO - NUNCA retornar Entity
@GetMapping("/{id}")
public ResponseEntity<Empresa> getById(@PathVariable Long id) {  // ❌ PROIBIDO!
    return ResponseEntity.ok(empresaRepository.findById(id).orElseThrow());
}
```

#### **3. MapStruct - Configuração Global**

```java
// ✅ CORRETO - Sempre usar MapStructConfig do shared
@Mapper(config = MapStructConfig.class)
public interface EmpresaMapper {
    EmpresaDTO toDto(Empresa entity);
    Empresa toEntity(EmpresaCreateDTO dto);

    // Para updates parciais
    void updateEntityFromDto(EmpresaUpdateDTO dto, @MappingTarget Empresa entity);
}
```

**Por quê?**

- `MapStructConfig` está em `shared/mapper`
- Define `componentModel = "spring"` globalmente
- Garante consistência em todos os mappers

#### **4. Validação**

```java
// ✅ CORRETO - @Valid em DTOs de entrada
@PostMapping
public ResponseEntity<EmpresaDTO> create(@Valid @RequestBody EmpresaCreateDTO dto) {
    return ResponseEntity
        .status(HttpStatus.CREATED)
        .body(empresaService.create(dto));
}

// ✅ CORRETO - Mensagens em messages.properties
javax.validation.constraints.NotBlank.message=Campo obrigatório
javax.validation.constraints.Email.message=Email inválido
```

#### **5. Exceções - Tratamento Global**

```java
// ✅ CORRETO - Lançar exceções de negócio
@Service
class EmpresaServiceImpl implements EmpresaService {
    public EmpresaDTO findById(Long id) {
        return repository.findById(id)
            .map(mapper::toDto)
            .orElseThrow(() -> new EntityNotFoundException("Empresa não encontrada: " + id));
    }
}

// ✅ Tratamento global em GlobalExceptionHandler
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(EntityNotFoundException ex) {
        ApiError error = new ApiError(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            Instant.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

### ⚠️ **Null-Safety em Classes Spring**

Quando implementar interfaces do Spring Framework (ex.: `WebMvcConfigurer`, `Converter`, `HandlerInterceptor`):

```java
// ✅ CORRETO - Aplicar política global de null-safety por pacote
// package-info.java
@org.springframework.lang.NonNullApi
@org.springframework.lang.NonNullFields
package com.auditoria.portalweb.config;
```

**Por quê?**

- Evita warnings do tipo: "Missing non-null annotation"
- Melhora análise estática de código
- Compatível com Kotlin null-safety

### 🚫 **Dependências entre Módulos**

```java
// ❌ ERRADO - Importar service/repository/domain de outro módulo
import com.auditoria.portalweb.modules.midia.repository.MediaRepository;  // ❌
import com.auditoria.portalweb.modules.corporate.service.EmpresaService;  // ❌
import com.auditoria.portalweb.modules.content.domain.Post;               // ❌

// ✅ CORRETO - Apenas via SPI ou domain (Named Interface)
import com.auditoria.portalweb.modules.corporate.spi.EmpresaApi;          // ✅
import com.auditoria.portalweb.modules.corporate.spi.dto.EmpresaSeoDTO;   // ✅
import com.auditoria.portalweb.modules.midia.domain.Media;                // ✅ (se @NamedInterface)
```

---

## 🏗️ **ARQUITETURA SPRING MODULITH**

### 📐 **Princípios Arquiteturais**

1. **Isolamento Modular**: Cada módulo é autossuficiente com suas próprias regras de negócio
2. **Comunicação via Contratos**: Módulos se comunicam apenas por interfaces públicas (`spi` ou `domain`)
3. **Fronteiras Validadas**: Testes automatizados garantem que não há violações de acesso
4. **Clean Architecture**: Separação clara entre camadas (web → service → repository)

### 🧩 **Anotações Spring Modulith**

#### **1. Módulo Básico (sem dependências)**

```java
// modules/midia/package-info.java
@org.springframework.modulith.ApplicationModule
package com.auditoria.portalweb.modules.midia;
```

#### **2. Módulo com Dependências**

```java
// modules/corporate/package-info.java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {
        "shared",                    // Módulo compartilhado
        "shared::mapper",            // Named interface do shared
        "shared::dto",               // Named interface do shared
        "modules.midia::domain"      // Named interface do módulo mídia
    }
)
package com.auditoria.portalweb.modules.corporate;
```

**Regras:**

- Nome completo do módulo: `modules.{nome}::interface`
- Spring Modulith deriva o nome do caminho do pacote
- Use `::` para referenciar Named Interfaces

#### **3. Expor Interface Pública (SPI)**

```java
// modules/corporate/spi/package-info.java
@org.springframework.modulith.NamedInterface("spi")
package com.auditoria.portalweb.modules.corporate.spi;

// modules/corporate/spi/dto/package-info.java
@org.springframework.modulith.NamedInterface
package com.auditoria.portalweb.modules.corporate.spi.dto;
```

#### **4. Expor Entidades de Domínio**

```java
// modules/midia/domain/package-info.java
@org.springframework.modulith.NamedInterface("domain")
package com.auditoria.portalweb.modules.midia.domain;
```

**Quando usar:**

- ✅ Para relacionamentos JPA cross-module (ex.: Empresa → Media)
- ✅ Com `EntityManager.getReference()` para respeitar fronteiras
- ❌ NÃO expor domain se houver alternativa via SPI/DTOs

#### **5. Módulo Aberto (Shared)**

```java
// shared/package-info.java
@org.springframework.modulith.ApplicationModule(
    type = org.springframework.modulith.ApplicationModule.Type.OPEN
)
package com.auditoria.portalweb.shared;

// shared/dto/package-info.java
@org.springframework.modulith.NamedInterface
package com.auditoria.portalweb.shared.dto;
```

**Tipo OPEN:**

- Permite acesso de qualquer módulo
- Use apenas para utilitários e contratos comuns
- NUNCA coloque lógica de negócio em `shared`

### 🔄 **Comunicação Entre Módulos**

#### **Exemplo: Layout consumindo Corporate**

```java
// 1. Corporate expõe interface pública
// modules/corporate/spi/EmpresaApi.java
public interface EmpresaApi {
    EmpresaSeoDTO obterSeoDaEmpresa();
}

// 2. Corporate implementa
// modules/corporate/internal/EmpresaServiceImpl.java
@Service
class EmpresaServiceImpl implements EmpresaApi {
    @Override
    public EmpresaSeoDTO obterSeoDaEmpresa() {
        // lógica privada usando repository e domain
        Empresa empresa = repository.findPrincipal().orElseThrow();
        return mapper.toSeoDto(empresa);
    }
}

// 3. Layout declara dependência
// modules/layout/package-info.java
@ApplicationModule(
    allowedDependencies = {"shared", "modules.corporate::spi"}
)
package com.auditoria.portalweb.modules.layout;

// 4. Layout consome
// modules/layout/service/LayoutServiceImpl.java
@Service
class LayoutServiceImpl {
    private final EmpresaApi empresaApi;  // ✅ Injeção via interface pública

    public HomePageDTO buildHomePage() {
        EmpresaSeoDTO seo = empresaApi.obterSeoDaEmpresa();
        // ... compor página
    }
}
```

---

## 🚀 **PASSO A PASSO: CRIAR NOVO MÓDULO**

### 📋 **Checklist de Criação**

#### **1. Criar Estrutura de Pastas**

```bash
mkdir -p src/main/java/com/auditoria/portalweb/modules/novomodulo/{domain,repository,service,internal,api/dto,api/mapper,web,spi/dto}
```

#### **2. Criar package-info.java Principal**

```java
// modules/novomodulo/package-info.java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {
        "shared",
        "shared::mapper",
        "shared::dto"
        // Adicionar outros módulos se necessário
    }
)
package com.auditoria.portalweb.modules.novomodulo;
```

#### **3. Criar Entidade JPA**

```java
// modules/novomodulo/domain/MinhaEntidade.java
@Entity
@Table(name = "minha_entidade")
public class MinhaEntidade {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nome;

    // getters, setters, equals, hashCode
}
```

#### **4. Criar Repository**

```java
// modules/novomodulo/repository/MinhaEntidadeRepository.java
public interface MinhaEntidadeRepository extends JpaRepository<MinhaEntidade, Long> {
    Optional<MinhaEntidade> findByNome(String nome);
}
```

#### **5. Criar DTOs**

```java
// modules/novomodulo/api/dto/MinhaEntidadeDTO.java
public record MinhaEntidadeDTO(Long id, String nome) { }

public record MinhaEntidadeCreateDTO(@NotBlank String nome) { }

public record MinhaEntidadeUpdateDTO(@NotBlank String nome) { }
```

#### **6. Criar Mapper**

```java
// modules/novomodulo/api/mapper/MinhaEntidadeMapper.java
@Mapper(config = MapStructConfig.class)
public interface MinhaEntidadeMapper {
    MinhaEntidadeDTO toDto(MinhaEntidade entity);
    MinhaEntidade toEntity(MinhaEntidadeCreateDTO dto);
    void updateEntity(MinhaEntidadeUpdateDTO dto, @MappingTarget MinhaEntidade entity);
}
```

#### **7. Criar Service**

```java
// modules/novomodulo/service/MinhaEntidadeService.java
public interface MinhaEntidadeService {
    MinhaEntidadeDTO findById(Long id);
    List<MinhaEntidadeDTO> findAll();
    MinhaEntidadeDTO create(MinhaEntidadeCreateDTO dto);
    MinhaEntidadeDTO update(Long id, MinhaEntidadeUpdateDTO dto);
    void delete(Long id);
}

// modules/novomodulo/internal/MinhaEntidadeServiceImpl.java
@Service
class MinhaEntidadeServiceImpl implements MinhaEntidadeService {
    private final MinhaEntidadeRepository repository;
    private final MinhaEntidadeMapper mapper;

    // Implementar métodos...
}
```

#### **8. Criar Controller**

```java
// modules/novomodulo/web/MinhaEntidadeController.java
@RestController
@RequestMapping("/api/minhas-entidades")
public class MinhaEntidadeController {
    private final MinhaEntidadeService service;

    @GetMapping
    public ResponseEntity<List<MinhaEntidadeDTO>> findAll() {
        return ResponseEntity.ok(service.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<MinhaEntidadeDTO> findById(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @PostMapping
    public ResponseEntity<MinhaEntidadeDTO> create(@Valid @RequestBody MinhaEntidadeCreateDTO dto) {
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(service.create(dto));
    }

    @PutMapping("/{id}")
    public ResponseEntity<MinhaEntidadeDTO> update(
        @PathVariable Long id,
        @Valid @RequestBody MinhaEntidadeUpdateDTO dto
    ) {
        return ResponseEntity.ok(service.update(id, dto));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

#### **9. (Opcional) Expor Interface Pública**

Se outros módulos precisarem consumir este módulo:

```java
// modules/novomodulo/spi/package-info.java
@org.springframework.modulith.NamedInterface("spi")
package com.auditoria.portalweb.modules.novomodulo.spi;

// modules/novomodulo/spi/NovoModuloApi.java
public interface NovoModuloApi {
    MinhaEntidadePublicDTO obterDados();
}

// modules/novomodulo/spi/dto/package-info.java
@org.springframework.modulith.NamedInterface
package com.auditoria.portalweb.modules.novomodulo.spi.dto;

// modules/novomodulo/spi/dto/MinhaEntidadePublicDTO.java
public record MinhaEntidadePublicDTO(Long id, String nome) { }
```

#### **10. Executar Testes de Arquitetura**

```bash
# Validar que o módulo respeita fronteiras
mvn test -Dtest=ModulithArchitectureTests

# Se passar, tudo certo! ✅
```

### ✅ **Checklist Final**

- [ ] `package-info.java` criado no módulo raiz
- [ ] Entidades JPA em `domain/`
- [ ] Repository em `repository/`
- [ ] Service + Impl em `service/` e `internal/`
- [ ] DTOs em `api/dto/`
- [ ] Mapper em `api/mapper/` usando `MapStructConfig`
- [ ] Controller em `web/` retornando apenas DTOs
- [ ] (Se necessário) SPI em `spi/` com `@NamedInterface`
- [ ] Testes de arquitetura passando
- [ ] Documentação do módulo em `/docs/Módulos/`

---

## 🛠️ **STACK TECNOLÓGICA**

### 📦 **Versões Principais**

| Componente | Versão | Observações |
|------------|--------|-------------|
| **Java** | 21 LTS | Recursos modernos (records, pattern matching, etc.) |
| **Spring Boot** | 3.5.7 | Framework principal |
| **Spring Modulith** | 1.3.2 | Arquitetura modular validada |
| **MariaDB** | 12.0.2+ | Collation: utf8mb4_uca1400_ai_ci |
| **MapStruct** | 1.6.3 | Mapeamento DTO ↔ Entity |
| **JJWT** | 0.12.6 | JWT moderno (sem SignatureAlgorithm) |
| **Maven** | 3.8+ | Build via Maven Wrapper |

### 🔧 **Build e Qualidade**

- **Spotless** - Formatação de código automática
- **Checkstyle** - Verificação de estilo (config/quality/checkstyle.xml)
- **PMD** - Análise estática (config/quality/pmd-ruleset.xml)
- **SpotBugs** - Detecção de bugs (config/quality/spotbugs-exclude.xml)
- **JaCoCo** - Cobertura de testes

### 🌐 **Perfis de Execução**

| Profile | Porta | DDL Mode | Database | Uso |
|---------|-------|----------|----------|-----|
| **dev** | 8080 | `update` | MariaDB | Desenvolvimento local |
| **test** | 8080 | `create-drop` | H2 (memória) | Testes automatizados |
| **prod** | 8080 | `validate` | MariaDB | Produção |

**Ativação:**

```bash
# Via variável de ambiente (RECOMENDADO)
export SPRING_PROFILES_ACTIVE=dev
mvn spring-boot:run

# Via linha de comando
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

### 🗄️ **Estratégia de Banco de Dados**

**❌ NÃO utilizamos:**

- Flyway
- Liquibase
- Migrations automáticas

**✅ Utilizamos:**

- **DEV**: Scripts SQL manuais idempotentes (`schema.sql`, `data.sql`)
  - `spring.jpa.hibernate.ddl-auto=update` (aceitável apenas em DEV)
- **PROD**: Scripts SQL revisados e controlados
  - `spring.jpa.hibernate.ddl-auto=validate` (apenas validação)
- **TEST**: H2 in-memory com modo MySQL
  - `spring.jpa.hibernate.ddl-auto=create-drop`

---

## 🎯 **HISTÓRICO DE ALTERAÇÕES**

## 🎯 **29/10/2025 - Atualização Spring Boot 3.5.7 + Modernização de APIs**

### ✅ **Implementado**

#### **🔧 Correções e Modernizações**

**1. Spring Boot - Atualização de Versão:**

```xml
<!-- pom.xml -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.5.7</version>  <!-- ✅ Atualizado de 3.5.6 -->
</parent>
```

**2. Spring Beans - Tipos Concretos (AOT/Native Compatibility):**

```java
// ✅ UsersConfig.java - BCryptPasswordEncoder
@Bean
public BCryptPasswordEncoder passwordEncoder() {  // Antes: PasswordEncoder
    return new BCryptPasswordEncoder();
}

// ✅ AuthSecurityConfig.java - DefaultSecurityFilterChain
@Bean
DefaultSecurityFilterChain authFilterChain(HttpSecurity http) throws Exception {  // Antes: SecurityFilterChain
    // ...
    return http.build();
}

// ✅ CorsConfig.java - UrlBasedCorsConfigurationSource
@Bean
public UrlBasedCorsConfigurationSource corsConfigurationSource() {  // Antes: CorsConfigurationSource
    // ...
    return source;
}
```

**Motivo:** Compatibilidade com Spring AOT e GraalVM Native Image.

**3. JJWT - API Moderna (0.12.x):**

```java
// ✅ JwtTokenProvider.java
// REMOVIDO: import io.jsonwebtoken.SignatureAlgorithm;

public String newAccessToken(Long userId, String email, String role) {
    return Jwts.builder()
        // ...
        .signWith(key)  // ✅ Algoritmo inferido automaticamente (antes: signWith(key, SignatureAlgorithm.HS256))
        .compact();
}
```

**4. Spring Data JPA - API Moderna (3.5+):**

```java
// ✅ AdminAuditController.java
Specification<AuditEvent> spec = Specification.allOf(  // ✅ Antes: Specification.where()
    AuditSpecs.pathLike(path),
    AuditSpecs.actorEmailLike(actorEmail),
    AuditSpecs.statusEq(status),
    AuditSpecs.createdBetween(from, to)
);
```

**5. Java 16+ - Stream API Moderna:**

```java
// ✅ CorsConfig.java
List<String> origins = Arrays.stream(allowedOrigins.split(","))
    .map(String::trim)
    .filter(s -> !s.isEmpty())
    .toList();  // ✅ Antes: .collect(Collectors.toList())
```

**6. Testes - Configuração JWT:**

```java
// ✅ PortalwebApplicationTests.java
@TestPropertySource(properties = {
    // H2 Database
    "spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE",
    "spring.datasource.driver-class-name=org.h2.Driver",

    // ✅ ADICIONADO: JWT Configuration (mínimo 32 caracteres para HS256)
    "app.auth.jwt.secret=TEST_SECRET_KEY_FOR_TESTING_PURPOSES_MIN_32_CHARS_REQUIRED",
    "app.auth.jwt.issuer=portalweb-test",
    "app.auth.jwt.access-ttl-sec=900",
    "app.auth.jwt.refresh-ttl-sec=1209600"
})
```

**Problema resolvido:** `Could not resolve placeholder 'app.auth.jwt.secret'` ao executar testes.

**7. VS Code - Configuração de Indexação:**

```json
// ✅ .vscode/settings.json
{
  "files.watcherExclude": {
    "**/target/**": true  // ✅ Evita indexar arquivos gerados (MapStruct)
  },
  "files.exclude": {
    "**/target": true
  }
}
```

**Problema resolvido:** ~100 erros falsos de MapStruct no VS Code.

**8. Markdown - Linting:**

```markdown
<!-- ✅ README_DEV.md - Removidas linhas em branco duplas -->
```

#### **📊 Impacto Técnico:**

- **Modernização**: APIs atualizadas para versões mais recentes
- **Compatibilidade**: Preparado para AOT/Native compilation
- **Manutenibilidade**: Código mais limpo e idiomático
- **Testes**: 100% passando (4/4)
- **Documentação**: Gerada automaticamente em `target/spring-modulith-docs/`

#### **📁 Arquivos Alterados:**

1. `pom.xml` - Spring Boot 3.5.6 → 3.5.7
2. `src/main/java/.../modules/users/internal/config/UsersConfig.java`
3. `src/main/java/.../modules/auth/internal/security/AuthSecurityConfig.java`
4. `src/main/java/.../modules/auth/internal/JwtTokenProvider.java`
5. `src/main/java/.../config/CorsConfig.java`
6. `src/main/java/.../modules/audit/web/admin/AdminAuditController.java`
7. `src/test/java/.../PortalwebApplicationTests.java`
8. `.vscode/settings.json`
9. `docs/Dev/README_DEV.md`

#### **✅ Resultados dos Testes:**

```text
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0

✅ ModulithArchitectureTests (3 testes)
   - verifyModularity
   - documentModules
   - verifyArchitecture

✅ PortalwebApplicationTests (1 teste)
   - contextLoads
```

#### **📄 Documentação Gerada:**

Spring Modulith gerou automaticamente:

- `target/spring-modulith-docs/components.puml` - Diagrama C4
- `target/spring-modulith-docs/module-*.adoc` - Documentação de cada módulo
- `target/spring-modulith-docs/module-*.puml` - Diagramas PlantUML

#### **🔧 Comandos Úteis:**

```bash
# Executar testes
mvn clean test

# Gerar documentação Modulith
mvn clean verify

# Executar aplicação
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Validar código (Spotless)
mvn spotless:check
mvn spotless:apply
```

#### **✅ Status Final:**

- **Spring Boot**: 3.5.7 ✅
- **Testes**: 100% passando ✅
- **APIs**: Modernizadas ✅
- **Documentação**: Gerada ✅
- **Warnings**: Todos corrigidos ✅

---

## 🎯 **28/10/2025 - Correção de Dependências Modulares (Mídia e Corporate)**

### ✅ **Implementado - Módulos**

#### **🔧 Correções de Arquitetura Modular**

**1. Módulo Mídia - Remoção de Dependência Invertida:**

```java
// ANTES (incorreto):
@ApplicationModule(allowedDependencies = {"content"})

// DEPOIS (correto):
@ApplicationModule // Sem dependências - módulo autossuficiente
```

**Problema corrigido:**

- ❌ **Dependência invertida**: Mídia não deve depender de Content
- ❌ **Dependência fantasma**: Código não usa nada do módulo Content
- ✅ **Arquitetura correta**: Módulo de infraestrutura autossuficiente

**2. Módulo Corporate - Validação de Dependências:**

```java
// Mantido (correto):
@ApplicationModule(allowedDependencies = {
    "shared", "shared::mapper", "shared::dto", "modules.midia::domain"
})
```

**Validação:**

- ✅ **Nome lógico completo**: `modules.midia::domain` (como detectado pelo Spring Modulith)
- ✅ **Derivação automática**: Spring Modulith usa o caminho completo do pacote como nome lógico
- ✅ **Fronteiras respeitadas**: Acesso apenas à interface `domain` do módulo Mídia

#### **📊 Impacto Técnico - Módulos:**

- **Arquitetura**: Dependências corretas e unidirecionais
- **Manutenibilidade**: Nomenclatura padronizada e consistente
- **Qualidade**: Sem dependências não utilizadas (código limpo)
- **Validação**: Testes de arquitetura garantem integridade

#### **📁 Arquivos Alterados - Dependências:**

1. `src/main/java/.../modules/midia/package-info.java`
2. `src/main/java/.../modules/corporate/package-info.java`
3. `docs/Módulos/Modulo_midia/Projeto_tecnico_midia.md`
4. `GUIA_DESENVOLVIMENTO.md`

#### **✅ Status Final - Dependências:**

- **Correções aplicadas**: 100% concluídas
- **Documentação**: Atualizada e sincronizada
- **Testes**: Pendente de validação via `ModulithArchitectureTests`

---

## 🎯 **28/10/2025 - Exposição do Domínio Mídia via Spring Modulith**

### ✅ Implementado - Named Interfaces

#### **🧩 Arquitetura Spring Modulith**

- **Exposição de Named Interfaces** no módulo Mídia
- **Configuração de dependências** entre módulos  
- **Validação de arquitetura** via testes automatizados

#### **📁 Arquivos Criados/Modificados:**

**1. Módulo Mídia - Configuração Principal:**

```java
// src/main/java/com/auditoria/portalweb/modules/midia/package-info.java
@ApplicationModule(allowedDependencies = {"content"})
```

**2. Domínio Exposto:**

```java
// src/main/java/com/auditoria/portalweb/modules/midia/domain/package-info.java
@NamedInterface("domain") 
// Expõe: Media, MediaKind, MediaStorage
```

**3. API Preparada:**

```java
// src/main/java/com/auditoria/portalweb/modules/midia/api/package-info.java
@NamedInterface("api")
// Para DTOs e mappers futuros
```

**4. Módulo Corporate - Consumidor:**

```java
// src/main/java/com/auditoria/portalweb/modules/corporate/package-info.java
@ApplicationModule(allowedDependencies = {
    "shared", "shared::mapper", "shared::dto", "modules.midia::domain"
})
```

**5. Correção de Violação Modular:**

```java
// src/main/java/.../corporate/service/EmpresaService.java
// ❌ Removido: MediaRepository (violação modular)
// ✅ Adicionado: EntityManager + em.getReference(Media.class, logoMediaId)
```

**6. Testes de Arquitetura:**

```java  
// src/test/java/.../ModulithArchitectureTests.java
// ✅ Corrigido: import PortalwebApplication
// ✅ Status: Todos os testes passando (3/3)
```

#### **🎯 Resultados Obtidos:**

- **✅ Spring Modulith** detectando módulos corretamente
- **✅ Interface domain** expondo entidades Media para Corporate
- **✅ Fronteiras modulares** respeitadas (sem acessos indevidos)  
- **✅ Relacionamentos JPA** funcionando via proxy EntityManager
- **✅ Testes de arquitetura** validando dependências (0 erros)

#### 📊 Impacto Técnico

- **Modularidade**: Módulos bem definidos com interfaces claras
- **Manutenibilidade**: Dependências explícitas e validadas
- **Escalabilidade**: Base para futuras integrações modulares
- **Qualidade**: Testes automatizados garantindo arquitetura

#### **🔧 Comandos de Validação:**

```bash
# Executar testes de arquitetura
./mvnw test -Dtest="*ArchitectureTests*"

# Gerar documentação dos módulos  
# Executado automaticamente nos testes
# Saída: target/modulith-docs/
```

✅ Status Final:

- **Implementação:** 100% concluída
- **Testes:** Todos passando (3/3)  
- **Documentação:** Atualizada e sincronizada
- **Arquitetura:** Validada pelo Spring Modulith

---

---

> Este documento consolida o histórico recente e mantém apenas pendências **ativas** e resoluções com impacto técnico.
