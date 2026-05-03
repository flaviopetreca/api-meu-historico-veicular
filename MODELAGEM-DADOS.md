# Modelagem de Dados — Meu Histórico Veicular

> Versão 1.1 — atualizada com vault de dados sensíveis.
> Total: 12 tabelas principais + tabelas do ASP.NET Core Identity.

---

## Lista de tabelas

1. PLANO
2. USUARIO (estende IdentityUser do ASP.NET Core Identity)
3. USUARIO_DADOS_SENSIVEIS (vault)
4. VEICULO
5. VEICULO_DADOS_SENSIVEIS (vault)
6. VEICULO_PROPRIETARIO
7. SERVICO
8. DOCUMENTO
9. PERMISSAO_OFICINA
10. TRANSFERENCIA
11. OTP_CODIGO
12. REFRESH_TOKEN

Tabelas criadas automaticamente pelo Identity (não detalhadas aqui):
- AspNetUsers, AspNetRoles, AspNetUserRoles, AspNetUserClaims,
  AspNetUserLogins, AspNetUserTokens, AspNetRoleClaims

---

## 1. PLANO

Define os limites de cada tipo de assinatura.

| Campo            | Tipo   | PK | Obrig | Observações                          |
|------------------|--------|----|-------|--------------------------------------|
| Id               | Guid   | X  | sim   |                                      |
| Nome             | string |    | sim   | Básico / Plus / Profissional         |
| MaxVeiculos      | int    |    | sim   | -1 = ilimitado                       |
| PermiteUpload    | bool   |    | sim   | Permite upload de notas fiscais      |
| PermiteOficinas  | bool   |    | sim   | Permite autorizar oficinas           |

---

## 2. USUARIO (ApplicationUser)

Estende IdentityUser do ASP.NET Core Identity. Email, senha, EmailConfirmed,
PhoneNumber etc. já vêm do Identity. Os campos abaixo são os customizados.

>>> CPF NÃO ESTÁ AQUI <<<
Está isolado em USUARIO_DADOS_SENSIVEIS (vault).

| Campo               | Tipo     | PK | Obrig | Observações                                |
|---------------------|----------|----|-------|--------------------------------------------|
| Id                  | Guid     | X  | sim   | Vem do Identity                            |
| Nome                | string   |    | sim   |                                            |
| Email               | string   |    | sim   | Único. Vem do Identity                     |
| EmailConfirmed      | bool     |    | sim   | Vem do Identity                            |
| Telefone            | string   |    | sim   | Formato E.164 (+5511999999999)             |
| TelefoneVerificado  | bool     |    | sim   | Para futuro 2FA                            |
| CanalRecuperacao    | enum     |    | não   | SMS ou WhatsApp                            |
| PlanoId             | Guid FK  |    | sim   | -> PLANO.Id                                |
| Ativo               | bool     |    | sim   | Soft delete                                |
| CriadoEm            | DateTime |    | sim   |                                            |

Relacionamentos:
- N:1 com PLANO
- 1:1 com USUARIO_DADOS_SENSIVEIS (vault)
- 1:N com VEICULO_PROPRIETARIO
- 1:N com SERVICO (como proprietário ou oficina executora)
- 1:N com PERMISSAO_OFICINA (como concedente ou recebedor)
- 1:N com TRANSFERENCIA (como remetente ou destinatário)
- 1:N com OTP_CODIGO
- 1:N com REFRESH_TOKEN

---

## 3. USUARIO_DADOS_SENSIVEIS (VAULT)

Tabela isolada para CPF e outros dados sensíveis do usuário.
Acessada APENAS por endpoints específicos com auditoria.

| Campo          | Tipo      | PK | Obrig | Observações                                 |
|----------------|-----------|----|-------|---------------------------------------------|
| Id             | Guid      | X  | sim   |                                             |
| UsuarioId      | Guid FK   |    | sim   | -> USUARIO.Id (UNIQUE — relação 1:1)        |
| Cpf            | string    |    | não   | Criptografado AES-256                       |
| RG             | string    |    | não   | Criptografado AES-256 (reservado p/ futuro) |
| AtualizadoEm   | DateTime  |    | sim   |                                             |
| AcessadoEm     | DateTime? |    | não   | Última leitura completa (auditoria)         |
| AcessadoPorId  | Guid?     |    | não   | -> USUARIO.Id (quem acessou)                |

Endpoints que acessam:
- GET /usuarios/{id}/dados-sensiveis (com rate limit 10/h)
- PUT /usuarios/{id}/dados-sensiveis

---

## 4. VEICULO

>>> RENAVAM E CHASSI NÃO ESTÃO AQUI <<<
Estão isolados em VEICULO_DADOS_SENSIVEIS (vault).

