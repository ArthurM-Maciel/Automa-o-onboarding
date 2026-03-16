⚡ n8n Onboarding Automation
Fluxo automático de onboarding de colaboradores
Stack: n8n  ·  Google Workspace  ·  Slack  ·  HubSpot  ·  Convenia

📋 O que é
Automação completa do processo de onboarding de novos colaboradores. O fluxo é disparado automaticamente pelo e-mail "CRIAÇÃO DE ACESSOS" enviado pelo Convenia (sistema de RH) e executa toda a jornada de provisionamento sem intervenção manual.

Uma vez recebido o e-mail, o fluxo:
	•	Cria o usuário no Google Workspace com senha aleatória segura
	•	Define o gestor/manager direto no Google Admin
	•	Adiciona o colaborador aos grupos corretos (todos@, sede-poa@ se presencial, grupo do departamento)
	•	Gera o card de boas-vindas (cópia do modelo Google Docs) com nome, e-mail e senha preenchidos
	•	Cria o contato no HubSpot se o colaborador for do Comercial
	•	Notifica o canal no Slack com um resumo completo

🗂️ Ponto de Partida
O projeto partiu de um fluxo v1/v2 parcialmente funcional. Apenas a criação do usuário no Google Workspace estava operando. Os demais componentes apresentavam os seguintes problemas:

Componente
Problema
Loop de grupos
Iterava sobre o objeto do usuário criado (único item), nunca sobre a lista de grupos — nunca executava
URL do grupo
Bug: $json.primaryEmail.emailGrupoDestino — mistura de dois campos distintos
Card boas-vindas
Nó Google Docs tentava editar um arquivo Google Slides — tipos incompatíveis
Senha
Hardcoded diretamente no JSON do fluxo (R@io@2025!!@)
Campo password
Declarado em additionalFields — ignorado pelo nó gSuiteAdmin (precisa ser campo raiz)
Manager/Gestor
Não existia no fluxo
HubSpot
Não existia no fluxo
Card para todos
Card só era gerado em alguns caminhos do fluxo
Parser de campos
Campos genéricos com fallbacks imprecisos, sem mapeamento dos campos reais do Convenia

🔨 Arquitetura do Fluxo v3
O fluxo v3 é linear, sem gates de aprovação humana. Chega o e-mail do Convenia → tudo é provisionado automaticamente.


Nó
O que faz
📧
Gmail Trigger
Polling a cada minuto — detecta e-mail do Convenia com assunto "CRIAÇÃO DE ACESSOS"
⚙️
Parsear Dados
Extrai campos do corpo do e-mail por regex linha a linha. Monta lista de grupos e resolve o nome do gestor para e-mail corporativo.
🔑
Gerar Senha
Gera senha segura aleatória (ex: RbxK9mTq2a!9) e salva no item para ser usada na criação e no card.
👤
Criar Usuário GWS
Cria o usuário no Google Workspace com firstName, lastName, username (nome.sobrenome), domain e senha gerada.
👔
Definir Manager
Atualiza o campo manager no Google Admin via Directory API (PUT /users/{email}) com o e-mail resolvido do gestor.
📋
Preparar Grupos
Expande o array todosOsGrupos em itens individuais para que o próximo nó processe cada grupo.
➕
Adicionar ao Grupo
Adiciona o colaborador a cada grupo via nó nativo GSuite Admin (addToGroup).
🔗
Consolidar
Agrega os múltiplos itens de volta para um único item e continua o fluxo.
📄
Copiar Modelo Doc
Copia o arquivo Google Docs modelo para a pasta de boas-vindas com nome "<Nome Completo> — Boas-vindas".
✏️
Preencher Card
Substitui {{NOME}}, {{EMAIL}} e {{SENHA}} no documento copiado com os dados reais do colaborador.
🏢
É Comercial?
IF: verifica se chaveDepto == COMERCIAL. Sim → HubSpot. Não → Slack diretamente. Card sempre é gerado antes deste ponto.
🟠
Criar no HubSpot
Cria contato no HubSpot com nome e e-mail corporativo. Executado somente para o departamento Comercial.
✅
Slack — Conclusão
Notifica o canal com resumo: nome, e-mail, departamento, gestor resolvido, grupos adicionados e confirmação do card.

Lógica de Grupos
Calculados dinamicamente no nó de parse com base nos dados do colaborador:
	•	todos@raiobeneficios.com — sempre, para todos os colaboradores
	•	sede-poa@raiobeneficios.com — somente se a jornada contiver "PRESENCIAL"
	•	Grupo do departamento (comercial@, rh@, tech@...) — se o departamento estiver no mapeamento

