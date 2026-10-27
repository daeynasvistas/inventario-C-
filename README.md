# TechRepair

Sistema de gestão de ordens de reparação para a empresa TechRepair.

## Pré-requisitos

- [.NET 10 SDK](https://dotnet.microsoft.com/download)
- Visual Studio Code ou outro editor compatível
- Terminal

## Tecnologias utilizadas

- C# e ASP.NET Core Web API (.NET 10)
- Entity Framework Core com SQLite
- ASP.NET Core Identity (autenticação por cookies)
- OpenAPI/Swagger
- Proteção Antiforgery (CSRF)

## Arquitetura

- API REST com prefixo `/api`
- Autenticação por cookies HttpOnly (SameSite=Lax), sessão de 8 horas sem renovação deslizante
- Preparado para alojamento conjunto com cliente Blazor WebAssembly PWA no mesmo endereço HTTPS:
  - `/` - Aplicação cliente (futura)
  - `/api` - API
- Base de dados SQLite em `App_Data/techrepair.db`
- Chaves de Data Protection em pasta configurável fora de `wwwroot` (excluída do Git)
- Validação, autorização por perfis e proteção antiforgery

## Configuração

### 1) Configurar ligação à base de dados

A ligação é configurada em `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=App_Data/techrepair.db"
  }
}
```

A pasta `App_Data/` é criada automaticamente. A integridade referencial do SQLite deve estar ativa.

### 2) Configurar administrador inicial (User Secrets ou variáveis de ambiente)

**Nunca** incluir credenciais no código ou no README. Utilize uma das opções:

#### Opção A - User Secrets (recomendado para desenvolvimento)

```bash
dotnet user-secrets init --project TechRepair.Api
dotnet user-secrets set "AdminUser:Email" "admin@techrepair.pt" --project TechRepair.Api
dotnet user-secrets set "AdminUser:Password" "PasswordSegura123!" --project TechRepair.Api
```

#### Opção B - Variáveis de ambiente

```bash
# Windows (PowerShell)
$env:AdminUser__Email = "admin@techrepair.pt"
$env:AdminUser__Password = "PasswordSegura123!"

# Linux/macOS
export AdminUser__Email="admin@techrepair.pt"
export AdminUser__Password="PasswordSegura123!"
```

Requisitos da palavra-passe (Identity):
- Mínimo 10 caracteres
- Pelo menos uma maiúscula, uma minúscula, um número e um carácter especial
- Bloqueio: 15 minutos após 5 tentativas falhadas

O administrador é criado de forma idempotente apenas se não existir nenhum administrador ativo. As passwords não são alteradas em cada arranque.

### 3) Chaves de Data Protection

Configurar o caminho para as chaves de Data Protection fora de `wwwroot` e excluído do Git (ex.: `DataProtection-Keys/`).

## Comandos

### Restaurar dependências
```bash
dotnet restore TechRepair.sln
```

### Compilar
```bash
dotnet build TechRepair.sln
```

### Criar e aplicar migration inicial
```bash
dotnet ef migrations add InitialCreate --project TechRepair.Api
dotnet ef database update --project TechRepair.Api
```

> Usar migrations. Não utilizar `EnsureCreated` em conjunto com migrations.

### Executar API
```bash
dotnet run --project TechRepair.Api
```

### Executar testes
```bash
dotnet test TechRepair.sln
```

## Documentação OpenAPI

Em ambiente Development, a documentação OpenAPI encontra-se disponível (ex.: Swagger UI) no endereço configurado pela API.

## Autenticação, Cookies e Antiforgery

### Autenticação por cookies
- Cookie HttpOnly
- Cookie Secure em HTTPS
- SameSite=Lax (cliente e API no mesmo endereço)
- Sessão de 8 horas, sem renovação deslizante
- Sem sessão persistente nesta versão

### Proteção CSRF (Antiforgery)
- Implementar proteção antiforgery do ASP.NET Core
- Obter request token através de `GET /api/auth/csrf` antes de operações que alteram dados
- Enviar o token no cabeçalho documentado nos pedidos `POST`, `PUT`, `PATCH` e `DELETE` (incluindo login e logout)
- É necessário obter um novo token após login e logout
- Pedidos `GET` não alteram dados

### Endpoints de autenticação
| Método | Endpoint | Descrição |
|---|---|---|
| GET | `/api/auth/csrf` | Obter token antiforgery |
| POST | `/api/auth/login` | Login (email/password). Devolve utilizador autenticado e perfis. |
| POST | `/api/auth/logout` | Logout |
| GET | `/api/auth/me` | Utilizador autenticado atual |
| POST | `/api/auth/change-password` | Alterar password própria (requer password atual) |

Regras:
- Não existe registo público
- Credenciais inválidas devolvem mensagem genérica
- Contas desativadas não conseguem aceder ou manter sessão
- Alterações de password/perfil/estado/associação invalidam sessões existentes
- Nunca são devolvidos hashes, passwords ou tokens internos

## Perfis e Permissões

### Perfis
- `Administrador`
- `Tecnico`

### Administrador
- Gere utilizadores, perfis, clientes, equipamentos e técnicos
- Consulta todas as reparações e intervenções
- Abre reparações, atribui/altera técnico responsável, altera estados
- Regista/edita/eliminação de intervenções em reparações abertas, identificando o técnico responsável
- Gestão de utilizadores: listar, consultar, criar (com perfil/associação/técnico), alterar, ativar/desativar (sem eliminação de contas)

### Técnico
- Consulta clientes, equipamentos e técnicos ativos
- Consulta apenas reparações que lhe estejam atribuídas
- Consulta apenas intervenções e histórico das reparações a que tenha acesso
- Regista/edita/elimina apenas as suas intervenções em reparações abertas que lhe estejam atribuídas
- Altera estados de trabalho e conclui as suas reparações
- Não pode marcar como `Entregue`, atribuir técnicos ou gerir utilizadores
- Não pode criar/editar/eliminar clientes ou equipamentos

Autorização aplicada no servidor. Para utilizadores `Tecnico`, o técnico é obtido a partir da sessão (não confiar em IDs enviados pelo cliente). Consultas a reparações de outros técnicos devolvem `404` sem revelar existência.

## Modelo de Dados (resumo)

- **Cliente**: Nome (obrigatório, máx. 150), NIF/Email/Telefone/Morada opcionais
- **Equipamento**: Cliente, Tipo/Marca/Modelo obrigatórios, Nº Série/Observações opcionais (não é permitido mudar cliente se tiver reparações)
- **Tecnico**: Nome (obrigatório), Email, Ativo, `ApplicationUserId` opcional e único; apenas técnicos ativos recebem novas atribuições/intervenções
- **ApplicationUser**: Deriva de `IdentityUser`, adiciona `Nome` (máx. 150) e `Ativo`. Email usado como username. Máximo 1 técnico por conta; conta com perfil `Tecnico` tem de estar associada a técnico ativo
- **Reparacao**: Equipamento, `TecnicoId` opcional até atribuição, `DescricaoAvaria` obrigatória, `DataEntrada` (UTC), `Estado`, `DataConclusao`/`DataEntrega` opcionais
- **Intervencao**: Reparacao, `TecnicoId` obrigatório, `Data`, `Descricao` obrigatória, `PecasUtilizadas` opcional, `TempoMinutos`, `CustoMaoObraCentimos`, `CustoPecasCentimos` (valores em cêntimos, `long`, não negativos). Técnico da intervenção tem de ser o responsável atual

## Estados da Reparação
`AguardarDiagnostico` | `EmReparacao` | `AguardarPecas` | `Concluida` | `Entregue`

Serializados por nome em JSON. Estados desconhecidos são rejeitados.

## Regras de Negócio (principais)

- Reparação começa em `AguardarDiagnostico`. `DataEntrada` definida no servidor (UTC).
- Transições permitidas entre `AguardarDiagnostico`, `EmReparacao`, `AguardarPecas`. Pode concluir a partir de qualquer estado de trabalho.
- Exigir técnico responsável antes de passar para `EmReparacao`, `AguardarPecas` ou `Concluida`. Ao concluir -> `DataConclusao` (servidor). Só de `Concluida` para `Entregue` -> `DataEntrega` (servidor).
- Não reabrir reparações concluídas/entregues. Não alterar dados de negócio/intervenções de reparações concluídas/entregues.
- Pedido para manter estado atual não altera datas.
- Apenas técnicos ativos recebem novas atribuições/intervenções. Alteração de responsável não modifica intervenções anteriores.
- Técnico da intervenção = responsável atual da reparação.
- Tempos/custos não negativos. Valores monetários em cêntimos (`long`). Custo por intervenção e total da reparação calculados no servidor (soma mão-de-obra + peças). Totais não aceites do cliente.
- Não gerir inventário (peças em texto).
- Não alterar cliente de equipamento com reparações. Não alterar equipamento de reparação.
- Não eliminar clientes/equipamentos com dependências. Eliminação de reparações não disponível nesta versão.
- Desativar técnicos/utilizadores em vez de eliminar (preservar histórico).
- Chaves estrangeiras e restrições de eliminação configuradas explicitamente.
- Transações em operações com múltiplos registos.
- Impedir desativação/alteração que remova o último administrador ativo.
- Respostas JSON sem referências circulares. Não expor entidades EF diretamente (usar DTOs).

## Endpoints de Negócio (resumo)

### Clientes
- Criar/consultar/editar/eliminar (quando permitido)
- Pesquisa por nome/NIF/telefone
- Equipamentos de um cliente

### Equipamentos
- Criar/consultar/editar/eliminar (quando permitido)
- Filtrar por cliente
- Histórico de reparações (respeitando permissões)

### Técnicos
- Criar/consultar/editar, ativar/desativar
- Reparações atribuídas (respeitando permissões)

### Reparações
- Abrir, listar/detalhe, editar descrição (enquanto aberta)
- Atribuir/alterar técnico responsável (endpoint próprio)
- Alterar estado (endpoint próprio)
- Filtros: cliente, equipamento, técnico, estado, datas de entrada
- Detalhe inclui custo total e dados essenciais de cliente/equipamento/técnico
- Resumo com quantidades por estado e custo total (filtros/permissões aplicados antes do cálculo)

### Intervenções
- Criar/listar via `/api/reparacoes/{id}/intervencoes`
- Consultar/editar/eliminar via `/api/intervencoes/{id}`
- Respeitam estado da reparação e permissões

## Validação, Paginação e Respostas

- Validação de obrigatórios/comprimentos/emails. Espaços trimados; strings opcionais vazias -> `null` (não remover espaços internos)
- Validação de referências existentes, intervalos de datas e valores numéricos
- DTOs específicos por operação
- Códigos HTTP: `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`
- Erros em `ProblemDetails`, mensagens em português de Portugal, com identificador de rastreio
- Sem detalhes internos fora de Development. Sem logs de passwords/cookies
- Paginação: `page`, `pageSize` (inicial 20, máximo 100). Resposta: `items`, `page`, `pageSize`, `totalCount`. Ordenação estável com `Id` como desempate. Parâmetros inválidos rejeitados.

## Base de Dados e Configuração

- SQLite em `App_Data/techrepair.db`
- Integridade referencial ativa
- Migrations (não usar `EnsureCreated` com migrations)
- Excluir do Git: base de dados, chaves de Data Protection, ficheiros temporários e segredos

## Dados de Demonstração

Ativação explícita apenas em Development. Criados clientes/equipamentos/técnicos/reparações/intervenções fictícias sem duplicação. Sem contas com passwords fixas.

## Acesso do Futuro Cliente Blazor

- Preferir cliente e API no mesmo endereço (sem necessidade de CORS)
- Se origens diferentes em desenvolvimento: lista explícita em configuração, credenciais apenas para essas origens. Nunca `AllowAnyOrigin` com `AllowCredentials`
- Usar HTTPS. Documentar requisitos de cookies e antiforgery
- Não guardar dados de autenticação em `localStorage`

## Testes

Testes de integração com SQLite temporário/isolado (sem `InMemory` provider). Cobrem: login/logout, 401/403, antiforgery, contas desativadas, registo público, CRUD, cálculos, validações, eliminações com dependências, transições/datas, restrições por técnico, permissões em pesquisas/históricos/resumos, proteção do último administrador ativo.

## Preservação de Dados e Segurança

- Preservar base de dados (`App_Data/techrepair.db`) e chaves de Data Protection
- Nunca commitar segredos, passwords, cookies, DB ou chaves
- Desativação em vez de eliminação
- Transações em operações multi-registo
- Autorização no servidor (ocultar botões não substitui autorização)

## Limitações Conhecidas

- Sem recuperação de password por email nesta versão
- Sem gestão de inventário nesta versão
- Sem eliminação de reparações nesta versão
- Sem sessão persistente nesta versão
- Cliente Blazor WebAssembly PWA a desenvolver em fase posterior
- Sem Views MVC nesta fase
