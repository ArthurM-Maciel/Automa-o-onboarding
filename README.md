# ⚡ Raio Benefícios — Onboarding Automation

> Automação completa de onboarding de colaboradores utilizando n8n, Google Workspace, Slack, HubSpot e Convenia.

---

## 📋 Sumário

- [Visão Geral](#visão-geral)
- [Ponto de Partida](#ponto-de-partida)
- [O Que Foi Construído](#o-que-foi-construído)
- [Arquitetura do Fluxo](#arquitetura-do-fluxo)
- [Arquivos do Repositório](#arquivos-do-repositório)
- [Pré-requisitos](#pré-requisitos)
- [Configuração](#configuração)
- [Como Usar](#como-usar)

---

## Visão Geral

Esta automação gerencia todo o processo de provisionamento de um novo colaborador a partir do e-mail enviado pelo Convenia (sistema de RH), executando todas as ações automaticamente sem intervenção manual.

**Stack utilizada:**
- **Orquestração:** n8n (self-hosted)
- **Identidade:** Google Workspace Admin API
- **Comunicação:** Slack (notificações)
- **CRM:** HubSpot
- **Formulários/Card:** Google Drive + Google Docs
- **RH:** Convenia (trigger via e-mail)

---

## Ponto de Partida

O fluxo original (`Fluxo_de_Onboarding_2.json`) já possuía uma base parcial com:

✅ Trigger via Gmail polling (assunto "CRIAÇÃO DE ACESSOS")  
✅ Nó Code com parser do corpo do e-mail  
✅ Criação do usuário no Google Workspace  

**Problemas identificados no fluxo original:**

| Problema | Impacto |
|---|---|
| Loop de grupos iterava sobre o objeto do usuário (item único), não sobre os grupos | Nenhum grupo era adicionado |
| URL do HTTP Request com bug: `$json.primaryEmail.emailGrupoDestino` | Chamada sempre falhava |
| Nó Google Docs tentava editar um arquivo Google Slides | Tipos incompatíveis, card nunca era gerado |
| Senha hardcoded `R@io@2025!!@` no JSON do fluxo | Risco de segurança |
| Campo `password` declarado em `additionalFields` | Ignorado pelo nó gSuiteAdmin — usuário criado sem senha |
| Sem definição do campo manager/gestor no Google Admin | Gestor não era registrado |
| Sem integração com HubSpot | Colaboradores do Comercial não eram criados no CRM |
| Card de boas-vindas gerado só em alguns caminhos | Colaboradores de outros departamentos não recebiam o card |
| Campos do parser genéricos com fallbacks imprecisos | Risco de extração incorreta dos dados |

---

## O Que Foi Construído

### v3 — Correções e novas funcionalidades

#### 🔧 Correções técnicas

| Problema | Correção |
|---|---|
| Loop de grupos quebrado | Code expande o array `todosOsGrupos` em itens individuais; nó nativo GSuite Admin processa cada um |
| URL com bug no HTTP Request | Substituído por nó nativo `addToGroup` com `mode: groupEmail` |
| Card usando nó errado | Google Drive copia o modelo; Google Docs substitui os placeholders |
| Senha hardcoded | Gerada aleatoriamente em nó dedicado e salva no item para reuso |
| `password` em `additionalFields` | Movido para campo raiz dos parâmetros do nó gSuiteAdmin |

#### ✨ Novas funcionalidades

**Geração de senha aleatória**  
Um nó dedicado gera a senha antes da criação do usuário (ex: `RbxK9mTq2a!9`) e a mantém no item para ser usada na criação do usuário e preenchida automaticamente no card de boas-vindas.

**Definição de Manager no Google Admin**  
O gestor informado pelo Convenia é resolvido para e-mail corporativo via mapa interno e registrado via `PUT /admin/directory/v1/users/{email}` com o campo `relations.type: manager`.

**Resolução automática de gestores**  
O campo GESTOR do e-mail pode chegar como primeiro nome ou nome completo. O parser resolve automaticamente para o e-mail correto cobrindo todas as variações (com/sem acento, com/sem sobrenome).

**Grupos dinâmicos por departamento e jornada**  
Os grupos são calculados automaticamente:
- `todos@raiobeneficios.com` — sempre
- `sede-poa@raiobeneficios.com` — somente se a jornada contiver "PRESENCIAL"
- Grupo do departamento (`comercial@`, `rh@`, `tech@`...) — se o departamento estiver mapeado

**Card de boas-vindas para todos**  
O modelo Google Docs é copiado para a pasta destino e os placeholders `{{NOME}}`, `{{EMAIL}}` e `{{SENHA}}` são substituídos com os dados reais — para todos os colaboradores, independente do departamento.

**Integração com HubSpot**  
Colaboradores do departamento Comercial são criados automaticamente como contatos no HubSpot.

**Notificação de conclusão no Slack**  
Mensagem final com resumo completo: nome, e-mail, departamento, gestor resolvido, grupos adicionados e confirmação do card gerado.

---

## Arquitetura do Fluxo
```
E-mail Convenia "CRIAÇÃO DE ACESSOS"
         │
         ▼
  Gmail Trigger (polling 1 min)
         │
         ▼
  Parsear Dados do Convenia
  (extrai campos, monta grupos, resolve gestor)
         │
         ▼
  Gerar Senha Temporária
  (salva senhaTemporaria no item)
         │
         ▼
  Criar Usuário GWS
  (firstName, lastName, username, domain, password)
         │
         ▼
  Definir Manager no GWS
  (PUT Directory API com gestorEmail)
         │
         ▼
  Preparar Lista de Grupos
  (expande array em itens individuais)
         │
         ▼
  Adicionar ao Grupo (por item)
  todos@ + sede-poa@ (se presencial) + grupo do depto
         │
         ▼
  Consolidar após Grupos
         │
         ▼
  Copiar Modelo Boas-Vindas (Google Drive)
         │
         ▼
  Preencher Card (Google Docs)
  {{NOME}} {{EMAIL}} {{SENHA}}
         │
         ▼
  É Comercial?
  ┌───────┴───────┐
 sim             não
  │               │
  ▼               │
Criar Contato     │
HubSpot           │
  │               │
  └───────┬───────┘
          ▼
  Slack — Onboarding Concluído
```

---

## Arquivos do Repositório

| Arquivo | Descrição |
|---|---|
| `Fluxo_Onboarding_v3.json` | Workflow n8n completo — importar diretamente |

---

## Pré-requisitos

### Credenciais n8n necessárias

| Credencial | Tipo | Usado em |
|---|---|---|
| Google Workspace Admin | gSuiteAdminOAuth2Api | Criar usuário, adicionar a grupos, definir manager |
| Google Drive | googleDriveOAuth2Api | Copiar modelo de boas-vindas |
| Google Docs | googleDocsOAuth2Api | Substituir placeholders no card |
| Gmail | gmailOAuth2 | Trigger do fluxo |
| Slack Bot | slackApi | Notificação de conclusão |
| HubSpot | hubspotOAuth2Api | Criar contato (Comercial) |

### Placeholders a preencher no fluxo

| Placeholder | Nó | O que colocar |
|---|---|---|
| `PLACEHOLDER_DRIVE_COMERCIAL` | Parsear Dados | ID da pasta Drive do depto Comercial |
| `PLACEHOLDER_DRIVE_RH` | Parsear Dados | ID da pasta Drive do depto RH |
| `PLACEHOLDER_DRIVE_FINANCEIRO` | Parsear Dados | ID da pasta Drive do depto Financeiro |
| `PLACEHOLDER_DRIVE_TECH` | Parsear Dados | ID da pasta Drive do depto Tech/TI |
| `PLACEHOLDER_DRIVE_GERAL` | Parsear Dados | ID da pasta Drive padrão (fallback) |
| `PLACEHOLDER_HUBSPOT_CRED` | Criar Contato HubSpot | ID da credencial HubSpot no n8n |

---

## Configuração

### 1. Importar o workflow no n8n
1. Acesse seu n8n → **Workflows** → **Import from file**
2. Selecione `Fluxo_Onboarding_v3.json`
3. Configure todas as credenciais nos nós marcados em vermelho

### 2. Preencher os placeholders
No nó **"Parsear Dados do Convenia"**, substitua os `PLACEHOLDER_DRIVE_*` pelos IDs reais das pastas de Drive de cada departamento.

### 3. Configurar o modelo de boas-vindas
O arquivo Google Docs modelo deve conter os seguintes placeholders com essa formatação exata:

| Placeholder | Substituído por |
|---|---|
| `{{NOME}}` | Primeiro nome do colaborador |
| `{{EMAIL}}` | E-mail corporativo completo |
| `{{SENHA}}` | Senha temporária gerada pelo fluxo |

IDs já configurados no fluxo:
- **Modelo:** `1lCtvWeVdhY-ilX5JMDKv7FM70XcnazaU40UX_gP24zA`
- **Pasta destino:** `1sEgVnqikEax4I2sLPGYEOYr0nC9vy8KZ`

### 4. Adicionar novos departamentos
Para mapear novos departamentos a grupos ou pastas, edite os objetos `mapaGrupos` e `mapaDrives` no nó **"Parsear Dados do Convenia"**.

### 5. Adicionar novos gestores
Para adicionar gestores ao mapa de resolução, edite o objeto `mapaGestores` no mesmo nó, adicionando o nome como aparece no Convenia (maiúsculo) e o e-mail corporativo correspondente.

---

## Como Usar

O fluxo é totalmente automático. Não é necessária nenhuma ação manual além de manter o fluxo ativo.

Quando o Convenia enviar um e-mail com assunto **"CRIAÇÃO DE ACESSOS"** para o Gmail vinculado, o fluxo executa automaticamente e ao final envia uma mensagem no Slack confirmando tudo que foi feito:
```
✅ Onboarding concluído!

Nome: ---------
E-mail: -------------
Departamento: Comercial
Gestor: ------------
Grupos: ---------------
Card de boas-vindas: criado 📄
```

---

## Observações

- O campo `manager` no Google Admin é preenchido automaticamente durante o onboarding com base no campo GESTOR do e-mail do Convenia. Isso garante que o fluxo de **offboarding** sempre encontre o gestor correto sem depender de input manual.
- Se o Convenia enviar um nome de gestor fora do mapa de resolução, o fluxo não quebra — usa o valor bruto e informa na mensagem do Slack para que o mapa seja atualizado.
- A senha temporária gerada segue o padrão `Rb` + 8 caracteres alfanuméricos + `!9`, atendendo requisitos de complexidade do Google Workspace.
- O fluxo **não** usa `changePasswordAtNextLogin` — a senha está no card de boas-vindas e a pessoa acessa normalmente na primeira vez.
- Todos os dados sensíveis (IDs de pastas, credenciais) devem ser preenchidos diretamente no fluxo ou via n8n Variables — nunca hardcoded em produção.
