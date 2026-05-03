# Arquitetura — Vault de Dados Sensíveis

> Padrão para isolar CPF, RENAVAM e Chassi do restante do sistema.
> Documento para revisão antes de aplicar na modelagem.

---

## O problema

Hoje a modelagem tinha CPF dentro de USUARIO e RENAVAM/Chassi dentro de VEICULO.
Mesmo criptografados, esses dados:

- aparecem em todo SELECT que carrega o objeto
- correm risco de leak por logs de SQL, cache de EF Core, serialização JSON
- precisam de mascaramento manual em CADA endpoint que retorna USUARIO ou VEICULO
- dificultam auditoria (quem acessou quê?)

---

## A solução: Vault separado

Criar duas tabelas isoladas (uma por tipo de entidade) que guardam APENAS
os dados sensíveis. As tabelas USUARIO e VEICULO referenciam o vault por ID.

Resultado:
- 99% das queries do sistema NUNCA tocam dados sensíveis
- Mascaramento deixa de ser por campo e passa a ser por endpoint
- Auditoria fica trivial: basta logar acessos ao vault
- Permissão de acesso fica granular

---

## Novas tabelas

### USUARIO_DADOS_SENSIVEIS

| Campo          | Tipo     | PK | Obrig | Observações                          |
|----------------|----------|----|-------|--------------------------------------|
| Id             | Guid     | X  | sim   |                                      |
| UsuarioId      | Guid FK  |    | sim   | -> USUARIO.Id (1:1, único)           |
| Cpf            | string   |    | não   | Criptografado AES-256                |
| RG             | string   |    | não   | Criptografado AES-256 (futuro)       |
| AtualizadoEm   | DateTime |    | sim   |                                      |
| AcessadoEm     | DateTime |    | não   | Última leitura do dado completo      |
| AcessadoPorId  | Guid?    |    | não   | Quem foi o último a ler (auditoria)  |

### VEICULO_DADOS_SENSIVEIS

| Campo          | Tipo     | PK | Obrig | Observações                          |
|----------------|----------|----|-------|--------------------------------------|
| Id             | Guid     | X  | sim   |                                      |
| VeiculoId      | Guid FK  |    | sim   | -> VEICULO.Id (1:1, único)           |
| Renavam        | string   |    | não   | Criptografado AES-256                |
| Chassi         | string   |    | não   | Criptografado AES-256                |
| AtualizadoEm   | DateTime |    | sim   |                                      |
| AcessadoEm     | DateTime |    | não   | Última leitura do dado completo      |
| AcessadoPorId  | Guid?    |    | não   | Quem foi o último a ler (auditoria)  |

---

## Mudanças nas tabelas existentes

### USUARIO (remover Cpf)

ANTES:
- ... Cpf (criptografado) ...

DEPOIS:
- Cpf NÃO existe mais aqui
- Existe USUARIO_DADOS_SENSIVEIS.UsuarioId apontando pra cá

### VEICULO (remover Renavam, Chassi)

ANTES:
- ... Renavam (criptografado) ...
- ... Chassi (criptografado) ...

DEPOIS:
- Renavam e Chassi NÃO existem mais aqui
- Existe VEICULO_DADOS_SENSIVEIS.VeiculoId apontando pra cá

---

## Como a API se comporta

### Listagem de veículos

Endpoint: GET /veiculos
Retorno (sem mascaramento, sem dados sensíveis na query):
```
[
  { "id": "...", "placa": "ABC1D23", "marca": "Fiat", "modelo": "Argo" }
]
```

A query nem chega perto dos dados sensíveis. Performance e segurança ganham.

### Detalhe de veículo (modo visualização — padrão)

Endpoint: GET /veiculos/{id}
Retorno mascarado:
```
{
  "id": "...",
  "placa": "ABC1D23",
  "marca": "Fiat",
  "modelo": "Argo",
  "renavam": "*********23",
  "chassi": "9BW***********1234"
}
```

Os dados sensíveis vêm mascarados. A API consulta o vault MAS aplica máscara
antes de retornar. O usuário comum nunca recebe os dados completos.

### Detalhe de veículo (modo edição — sob demanda)

Endpoint: GET /veiculos/{id}/dados-sensiveis
Permissões: apenas proprietário atual ou admin
Retorno completo (sem máscara):
```
{
  "id": "...",
  "renavam": "12345678923",
  "chassi": "9BWZZZ377VT004251234"
}
```

