# Estrutura Backend - Portal Auditoria

**Versao**: 1.0.0
**Framework**: Spring Boot 3.5.7
**Java**: 21
**Arquitetura**: Spring Modulith 1.3.2
**Banco**: MariaDB 12.0.2
**Total de Arquivos**: 196 classes Java

---

## Visao Geral

Backend da plataforma Portal Auditoria, construido como **monolito modular** usando Spring Modulith. Arquitetura API-first com suporte multi-tenant (empresaId).

**Caracteristicas principais**:
- **8 modulos Spring Modulith** (audit, auth, content, corporate, layout, midia, users, webhooks)
- Multi-tenancy nativo
- Modulos desacoplados com dependencias declaradas
- REST API com OpenAPI/Swagger
- Event-driven architecture
- Testes de integracao

**Estatisticas**:
- 176 arquivos em modulos
- 13 arquivos shared (utilitarios)
- 5 arquivos config
- 196 arquivos total

---

## Stack Tecnologico

### Core
- **Java 21** (LTS)
- **Spring Boot 3.5.7**
- **Spring Modulith 1.3.2** - Modular monolith
- **Maven** - Build e dependencias

### Persistencia
- **Spring Data JPA** - ORM com repositories
- **MariaDB 12.0.2** - Banco de dados
- **Hibernate** - JPA provider
- **JPA Specifications** - Queries dinamicas

### Seguranca
- **JWT (JSON Web Tokens)** - Autenticacao stateless
- **HMAC-SHA256** - Webhook validation
- **Password Hashing** - BCrypt
- **Token Revocation** - Blacklist de tokens

### Mapeamento
- **MapStruct 1.5.x** - DTO <-> Entity mapping (code generation)
- **Lombok** - Boilerplate reduction

### Utilitarios
- **Jackson** - JSON serialization
- **SLF4J + Logback** - Logging
- **Slugify** - URL-friendly slugs
- **AspectJ** - AOP para auditoria

### Testes
- **JUnit 5** - Framework de testes
- **Spring Boot Test** - Integration tests
- **H2** - In-memory database (testes)
- **MockWebServer (OkHttp3)** - HTTP mocking
- **Mockito** - Mocking framework

### Documentacao
- **OpenAPI 3.0** - Documentacao de API
- **Springdoc** - Geracao automatica

### Storage
- **Local Filesystem** - Armazenamento local
- **AWS S3** - (suportado, nao configurado)
- **Google Cloud Storage** - (suportado, nao configurado)

---

## Arquitetura Spring Modulith

### Conceito

Spring Modulith permite organizar monolitos em modulos logicos com boundaries claros:

```
backend/src/main/java/com/auditoria/portalweb/
├── modules/                    (8 modulos)
│   ├── audit/                  (10 arquivos)
│   ├── auth/                   (25 arquivos)
│   ├── content/                (31 arquivos)
│   ├── corporate/              (33 arquivos)
│   ├── layout/                 (5 arquivos)
│   ├── midia/                  (17 arquivos)
│   ├── users/                  (16 arquivos)
│   └── webhooks/               (39 arquivos)
├── shared/                     (13 arquivos compartilhados)
├── config/                     (5 configuracoes globais)
└── web/                        (1 controller publico)
```

### Beneficios

1. **Isolamento**: Modulos nao acessam internals de outros modulos
2. **Dependencias Declaradas**: package-info.java define o que pode ser acessado
3. **Events**: Comunicacao assincrona via Spring Events
4. **Documentacao**: Geracao automatica de diagramas de modulos
5. **Testes**: Testes de modulos isolados
6. **Migracao**: Facil extracao para microservicos

### Package Convention

```
com.auditoria.portalweb.modules.<modulo>/
├── api/                 (DTOs, mappers, interfaces publicas)
├── domain/              (entities JPA - publico)
├── repository/          (JPA repos - interno)
├── internal/            (services, handlers - interno/privado)
├── web/                 (controllers - publico)
│   └── admin/           (endpoints admin)
├── spi/                 (Service Provider Interface - publico)
└── package-info.java    (@ApplicationModule com dependencias)
```

