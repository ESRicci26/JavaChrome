# JavaChrome 🌐

Um navegador web ao estilo **Google Chrome**, construído inteiramente com tecnologias Java: **Java 11**, **Spring Boot** e **Thymeleaf** no frontend, com layout responsivo e arquitetura em camadas.

> **package base:** `com.javaricci` &nbsp;|&nbsp; **artifactId:** `javachrome`

---

## No projeto consta o arquivo JavaChrome.zip porque o projeto é muito grande.

## ⚠️ Como funciona (leia antes de tudo)

Spring Boot + Thymeleaf é uma stack **server-side**: ela gera HTML/CSS/JS que rodam dentro de um navegador já existente (Chrome, Firefox, Edge...). Não é possível, com essa stack, recriar um motor de renderização nativo como o Chromium.

O que o **JavaChrome** entrega é o que se costuma chamar de **"browser shell"**: uma aplicação web completa que se comporta como um navegador de verdade, dentro do seu navegador real:

- Interface idêntica à de um navegador Chrome: abas, barra de endereço (omnibox), botões voltar/avançar/recarregar, favoritos, histórico, menu.
- O conteúdo dos sites visitados é carregado dentro de um `<iframe>`.
- Todo o back-end (Spring Boot) cuida de favoritos, histórico e da descoberta de título/ícone de cada página visitada.

### O problema do X-Frame-Options/CSP — e como o JavaChrome contorna

Muitos sites (redes sociais, bancos, buscadores...) enviam os cabeçalhos `X-Frame-Options` ou `Content-Security-Policy: frame-ancestors` para impedir que sejam exibidos dentro de um `<iframe>` de **outra origem** — é uma proteção legítima contra clickjacking, imposta pelo próprio navegador do usuário, e nenhum JavaScript do lado do cliente pode contornar isso diretamente.

Para resolver isso, o JavaChrome usa um **proxy server-side** (`ProxyController` + `ProxyService`, via `/proxy/view?url=...`):

1. O **backend** (não o navegador do usuário) busca o HTML da página externa com Jsoup.
2. Reescreve a página: insere uma tag `<base>` para que imagens/CSS/JS continuem carregando do site original, remove `<meta>` de CSP embutidas no HTML, e reescreve os links (`<a href>`) para continuarem passando pelo proxy.
3. Devolve esse HTML como se fosse conteúdo do **próprio domínio do JavaChrome** — sem repassar os cabeçalhos de bloqueio do site original.
4. O iframe, agora carregando conteúdo do mesmo domínio da aplicação, exibe a página normalmente.

Com isso, sites como Wikipedia, GitHub, MDN, blogs e a maioria dos sites de conteúdo passam a funcionar dentro do JavaChrome.

**Limitações que continuam existindo (inerentes à técnica, não são bugs):**
- **Sessões/cookies não são levados** — a página é sempre buscada de forma pública/anônima pelo servidor. Sites que exigem login (ex: e-mail, redes sociais logadas) aparecerão na versão pública ou de login.
- **Sites com forte proteção anti-bot** (Cloudflare, Akamai, captcha) podem recusar a requisição do servidor, mesmo com um `User-Agent` de navegador comum.
- **Aplicações de página única (SPA)** que dependem pesadamente de JavaScript para renderizar o conteúdo principal (o Jsoup não executa JavaScript) podem aparecer incompletas ou em branco.
- **Formulários de busca** do site original não são reescritos — use a própria barra de endereço do JavaChrome para pesquisar.
- Quando o proxy falha, o JavaChrome mostra um aviso amigável com opção de abrir o site em uma aba real do seu navegador.

---

## ✨ Funcionalidades

- Múltiplas abas (abrir, fechar, alternar) com favicon e título dinâmicos
- Barra de endereço (omnibox) que detecta se o texto é uma URL ou um termo de busca
- Botões Voltar / Avançar / Recarregar com **histórico de navegação por aba**
- Favoritos (adicionar/remover pela estrela na barra de endereço, página completa em `/bookmarks`)
- Histórico de navegação persistido em banco (`/history`), com opção de limpar tudo
- Página de "Nova Aba" com busca e atalhos para os favoritos
- Layout **responsivo** (desktop, tablet e celular) e suporte a modo escuro (`prefers-color-scheme`)
- Aviso automático quando um site bloqueia sua exibição em iframe

---

## 🏗️ Arquitetura

Arquitetura em camadas, com responsabilidades bem separadas:

```
com.javaricci.javachrome
├── JavaChromeApplication      # bootstrap da aplicação Spring Boot
├── config/                    # configurações e propriedades customizadas
│   ├── JavaChromeProperties   # propriedades externalizadas (application.yml)
│   ├── WebConfig
│   └── DataInitializer        # seed de favoritos padrão (CommandLineRunner)
├── controller/                 # controllers MVC (renderizam páginas Thymeleaf)
│   ├── BrowserController
│   ├── BookmarkPageController
│   ├── HistoryPageController
│   ├── ProxyController         # busca e reescreve paginas externas (bypass de X-Frame-Options)
│   └── api/                    # controllers REST (consumidos via AJAX)
│       ├── BookmarkApiController
│       ├── HistoryApiController
│       └── PageApiController
├── service/                    # regras de negócio (interfaces + impl)
│   ├── BookmarkService / impl/BookmarkServiceImpl
│   ├── HistoryService  / impl/HistoryServiceImpl
│   ├── PageMetadataService / impl/PageMetadataServiceImpl  (usa Jsoup)
│   └── ProxyService / impl/ProxyServiceImpl                (usa Jsoup)
├── repository/                 # Spring Data JPA
│   ├── BookmarkRepository
│   └── HistoryRepository
├── model/                      # entidades JPA
│   ├── Bookmark
│   └── HistoryEntry
├── dto/                        # objetos de transferência (nunca expõe entidade JPA)
├── mapper/                     # conversão entidade <-> DTO
└── exception/                  # exceções de negócio + handler global (@RestControllerAdvice)
```