Este endpoint:
- Exige autenticação JWT
- Exige que o usuário seja o proprietário ATIVO do veículo (ou admin)
- Registra na tabela do vault: AcessadoEm = agora, AcessadoPorId = usuario
- Pode ter rate limit (ex: 10 acessos/hora por usuário)
- Loga em sistema de auditoria separado

### Atualização de dados sensíveis

Endpoint: PUT /veiculos/{id}/dados-sensiveis
Permissões: apenas proprietário atual ou admin
Body:
```
{ "renavam": "12345678923", "chassi": "9BWZZZ377VT004251234" }
```

Atualiza apenas a tabela vault. Nada muda em VEICULO.

---

## Fluxo na prática (exemplo: editar RENAVAM)

```
1. App: GET /veiculos/abc-123
   API: retorna veículo com renavam mascarado: "*********23"

2. Usuário clica "Editar dados sensíveis"
   App: GET /veiculos/abc-123/dados-sensiveis
   API: valida permissão (é proprietário?), registra acesso, retorna sem máscara
   App: exibe formulário com valores reais

3. Usuário altera o RENAVAM e salva
   App: PUT /veiculos/abc-123/dados-sensiveis
   API: atualiza apenas VEICULO_DADOS_SENSIVEIS, registra acesso

4. App volta para a tela de detalhe
   API: retorna novamente com máscara aplicada
```

---

## Vantagens dessa arquitetura

1. **Princípio do menor privilégio aplicado por padrão**
   A query padrão NÃO traz dados sensíveis. Tem que ser explícito pra trazer.

2. **Auditoria nativa**
   AcessadoEm + AcessadoPorId no próprio vault. Sem precisar de tabela
   separada para Fase 1 (pode evoluir depois).

3. **Mascaramento simplificado**
   Só dois endpoints retornam dados completos. Resto é genérico.

4. **Conformidade LGPD**
   Mais fácil responder "quem acessou meu CPF nos últimos 90 dias?"

5. **Rotação de chave de criptografia**
   Como dados sensíveis estão isolados, re-criptografar é só rodar UPDATE
   nas duas tabelas vault.

6. **Cache mais seguro**
   Dá pra cachear veículos no Redis sem medo. Vault não vai pra cache.

---

## Considerações

### O vault NÃO é uma tabela 1:1 obrigatória

Se um usuário não tem CPF cadastrado (ex: conta criada via Google sem CPF),
simplesmente não existe linha em USUARIO_DADOS_SENSIVEIS. Não é JOIN forçado.

### Performance

- Queries normais: idênticas às de hoje (até melhores, sem campos pesados)
- Acesso ao vault: 1 query extra, mas só nos endpoints específicos

### Backup

Backups do banco devem tratar tabelas vault com cuidado especial
(criptografia em trânsito, controle de acesso).

### Quem pode ver dados completos

| Quem            | Pode ler vault? | Observação                              |
|-----------------|------------------|----------------------------------------|
| Proprietário    | Sim             | Apenas dos PRÓPRIOS veículos/perfil    |
| Oficina         | Não             | Mesmo autorizada, NÃO vê RENAVAM/Chassi|
| Admin           | Sim             | Acesso total + auditado                |
| UsuarioSistema  | Não             | Não acessa vault                       |

---

## Resumo das mudanças no schema

ADICIONAR:
- USUARIO_DADOS_SENSIVEIS (nova tabela)
- VEICULO_DADOS_SENSIVEIS (nova tabela)

REMOVER:
- USUARIO.Cpf
- VEICULO.Renavam
- VEICULO.Chassi

ADICIONAR (endpoints novos):
- GET /usuarios/{id}/dados-sensiveis
- PUT /usuarios/{id}/dados-sensiveis
- GET /veiculos/{id}/dados-sensiveis
- PUT /veiculos/{id}/dados-sensiveis

---

## Pendências de aprovação

[ ] Aprovar a separação em vault?
[ ] Aprovar os endpoints específicos pra leitura/edição?
[ ] Auditoria simples (campos no vault) ou tabela de log dedicada?
    Sugestão: começar com campos no vault e evoluir se necessário.
[ ] Rate limit no endpoint de leitura completa?
    Sugestão: 10 acessos/hora por usuário.