| Campo          | Tipo     | PK | Obrig | Observações                  |
|----------------|----------|----|-------|------------------------------|
| Id             | Guid     | X  | sim   |                              |
| Placa          | string   |    | sim   | Única no sistema             |
| Marca          | string   |    | sim   |                              |
| Modelo         | string   |    | sim   |                              |
| AnoFabricacao  | int      |    | sim   |                              |
| AnoModelo      | int      |    | sim   |                              |
| Cor            | string   |    | sim   |                              |
| CriadoEm       | DateTime |    | sim   |                              |

Relacionamentos:
- 1:1 com VEICULO_DADOS_SENSIVEIS (vault)
- 1:N com VEICULO_PROPRIETARIO
- 1:N com SERVICO
- 1:N com PERMISSAO_OFICINA
- 1:N com TRANSFERENCIA

---

## 5. VEICULO_DADOS_SENSIVEIS (VAULT)

Tabela isolada para RENAVAM e Chassi.
Acessada APENAS por endpoints específicos com auditoria.

| Campo          | Tipo      | PK | Obrig | Observações                                 |
|----------------|-----------|----|-------|---------------------------------------------|
| Id             | Guid      | X  | sim   |                                             |
| VeiculoId      | Guid FK   |    | sim   | -> VEICULO.Id (UNIQUE — relação 1:1)        |
| Renavam        | string    |    | não   | Criptografado AES-256                       |
| Chassi         | string    |    | não   | Criptografado AES-256                       |
| AtualizadoEm   | DateTime  |    | sim   |                                             |
| AcessadoEm     | DateTime? |    | não   | Última leitura completa (auditoria)         |
| AcessadoPorId  | Guid?     |    | não   | -> USUARIO.Id (quem acessou)                |

Endpoints que acessam:
- GET /veiculos/{id}/dados-sensiveis (com rate limit 10/h)
- PUT /veiculos/{id}/dados-sensiveis

Permissão: APENAS proprietário ATIVO do veículo ou Admin.
Oficinas NÃO acessam, mesmo as autorizadas.

---

## 6. VEICULO_PROPRIETARIO

Histórico de proprietários por veículo.

| Campo            | Tipo      | PK | Obrig | Observações                  |
|------------------|-----------|----|-------|------------------------------|
| Id               | Guid      | X  | sim   |                              |
| VeiculoId        | Guid FK   |    | sim   | -> VEICULO.Id                |
| ProprietarioId   | Guid FK   |    | sim   | -> USUARIO.Id                |
| DataInicio       | DateTime  |    | sim   | Quando se tornou dono        |
| DataFim          | DateTime? |    | não   | null = proprietário atual    |
| Ativo            | bool      |    | sim   | true = posse vigente         |

---

## 7. SERVICO

Manutenção ou serviço realizado em um veículo.

| Campo             | Tipo      | PK | Obrig | Observações                         |
|-------------------|-----------|----|-------|-------------------------------------|
| Id                | Guid      | X  | sim   |                                     |
| VeiculoId         | Guid FK   |    | sim   | -> VEICULO.Id                       |
| ProprietarioId    | Guid FK   |    | sim   | -> USUARIO.Id (dono na época)       |
| RealizadoPorId    | Guid? FK  |    | não   | -> USUARIO.Id (oficina executora)   |
| Descricao         | string    |    | sim   |                                     |
| LocalRealizacao   | string    |    | sim   | Nome da oficina/estabelecimento     |
| DataRealizacao    | DateTime  |    | sim   |                                     |
| KmNaRealizacao    | int?      |    | não   |                                     |
| ValorTotal        | decimal?  |    | não   |                                     |
| GarantiaMeses     | int?      |    | não   | Tempo de garantia                   |
| DataFimGarantia   | DateTime? |    | não   | Calculado automaticamente           |
| Observacoes       | string?   |    | não   |                                     |
| CriadoEm          | DateTime  |    | sim   |                                     |

---

## 8. DOCUMENTO

Anexos de notas fiscais, orçamentos e comprovantes.

| Campo           | Tipo     | PK | Obrig | Observações                          |
|-----------------|----------|----|-------|--------------------------------------|
| Id              | Guid     | X  | sim   |                                      |
| ServicoId       | Guid FK  |    | sim   | -> SERVICO.Id                        |
| Tipo            | enum     |    | sim   | NotaFiscal / Orcamento / Garantia / Outro |
| NomeArquivo     | string   |    | sim   |                                      |
| CaminhoArquivo  | string   |    | sim   | Caminho no volume Docker             |
| TamanhoBytes    | long     |    | sim   |                                      |
| MimeType        | string   |    | sim   |                                      |
| CriadoEm        | DateTime |    | sim   |                                      |

---

## 9. PERMISSAO_OFICINA