**Regras**:
- `internal/` - Nao pode ser acessado de fora do modulo
- `api/`, `domain/`, `web/` - API publica do modulo
- `spi/` - Interface publica para integracao entre modulos
- Comunicacao inter-modulos via eventos ou SPI

---

## Modulos Implementados (8)

### 1. AUDIT - Auditoria e Rastreamento (10 arquivos)

**Package**: `com.auditoria.portalweb.modules.audit`
**Status**: Producao
**Dependencias**: users, auth::api, content, corporate, midia

**Funcionalidades**:
- Rastreamento de todas as operacoes HTTP
- Auditoria de mudancas em entidades (via AOP)
- Registro de acoes administrativas
- Queries para consulta de historico

**Estrutura**:
```
audit/
├── api/
│   └── Audited.java                  (anotacao para metodos auditados)
├── domain/
│   └── AuditEvent.java               (entidade de auditoria)
├── internal/
│   ├── aop/
│   │   └── AuditAspect.java          (AspectJ interceptor)
│   ├── spec/
│   │   └── AuditSpecs.java           (JPA specifications)
│   ├── web/
│   │   ├── AuditWebMvcConfig.java
│   │   └── HttpAuditInterceptor.java (interceptor HTTP)
│   └── AuditService.java
├── repository/
│   └── AuditEventRepository.java
└── web/admin/
    └── AdminAuditController.java     (GET /admin/audit)
```

**Endpoints**:
- `GET /api/v1/admin/audit` - Listar eventos de auditoria

---

### 2. AUTH - Autenticacao e Autorizacao (25 arquivos)

**Package**: `com.auditoria.portalweb.modules.auth`
**Status**: Producao
**Dependencias**: users::api

**Funcionalidades**:
- Autenticacao JWT (login, refresh, logout)
- Recuperacao de senha via email
- Revogacao de tokens (blacklist)
- Gerenciamento de sessoes

**Estrutura**:
```
auth/
├── api/
│   └── dto/
│       ├── AuthMeResponse.java
│       ├── LoginRequest.java
│       ├── LoginResponse.java
│       ├── RefreshRequest.java
│       ├── ResetPasswordRequest.java
│       ├── RevokeRequest.java
│       └── ForgotPasswordRequest.java
├── domain/
│   ├── PasswordResetToken.java       (tokens de recuperacao)
│   └── TokenRevogado.java            (tokens revogados)
├── internal/
│   ├── security/
│   │   ├── JwtAuthenticationFilter.java
│   │   ├── AuthSecurityConfig.java   (Spring Security config)
│   │   ├── UserPrincipal.java
│   │   └── AuthenticatedUser.java
│   ├── JwtTokenProvider.java         (geracao/validacao JWT)
│   ├── AuthService.java
│   ├── EmailService.java
│   └── PasswordRecoveryService.java
├── repository/
│   ├── PasswordResetTokenRepository.java
│   └── TokenRevogadoRepository.java
└── web/
    ├── AuthController.java           (POST /auth/login, /auth/refresh)
    ├── PasswordRecoveryController.java
    └── AuthExceptionHandler.java
```

**Endpoints**:
- `POST /api/v1/auth/login` - Login (retorna JWT)
- `POST /api/v1/auth/refresh` - Refresh token
- `POST /api/v1/auth/logout` - Revoga token
- `GET /api/v1/auth/me` - Dados do usuario autenticado
- `POST /api/v1/auth/forgot-password` - Solicita recuperacao
- `POST /api/v1/auth/reset-password` - Reseta senha

**Seguranca**:
- JWT com expiracao configuravel
- Refresh tokens
- Revogacao imediata de tokens
- Hashing de senhas com BCrypt

---

### 3. CONTENT - Conteudo e Publicacoes (31 arquivos)

**Package**: `com.auditoria.portalweb.modules.content`
**Status**: Producao
**Dependencias**: shared, corporate::spi, corporate::spi.dto

