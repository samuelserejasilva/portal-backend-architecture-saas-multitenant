# Portal Auditoria 2.0 — Backend Modular (Spring Boot + Spring Modulith)
## Plataforma SaaS Multi-tenant, API-first

## 🚀 Stack Tecnológica

| Componente | Versão | Observações |
|------------|--------|-------------|
| **Java** | 21 LTS | Compatível com 25 |
| **Spring Boot** | 3.5.7 | Core + Web + Data JPA |
| **Spring Modulith** | 1.3.2 | Modularização por pacote |
| **MariaDB** | 12.0.2+ | Collation: utf8mb4_uca1400_ai_ci |
| **MapStruct** | 1.5.x | Mapeamento DTO ↔ Entity |
| **Maven** | 3.8+ | Build via Maven Wrapper |

---

## 🚀 Visão Geral

Este projeto é o backend principal do **Portal Auditoria 2.0**, desenvolvido com **Spring Boot 3.5.x** e arquitetura **Spring Modulith 1.3.x**, garantindo modularidade, escalabilidade e rastreabilidade total de dependências.

O objetivo é modernizar o ecossistema **Portal Auditoria** substituindo o legado JSP/AMP por uma arquitetura Java modular limpa, com persistência 100% em banco de dados **MariaDB 12.0.2**, e integração completa com serviços internos de conteúdo, mídia e usuários.

---

## 🧩 Governança Modular (Spring Modulith)

O projeto adota o **Spring Modulith** para garantir isolamento e integridade das dependências entre módulos.

Cada módulo (ex.: `modules.midia`, `modules.corporate`, `modules.content`, etc.) possui:

* Um `package-info.java` com **@ApplicationModule** e interfaces nomeadas (`api`, `domain`, `spi`);
* Testes automatizados `*ArchitectureTests*` que verificam fronteiras e dependências;
* Documentação própria em `/docs/Módulos/Projeto_tecnico_<modulo>.md`.

### 🧱 Regras Gerais

* Nenhum módulo pode acessar classes `internal`, `impl` ou `repository` de outro.
* A comunicação ocorre apenas através de interfaces expostas (`api`, `domain`, `spi`).
* Toda alteração estrutural (dependência, exposição, integração) deve ser documentada.

---

## 🌐 API — Padrão e Estado Atual

- Base: **`/api/v1`** para todos os recursos.
- Admin: **`/api/v1/admin/**`** (users, posts, serviços, media, audit/logs).
- Público: **`/api/v1/auth/*`**, **`/api/v1/users/register`**, **`/api/v1/layout/**`**, **`/api/v1/posts/**`**, **`/api/v1/servicos/**`**, **`/api/v1/media/**`**, **`/api/v1/empresas/**`**, **`/api/v1/pessoas/**`**, **`/api/v1/errors/frontend`**.
- **OpenAPI/Swagger**:
  - **Especificação**: `C:\portal-auditoria\backend\openapi\openapi.json` (padronizado em `/api/v1`)
  - **Sincronização**: Use `scripts/sync-api-types.ps1` (na raiz do monorepo) para atualizar tipos no frontend
  - **Documentação interativa**: Disponível em `/swagger-ui.html` quando a aplicação está rodando
- Segurança: JWT nas rotas protegidas; whitelist configurada em `AuthSecurityConfig` cobre apenas as rotas públicas acima.

### Usuários e papéis disponíveis
- `super_admin` global já existente: `admin@portalauditoria.com.br` (tenant nulo).
- `company_admin` (empresa 1): `samuel.cereja@gmail.com`.
- Usuário comum: fluxo de **registro** (`/api/v1/users/register`) + **login** + **esqueci/reset de senha** (`/api/v1/auth/forgot-password` e `/api/v1/auth/reset-password`).

### Convites (opcional, ainda não implementado)
- Não há módulo de convite no backend. Se quiser delegar criação de `company_admin`/`user` via link:
  1) Criar endpoints `/api/v1/invites` (criar/listar/cancelar) gerando token com expiração.
  2) Endpoint público `/api/v1/invites/accept` (valida token, define senha, cria usuário e associa tenant/role).
  3) Envio de e-mail com link `https://.../invite/accept?token=...`.