| Campo            | Tipo      | PK | Obrig | Observações                            |
|------------------|-----------|----|-------|----------------------------------------|
| Id               | Guid      | X  | sim   |                                        |
| VeiculoId        | Guid FK   |    | sim   | -> VEICULO.Id                          |
| ProprietarioId   | Guid FK   |    | sim   | -> USUARIO.Id (concede)                |
| OficinaId        | Guid FK   |    | sim   | -> USUARIO.Id (recebe)                 |
| Ativo            | bool      |    | sim   |                                        |
| CriadoEm         | DateTime  |    | sim   |                                        |
| RevogadoEm       | DateTime? |    | não   |                                        |

---

## 10. TRANSFERENCIA

| Campo               | Tipo      | PK | Obrig | Observações                              |
|---------------------|-----------|----|-------|------------------------------------------|
| Id                  | Guid      | X  | sim   |                                          |
| VeiculoId           | Guid FK   |    | sim   | -> VEICULO.Id                            |
| DeProprietarioId    | Guid FK   |    | sim   | -> USUARIO.Id (remetente)                |
| ParaProprietarioId  | Guid FK   |    | sim   | -> USUARIO.Id (destinatário)             |
| Status              | enum      |    | sim   | Pendente / Aceita / Recusada / Cancelada |
| SolicitadaEm        | DateTime  |    | sim   |                                          |
| RespondidaEm        | DateTime? |    | não   |                                          |

---

## 11. OTP_CODIGO

| Campo        | Tipo     | PK | Obrig | Observações                                  |
|--------------|----------|----|-------|----------------------------------------------|
| Id           | Guid     | X  | sim   |                                              |
| UsuarioId    | Guid FK  |    | sim   | -> USUARIO.Id                                |
| CodigoHash   | string   |    | sim   | Hash do código                               |
| Canal        | enum     |    | sim   | SMS ou WhatsApp                              |
| Finalidade   | enum     |    | sim   | RecuperacaoSenha / Verificacao2FA            |
| ExpiraEm     | DateTime |    | sim   | 10 minutos após criação                      |
| Utilizado    | bool     |    | sim   |                                              |
| Tentativas   | int      |    | sim   | Para rate limiting                           |

---

## 12. REFRESH_TOKEN

| Campo       | Tipo     | PK | Obrig | Observações                            |
|-------------|----------|----|-------|----------------------------------------|
| Id          | Guid     | X  | sim   |                                        |
| UsuarioId   | Guid FK  |    | sim   | -> USUARIO.Id                          |
| TokenHash   | string   |    | sim   |                                        |
| ExpiraEm    | DateTime |    | sim   | 7 dias                                 |
| Revogado    | bool     |    | sim   |                                        |
| CriadoEm    | DateTime |    | sim   |                                        |

---

## Mapa de relacionamentos atualizado

```
PLANO (1) ----< (N) USUARIO

USUARIO (1) ----- (1) USUARIO_DADOS_SENSIVEIS  [VAULT]

USUARIO (1) ----< (N) VEICULO_PROPRIETARIO >---- (1) VEICULO
                                                      |
                                                     (1)
                                                      |
                                            VEICULO_DADOS_SENSIVEIS  [VAULT]

USUARIO (1) ----< (N) SERVICO >---- (1) VEICULO
                          |
                          (N) >---- (1) USUARIO  (oficina, opcional)

SERVICO (1) ----< (N) DOCUMENTO

USUARIO (1) ----< (N) PERMISSAO_OFICINA >---- (1) VEICULO
                              |
                              (N) >---- (1) USUARIO

USUARIO (1) ----< (N) TRANSFERENCIA >---- (1) VEICULO

USUARIO (1) ----< (N) OTP_CODIGO
USUARIO (1) ----< (N) REFRESH_TOKEN
```

---

## Endpoints novos (vault)

| Endpoint                              | Método | Permissão                  | Rate Limit |
|---------------------------------------|--------|----------------------------|------------|
| /usuarios/{id}/dados-sensiveis        | GET    | Próprio user / Admin       | 10/hora    |
| /usuarios/{id}/dados-sensiveis        | PUT    | Próprio user / Admin       | -          |
| /veiculos/{id}/dados-sensiveis        | GET    | Proprietário ativo / Admin | 10/hora    |
| /veiculos/{id}/dados-sensiveis        | PUT    | Proprietário ativo / Admin | -          |

Todos os acessos atualizam AcessadoEm e AcessadoPorId no vault.

---

## Ganhos da arquitetura vault

1. Queries normais NUNCA tocam dados sensíveis
2. Auditoria nativa (campos no próprio vault)
3. Mascaramento por padrão, dado completo só sob demanda
4. Rotação de chave de criptografia trivial
5. Cache seguro (vault não vai pro Redis)
6. Conformidade LGPD facilitada
7. Oficinas não veem RENAVAM/Chassi mesmo autorizadas