**Funcionalidades**:
- Gerenciamento de posts/artigos
- Gerenciamento de servicos oferecidos
- Upload e vinculacao de midias
- Geracao automatica de slugs

**Estrutura**:
```
content/
├── api/
│   ├── dto/
│   │   ├── PostCreateDTO.java
│   │   ├── PostDTO.java
│   │   ├── PostUpdateDTO.java
│   │   ├── ServicoCreateDTO.java
│   │   ├── ServicoDTO.java
│   │   └── ServicoUpdateDTO.java
│   └── mapper/
│       ├── PostMapper.java           (MapStruct)
│       └── ServicoMapper.java        (MapStruct)
├── domain/
│   ├── Post.java                     (posts/artigos)
│   ├── Servico.java                  (servicos oferecidos)
│   └── Media.java                    (referencia de midia)
├── internal/
│   ├── impl/
│   │   ├── PostServiceImpl.java
│   │   ├── ServicoServiceImpl.java
│   │   └── MediaServiceImpl.java
│   ├── support/
│   │   └── SlugUtil.java             (geracao de slugs)
│   ├── PostService.java
│   ├── ServicoService.java
│   └── MediaService.java
├── repository/
│   ├── PostRepository.java
│   ├── ServicoRepository.java
│   ├── MediaRepository.java
│   └── MediaLiteRepository.java
└── web/
    ├── admin/
    │   ├── AdminPostController.java
    │   └── AdminServicoController.java
    ├── PostController.java           (GET /posts)
    ├── ServicoController.java        (GET /servicos)
    ├── MediaController.java
    └── ApiExceptionHandler.java
```

**Endpoints**:
- `GET /api/v1/posts` - Listar posts
- `GET /api/v1/posts/{slug}` - Post por slug
- `POST /api/v1/admin/posts` - Criar post
- `PUT /api/v1/admin/posts/{id}` - Atualizar post
- `GET /api/v1/servicos` - Listar servicos
- `POST /api/v1/admin/servicos` - Criar servico

---

### 4. CORPORATE - Dados Corporativos (33 arquivos)

**Package**: `com.auditoria.portalweb.modules.corporate`
**Status**: Producao
**Dependencias**: shared, shared::mapper, shared::dto, midia::domain

**Funcionalidades**:
- Gerenciamento de dados de empresas (CNPJ, enderecos, contatos)
- Estrutura societaria (participacoes)
- Cadastro de pessoas (fisicas/juridicas)
- Expoe SPI publica para outros modulos

**Estrutura**:
```
corporate/
├── api/
│   ├── dto/
│   │   ├── EmpresaCreateDTO.java
│   │   ├── EmpresaDTO.java
│   │   ├── EmpresaResumoDTO.java
│   │   ├── EmpresaSeoDTO.java
│   │   ├── EmpresaUpdateDTO.java
│   │   ├── ParticipacaoCreateDTO/DTO/UpdateDTO.java
│   │   └── PessoaCreateDTO/DTO/UpdateDTO.java
│   ├── mapper/
│   │   ├── EmpresaMapper.java        (MapStruct)
│   │   ├── ParticipacaoMapper.java   (MapStruct)
│   │   └── PessoaMapper.java         (MapStruct)
│   └── EmpresaApi.java               (interface publica)
├── domain/
│   ├── Empresa.java                  (CNPJ, endereco, contatos, redes sociais)
│   ├── ParticipacaoSocietaria.java
│   └── Pessoa.java
├── internal/
│   └── EmpresaServiceImpl.java
├── repository/
│   ├── EmpresaRepository.java
│   ├── ParticipacaoSocietariaRepository.java
│   └── PessoaRepository.java
├── service/
│   ├── EmpresaService.java
│   ├── ParticipacaoSocietariaService.java
│   └── PessoaService.java
├── spi/                              (Service Provider Interface)
│   ├── dto/
│   │   └── EmpresaSeoDTO.java
│   └── EmpresaApi.java               (usado por outros modulos)
└── web/
    ├── EmpresaController.java
    ├── ParticipacaoSocietariaController.java
    └── PessoaController.java
```

