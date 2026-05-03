# Prompt Fase 1 — Setup do projeto

> Cole o conteúdo abaixo no Claude Code estando no diretório do projeto, na branch `feature/fase-1-setup`.

---

## Prompt para o agente

```
Olá! Você é o agente responsável por implementar a API "Meu Histórico Veicular".

ANTES DE COMEÇAR, leia obrigatoriamente os dois arquivos da raiz do projeto:
1. README.md
2. SPEC.md

Eles contêm todas as decisões de arquitetura, regras de negócio e fases do projeto.

CONTEXTO:
- Estamos na branch feature/fase-1-setup
- Esta é a Fase 1 do roadmap (ver seção 13 do SPEC.md)
- Foque APENAS no escopo da Fase 1. Não adiante código de outras fases.

ESCOPO DA FASE 1 (conforme SPEC.md seção 13):
- Setup Docker + .NET 9
- Estrutura Clean Architecture (5 projetos: API, Domain, Application, Infrastructure, CrossCutting)
- ASP.NET Core Identity configurado
- JWT Bearer com refresh token
- OAuth Google e Facebook (endpoints prontos, validando token do provedor)
- Migrations base + Seed inicial (planos, roles, usuário admin)
- Verificação obrigatória de e-mail (bloqueia login até confirmar)
- Recuperação de senha por e-mail (forgot/reset)
- Recuperação de senha por telefone (OTP via SMS/WhatsApp — pode usar mock/log inicialmente)
- Telefone como campo obrigatório no Usuario
- docker-compose.yml com api + db (MySQL 8) + adminer
- docker-compose.test.yml para testes E2E
- .env.example com todas as variáveis listadas no SPEC seção 10

INSTRUÇÕES DE TRABALHO:
1. PRIMEIRO, faça um plano detalhado dos arquivos que você vai criar e me mostre antes de executar
2. AGUARDE minha aprovação do plano antes de começar a criar arquivos
3. Trabalhe em commits pequenos e atômicos (ex: "chore: setup solution structure", "feat: configura Identity", "feat: adiciona JWT")
4. Use português nos nomes de entidades de domínio (Usuario, Veiculo, Servico) conforme SPEC
5. Use inglês em código de infraestrutura (Program.cs, Startup, configurações técnicas)
6. Para o provedor de SMS/WhatsApp, crie uma INTERFACE IMensagemService e uma implementação MOCK que loga no console. A escolha do provedor real é uma decisão em aberto.
7. Inclua testes E2E básicos para os fluxos da Fase 1 (lista no SPEC seção 12)
8. NÃO faça push automático. Eu reviso os commits localmente antes de subir.

OBSERVAÇÕES IMPORTANTES:
- .NET 9 é obrigatório
- MySQL 8 com Pomelo.EntityFrameworkCore.MySql
- Senha do admin padrão deve vir de variável de ambiente ADMIN_PASSWORD
- Criptografia AES-256 para CPF, RENAVAM e Chassi (campos sensíveis)
- Mascaramento na camada Application

Comece lendo os arquivos e me apresentando o plano.
```

---

## Dicas para economizar créditos

### O que faz o Claude Code consumir menos:
- **Plano antes da execução** — o prompt acima força isso, evitando refazer trabalho
- **Commits pequenos** — você pode interromper e retomar sem perder contexto
- **Escopo travado por fase** — não deixar o agente "melhorar" coisas fora do combinado
- **Você revisa localmente antes do push** — evita ciclos longos de correção

### O que consome muito:
- Pedir refatorações grandes depois que o código está pronto
- Alterar decisões de arquitetura no meio da fase
- Não ler o que o agente gerou e pedir para "fazer de novo"

---

## Após a Fase 1

Quando o agente terminar e você aprovar localmente:

```bash
git push origin feature/fase-1-setup
```

Abra um Pull Request no GitHub de `feature/fase-1-setup` → `develop`. Revise o diff visualmente, faça o merge, e voltamos aqui para preparar o prompt da Fase 2.
