# 📁 Projeto Técnico - Módulo Mídia
## Plataforma SaaS Multi-tenant, API-first

**Versão:** 2.1.0
**Última Atualização:** 06/12/2025
**Status:** ✅ Backend 100%
**Arquitetura:** Plataforma SaaS multi-tenant, API-first

---

## 📋 Visão Geral

O módulo **Mídia** é responsável por **gerenciamento de arquivos de mídia** no Portal Auditoria, oferecendo upload seguro, deduplicação automática, armazenamento eficiente e API pública para acesso.

### 🎯 Responsabilidades

- ✅ Upload seguro com validação e sanitização
- ✅ Deduplicação automática via SHA-256
- ✅ Armazenamento local com suporte para S3/GCS (futuro)
- ✅ Extração automática de metadados (dimensões, MIME type)
- ✅ Organização temporal de arquivos (ano/mês)
- ✅ API pública para redirecionamento e consulta
- ✅ API administrativa para upload e remoção

---

## 🏗️ Arquitetura do Módulo

### 📁 Estrutura de Diretórios

```txt
modules/midia/
├── package-info.java                    # @ApplicationModule (sem dependências)
├── api/                                # Interface pública (3 arquivos)
│   ├── package-info.java               # @NamedInterface("api")
│   ├── dto/
│   │   ├── MediaDTO.java               # ✅ record (público)
│   │   └── UploadResponseDTO.java      # ✅ record (admin)
│   └── mapper/
│       └── MediaMapper.java            # ✅ Entity → DTO
├── domain/                             # Entidades JPA (4 arquivos)
│   ├── package-info.java               # @NamedInterface("domain")
│   ├── Media.java                      # ✅ Entidade principal
│   ├── MediaKind.java                  # ✅ Enum (IMAGE, VIDEO, DOC, OTHER)
│   └── MediaStorage.java               # ✅ Enum (LOCAL, S3, GCS)
├── repository/                         # Repositório JPA (1 arquivo)
│   └── MediaRepository.java            # ✅ JpaRepository
├── internal/                           # Implementações privadas (5 arquivos)
│   ├── MediaStorageService.java        # ✅ Interface
│   ├── impl/
│   │   ├── LocalStorageService.java    # ✅ Armazenamento local
│   │   └── MediaAdminServiceImpl.java  # ✅ Lógica de upload/delete
│   └── support/
│       └── FileNameUtil.java           # ✅ Sanitização de nomes
└── web/                               # Controllers REST (3 arquivos)
    ├── MediaPublicController.java      # ✅ GET /api/v1/media
    ├── MediaExceptionHandler.java      # ✅ Exception handling
    └── admin/
        └── AdminMediaController.java   # ✅ POST/DELETE /api/v1/admin/media

Total: 17 arquivos Java
```

### 🔗 Spring Modulith

```java
@ApplicationModule  // Sem dependências - módulo autossuficiente
```

**Named Interfaces:**
- `domain` → Exporta: Media, MediaKind, MediaStorage
- `api` → Exporta: DTOs, Mappers

**Uso por outros módulos:**
```java
// Corporate module
@ApplicationModule(allowedDependencies = {"midia::domain"})

// Relacionamento JPA (proxy via EntityManager)
empresa.setLogoMedia(entityManager.getReference(Media.class, logoMediaId));
```

---

## 🗄️ Modelo de Dados

### Entidade `Media`

```java
@Entity
@Table(name = "media")
public class Media {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Enumerated(EnumType.STRING)
  private MediaKind kind = MediaKind.IMAGE;

  @Enumerated(EnumType.STRING)
  private MediaStorage storage = MediaStorage.LOCAL;

  @Column(name = "path_key", length = 400, nullable = false)
  private String pathKey;          // Caminho físico

  @Column(length = 600, nullable = false)
  private String url;              // URL pública

  @Column(length = 100, nullable = false)
  private String mime;             // MIME type

  @Column(name = "bytes_size", nullable = false)
  private Long bytesSize = 0L;

  // Metadados de imagem
  private Integer width;
  private Integer height;

  // Acessibilidade
  @Column(name = "alt_text", length = 255)
  private String altText;

  @Column(name = "title_text", length = 255)
  private String titleText;

  // Deduplicação e integridade
  @Column(length = 64)
  private String sha256;

  // Design (futuro)
  @Column(name = "dominant_hex", length = 7)
  private String dominantHex;

  @Column(name = "created_at")
  private LocalDateTime createdAt = LocalDateTime.now();
}
```

### Enums