**Endpoints**:
- `GET /api/v1/empresas` - Listar empresas
- `POST /api/v1/admin/empresas` - Criar empresa
- `GET /api/v1/participacoes` - Listar participacoes
- `GET /api/v1/pessoas` - Listar pessoas

**Campos de Empresa**:
- Dados basicos: CNPJ, razao social, nome fantasia
- Endereco: CEP, logradouro, numero, complemento, bairro, cidade, UF
- Contatos: telefone, email, site
- Redes sociais: Facebook, Instagram, LinkedIn, Twitter
- SEO: missao, visao, valores, descricao

---

### 5. LAYOUT - Estrutura de Paginas (5 arquivos)

**Package**: `com.auditoria.portalweb.modules.layout`
**Status**: Producao
**Dependencias**: shared, corporate::spi, corporate::spi.dto

**Funcionalidades**:
- Estrutura de paginas publicas (home, about, etc)
- Integracao com dados corporativos
- Renderizacao de layout

**Estrutura**:
```
layout/
├── api/dto/
│   └── HomePageDTO.java              (estrutura da home)
├── service/
│   ├── LayoutService.java            (interface)
│   └── LayoutServiceImpl.java
└── web/
    └── LayoutController.java         (GET /layout/home)
```

**Endpoints**:
- `GET /api/v1/layout/home` - Dados da pagina home

---

### 6. MIDIA - Gerenciamento de Arquivos (17 arquivos)

**Package**: `com.auditoria.portalweb.modules.midia`
**Status**: Producao
**Dependencias**: Nenhuma (modulo autossuficiente)

**Funcionalidades**:
- Upload de arquivos (imagens, videos, documentos)
- Deduplicacao via SHA-256
- Suporte a multiplos provedores (Local, S3, GCS)
- Extracao de metadados (dimensoes, MIME, cor dominante)
- Geracao automatica de URLs publicas

**Estrutura**:
```
midia/
├── api/
│   ├── dto/
│   │   ├── MediaDTO.java
│   │   └── UploadResponseDTO.java
│   └── mapper/
│       └── MediaMapper.java          (MapStruct)
├── domain/
│   ├── Media.java                    (URL, MIME, dimensoes, SHA-256)
│   ├── MediaKind.java                (enum: IMAGE, VIDEO, DOCUMENT)
│   └── MediaStorage.java             (enum: LOCAL, S3, GCS)
├── internal/
│   ├── impl/
│   │   ├── LocalStorageService.java  (armazenamento local)
│   │   └── MediaAdminServiceImpl.java
│   ├── support/
│   │   └── FileNameUtil.java
│   └── MediaStorageService.java      (interface)
├── repository/
│   └── MediaRepository.java
└── web/
    ├── admin/
    │   └── AdminMediaController.java (POST /admin/media/upload)
    ├── public/
    │   └── MediaPublicController.java (GET /media/{sha256})
    └── MediaExceptionHandler.java
```

**Endpoints**:
- `POST /api/v1/admin/media/upload` - Upload de arquivo
- `GET /api/v1/admin/media` - Listar midias
- `GET /api/v1/media/{sha256}` - Servir arquivo publico
- `DELETE /api/v1/admin/media/{id}` - Deletar midia

**Deduplicacao**:
- SHA-256 do conteudo calculado no upload
- Arquivos identicos apontam para o mesmo storage
- Economia de espaco e banda

---

### 7. USERS - Gerenciamento de Usuarios (16 arquivos)

**Package**: `com.auditoria.portalweb.modules.users`
**Status**: Producao
**Dependencias**: corporate::domain, midia::api

**Funcionalidades**:
- CRUD de usuarios
- Roles e permissoes
- Integracao com empresas
- Avatar via modulo midia