- Enquanto não existir, continue criando `company_admin` via admin ou seed/SQL.

### Prioridades curtas (cadastro/recuperação)
- Frontend: telas já usam `/api/v1/users/register`, `/api/v1/auth/forgot-password`, `/api/v1/auth/reset-password`.
- Backend: fluxos de auth prontos; apenas manter OpenAPI atualizado ao ajustar contratos.

### ✅ Status da Arquitetura Modular

✔️ **Arquitetura validada** — `ModulithArchitectureTests` aprovados após correções de dezembro/2025.
✔️ **Módulos em conformidade** — `Corporate`, `Mídia`, `Content`, `Users`, `Audit` com nomenclatura correta.
✔️ **Dependências padronizadas** — Todos os módulos usam prefixo `modules.*` nas declarações.
✔️ **Spotless e compile** — Formatação e compilação validadas.
⚠️ **Build completo** — Requer workaround devido a problema de compilação incremental (investigação em andamento).

---

## 🧠 Nota Técnica — Governança Modular (Spring Modulith)

> 🧠 **Importante para desenvolvedores e mantenedores:**
> Toda **alteração de fronteira entre módulos** (por exemplo):
>
> * substituição de dependência direta por `EntityManager#getReference()`,
> * criação de **NamedInterface** (`api`, `domain`, `spi`),
> * inclusão ou remoção de `allowedDependencies`,
> * nova exposição pública entre módulos,
>
> deve ser **documentada imediatamente** nos seguintes arquivos:
>
> * `/docs/Módulos/Projeto_tecnico_<modulo>.md`
> * `/GUIA_DESENVOLVIMENTO.md`
>
> ✅ Isso garante que a documentação técnica acompanhe a evolução do código e que os testes
> `ModulithArchitectureTests` continuem 100% válidos (`mvnw test -Dtest="*ArchitectureTests*"`).
>
> 🧱 Cada módulo segue o princípio **Clean Architecture + Spring Modulith**, assegurando isolamento, reuso controlado e rastreabilidade total de dependências internas.

---

## 📘 Documentação Completa

Documentação detalhada do sistema, arquitetura, convenções de código e padrões de implementação:

📄 [GUIA_DESENVOLVIMENTO.md](GUIA_DESENVOLVIMENTO.md) - Guia completo de desenvolvimento
📄 [docs/Módulos/](docs/Módulos/)  — Documentação técnica individual de cada módulo.

---

## 🏁 Status Geral do Build

| Etapa                  |      Resultado      | Observações |
| ---------------------- | :-----------------: | ----------- |
| **Spotless**           |       ✅ Limpo       | Formatação validada |
| **Compile**            |      ✅ Sucesso      | `mvn clean compile` funcional |
| **Architecture Tests** |   ✅ 100% aprovados  | Spring Modulith validado |
| **Jacoco Report**      | ✅ Geração concluída | Cobertura de código |
| **Full Build (install)** | ⚠️ Com workaround | Ver seção "Comandos Úteis" |

---

> 💬 **Observação Final:**
> O Portal Auditoria 2.0 segue uma estrutura profissional baseada em padrões de mercado (Spring Modulith, Clean Architecture, DTOs tipados, testes automatizados e rastreabilidade).
> Toda alteração deve ser validada via `mvn spotless:apply`, `mvn compile` e `mvn test -Dtest="*ArchitectureTests*"` antes do commit final.
---

### 2️⃣ Configure o Banco de Dados

