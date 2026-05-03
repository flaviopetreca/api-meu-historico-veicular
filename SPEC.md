# Especificação Técnica — Meu Histórico Veicular

> **Versão:** 1.4  
> **Data:** 2026-05-03  
> **Status:** Em refinamento  
> **Alterações v1.1:** Adicionado suporte a login via Google e Facebook (OAuth 2.0)  
> **Alterações v1.2:** Adicionada verificação obrigatória de e-mail para contas e-mail/senha; login social confia na verificação do provedor  
> **Alterações v1.3:** Telefone tornado campo obrigatório; adicionada recuperação de senha via SMS ou WhatsApp (usuário escolhe); 2FA via telefone previsto para fase futura  
> **Alterações v1.4:** Dados sensíveis (CPF, RENAVAM, Chassi) movidos para tabelas vault isoladas com auditoria nativa; endpoints específicos para leitura/edição com rate limiting

---

## Índice

- [1. Visão do Produto](#1-visão-do-produto)
- [2. Stack Tecnológica](#2-stack-tecnológica)
- [3. Arquitetura](#3-arquitetura)
- [4. Perfis de Usuário e Permissões](#4-perfis-de-usuário-e-permissões)
- [5. Planos de Acesso](#5-planos-de-acesso)
- [6. Modelagem de Dados](#6-modelagem-de-dados)
- [7. Módulos e Regras de Negócio](#7-módulos-e-regras-de-negócio)
- [8. Segurança e Dados Sensíveis](#8-segurança-e-dados-sensíveis)
- [9. Integrações Externas](#9-integrações-externas)
- [10. Infraestrutura Docker](#10-infraestrutura-docker)
- [11. Migrations e Seed](#11-migrations-e-seed)
- [12. Testes](#12-testes)
- [13. Fases de Entrega](#13-fases-de-entrega)
- [14. Decisões em Aberto](#14-decisões-em-aberto)

---

## 1. Visão do Produto

### Objetivo

Centralizar as informações de veículos que uma pessoa possui ou já possuiu, garantindo que dados, manutenções e documentos estejam sempre acessíveis de forma prática e segura.

### Problemas que resolve

- Histórico de manutenções perdido na troca de veículo ou oficina
- Documentos (notas fiscais, garantias) dispersos ou extraviados
- Falta de controle sobre serviços realizados e seus prazos de garantia
- Dificuldade em transferir o histórico ao vender o veículo

### Público-alvo

Proprietários de veículos individuais e frotas pequenas, oficinas mecânicas e concessionárias autorizadas.

---

## 2. Stack Tecnológica

| Camada | Tecnologia | Versão | Justificativa |
|---|---|---|---|
| Runtime | .NET | 9 | Última versão estável, suporte LTS |
| Framework Web | ASP.NET Core | 9 | Padrão para APIs .NET |
| ORM | Entity Framework Core | 9 | Integração nativa com Identity e migrations |
| Autenticação | ASP.NET Core Identity | 9 | Gerenciamento de usuários, roles e claims |
| Login Social | OAuth 2.0 (Google, Facebook) | — | Autenticação federada via provedores externos |
| Token | JWT Bearer | — | Stateless, compatível com FlutterFlow |
| Banco de dados | MySQL | 8.x | Já disponível na infraestrutura do cliente |
| Containerização | Docker + Docker Compose | latest | Portabilidade e isolamento de ambiente |
| Documentação | Swagger / OpenAPI | — | Contrato da API para integração com FlutterFlow |
| Testes E2E | xUnit | — | Padrão .NET, integração com CI |
| Frontend | FlutterFlow | — | App iOS/Android + Web a partir de um único projeto |

---

## 3. Arquitetura

### Padrão

**Clean Architecture** — separação em camadas com dependências sempre apontando para o domínio.

```
MeuHistoricoVeicular/
├── src/
│   ├── MHV.API/              # Controllers, Middlewares, Filters, Program.cs
│   ├── MHV.Domain/           # Entidades, Enums, Interfaces de repositório, Value Objects
│   ├── MHV.Application/      # Casos de uso, Services, DTOs, mapeamentos, validações
│   ├── MHV.Infrastructure/   # EF Core DbContext, Repositórios, Identity, acesso a arquivos
│   └── MHV.CrossCutting/     # JWT, Logging, extensões, helpers globais
├── tests/
│   └── MHV.E2E.Tests/        # Testes end-to-end com banco MySQL real em container
├── docker-compose.yml
├── docker-compose.test.yml
├── Dockerfile
└── .env.example
```

### Fluxo de uma requisição

```
FlutterFlow
  → HTTPS + JWT
    → MHV.API (Controller)
      → MHV.Application (Service / Caso de uso)
        → MHV.Domain (validação de regra)
        → MHV.Infrastructure (repositório / EF Core)
          → MySQL
```

### Containers

| Container | Imagem | Função |
|---|---|---|
| `api` | `mcr.microsoft.com/dotnet/aspnet:9.0` | Aplicação principal |
| `db` | `mysql:8` | Banco de dados |
| `adminer` | `adminer` | Interface web do banco (dev only) |

Volume nomeado `mhv_uploads` para persistência de arquivos (notas fiscais e documentos).

---

## 4. Perfis de Usuário e Permissões

Implementado via **ASP.NET Core Identity Roles + Claims**.

### Roles

| Role | Descrição |
|---|---|
| `Admin` | Acesso irrestrito. Cria usuários do sistema, define planos e permissões |
| `UsuarioSistema` | Gerencia contas de proprietários e oficinas |
| `Proprietario` | CRUD de veículos e serviços, transferência, autoriza oficinas |
| `Oficina` | Insere serviços em veículos autorizados pelo proprietário |
| `OficinaAutorizada` | Idem `Oficina` + flag de autorização pela montadora |

### Matriz de permissões por recurso

| Recurso | Admin | UsuarioSistema | Proprietario | Oficina | OficinaAutorizada |
|---|:---:|:---:|:---:|:---:|:---:|
| Gerenciar usuários | ✅ | ✅ | ❌ | ❌ | ❌ |
| CRUD veículos | ✅ | ✅ | ✅ (próprios) | ❌ | ❌ |
| Ver histórico do veículo | ✅ | ✅ | ✅ (próprios) | ✅ (autorizados) | ✅ (autorizados) |
| Inserir serviço | ✅ | ✅ | ✅ | ✅ (autorizados) | ✅ (autorizados) |
| Upload de documentos | ✅ | ✅ | ✅ | ✅ (autorizados) | ✅ (autorizados) |
| Transferir veículo | ✅ | ✅ | ✅ (próprios) | ❌ | ❌ |
| Ver dados completos (sem máscara) | ✅ | ❌ | ✅ (próprios) | ❌ | ❌ |
| Gerenciar planos | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## 5. Planos de Acesso

O plano é vinculado ao usuário `Proprietario` e validado via middleware na API.

| Plano | Veículos | Upload NF | Permissão para oficinas | Observações |
|---|---|---|---|---|
| **Básico** | Até 2 | ❌ | ❌ | Consulta por placa, serviços manuais |
| **Plus** | Até 5 | ✅ | ❌ | Histórico completo, controle de garantia |
| **Profissional** | Ilimitado | ✅ | ✅ | Todos os recursos, API para oficinas |

> ⚠️ **Decisão em aberto:** valores de assinatura e forma de cobrança (ver seção 14).

---

## 6. Modelagem de Dados

### Entidades principais

#### `Usuario` (gerenciado pelo ASP.NET Core Identity — `ApplicationUser`)
| Campo | Tipo | Obrigatório | Observações |
|---|---|:---:|---|
| `Id` | `Guid` | ✅ | PK |
| `Nome` | `string` | ✅ | |
| `Email` | `string` | ✅ | Único no sistema |
| `Telefone` | `string` | ✅ | Formato E.164 (ex: `+5511999999999`). Usado para recuperação de senha e futuro 2FA |
| `TelefoneVerificado` | `bool` | ✅ | Indica se o telefone passou por verificação via código |
| `CanalRecuperacao` | `enum` | ❌ | `SMS` ou `WhatsApp`. Preferência do usuário para envio de código |
| `PlanoId` | `Guid` | ✅ | FK → Plano |
| `Ativo` | `bool` | ✅ | Soft delete |
| `CriadoEm` | `DateTime` | ✅ | |

> Logins externos (Google, Facebook) são armazenados na tabela padrão do Identity `AspNetUserLogins`, com os campos `LoginProvider` (`Google`/`Facebook`) e `ProviderKey` (ID do usuário no provedor).

> O campo `PhoneNumber` nativo do Identity é reutilizado para `Telefone`. `TelefoneVerificado` e `CanalRecuperacao` são campos customizados adicionados ao `ApplicationUser`.

> ⚠️ **CPF não está nesta tabela.** Dados sensíveis ficam isolados em `UsuarioDadosSensiveis` (ver seção do vault abaixo).

#### `UsuarioDadosSensiveis` _(vault isolado)_
| Campo | Tipo | Obrigatório | Observações |
|---|---|:---:|---|
| `Id` | `Guid` | ✅ | PK |
| `UsuarioId` | `Guid` | ✅ | FK → Usuario (relacionamento 1:1, único) |
| `Cpf` | `string` | ❌ | Criptografado AES-256. Opcional para contas OAuth |
| `RG` | `string` | ❌ | Criptografado AES-256 (reservado para uso futuro) |
| `AtualizadoEm` | `DateTime` | ✅ | |
| `AcessadoEm` | `DateTime?` | ❌ | Última leitura completa do dado (auditoria) |
| `AcessadoPorId` | `Guid?` | ❌ | FK → Usuario que fez o último acesso |

#### `Veiculo`
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `Placa` | `string` | Única no sistema |
| `Marca` | `string` | |
| `Modelo` | `string` | |
| `AnoFabricacao` | `int` | |
| `AnoModelo` | `int` | |
| `Cor` | `string` | |
| `CriadoEm` | `DateTime` | |

> ⚠️ **RENAVAM e Chassi não estão nesta tabela.** Dados sensíveis ficam isolados em `VeiculoDadosSensiveis` (ver vault abaixo).

#### `VeiculoDadosSensiveis` _(vault isolado)_
| Campo | Tipo | Obrigatório | Observações |
|---|---|:---:|---|
| `Id` | `Guid` | ✅ | PK |
| `VeiculoId` | `Guid` | ✅ | FK → Veiculo (relacionamento 1:1, único) |
| `Renavam` | `string` | ❌ | Criptografado AES-256 |
| `Chassi` | `string` | ❌ | Criptografado AES-256 |
| `AtualizadoEm` | `DateTime` | ✅ | |
| `AcessadoEm` | `DateTime?` | ❌ | Última leitura completa do dado (auditoria) |
| `AcessadoPorId` | `Guid?` | ❌ | FK → Usuario que fez o último acesso |

#### `VeiculoProprietario` _(tabela de relacionamento com histórico)_
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `VeiculoId` | `Guid` | FK → Veiculo |
| `ProprietarioId` | `Guid` | FK → Usuario |
| `DataInicio` | `DateTime` | Início da posse |
| `DataFim` | `DateTime?` | Null = proprietário atual |
| `Ativo` | `bool` | |

#### `Servico`
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `VeiculoId` | `Guid` | FK → Veiculo |
| `ProprietarioId` | `Guid` | FK → Usuario (quem era dono na época) |
| `RealizadoPorId` | `Guid?` | FK → Usuario (oficina, se aplicável) |
| `Descricao` | `string` | |
| `LocalRealizacao` | `string` | Nome da oficina/estabelecimento |
| `DataRealizacao` | `DateTime` | |
| `KmNaRealizacao` | `int?` | |
| `ValorTotal` | `decimal?` | |
| `GarantiaMeses` | `int?` | Prazo de garantia em meses |
| `DataFimGarantia` | `DateTime?` | Calculado automaticamente |
| `Observacoes` | `string?` | |
| `CriadoEm` | `DateTime` | |

#### `Documento`
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `ServicoId` | `Guid` | FK → Servico |
| `Tipo` | `enum` | `NotaFiscal`, `Orcamento`, `Garantia`, `Outro` |
| `NomeArquivo` | `string` | Nome original do arquivo |
| `CaminhoArquivo` | `string` | Caminho no volume Docker |
| `TamanhoBytes` | `long` | |
| `MimeType` | `string` | |
| `CriadoEm` | `DateTime` | |

#### `PermissaoOficina`
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `VeiculoId` | `Guid` | FK → Veiculo |
| `ProprietarioId` | `Guid` | FK → Usuario (quem concedeu) |
| `OficinaId` | `Guid` | FK → Usuario (quem recebeu) |
| `Ativo` | `bool` | Permite revogar sem deletar |
| `CriadoEm` | `DateTime` | |
| `RevogadoEm` | `DateTime?` | |

#### `Plano`
| Campo | Tipo | Observações |
|---|---|---|
| `Id` | `Guid` | PK |
| `Nome` | `string` | Básico, Plus, Profissional |
| `MaxVeiculos` | `int` | -1 = ilimitado |
| `PermiteUpload` | `bool` | |
| `PermiteOficinas` | `bool` | |

---

## 7. Módulos e Regras de Negócio

### 7.1 Autenticação (`/auth`)

#### Métodos de login suportados

| Método | Endpoint | Observações |
|---|---|---|
| E-mail + senha | `POST /auth/login` | Credenciais gerenciadas pelo Identity |
| Google | `POST /auth/google` | Recebe o `id_token` do Google, valida e emite JWT interno |
| Facebook | `POST /auth/facebook` | Recebe o `access_token` do Facebook, valida e emite JWT interno |

#### Fluxo OAuth (Google e Facebook)

1. O FlutterFlow realiza o login social no lado do cliente e obtém o token do provedor
2. O app envia o token para a API (`id_token` para Google, `access_token` para Facebook)
3. A API valida o token diretamente com o provedor (Google APIs / Facebook Graph API)
4. Se válido, a API verifica se já existe um usuário com aquele e-mail no banco:
   - **Existe:** emite JWT interno normalmente
   - **Não existe:** cria automaticamente o usuário com role `Proprietario` e plano `Básico`, depois emite JWT
5. O JWT interno é retornado ao app — a partir daí o fluxo é idêntico ao login por e-mail

> O Identity armazena os logins externos na tabela `AspNetUserLogins` (padrão do framework), vinculando `provider` + `providerKey` ao usuário local.

#### Verificação de e-mail

| Método de login | Exige verificação de e-mail? | Comportamento |
|---|---|---|
| E-mail + senha | ✅ Sim | Acesso **bloqueado** até o e-mail ser confirmado |
| Google | ❌ Não | E-mail considerado verificado pelo provedor |
| Facebook | ❌ Não | E-mail considerado verificado pelo provedor |

**Fluxo de verificação para contas e-mail/senha:**

1. Usuário se registra com e-mail e senha
2. API envia e-mail com link de confirmação contendo token gerado pelo Identity (`EmailConfirmationToken`)
3. Qualquer tentativa de login antes da confirmação retorna `HTTP 403` com mensagem `"E-mail não verificado. Verifique sua caixa de entrada."`
4. O link de confirmação aponta para `POST /auth/confirm-email?userId=&token=`
5. Após confirmação, o campo `EmailConfirmed = true` é gravado no Identity e o acesso é liberado
6. O usuário pode solicitar reenvio do e-mail via `POST /auth/resend-confirmation`

> O Identity já possui o campo `EmailConfirmed` na tabela `AspNetUsers` — nenhuma coluna extra é necessária.

#### Recuperação de senha

Disponível apenas para contas criadas com e-mail/senha. Contas OAuth (Google, Facebook) não possuem senha no sistema e não são elegíveis para este fluxo.

O usuário pode recuperar a senha por **dois canais**, à sua escolha: **e-mail** ou **telefone** (SMS ou WhatsApp, conforme preferência cadastrada em `CanalRecuperacao`).

---

**Fluxo por e-mail:**

1. Usuário informa o e-mail em "Esqueci minha senha"
2. `POST /auth/forgot-password/email` — API verifica o e-mail e envia link com `PasswordResetToken`
3. O link abre a tela de redefinição com `userId` e `token` como parâmetros
4. `POST /auth/reset-password` — API valida o token e redefine a senha
5. Token expira após uso único ou **24 horas**
6. Após redefinição, todos os refresh tokens ativos são invalidados

---

**Fluxo por telefone (SMS ou WhatsApp):**

1. Usuário informa o e-mail ou telefone em "Esqueci minha senha" e escolhe receber código por telefone
2. `POST /auth/forgot-password/phone` — API localiza o usuário e envia um **código OTP de 6 dígitos** via SMS ou WhatsApp (conforme `CanalRecuperacao` ou escolha pontual)
3. Código tem validade de **10 minutos** e é de uso único — armazenado em cache (Redis ou tabela temporária) com hash
4. `POST /auth/verify-phone-otp` — usuário informa o código recebido; API valida e retorna um `reset_token` temporário (válido por 5 minutos)
5. `POST /auth/reset-password` — usuário informa nova senha + `reset_token`; API redefine a senha
6. Após redefinição, todos os refresh tokens ativos são invalidados

> A API **nunca informa** se o e-mail ou telefone existe no sistema — sempre retorna `HTTP 200` para evitar enumeração de usuários.

> **Limite de tentativas:** máximo de 5 envios de OTP por hora por usuário para evitar abuso (rate limiting).

---

**Provedor de mensagens (SMS/WhatsApp):**

| Canal | Provedor sugerido | Observações |
|---|---|---|
| SMS | Twilio ou Zenvia | APIs consolidadas no Brasil |
| WhatsApp | Twilio (API oficial) ou Z-API | Z-API é opção mais acessível para o Brasil |

> ⚠️ **Decisão em aberto:** escolha do provedor de SMS/WhatsApp (ver seção 14).

---

**2FA via telefone (previsto para fase futura):**

A infraestrutura de OTP por telefone construída para recuperação de senha será reutilizada para autenticação em dois fatores. O campo `TelefoneVerificado` já prepara o usuário para habilitar o 2FA quando implementado.

#### Endpoints

- `POST /auth/login` — login por e-mail/senha, retorna JWT + refresh token
- `POST /auth/google` — login via Google (recebe `id_token`)
- `POST /auth/facebook` — login via Facebook (recebe `access_token`)
- `POST /auth/register` — registro manual por e-mail/senha (dispara e-mail de confirmação)
- `POST /auth/confirm-email` — confirmação do e-mail via token (query params: `userId`, `token`)
- `POST /auth/resend-confirmation` — reenvio do e-mail de confirmação
- `POST /auth/forgot-password/email` — solicita recuperação de senha por e-mail
- `POST /auth/forgot-password/phone` — solicita código OTP por SMS ou WhatsApp
- `POST /auth/verify-phone-otp` — valida o OTP e retorna reset_token temporário
- `POST /auth/reset-password` — redefine a senha (aceita token por e-mail ou reset_token por telefone)
- `POST /auth/refresh` — renovação de token (refresh token rotacionado)
- `POST /auth/change-password` — troca de senha autenticado (apenas contas com senha cadastrada)

#### JWT (todos os métodos)

- Algoritmo: **HS256**
- Chave mínima: 256 bits, configurada via variável de ambiente
- Claims incluídas: `sub`, `email`, `role`, `plano`, `jti` (para blacklist de refresh tokens)
- Expiração: **60 minutos** para access token, **7 dias** para refresh token
- Refresh token é rotacionado a cada uso (invalidado após uso)

### 7.2 Veículos (`/veiculos`)

- Ao cadastrar, o usuário informa a placa
- O sistema consulta a API pública de placas e pré-preenche os campos
- Os dados retornados podem ser editados antes de salvar
- O veículo não precisa estar no nome do usuário que cadastra
- Um veículo pode ter múltiplos proprietários ao longo do tempo (histórico via `VeiculoProprietario`)
- O limite de veículos é validado pelo plano do usuário no momento do cadastro

### 7.3 Serviços e Manutenções (`/servicos`)

- Serviços são sempre vinculados a um veículo e ao proprietário ativo no momento
- Se `GarantiaMeses` for informado, `DataFimGarantia` é calculada automaticamente
- Oficinas só podem inserir serviços em veículos para os quais foram autorizadas pelo proprietário
- Não é permitido editar ou excluir serviços inseridos por outro usuário (apenas o Admin)

### 7.4 Documentos (`/documentos`)

- Upload aceito: PDF, JPG, PNG, até **10 MB** por arquivo
- Arquivos armazenados em volume Docker no caminho: `/app/uploads/{veiculoId}/{servicoId}/`
- Acesso ao arquivo protegido por autenticação — não há URL pública direta
- Download retorna o arquivo via streaming com o nome original

### 7.5 Transferência de Veículo

**Fluxo completo:**

1. Proprietário A inicia a transferência informando o email do Proprietário B
2. Sistema valida se o email existe e pertence a um `Proprietario`
3. Sistema valida se B tem capacidade no plano para receber mais um veículo
4. Uma solicitação de transferência é criada com status `Pendente`
5. Proprietário B recebe notificação e confirma (ou recusa) no app
6. Ao confirmar:
   - `VeiculoProprietario` de A recebe `DataFim = agora` e `Ativo = false`
   - Novo `VeiculoProprietario` criado para B com `DataInicio = agora`
7. Histórico anterior fica visível para ambos (somente leitura para A)
8. Permissões de oficinas concedidas por A **não são transferidas**

### 7.6 Permissões para Oficinas

- Proprietário acessa o app, seleciona um veículo e autoriza uma oficina (por email ou ID)
- A permissão é por veículo — não é global
- O proprietário pode revogar a permissão a qualquer momento
- Após revogação, a oficina perde acesso imediatamente, mas os serviços já inseridos permanecem

---

## 8. Segurança e Dados Sensíveis

### Arquitetura de Vault Isolado

Dados sensíveis (CPF, RG, RENAVAM, Chassi) **não** ficam nas tabelas principais. Eles são isolados em tabelas vault dedicadas (`UsuarioDadosSensiveis` e `VeiculoDadosSensiveis`) referenciadas por ID.

**Princípios:**

1. Queries normais do sistema **nunca tocam dados sensíveis** — performance e segurança ganham por padrão.
2. Acesso aos dados completos exige **endpoint específico** com autenticação reforçada.
3. Cada acesso ao dado completo é **registrado nativamente no próprio vault** (campos `AcessadoEm` e `AcessadoPorId`).
4. **Mascaramento aplicado por padrão** na exibição. Dado completo só é retornado em endpoints explícitos de edição.
5. **Rate limit** de 10 acessos/hora por usuário nos endpoints de leitura completa.
6. Rotação de chave de criptografia fica trivial (afeta apenas duas tabelas).

### Endpoints do vault

| Endpoint | Permissão | Comportamento |
|---|---|---|
| `GET /usuarios/{id}/dados-sensiveis` | Próprio usuário ou Admin | Retorna dado completo, registra acesso, aplica rate limit |
| `PUT /usuarios/{id}/dados-sensiveis` | Próprio usuário ou Admin | Atualiza apenas o vault, registra acesso |
| `GET /veiculos/{id}/dados-sensiveis` | Proprietário ativo ou Admin | Retorna dado completo, registra acesso, aplica rate limit |
| `PUT /veiculos/{id}/dados-sensiveis` | Proprietário ativo ou Admin | Atualiza apenas o vault, registra acesso |

### Fluxo de edição (exemplo: alterar RENAVAM)

1. App chama `GET /veiculos/{id}` → retorno mascarado (`renavam: "*********23"`)
2. Usuário clica "Editar dados sensíveis"
3. App chama `GET /veiculos/{id}/dados-sensiveis` → API valida permissão, registra acesso, retorna sem máscara
4. Usuário edita e salva → App chama `PUT /veiculos/{id}/dados-sensiveis`
5. App volta para tela de detalhe com dados mascarados novamente

### Mascaramento de dados (exibição padrão)

O mascaramento ocorre na camada **Application** (nos DTOs de resposta).

| Campo | Exibição mascarada | Acesso ao valor completo |
|---|---|---|
| CPF | `***.***.789-09` | `GET /usuarios/{id}/dados-sensiveis` |
| RENAVAM | `*********23` (últimos 2) | `GET /veiculos/{id}/dados-sensiveis` |
| Chassi | `9BW***********1234` (1º + últimos 4) | `GET /veiculos/{id}/dados-sensiveis` |
| Telefone | `(**) *****-5678` | Admin, próprio usuário |
| Placa | Completa | Admin, proprietário, oficina autorizada |

### Permissões de acesso ao vault

| Perfil | CPF próprio | CPF de terceiros | RENAVAM/Chassi |
|---|:---:|:---:|:---:|
| Proprietário | ✅ (próprio) | ❌ | ✅ (próprios veículos) |
| Oficina | ✅ (próprio) | ❌ | ❌ (mesmo autorizada) |
| Admin | ✅ | ✅ (auditado) | ✅ (auditado) |
| UsuarioSistema | ❌ | ❌ | ❌ |

> ⚠️ **Oficinas autorizadas não veem RENAVAM ou Chassi**, mesmo dos veículos para os quais têm permissão de inserir serviços.

### Criptografia em repouso

Campos sensíveis (`Cpf`, `RG`, `Renavam`, `Chassi`) são armazenados criptografados nas tabelas vault usando **AES-256**, com chave configurada via variável de ambiente `ENCRYPTION_KEY`.

### JWT

- Algoritmo: **HS256**
- Chave mínima: 256 bits, configurada via variável de ambiente
- Claims incluídas: `sub`, `email`, `role`, `plano`, `jti` (para blacklist de refresh tokens)
- Válido para todos os métodos de autenticação (e-mail, Google, Facebook)

### Credenciais OAuth (variáveis de ambiente)

As credenciais dos provedores são configuradas via `.env` e nunca expostas no código:

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
FACEBOOK_APP_ID=
FACEBOOK_APP_SECRET=
```

---

## 9. Integrações Externas

### API de Consulta por Placa

- **Finalidade:** Buscar dados do veículo (marca, modelo, ano, cor) a partir da placa
- **Abordagem:** API pública (a definir entre opções disponíveis no mercado brasileiro)
- **Comportamento:** A consulta é feita no momento do cadastro do veículo
- **Fallback:** Se a API estiver indisponível ou não retornar dados, o cadastro manual segue normalmente
- **Custo:** A ser avaliado conforme API escolhida

> ⚠️ **Decisão em aberto:** qual API de placas utilizar (ver seção 14).

---

## 10. Infraestrutura Docker

### `docker-compose.yml` (desenvolvimento)

```yaml
services:
  api:
    build: .
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Server=db;Database=${DB_NAME};User=${DB_USER};Password=${DB_PASSWORD}
      - Jwt__Secret=${JWT_SECRET}
      - Upload__Path=/app/uploads
    volumes:
      - mhv_uploads:/app/uploads
    depends_on:
      - db

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mhv_db:/var/lib/mysql

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  mhv_db:
  mhv_uploads:
```

### Variáveis de ambiente (`.env.example`)

```env
DB_NAME=mhv_db
DB_USER=mhv_user
DB_PASSWORD=senha_segura
DB_ROOT_PASSWORD=root_senha_segura
JWT_SECRET=chave_jwt_minimo_32_caracteres
JWT_EXPIRY_MINUTES=60
JWT_REFRESH_EXPIRY_DAYS=7
UPLOAD_PATH=/app/uploads
ENCRYPTION_KEY=chave_aes_256_bits
PLACA_API_URL=https://api-de-placas.com.br
PLACA_API_KEY=sua_chave_aqui
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
FACEBOOK_APP_ID=
FACEBOOK_APP_SECRET=
```

---

## 11. Migrations e Seed

### Estratégia

- Todas as alterações de schema via **EF Core Migrations** — nunca scripts SQL manuais
- Migration inicial cria o schema completo + seed de dados
- Migrations de testes rodam em banco isolado e são revertidas após cada suite

### Seed inicial (migration `InitialSeed`)

- Planos: Básico, Plus, Profissional
- Roles: Admin, UsuarioSistema, Proprietario, Oficina, OficinaAutorizada
- Usuário admin padrão: `admin@mhv.com.br` / senha via variável de ambiente

### Comandos

```bash
# Aplicar migrations
dotnet ef database update \
  --project src/MHV.Infrastructure \
  --startup-project src/MHV.API

# Nova migration
dotnet ef migrations add NomeDaMigration \
  --project src/MHV.Infrastructure \
  --startup-project src/MHV.API

# Reverter última migration
dotnet ef database update PreviousMigrationName \
  --project src/MHV.Infrastructure \
  --startup-project src/MHV.API
```

---

## 12. Testes

### Estratégia de testes

| Tipo | Ferramenta | Escopo |
|---|---|---|
| E2E | xUnit + `docker-compose.test.yml` | Fluxos completos via HTTP |
| Integração | xUnit + EF Core InMemory | Repositórios e services |
| Unitário | xUnit | Regras de domínio puras |

### Cenários E2E obrigatórios por fase

- **Fase 1:** Login por e-mail, bloqueio antes da verificação, confirmação de e-mail, reenvio de confirmação, recuperação por e-mail (forgot + reset), recuperação por telefone (OTP via SMS/WhatsApp + reset), login via Google, login via Facebook, registro, refresh token
- **Fase 2:** CRUD veículo, consulta por placa
- **Fase 3:** CRUD serviço, upload e download de documento
- **Fase 4:** Transferência de veículo (fluxo completo), autorização de oficina
- **Fase 5:** Validação de limites de plano, mascaramento de dados
- **Fase 6:** Cobertura completa de todos os endpoints documentados no Swagger

---

## 13. Fases de Entrega

| Fase | Entregável | Dependências |
|---|---|---|
| **1** | Setup Docker + .NET 9 + Identity + JWT + OAuth (Google, Facebook) + Migrations base + Seed | — |
| **2** | Módulo Veículos: CRUD + consulta por placa + validação de plano | Fase 1 |
| **3** | Módulo Serviços + Upload/download de documentos | Fase 2 |
| **4** | Transferência de veículo + Permissões para oficinas | Fase 3 |
| **5** | Planos de acesso + Mascaramento de dados sensíveis | Fase 4 |
| **6** | Testes E2E completos + Swagger documentado | Fase 5 |
| **7** | Guia de integração FlutterFlow (autenticação JWT, endpoints, exemplos) | Fase 6 |

### Convenção de branches

```
main          ← código aprovado e estável
develop       ← integração contínua
feature/fase-1-setup
feature/fase-2-veiculos
feature/fase-3-servicos
feature/fase-4-transferencia
feature/fase-5-planos
feature/fase-6-testes
feature/fase-7-flutterflow
```

---

## 14. Decisões em Aberto

| # | Decisão | Opções | Impacto |
|---|---|---|---|
| 1 | API de consulta por placa | FIPE, Detran estaduais, APIs de mercado (ex: PlacaFipe, ApiPlaca) | Custo, disponibilidade, dados retornados |
| 2 | Valores dos planos de assinatura | A definir pelo cliente | Regras de upgrade/downgrade |
| 3 | Forma de cobrança/billing | Stripe, PagSeguro, Mercado Pago, manual | Integração adicional fora do escopo atual |
| 4 | Notificações (transferência, garantia vencendo) | Push via FlutterFlow, e-mail, SMS | Complexidade e custo de infraestrutura |
| 5 | Limite de tamanho por upload e total por usuário | Definir por plano | Custo de armazenamento a longo prazo |
| 6 | Versão exata do MySQL disponível | Confirmar com o cliente | Compatibilidade com EF Core provider |
| 7 | Provedor de SMS/WhatsApp para OTP | Twilio, Zenvia (SMS) · Twilio API oficial, Z-API (WhatsApp) | Custo por mensagem, facilidade de integração no Brasil |

---

> Este documento deve ser atualizado a cada decisão tomada e a cada fase concluída.