**Estrutura**:
```
users/
├── api/
│   ├── dto/
│   │   ├── UsuarioCreateDTO.java
│   │   ├── UsuarioUpdateDTO.java
│   │   ├── UsuarioDTO.java
│   │   └── UserAuthDTO.java
│   ├── mapper/
│   │   └── UsuarioMapper.java        (MapStruct)
│   └── UserPublicService.java        (interface publica)
├── domain/
│   └── Usuario.java                  (usuarios do sistema)
├── internal/
│   ├── config/
│   │   └── UsersConfig.java
│   ├── UserPublicServiceImpl.java
│   └── UsuarioService.java
├── repository/
│   └── UsuarioRepository.java
└── web/
    ├── admin/
    │   └── AdminUsuarioController.java
    ├── PublicUsuarioController.java
    └── UsersExceptionHandler.java
```

**Endpoints**:
- `GET /api/v1/admin/usuarios` - Listar usuarios
- `POST /api/v1/admin/usuarios` - Criar usuario
- `PUT /api/v1/admin/usuarios/{id}` - Atualizar usuario
- `DELETE /api/v1/admin/usuarios/{id}` - Deletar usuario

---

### 8. WEBHOOKS - Webhooks Entrantes e Saintes (39 arquivos)

**Package**: `com.auditoria.portalweb.modules.webhooks`
**Status**: Producao (92% completo)
**Dependencias**: Nenhuma

**Funcionalidades**:
- Recepcao de webhooks (Stripe, Asaas)
- Envio de webhooks para clientes
- HMAC-SHA256 validation (incoming/outgoing)
- Rate limiting (60 req/min)
- Retry com backoff exponencial
- Event-driven architecture
- Multi-tenancy support

**Estrutura completa**: Ver [docs/Modulos/webhooks/](./Modulos/webhooks/)

**Endpoints**:
- `POST /api/v1/webhooks/receive/{source}` - Receber webhook (publico)
- `GET /api/v1/admin/webhooks/subscriptions` - Listar subscricoes
- `POST /api/v1/admin/webhooks/subscriptions` - Criar subscricao
- `GET /api/v1/admin/webhooks/deliveries` - Listar entregas
- `POST /api/v1/admin/webhooks/deliveries/{id}/retry` - Retry manual

**Documentacao**: [Modulo Webhooks](./Modulos/webhooks/bakend-modulo_webhooks.md)

---

## Shared - Codigo Compartilhado (13 arquivos)

**Package**: `com.auditoria.portalweb.shared`

### Estrutura

```
shared/
├── dto/
│   ├── IdNameDTO.java                (DTO generico ID+Nome)
│   └── PageResponse.java             (resposta paginada)
├── exception/
│   └── ApiError.java                 (formato de erro padronizado)
├── mapper/
│   └── MapStructConfig.java          (config global MapStruct)
├── util/
│   ├── DateUtils.java                (manipulacao de datas)
│   ├── JsonUtils.java                (operacoes JSON)
│   └── Slugify.java                  (geracao de slugs)
└── validation/
    ├── Slug.java                     (anotacao de validacao)
    └── SlugValidator.java            (validador customizado)
```

**Uso**: Todos os modulos podem importar classes de `shared`

---

## Config - Configuracoes Globais (5 arquivos)

**Package**: `com.auditoria.portalweb.config`

### Arquivos

**CorsConfig.java**
- Configuracao CORS para acesso cross-origin
- Permite frontend consumir API

**GlobalExceptionHandler.java**
- Tratamento centralizado de exceções
- Retorna ApiError padronizado

**OpenApiConfig.java**
- Configuracao Swagger/OpenAPI
- Documentacao automatica da API

**SchedulingConfig.java**
- Habilita `@Scheduled` tasks
- Configuracao de thread pool

---

## Diagrama de Dependencias Entre Modulos