```sql
CREATE DATABASE portalweb CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

Configure as credenciais em:

* **DEV:** `src/main/resources/application-dev.properties`
* **PROD:** `C:\portalweb\config\application-prod.properties` (externo ao projeto)

### 🧱 Regras Principais

* Nenhum módulo pode acessar código interno (`internal`, `repository`, `impl`) de outro módulo;
* Comunicação entre módulos ocorre apenas por **interfaces expostas** (`api` ou `domain`);
* Alterações de fronteira (ex.: remoção de dependência, nova exposição pública) devem ser registradas em:
  * `docs/Módulos/Projeto_tecnico_<modulo>.md`
  * `GUIA_DESENVOLVIMENTO.md`

### ✅ Status Atual (Dezembro 2025)

✔️ **Arquitetura validada** — `ModulithArchitectureTests` 100% aprovados
✔️ **Dependências corrigidas** — Todos os módulos usam nomenclatura `modules.*`
✔️ **Corporate** acessa **Mídia** apenas via `modules.midia::domain`
✔️ **Audit** configurado para auditoria cross-module
✔️ **Spotless / Compile** — Formatação e compilação funcionais
⚠️ **Workaround necessário** — Ver seção "Comandos Úteis" para execução

```txt
src/main/java/com/auditoria/portalweb/
├── PortalwebApplication.java          # Classe principal Spring Boot
├── package-info.java                  # Documentação do pacote raiz
├── config/                            # Configurações globais da aplicação
│   ├── GlobalExceptionHandler.java    # Tratamento centralizado de exceções
│   ├── OpenApiConfig.java             # Configuração Swagger/OpenAPI
│   ├── SecurityConfig.java            # Configuração Spring Security
│   ├── WebConfig.java                 # Configuração Web MVC (CORS, etc.)
│   └── package-info.java
├── shared/                            # Utilitários e contratos compartilhados
│   ├── package-info.java
│   ├── dto/                           # DTOs compartilhados
│   │   ├── package-info.java
│   │   ├── IdNameDTO.java             # DTO genérico ID + Nome
│   │   └── PageResponse.java          # Wrapper de paginação
│   ├── exception/                     # Modelos de exceção
│   │   ├── package-info.java
│   │   └── ApiError.java              # Modelo de erro de API
│   ├── mapper/                        # Configuração MapStruct
│   │   ├── package-info.java
│   │   └── MapStructConfig.java       # Configuração global de mapeamento
│   ├── util/                          # Classes utilitárias
│   │   ├── DateUtils.java             # Utilitários de data
│   │   ├── JsonUtils.java             # Utilitários JSON
│   │   └── Slugify.java               # Geração de slugs para URL
│   └── validation/                    # Validadores customizados
│       ├── Slug.java                  # Anotação de validação de slug
│       └── SlugValidator.java         # Implementação do validador
└── modules/                           # Módulos de negócio (Spring Modulith)
    ├── corporate/                     # MÓDULO CORPORATE (empresas, pessoas)
    │   ├── package-info.java
    │   ├── api/                       # Camada API REST interna
    │   │   ├── dto/                   # DTOs para requisições/respostas
    │   │   │   ├── EmpresaDTO.java
    │   │   │   ├── EmpresaCreateDTO.java
    │   │   │   ├── EmpresaUpdateDTO.java
    │   │   │   ├── EmpresaSeoDTO.java
    │   │   │   ├── PessoaDTO.java
    │   │   │   ├── PessoaCreateDTO.java
    │   │   │   └── PessoaUpdateDTO.java
    │   │   └── mapper/                # Mapeadores MapStruct
    │   │       ├── EmpresaMapper.java
    │   │       └── PessoaMapper.java
    │   ├── domain/                    # Entidades JPA
    │   │   ├── Empresa.java           # Entidade Empresa
    │   │   └── Pessoa.java            # Entidade Pessoa
    │   ├── repository/                # Repositórios Spring Data JPA
    │   │   ├── EmpresaRepository.java
    │   │   └── PessoaRepository.java
    │   ├── service/                   # Interfaces de serviço
    │   │   ├── EmpresaService.java
    │   │   └── PessoaService.java
    │   ├── internal/                  # Implementações de serviço
    │   │   └── EmpresaServiceImpl.java
    │   ├── web/                       # Controllers REST
    │   │   ├── EmpresaController.java
    │   │   └── PessoaController.java
    │   └── spi/                       # SPI - Interface pública do módulo
    │       ├── package-info.java
    │       ├── EmpresaApi.java        # API de fronteira do módulo
    │       └── dto/
    │           ├── package-info.java
    │           └── EmpresaSeoDTO.java # DTO público para outros módulos
    ├── layout/                        # MÓDULO LAYOUT (gerenciamento de páginas)
    │   ├── package-info.java
    │   ├── api/
    │   │   └── dto/
    │   │       └── HomePageDTO.java   # DTO da página inicial
    │   ├── service/
    │   │   ├── LayoutService.java     # Interface de serviço
    │   │   └── LayoutServiceImpl.java # Implementação do serviço
    │   └── web/
    │       └── LayoutController.java  # Controller de layout
    ├── content/                       # MÓDULO CONTENT (conteúdo e posts)
    │   ├── package-info.java
    │   ├── api/
    │   │   ├── dto/
    │   │   │   ├── AutorDTO.java
    │   │   │   ├── PostDTO.java
    │   │   │   ├── ServicoDTO.java
    │   │   │   └── [outros DTOs...]
    │   │   └── mapper/
    │   │       ├── AutorMapper.java
    │   │       ├── PostMapper.java
    │   │       └── [outros mappers...]
    │   ├── domain/
    │   │   ├── Autor.java
    │   │   ├── Post.java
    │   │   └── Servico.java
    │   ├── repository/
    │   │   ├── AutorRepository.java
    │   │   ├── PostRepository.java
    │   │   └── ServicoRepository.java
    │   ├── internal/
    │   │   └── ContentServiceImpl.java
    │   ├── spi/
    │   │   ├── ContentApi.java
    │   │   └── dto/
    │   │       ├── PostSummaryDTO.java
    │   │       └── ServicoSummaryDTO.java
    │   └── web/
    │       ├── AutorController.java
    │       ├── PostController.java
    │       └── ServicoController.java
    ├── audit/                         # MÓDULO AUDIT (placeholder)
    │   └── package-info.java
    ├── social/                        # MÓDULO SOCIAL (placeholder)
    │   └── package-info.java
    └── users/                         # MÓDULO USERS (placeholder)
        └── package-info.java