### Boas práticas aplicadas

- **Injeção de dependência via construtor** (`@RequiredArgsConstructor`), nunca por campo.
- **Separação Controller → Service → Repository**, com interfaces de serviço (facilita testes e troca de implementação).
- **DTOs e Mappers**: as entidades JPA nunca "escapam" para a camada web.
- **Validação de entrada** com Bean Validation (`@NotBlank`, `@URL`, `@Size`) direto nos DTOs de requisição.
- **Tratamento de erros centralizado** com `@RestControllerAdvice`, retornando um payload de erro padronizado.
- **Configuração externalizada** (`application.yml` + `@ConfigurationProperties`), sem valores fixos ("magic strings") no código.
- **Testes unitários** de serviço (JUnit 5 + Mockito) e teste de camada web (`@WebMvcTest` + MockMvc).
- **Frontend organizado**: CSS com variáveis (fácil tema escuro), JS modularizado (`api-client.js`, `tabs.js`, `browser.js`), sem duplicação de chamadas `fetch`.

---

## 🛠️ Tecnologias

| Camada        | Tecnologia                          |
|---------------|--------------------------------------|
| Linguagem     | Java 11                              |
| Framework     | Spring Boot 2.7.18                   |
| Build         | Maven                                |
| Persistência  | Spring Data JPA + H2 (arquivo local) |
| Frontend      | Thymeleaf + CSS puro (responsivo) + JavaScript (vanilla) |
| Scraping leve | Jsoup (título/favicon das páginas)   |
| Testes        | JUnit 5, Mockito, AssertJ            |
| Boilerplate   | Lombok                               |

---

## ▶️ Como executar

Pré-requisitos: **JDK 11+** e **Maven 3.6+** instalados, com acesso à internet (para baixar as dependências do Maven Central na primeira execução).

```bash
# Na raiz do projeto
mvn clean install
mvn spring-boot:run
```

Depois, acesse: **http://localhost:8080**

> O banco H2 é criado automaticamente em `./data/javachrome.mv.db` na primeira execução, já populado com alguns favoritos padrão (Wikipedia, GitHub, Spring, DuckDuckGo). O console do H2, se precisar inspecionar os dados, fica em `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:file:./data/javachrome`).

### Rodando os testes

```bash
mvn test
```

> **Nota sobre este ambiente de geração:** o sandbox usado para gerar este projeto não tem acesso ao repositório Maven Central, então não foi possível rodar `mvn compile`/`mvn test` aqui dentro para uma verificação de build 100% ao vivo. Todo o código foi revisado manualmente com atenção a imports, assinaturas de métodos e consistência entre back-end e front-end. Ao rodar `mvn clean install` na sua máquina (com internet liberada), qualquer eventual ajuste fino de dependências será resolvido automaticamente pelo Maven; se algo não compilar, me mostre o erro e eu corrijo rapidamente.

---

## 📁 Endpoints principais

**Páginas (MVC/Thymeleaf):**
| Método | Rota          | Descrição                         |
|--------|---------------|------------------------------------|
| GET    | `/`           | Shell do navegador (abas, omnibox) |
| GET    | `/bookmarks`  | Página de gerenciamento de favoritos |
| GET    | `/history`    | Página de histórico de navegação   |
| GET    | `/proxy/view?url=` | Busca e devolve uma página externa reescrita (usada dentro do iframe) |

**API REST (`/api/**`, JSON):**
| Método | Rota                     | Descrição                              |
|--------|--------------------------|------------------------------------------|
| GET    | `/api/bookmarks`         | Lista favoritos                          |
| POST   | `/api/bookmarks`         | Cria um favorito                         |
| DELETE | `/api/bookmarks/{id}`    | Remove um favorito                       |
| GET    | `/api/history`           | Lista histórico (parâmetro opcional `limit`) |
| POST   | `/api/history`           | Registra uma visita                      |
| DELETE | `/api/history/{id}`      | Remove um item do histórico              |
| DELETE | `/api/history`           | Limpa todo o histórico                   |
| GET    | `/api/page/metadata?url=`| Retorna título/favicon de uma URL        |
| GET    | `/api/page/resolve?query=`| Resolve o texto da omnibox em uma URL de navegação ou busca |

---

## 🚀 Possíveis evoluções

- Autenticação/multiusuário (Spring Security) para favoritos e histórico por conta
- Sincronização de abas entre dispositivos (WebSocket)
- Modo anônimo (aba que não grava histórico)
- Bloqueador de anúncios básico via regras de CSS/JS injetadas no iframe (quando same-origin)
- Extensão para navegador real via Chrome Extensions API (fora do escopo desta stack)

## https://angular.io
## https://react.dev/
## https://www.uol.com.br/
## https://claude.ai/new