```
                    SHARED (13 arquivos)
            DTOs, Excecoes, Utils, Validacoes
                           |
        +------------------+------------------+
        |                  |                  |
    CONFIG            MIDIA (17)         CORPORATE (33)
    (5 arq)           Infra              CNPJ, Pessoas
        |                  |                  |
        |                  |                  |
        +--+-------+-------+-------+----------+
           |       |       |       |          |
       CONTENT  LAYOUT  USERS   AUTH      AUDIT
       (31 arq) (5 arq) (16 arq) (25 arq) (10 arq)
        Posts    Pages   Account  JWT      Logs
           |       |       |       |          |
           +-------+-------+-------+----------+
                           |
                      WEBHOOKS (39)
                      Integrações
```

**Observacoes**:
- `SHARED` - Base para todos
- `MIDIA` - Independente, usado por varios modulos
- `CORPORATE` - Nucleo de dados (SPI publica)
- `WEBHOOKS` - Independente, nao depende de nenhum modulo

---

## Banco de Dados

### Configuracao (MariaDB)

```properties
spring.datasource.url=jdbc:mariadb://localhost:3306/portal_auditoria
spring.datasource.username=root
spring.datasource.password=senha
spring.jpa.hibernate.ddl-auto=update  # Producao: validate
spring.jpa.show-sql=false             # Producao: false
```

### Tabelas por Modulo

**AUDIT**:
- `audit_event` - Eventos de auditoria

**AUTH**:
- `password_reset_token` - Tokens de recuperacao de senha
- `token_revogado` - Blacklist de tokens JWT

**CONTENT**:
- `post` - Posts/artigos
- `servico` - Servicos oferecidos
- `media` - Referencia de midias

**CORPORATE**:
- `empresa` - Dados de empresas
- `participacao_societaria` - Estrutura societaria
- `pessoa` - Pessoas fisicas/juridicas

**MIDIA**:
- `media` - Metadados de arquivos

**USERS**:
- `usuario` - Usuarios do sistema

**WEBHOOKS**:
- `webhook_received` - Webhooks recebidos
- `webhook_subscription` - Subscricoes
- `webhook_delivery` - Entregas
- `webhook_hmac_failure_log` - Falhas de validacao

**Total**: ~15 tabelas

### Convencoes

**Nomenclatura**: snake_case
```sql
webhook_received
participacao_societaria
password_reset_token
```

**Colunas comuns**:
```sql
id BIGINT AUTO_INCREMENT PRIMARY KEY
empresa_id INT                          # Multi-tenancy
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

**Collation**: `utf8mb4_uca1400_ai_ci`

---

## Padroes de Codigo

### 1. Nomenclatura

**Entities**: Substantivos singulares
```java
Empresa, Usuario, Post, Servico
```

**Repositories**: Entity + Repository
```java
EmpresaRepository extends JpaRepository<Empresa, Long>
```

**Services**: Entity + Service
```java
EmpresaService, UsuarioService
```

**Controllers**: Entity + Controller
```java
EmpresaController, PostController
```

**DTOs**: Entity + Create/Update/Response + DTO
```java
EmpresaCreateDTO, EmpresaDTO, PostUpdateDTO
```

**Mappers**: Entity + Mapper (MapStruct)
```java
@Mapper(componentModel = "spring")
public interface EmpresaMapper {
  EmpresaDTO toDTO(Empresa entity);
  Empresa toEntity(EmpresaCreateDTO dto);
}
```

### 2. Anotacoes Lombok

**Entities**:
```java
@Entity
@Table(name = "empresa")
@Getter @Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
```

**Services**:
```java
@Service
@RequiredArgsConstructor  // Injeta final fields
@Slf4j                    // Logger
```

**Controllers**:
```java
@RestController
@RequestMapping("/api/v1/empresas")
@RequiredArgsConstructor
@Slf4j
```

### 3. MapStruct

**Configuracao global**: `shared/mapper/MapStructConfig.java`

```java
@Mapper(componentModel = "spring", config = MapStructConfig.class)
public interface EmpresaMapper {
  EmpresaDTO toDTO(Empresa entity);
  List<EmpresaDTO> toDTOList(List<Empresa> entities);
  Empresa toEntity(EmpresaCreateDTO dto);
  void updateEntityFromDTO(EmpresaUpdateDTO dto, @MappingTarget Empresa entity);
}
```

### 4. Exception Handling

**Global Handler**: `config/GlobalExceptionHandler.java`

**Modulo Handler**: `<modulo>/web/<Modulo>ExceptionHandler.java`

```java
@RestControllerAdvice
public class ContentExceptionHandler {