```

### � **Estatísticas do Projeto**

* **Total de arquivos Java:** 154+ arquivos
* **Módulos ativos:** Corporate, Layout, Content, Mídia, Auth, Users, Audit ✅
* **Módulos placeholder:** Social
* **Arquitetura:** Spring Modulith + Clean Architecture
* **Testes de Arquitetura:** ModulithArchitectureTests validando fronteiras modulares

### 📦 **Estrutura Interna Padrão de Módulo**

```txt
modules/{nome-modulo}/
├── package-info.java     # Configuração Spring Modulith
├── domain/               # Entidades JPA (internas)
├── repository/           # Acesso ao banco (interno)
├── service/              # Interfaces de serviço (internas)
├── internal/             # Implementações de serviço (package-private)
├── api/                  # Interface REST interna
│   ├── dto/              # DTOs para API REST do módulo
│   └── mapper/           # Mappers MapStruct (Entity ↔ DTO)
├── web/                  # Controllers REST
├── spi/                  # ✅ INTERFACE PÚBLICA (para outros módulos)
│   ├── package-info.java # @NamedInterface("spi")
│   ├── {Nome}Api.java    # Contratos de serviço públicos
│   └── dto/              # DTOs para comunicação inter-módulos
└── **domain/             # ✅ NOVA - Exposição de entidades (Spring Modulith)
    └── package-info.java # @NamedInterface("domain")
```

---

## 🏗️ Princípios Arquiteturais

### ✅ Regras Obrigatórias

1. **Controller → Service → Repository** (nunca Controller → Repository direto)
2. **DTOs sempre obrigatórios** (nunca expor Entities na API)
3. **Comunicação entre módulos via Named Interfaces** (SPI ou domain)
4. **Frontend independente** (HTML + JavaScript + REST API)
5. **Testes obrigatórios** (código sem testes será rejeitado)

### 🚫 Proibições

* ❌ **Thymeleaf** (usar HTML estático + JavaScript)
* ❌ **Controller acessando Repository** direto
* ❌ **Expor Entities** nas APIs REST
* ❌ **Acessar service/repository** de outros módulos diretamente
* ✅ **Usar EntityManager.getReference()** para relacionamentos entre módulos
* ✅ **Respeitar Named Interfaces** (@NamedInterface) do Spring Modulith

### 📐 Camadas e Responsabilidades

```txt
Browser (HTML+JS) → Controller (web/) → Service → Repository → Database
                         ↓
                    DTOs (api/dto/)
                    