```java
public enum MediaKind {
  IMAGE,    // Imagens (jpg, png, gif, webp)
  VIDEO,    // Vídeos (mp4, webm, avi)
  DOC,      // Documentos (pdf, docx, xlsx)
  OTHER     // Outros arquivos
}

public enum MediaStorage {
  LOCAL,    // Filesystem local
  S3,       // Amazon S3
  GCS       // Google Cloud Storage
}
```

---

## 🔧 Componentes Principais

### 1. MediaStorageService (Interface)

Abstração para diferentes provedores de armazenamento:

```java
public interface MediaStorageService {
  record SavedFile(
      String pathKey,      // Caminho relativo
      String absolutePath, // Caminho absoluto
      String mime,
      long size,
      Integer width,       // null se não for imagem
      Integer height,      // null se não for imagem
      String sha256) {}

  SavedFile save(MultipartFile file, String subfolder) throws Exception;
  boolean delete(String absolutePath);
}
```

### 2. LocalStorageService

Implementação para armazenamento local:

**Funcionalidades:**
- Organização: `subfolder/YYYY/MM/filename-uuid.ext`
- Sanitização: Remove caracteres perigosos
- Hash: SHA-256 calculado durante escrita
- Metadados: Extrai dimensões de imagens
- Unicidade: UUID de 8 chars no nome

### 3. MediaAdminServiceImpl

Serviço de alto nível para operações administrativas:

```java
public Media createFromUpload(MultipartFile file, String subfolder,
                             String alt, String title) throws Exception {
  // 1. Salvar arquivo via LocalStorageService
  // 2. Verificar deduplicação por SHA-256
  // 3. Inferir MediaKind pelo MIME type
  // 4. Construir URL pública
  // 5. Persistir no banco
}

public void deleteMedia(Long id) {
  // 1. Buscar mídia
  // 2. Remover do banco
  // 3. Tentar remover arquivo físico (best-effort)
}
```

### 4. FileNameUtil

Sanitização segura de nomes de arquivo:

```java
public String sanitize(String original) {
  // 1. Separar nome e extensão
  // 2. Normalizar Unicode (NFD)
  // 3. Remover caracteres especiais
  // 4. Converter espaços para hífens
  // 5. Adicionar UUID único (8 chars)
  // 6. Retornar: nome-limpo-a1b2c3d4.ext
}
```

---

## 🌐 API REST

### API Pública (Somente Leitura)

**Base Path:** `/api/v1/media`

| Método | Endpoint | Descrição | Response |
|--------|----------|-----------|----------|
| `GET` | `/{id}` | Redireciona para URL da mídia | 302 Found + Location |
| `GET` | `/{id}/info` | Retorna metadados | MediaDTO (JSON) |

**Exemplo - Redirecionamento:**
```http
GET /api/v1/media/123

HTTP/1.1 302 Found
Location: /uploads/logos/2025/12/empresa-logo-a1b2c3d4.jpg
```

**Exemplo - Info:**
```http
GET /api/v1/media/123/info

{
  "id": 123,
  "url": "/uploads/logos/2025/12/empresa-logo-a1b2c3d4.jpg",
  "mime": "image/jpeg",
  "bytesSize": 45678,
  "width": 800,
  "height": 600,
  "altText": "Logo da Empresa XYZ",
  "titleText": "Logotipo oficial",
  "sha256": "abc123..."
}
```

### API Administrativa (ADMIN)

**Base Path:** `/api/v1/admin/media`

| Método | Endpoint | Descrição | Request | Response |
|--------|----------|-----------|---------|----------|
| `POST` | `/upload` | Upload de arquivo | multipart/form-data | UploadResponseDTO (201) |
| `DELETE` | `/{id}` | Remove mídia | - | 204 No Content |

**Exemplo - Upload:**
```http
POST /api/v1/admin/media/upload
Content-Type: multipart/form-data

file: [arquivo.jpg]
subfolder: "logos"
alt: "Logo da Empresa"
title: "Logotipo oficial"

---

HTTP/1.1 201 Created
{
  "id": 123,
  "url": "/uploads/logos/2025/12/empresa-logo-a1b2c3d4.jpg",
  "mime": "image/jpeg",
  "bytesSize": 45678,
  "width": 800,
  "height": 600,
  "sha256": "abc123...",
  "fileName": "empresa-logo.jpg"
}
```

**Exemplo - Delete:**
```http
DELETE /api/v1/admin/media/123

HTTP/1.1 204 No Content
```

---

## 📊 DTOs (Records)

### MediaDTO