  @ExceptionHandler(PostNotFoundException.class)
  public ResponseEntity<ApiError> handleNotFound(Exception e) {
    return ResponseEntity.status(404).body(
      new ApiError(404, e.getMessage())
    );
  }
}
```

---

## Multi-Tenancy

### Implementacao

**Campo em entities**:
```java
@Column(name = "empresa_id")
private Integer empresaId;
```

**Repositories com filtro**:
```java
List<Post> findByEmpresaId(Integer empresaId);
Page<Servico> findByEmpresaIdAndStatus(Integer empresaId, Status status, Pageable p);
```

**Controllers verificam tenant**:
```java
@GetMapping("/posts")
public Page<PostDTO> list(@RequestParam Integer empresaId, Pageable p) {
  // Validar que usuario tem acesso ao empresaId
  return postService.findByEmpresaId(empresaId, p);
}
```

### Isolamento

- [OK] Banco de dados compartilhado
- [OK] Filtragem via `empresaId`
- [ATENCAO] Sem isolamento fisico (shared schema)
- [ATENCAO] Aplicacao deve garantir filtragem

---

## Testes

### Estrategia

**Integration Tests** (preferido):
```java
@SpringBootTest
@AutoConfigureMockMvc
class PostControllerIT {

  @Autowired
  private MockMvc mockMvc;

  @Test
  void testListPosts() throws Exception {
    mockMvc.perform(get("/api/v1/posts?empresaId=1"))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$.content").isArray());
  }
}
```

**Unit Tests** (services):
```java
@ExtendWith(MockitoExtension.class)
class PostServiceTest {

  @Mock
  private PostRepository repository;

  @InjectMocks
  private PostServiceImpl service;
}
```

### Coverage

**Target**:
- Services: 80%+
- Controllers: 70%+
- Mappers: Gerados (nao testar)
- Entities: 50%+ (getters/setters)

---

## Configuracao

### application.properties (Producao)

```properties
# Server
server.port=8080

# Database
spring.datasource.url=jdbc:mariadb://localhost:3306/portal_auditoria
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASS}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false

# JWT
app.jwt.secret=${JWT_SECRET}
app.jwt.expiration=86400000       # 24h
app.jwt.refresh-expiration=604800000  # 7 dias

# Webhooks
portalweb.webhooks.incoming-secrets.stripe=${STRIPE_WEBHOOK_SECRET}
portalweb.webhooks.incoming-secrets.asaas=${ASAAS_WEBHOOK_SECRET}

# Storage
app.storage.type=LOCAL
app.storage.local.path=/var/uploads

# Email
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=${EMAIL_USER}
spring.mail.password=${EMAIL_PASS}

# Logging
logging.level.root=INFO
logging.level.com.auditoria.portalweb=INFO
```

### application-dev.properties

```properties
# Database local
spring.datasource.url=jdbc:mariadb://localhost:3306/portal_auditoria_dev
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update

# Logging verbose
logging.level.com.auditoria.portalweb=DEBUG
logging.level.org.hibernate.SQL=DEBUG

# Storage local
app.storage.local.path=./uploads-dev
```

---

## APIs Disponiveis

### Publicas (sem autenticacao)

```
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password

GET    /api/v1/posts
GET    /api/v1/posts/{slug}
GET    /api/v1/servicos
GET    /api/v1/empresas
GET    /api/v1/layout/home

POST   /api/v1/webhooks/receive/{source}  # Stripe, Asaas

GET    /api/v1/media/{sha256}              # Servir arquivos
```

### Admin (requer JWT)

```
# Posts
GET    /api/v1/admin/posts
POST   /api/v1/admin/posts
PUT    /api/v1/admin/posts/{id}
DELETE /api/v1/admin/posts/{id}