Comunicação entre módulos: Módulo A (spi/) ← Módulo B
```

---

## 🔒 Perfis de Execução

| Profile | Porta | Uso | DDL Mode |
|---------|-------|-----|----------|
| **dev** | 8080 | Desenvolvimento | `update` |
| **prod** | 8080 | Produção | `validate` |

> 💡 **Dica:** Configure via variável de ambiente: `SPRING_PROFILES_ACTIVE=dev`

---

## 🗄️ Estratégia de Banco de Dados

### ❌ NÃO utilizamos

* Flyway
* Liquibase
* Ferramentas de migration automática

### ✅ Utilizamos

* **Scripts SQL manuais** e idempotentes
* **DEV:** `spring.jpa.hibernate.ddl-auto=update` (+ scripts em `schema.sql` / `data.sql`)
* **PROD:** `spring.jpa.hibernate.ddl-auto=validate` (alterações via scripts controlados)

### 📊 Armazenamento de Conteúdo

Todo conteúdo (textos, imagens, páginas) é armazenado **no banco de dados**:

* 📄 Páginas/posts em tabelas específicas
* 🖼️ Imagens em campos BLOB ou caminhos relativos
* 🔗 Relacionamentos via chaves estrangeiras
* 💡 **Benefício:** Portabilidade total, backup simplificado

---

## 🧪 Padrões de Testes

### 📋 Estrutura Obrigatória

```txt
src/test/java/.../modules/{modulo}/
├── domain/           {Entity}Test.java
├── repository/       {Entity}RepositoryTest.java
├── service/          {Entity}ServiceTest.java
├── web/              {Entity}ControllerTest.java
└── integration/      {Entity}IntegrationTest.java
```

### ⚠️ Regra Crítica

Código sem testes = PR REJEITADO

---

## 📚 Documentação Adicional

Documentação completa disponível em [`/docs`](/docs):

* 📋 [**Guia de Desenvolvimento**](/GUIA_DESENVOLVIMENTO.md) - Boas práticas, convenções, arquitetura e histórico
* 👨‍💻 [**README Desenvolvedores**](/docs/README_DEV.md) - Guia para desenvolvedores
* 🛠️ [**Suporte Desenvolvimento**](/docs/SUPORTE_DEV.md) - Troubleshooting e dicas
* 🗄️ [**Database**](/docs/database/) - Scripts SQL e estrutura do banco

### 📂 Documentação de Módulos

Localizada em [`/docs/Módulos`](/docs/Módulos):

* [**Módulo Corporate**](/docs/Módulos/Módulo%20Corporate/) - ✅ Gestão de empresas e pessoas (ativo)
* [**Módulo Layout**](/docs/Módulos/Módulo%20Layout/) - ✅ Templates e páginas (ativo)
* [**Módulo Content**](/docs/Módulos/Módulo%20Content/) - ✅ Gestão de conteúdo, posts e serviços (ativo)
* [**Módulo Mídia**](/docs/Módulos/Modulo_midia/) - ✅ Gerenciamento de arquivos de mídia (ativo)
* [**Módulo Audit**](/docs/Módulos/Módulo%20Audit/) - 🚧 Auditoria e logs (placeholder)
* [**Módulo Social**](/docs/Módulos/Módulo%20Social/) - 🚧 Integração redes sociais (placeholder)
* [**Módulo Users**](/docs/Módulos/Módulo%20Users/) - 🚧 Gestão de usuários (placeholder)

---

## 🤝 Como Contribuir

### 📋 Antes de Criar um Pull Request

1. ✅ Leia [`GUIA_DESENVOLVIMENTO.md`](/GUIA_DESENVOLVIMENTO.md)
2. ✅ Siga a estrutura de módulos definida
3. ✅ Crie testes unitários e de integração
4. ✅ Documente em [`/docs/Módulos/`](/docs/Módulos/)
5. ✅ Valide com `./mvnw clean verify`

### 🚨 Checklist de Conformidade

* [ ] Controller não acessa Repository diretamente
* [ ] Comunicação entre módulos via SPI
* [ ] APIs REST expõem apenas DTOs
* [ ] Testes unitários + integração criados
* [ ] Documentação do módulo criada
* [ ] Frontend usa HTML + JS (sem Thymeleaf)

---

## 🔧 Comandos Úteis

```bash
# Compilar (sem testes de integração)
./mvnw clean compile