Resolução de Gestores
O campo GESTOR no e-mail do Convenia pode chegar como primeiro nome ou nome completo. O nó de parse converte para e-mail corporativo via mapa interno com todas as variações cobertas (com e sem acento, com e sem sobrenome). Se vier um nome fora do mapa, o fluxo não quebra — usa o valor bruto e informa no Slack.

📧 Formato do E-mail Convenia
O parser extrai os seguintes campos do corpo do e-mail por regex (uma chave por linha):

Campo no e-mail
Variável gerada
Observação
COLABORADOR
nomeCompleto / primeiroNome / sobrenome
Dividido em partes automaticamente
E-MAIL CORPORATIVO
emailCorporativo / usernameGoogle
Username = parte antes do @
E-MAIL PESSOAL
emailPessoal
—
DATA ADMISSÃO
dataAdmissao
Aceita com ou sem acento
DEPARTAMENTO
departamento / chaveDepto
chaveDepto = uppercase para comparação
GESTOR
gestorRaw / gestorEmail
gestorEmail = e-mail resolvido via mapa
TELEFONE
telefone
—
JORNADA
jornada / ehPresencial
ehPresencial = true se contiver PRESENCIAL
Necessidade de equipamento
equipamento
—

📄 Card de Boas-Vindas
O modelo Google Docs deve conter os seguintes placeholders (case-insensitive, chaves duplas obrigatórias):

{{NOME}}   →  primeiro nome do colaborador
{{EMAIL}}  →  e-mail corporativo completo
{{SENHA}}  →  senha temporária gerada pelo fluxo

O arquivo copiado é salvo na pasta destino com o nome:
<Nome Completo> — Boas-vindas

IDs configurados:
	•	Modelo: 1lCtvWeVdhY-ilX5JMDKv7FM70XcnazaU40UX_gP24zA
	•	Pasta destino: 1sEgVnqikEax4I2sLPGYEOYr0nC9vy8KZ

⚙️ Configuração antes de ativar
Os seguintes placeholders precisam ser preenchidos no fluxo antes de ativar:

Placeholder
Nó
O que colocar
PLACEHOLDER_DRIVE_COMERCIAL
Parsear Dados
ID da pasta Drive do depto Comercial
PLACEHOLDER_DRIVE_RH
Parsear Dados
ID da pasta Drive do depto RH
PLACEHOLDER_DRIVE_FINANCEIRO
Parsear Dados
ID da pasta Drive do depto Financeiro
PLACEHOLDER_DRIVE_TECH
Parsear Dados
ID da pasta Drive do depto Tech/TI
PLACEHOLDER_DRIVE_GERAL
Parsear Dados
ID da pasta Drive padrão (fallback)
PLACEHOLDER_HUBSPOT_CRED
Criar Contato HubSpot
ID da credencial HubSpot OAuth2 no n8n

Credenciais já vinculadas no n8n:
	•	Google Workspace Admin — cyjYpE0w4cxeFq7g
	•	Google Drive — EXMJ9Ve964QDFP6z
	•	Google Docs — BRv9VMdA1NSIArnq
	•	Gmail OAuth2 — 1sYwFViJfCbFb4qk
	•	Slack Bot — R2Kc8jZbzr4RLPPO

🚀 Como importar e ativar
	•	Acesse o n8n em https://n8n-tech.raiobeneficios.com
	•	Menu lateral → Workflows → Import from file
	•	Selecione o arquivo Fluxo_Onboarding_v3.json
	•	Preencha os PLACEHOLDERs listados acima no nó "Parsear Dados do Convenia"
	•	Vincule a credencial HubSpot no nó "Criar Contato HubSpot"
	•	Verifique os placeholders {{NOME}}, {{EMAIL}}, {{SENHA}} no arquivo modelo Google Docs
	•	Ative o fluxo com o toggle no canto superior direito

Dica: rode um teste manual enviando um e-mail com o assunto "CRIAÇÃO DE ACESSOS" para o Gmail vinculado e observe a execução no painel do n8n.

📁 Arquivos do repositório
Arquivo
Descrição
Fluxo_Onboarding_v3.json
Fluxo n8n pronto para importar
README.md / README.docx
Esta documentação

🛠️ Stack
Ferramenta
Papel no fluxo
n8n (self-hosted)
Motor de automação
Google Workspace Admin API
Criação de usuário, grupos, manager
Google Drive API
Cópia do modelo de boas-vindas
Google Docs API
Substituição de placeholders no card
Gmail
Trigger do fluxo via polling
Slack
Notificação de conclusão
HubSpot
Criação de contato (Comercial)
Convenia
Sistema de RH — origem do e-mail de criação