```java
public record MediaDTO(
    Long id,
    String url,
    String mime,
    Long bytesSize,
    Integer width,
    Integer height,
    String altText,
    String titleText,
    String sha256) {}
```

**Uso:** Endpoint `/api/v1/media/{id}/info`

### UploadResponseDTO

```java
public record UploadResponseDTO(
    Long id,
    String url,
    String mime,
    Long bytesSize,
    Integer width,
    Integer height,
    String sha256,
    String fileName) {}
```

**Uso:** Endpoint `/api/v1/admin/media/upload` (response 201)

---

## 🔒 Segurança

### Validações de Upload

1. **Arquivo não vazio:** Valida antes de processar
2. **Sanitização de nome:** Remove path traversal e chars perigosos
3. **Organização forçada:** Estrutura `subfolder/YYYY/MM/` imposta
4. **Hash SHA-256:** Integridade e deduplicação

### Controle de Acesso

- **`/api/v1/media/**`** → Público (somente leitura)
- **`/api/v1/admin/media/**`** → Requer role ADMIN

### Boas Práticas LGPD

- **Alt/Title manuais:** Não extrai automaticamente (privacidade)
- **Metadados limitados:** Apenas técnicos (dimensões, MIME)
- **Sem EXIF:** Não expõe geolocalização ou dados sensíveis

---

## ⚙️ Configuração

### Propriedades

```properties
# Armazenamento local
app.media.local.base-dir=./uploads
app.media.public-base-url=/uploads/

# Futuro: S3, GCS
# app.media.s3.bucket=
# app.media.s3.region=
# app.media.gcs.bucket=
```

### Estrutura de Pastas

```
uploads/
├── logos/
│   ├── 2025/
│   │   ├── 01/
│   │   ├── 02/
│   │   └── 12/
│   │       └── empresa-logo-a1b2c3d4.jpg
├── documentos/
└── imagens/
```

---

## 🚀 Funcionalidades

### 1. Upload com Deduplicação

**Fluxo:**
1. Recebe MultipartFile
2. Calcula SHA-256 durante escrita
3. Verifica se hash já existe no banco
4. Se existir: retorna mídia existente
5. Se novo: persiste e retorna novo registro

**Benefício:** Economia de espaço, consistência de dados

### 2. Categorização Automática

**Por MIME type:**
- `image/*` → MediaKind.IMAGE
- `video/*` → MediaKind.VIDEO
- `application/pdf` → MediaKind.DOC
- Outros → MediaKind.OTHER

### 3. Extração de Metadados

**Automático (imagens):**
- Dimensões (width, height)
- MIME type
- Tamanho em bytes
- Hash SHA-256

**Manual (via upload):**
- Alt text (acessibilidade)
- Title text (tooltip)

### 4. Organização Temporal

Arquivos organizados automaticamente por:
```
subfolder/YYYY/MM/nome-sanitizado-uuid.ext
```

---

## 🔮 Extensibilidades Futuras

### 1. Novos Provedores de Armazenamento

```java
@Service
@ConditionalOnProperty(name="app.media.storage.type", havingValue="s3")
public class S3StorageService implements MediaStorageService {
  // AWS S3 implementation
}
```

### 2. Processamento de Imagens

```java
public interface ImageProcessingService {
  SavedFile createThumbnail(Media original, int width, int height);
  SavedFile applyWatermark(Media original, String text);
  String extractDominantColor(Media image);
}
```

### 3. Cache e CDN

```java
public interface MediaCacheService {
  String getCdnUrl(Media media);
  void invalidateCache(Long mediaId);
}
```

---

## 🎯 Resumo Final

**Status:** Módulo 100% funcional

**Funcionalidades:**
- ✅ Upload seguro com validação
- ✅ Deduplicação automática (SHA-256)
- ✅ Extração de metadados
- ✅ API pública (redirecionamento + info) - `/api/v1/media`
- ✅ API admin (upload + delete) - `/api/v1/admin/media`
- ✅ DTOs usando records

**Integração:**
- Usado por Corporate (logos de empresas via EntityManager.getReference)
- Named Interface `domain` expõe entidades
- Named Interface `api` expõe DTOs/Mappers

**Qualidade:**
- ✅ DTOs são records (moderno)
- ✅ Deduplicação por SHA-256
- ✅ Sanitização de nomes
- ✅ Paths seguem padrão do projeto (`/api/v1/admin/...`)

---

**📅 Última Atualização:** 06/12/2025
**👥 Desenvolvido:** Equipe Portal Auditoria
**🏗️ Arquitetura:** Spring Modulith + Armazenamento Local + Deduplicação SHA-256
**🌐 Versão:** 2.1.0