# Rodar testes unitários
./mvnw -q -DskipITs clean test

# Verificar tudo (testes + build)
./mvnw clean verify

# Gerar documentação do Spring Modulith
./mvnw spring-boot:run
# Veja em: target/modulith-docs/
```

### ⚠️ Workaround - Problema de Compilação Incremental

Devido a um problema conhecido com compilação incremental, use os seguintes comandos para execução:

```bash
# Opção 1: Compilar e executar separadamente (RECOMENDADO)
mvn clean compile
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Opção 2: Gerar JAR e executar diretamente
mvn clean package -DskipTests
java -jar target/devmulti-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev

# Opção 3: Build completo com testes (pode falhar - investigação em andamento)
mvn clean install
```

> 💡 **Nota**: O problema afeta apenas o ciclo completo de build. A compilação direta (`mvn compile`) funciona perfeitamente.

---

## ⚠️ Problemas Conhecidos e Soluções

### Problema de Compilação Incremental

**Sintoma**: Ao executar `mvn clean install` ou `mvn spring-boot:run`, o erro `ClassNotFoundException: PortalwebApplication` aparece.

**Causa**: Bug na compilação incremental do Maven (`useIncrementalCompilation=true`) causa perda intermitente de arquivos `.class` durante o ciclo completo de build.

**Solução Temporária**:
```bash
# Opção 1: Compilar e executar em etapas separadas
mvn clean compile
mvn spring-boot:run -Dspring-boot.run.profiles=dev

# Opção 2: Gerar JAR e executar
mvn clean package -DskipTests
java -jar target/devmulti-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev
```

**Status**: 🔄 Investigação em andamento para correção permanente

---

## 📝 Registro de Alterações

**Importante:** Não edite este README diretamente para sugestões.

Registre alterações propostas em: **[`/GUIA_DESENVOLVIMENTO.md`](/GUIA_DESENVOLVIMENTO.md)**

O maintainer revisará e atualizará o README conforme necessário.

---

## 🎯 Próximos Passos

1. ✅ Completar módulo `corporate` (empresas)
2. ✅ Criar módulo `content` (posts e páginas)
3. ✅ Integrar `layout` com dados reais
4. ✅ **Implementar módulo `midia`** (gerenciamento de arquivos)
5. ✅ **Corrigir dependências Spring Modulith** (audit, content, users)
6. 🔄 Resolver problema de compilação incremental
7. ⏳ Expandir módulo `users` (gestão completa de usuários)
8. ⏳ Expandir módulo `audit` (auditoria cross-module)
9. ⏳ Implementar módulo `social` (integração redes sociais)

## 🆕 **Atualizações Recentes**

### **✅ Dezembro 2025 - Correções Spring Modulith**

* **Correção crítica**: Ajustados `package-info.java` dos módulos `audit`, `content` e `users`
* **Nomenclatura padronizada**: Todas as dependências agora usam prefixo `modules.*` (ex: `modules.users::api`)
* **Módulo Audit expandido**: Agora com dependências formalizadas para auditoria cross-module
* **Testes validados**: `ModulithArchitectureTests` 100% aprovados após correções
* **Build funcional**: `mvn clean package -DskipTests` operacional
* **Workaround documentado**: Problema de compilação incremental identificado e documentado

### **🔧 Mudanças Técnicas - Dezembro 2025**

* ✅ `modules/audit/package-info.java` - Dependências corrigidas com prefixo `modules.*`
* ✅ `modules/content/package-info.java` - Dependência `modules.corporate::spi` corrigida
* ✅ `modules/users/package-info.java` - Todas as referências atualizadas
* ✅ `PortalwebApplication.java` - Recriado para resolver problemas de encoding
* ⚠️ Identificado bug de compilação incremental - workaround documentado

### **✅ Outubro 2025 - Spring Modulith (Histórico)**

* **Módulo Mídia** totalmente implementado com Named Interfaces (`@NamedInterface`)
* **Integração Corporate ↔ Mídia** via `modules.midia::domain`
* **Testes de arquitetura** validando fronteiras modulares
* **EntityManager.getReference()** substituindo acesso direto a repositórios
* **Documentação técnica** atualizada com padrões Spring Modulith

---

## 📄 Licença

Este projeto é proprietário. Todos os direitos reservados.

---

## 💬 Suporte

* 📧 Email: <suporte@auditoria.com>
* 📝 Issues: Use o sistema de tickets interno
* 📚 Wiki: [Documentação Interna](https://wiki.auditoria.com)

---

---

## 🧩 Nota Técnica — Governança Modular (Spring Modulith)

> 🧠 **Importante para desenvolvedores e mantenedores:**  
> Toda **alteração de fronteira entre módulos** (por exemplo):
>
> * substituição de dependência direta por `EntityManager#getReference()`,
> * criação de **NamedInterface** (`api`, `domain`, `spi`),
> * inclusão ou remoção de `allowedDependencies`,
> * nova exposição pública entre módulos,
>
> deve ser **documentada imediatamente** nos seguintes arquivos:
>
> * `/docs/Módulos/Projeto_tecnico_<modulo>.md`
> * `/GUIA_DESENVOLVIMENTO.md`
>
> ✅ Isso garante que a documentação técnica acompanhe a evolução do código e que os testes  
> `ModulithArchitectureTests` continuem 100% válidos em todos os builds (`mvnw test -Dtest="*ArchitectureTests*"`).
>
> 🧱 Cada módulo mantém sua própria fronteira, seguindo o princípio de **Clean Architecture + Spring Modulith**  
> (isolamento, reuso controlado e auditoria completa das dependências internas).

