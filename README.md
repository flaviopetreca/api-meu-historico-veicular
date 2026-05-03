# Meu Histórico Veicular

Sistema para controle e gestão do histórico de veículos — manutenções, documentos e transferências.

> Para detalhes de arquitetura, regras de negócio e decisões técnicas, consulte [SPEC.md](./SPEC.md).

---

## Pré-requisitos

- [Docker](https://www.docker.com/) e Docker Compose
- [.NET 9 SDK](https://dotnet.microsoft.com/download) (opcional, para rodar fora do container)

---

## Como rodar

```bash
# 1. Clone o repositório
git clone https://github.com/seu-usuario/meu-historico-veicular.git
cd meu-historico-veicular

# 2. Configure as variáveis de ambiente
cp .env.example .env
# Edite o .env com suas configurações antes de continuar

# 3. Suba os containers
docker-compose up -d

# 4. Acesse a documentação da API
# http://localhost:5000/swagger
```

As migrations e o seed inicial são aplicados automaticamente na primeira inicialização da API.

---

## Variáveis de ambiente

Copie `.env.example` para `.env` e preencha os valores:

```env
DB_NAME=mhv_db
DB_USER=mhv_user
DB_PASSWORD=
DB_ROOT_PASSWORD=
JWT_SECRET=                  # mínimo 32 caracteres
JWT_EXPIRY_MINUTES=60
JWT_REFRESH_EXPIRY_DAYS=7
ENCRYPTION_KEY=              # chave AES-256
PLACA_API_URL=
PLACA_API_KEY=
```

---

## Testes E2E

```bash
docker-compose -f docker-compose.test.yml up --abort-on-container-exit
```

---

## Stack

- **API:** ASP.NET Core 9 + Entity Framework Core 9 + ASP.NET Core Identity
- **Banco:** MySQL 8.x
- **Auth:** JWT Bearer
- **Container:** Docker + Docker Compose
- **Frontend:** FlutterFlow (App + Web)

---

## Usuário admin padrão

Após o seed inicial:

- **Email:** `admin@mhv.com.br`
- **Senha:** definida via variável de ambiente `ADMIN_PASSWORD`