# Servicos
GET    /api/v1/admin/servicos
POST   /api/v1/admin/servicos

# Empresas
POST   /api/v1/admin/empresas
PUT    /api/v1/admin/empresas/{id}

# Usuarios
GET    /api/v1/admin/usuarios
POST   /api/v1/admin/usuarios
PUT    /api/v1/admin/usuarios/{id}

# Midia
POST   /api/v1/admin/media/upload
DELETE /api/v1/admin/media/{id}

# Webhooks
GET    /api/v1/admin/webhooks/subscriptions
POST   /api/v1/admin/webhooks/subscriptions
GET    /api/v1/admin/webhooks/deliveries

# Auditoria
GET    /api/v1/admin/audit
```

---

## Build e Deploy

### Build Local

```bash
# Compilar
mvn clean compile

# Testes
mvn test

# Package (JAR)
mvn package

# Pular testes (nao recomendado)
mvn package -DskipTests
```

### Executar Local

```bash
# Com Maven
mvn spring-boot:run

# JAR compilado
java -jar target/portal-web-backend-1.0.0.jar

# Com profile dev
java -jar target/portal-web-backend-1.0.0.jar --spring.profiles.active=dev
```

---

## Proximos Passos

### Melhorias Tecnicas

**1. Observabilidade**
- [ ] Adicionar Micrometer metrics
- [ ] Exportar para Prometheus
- [ ] Distributed tracing (Zipkin/Jaeger)
- [ ] Correlation IDs nos logs

**2. Performance**
- [ ] Redis cache para queries frequentes
- [ ] Paginacao em todas as listagens
- [ ] Indices otimizados no banco
- [ ] Async processing com @Async

**3. DevOps**
- [ ] Docker containerization
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Health checks customizados
- [ ] Graceful shutdown

**4. Funcionalidades**
- [ ] Versionamento de API (v2)
- [ ] GraphQL endpoint
- [ ] WebSocket para notificacoes real-time
- [ ] Exportacao de relatorios (PDF, Excel)

---

## Documentacao Adicional

- **Modulo Webhooks**: [docs/Modulos/webhooks/bakend-modulo_webhooks.md](./Modulos/webhooks/bakend-modulo_webhooks.md)
- **README Webhooks**: [src/main/java/.../webhooks/README.md](../src/main/java/com/auditoria/portalweb/modules/webhooks/README.md)
- **Spring Modulith**: https://docs.spring.io/spring-modulith/reference/
- **Spring Boot**: https://docs.spring.io/spring-boot/docs/3.5.7/reference/html/
- **MapStruct**: https://mapstruct.org/

---

## Resumo Final

### Estatisticas

| Componente | Arquivos | Tipo |
|-----------|----------|------|
| WEBHOOKS | 39 | Modulo Spring Modulith |
| CORPORATE | 33 | Modulo Spring Modulith |
| CONTENT | 31 | Modulo Spring Modulith |
| AUTH | 25 | Modulo Spring Modulith |
| MIDIA | 17 | Modulo Spring Modulith |
| USERS | 16 | Modulo Spring Modulith |
| AUDIT | 10 | Modulo Spring Modulith |
| LAYOUT | 5 | Modulo Spring Modulith |
| **Subtotal Modulos** | **176** | - |
| SHARED | 13 | Codigo Compartilhado |
| CONFIG | 5 | Configuracoes Globais |
| WEB | 1 | Controllers Publicos |
| **TOTAL BACKEND** | **195** | **100% Java** |

### Tecnologias Principais

- Spring Boot 3.5.7
- Spring Modulith 1.3.2
- Spring Data JPA
- MapStruct 1.5.x
- JWT Authentication
- MariaDB 12.0.2
- Lombok
- AspectJ AOP

---

**Autor**: Equipe Backend Portal Auditoria
**Ultima Atualizacao**: 2025-12-10
**Versao**: 1.0.0