---

---

## 🧾 Histórico Arquitetural — Validações de Módulos

| Data       | Alteração                                                                                                   | Status |
|-------------|-------------------------------------------------------------------------------------------------------------|:-------:|
| **09/12/2025** | 🔧 **Correção crítica Spring Modulith**: Ajustados `package-info.java` dos módulos `audit`, `content` e `users` com prefixos corretos (`modules.*`). Dependências inter-modulares agora respeitam nomenclatura completa (ex: `modules.users::api` ao invés de `users`). | ✅ |
| **09/12/2025** | 🧩 Módulo **Audit** expandido com dependências formalizadas: `modules.users::api`, `modules.auth::api`, `modules.content`, `modules.corporate`, `modules.midia`. Permite auditoria cross-module mantendo isolamento. | ✅ |
| **09/12/2025** | ⚠️ **Identificado**: Problema de compilação incremental (`useIncrementalCompilation=true`) causa perda de `PortalwebApplication.class` em alguns ciclos de build. **Workaround**: `mvn clean compile` + `mvn spring-boot:run` ou executar JAR diretamente. Investigação em andamento. | 🔄 |
| **09/12/2025** | ✅ Testes de arquitetura `ModulithArchitectureTests.verifiesModularStructure` validados após correções. Build com `mvn clean package -DskipTests` funcional. | ✅ |
| **28/10/2025** | 🔄 Removida dependência direta **Corporate → Mídia.repository**. Substituído por `EntityManager#getReference()` para acesso ao tipo `Media` via `modules.midia::domain`. | ✅ |
| **28/10/2025** | 🧩 Atualizados `package-info.java` de `Corporate` e `Mídia`. `ModulithArchitectureTests` executados e aprovados 100%. | ✅ |
| **28/10/2025** | 🧹 `Spotless`, `Compile`, `Jacoco` e `Modulith` testados — arquitetura confirmada como íntegra e modular. | ✅ |

> 📚 *Este histórico resume apenas alterações de arquitetura e fronteiras modulares.*
> Mudanças de negócio ou comportamento de API devem ser registradas nos documentos técnicos específicos de cada módulo.

---

Desenvolvido com ☕ + 💻 pela equipe de Auditoria

**Última atualização:** Dezembro 2025

---

## Frontend Externo (SPA) — Status e Guia de Testes

Este repositório é o backend. O frontend público roda como SPA estática em uma pasta separada.

- Local do front: `C:\portal-frontend-externo`
- Servidor SPA: `node server.js` (porta 8000, fallback para `index.html` em rotas)
- API base: configurada em `C:\portal-frontend-externo\config\api-config.js` (`baseURL: 'http://localhost:8080'`)
- Empresa padrão (layout): `defaultEmpresa: 'portalauditoria'`

### O que foi implementado (front)

- Autenticação JWT integrada ao backend
  - Login (`POST /api/v1/auth/login`) salvando `access_token` em `localStorage.auth_token`.
  - Header automático `Authorization: Bearer <token>` via `assets/js/api-client.js`.
  - Perfil do usuário via `GET /api/v1/auth/me` (protegiddo). Avatar carregado no header.
  - Logout com revogação (`POST /api/v1/auth/revoke`) + limpeza de storage.
  - Redirecionamento pós‑login por perfil (role no JWT):
    - ADMIN → `/dashboard/administrador`
    - SUPERVISOR → `/dashboard/supervisor`
    - demais → `/dashboard/cliente`
    - Implementado em `assets/js/auth-header.js` (`getDashboardUrl()`).

- SPA e navegação
  - Servidor SPA (`server.js`) com fallback para `index.html` ao recarregar rotas profundas.
  - Router client‑side (`assets/js/router.js`) carrega templates de `templates/*.html`.

- Layout dinâmico (header/footer)
  - Chamada `GET /api/layout/header-footer?empresa=<slugOuId>` com fallback:
    - `localStorage.empresa_slug` → `API_CONFIG.defaultEmpresa` → `'1'`
  - Implementado em `assets/js/layout-components.js` (usa `apiClient.getHeaderFooter(...)`).

- Acessibilidade
  - Aumento de contraste em textos do botão e do menu de usuário (`assets/css/auth-header.css`).

### Como rodar (dev)

1) Backend (porta 8080)

```
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

2) Frontend (porta 8000)

```
cd C:\portal-frontend-externo
node server.js
```

3) (Opcional) Definir empresa ativa no navegador

```
// Console do browser (F12)
localStorage.setItem('empresa_slug','portalauditoria');
location.reload();
```

### Roteiro de testes (E2E)

1. Abrir `http://localhost:8000` — ver header e footer carregados; sem erros críticos no console.
2. Login (modal): informar email/senha válidos.
   - Esperado: 200 no `/auth/login`, `localStorage.auth_token` preenchido.
   - Em seguida: 200 no `/auth/me` e avatar visível no header.
3. Redirecionamento pós‑login: navegador vai para `/dashboard/{perfil}` conforme role do JWT.
   - Se recarregar a página nessa rota, o `server.js` deve servir `index.html` (sem 404) e o router deve montar o template.
4. Layout: `/api/layout/header-footer?empresa=portalauditoria` → 200 e dados mínimos (título, descrição, logo).
5. Logout: clicar no menu do usuário → “Sair”.
   - Esperado: chamada a `/auth/revoke` e limpeza de storage, header volta a mostrar “Entrar”.

### Observações e dicas

- Se `/api/v1/auth/me` falhar, verifique se o front envia `Authorization: Bearer <access_token>` (token no `localStorage`).
- Se aparecer 404 ao abrir uma rota `/dashboard/...` diretamente, confirme que o front foi iniciado com `node server.js` (fallback SPA ativo).
- Se `/api/layout/header-footer` retornar 400/500, defina `empresa_slug` no `localStorage` ou ajuste `defaultEmpresa` no `config/api-config.js`.

### Próximos passos (sugestão)

- Completar páginas dos dashboards (`templates/dashboard_*.html`).
- Adicionar proteção de rotas no router conforme autenticação/role.
- Habilitar fluxo “Esqueci minha senha” no front para usar os novos endpoints (`/api/v1/auth/forgot-password` e `/api/v1/auth/reset-password`).
