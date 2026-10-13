# EducaTech

Plataforma educacional (protótipo funcional) feita para o **Desafio Liga Jovem**: alunos, professores e administração em áreas separadas, atividades escolares reais (publicar → entregar → corrigir) e um **Assistente de IA** que age como tutor.

> **Importante:** este é um protótipo para demonstração e testes controlados. Antes de usar com alunos reais (principalmente menores de 18 anos), a escola precisa avaliar a LGPD: termo de consentimento dos responsáveis, política de privacidade e contrato com o provedor de IA.

---

## 1. O que está pronto (e o que não está)

| Recurso | Situação |
|---|---|
| Página inicial, identidade visual azul/branco/cinza, responsivo | ✅ Pronto |
| Cadastro, login, recuperar senha, trocar senha, sair | ✅ Pronto (usa Supabase Auth) |
| Aluno e professor com perfis; professor só após aprovação do admin | ✅ Pronto |
| Painel do aluno: atividades, entregas, devolutivas, metas de estudo, calendário, avisos | ✅ Pronto |
| Painel do professor: turmas, criar/editar/excluir atividades, corrigir, avisos, indicadores | ✅ Pronto |
| Administração: aprovar/suspender contas, escolas, turmas, vincular professores, eventos de segurança | ✅ Pronto |
| Notificações automáticas (nova atividade, prazo alterado, devolutiva, aviso) | ✅ Pronto (gatilhos no banco) |
| Políticas de acesso (RLS) em todas as tabelas | ✅ Pronto e testado em PostgreSQL real |
| Assistente EducaTech com IA (Anthropic ou OpenAI) via rota segura | ✅ Pronto; **precisa da sua chave de API** |
| Assistente em **modo demonstrativo** (sem chave), identificado na tela | ✅ Pronto |
| Ajuste de texto, alto contraste, teclado, leitores de tela | ✅ Pronto (revise com usuários reais) |
| Anexar arquivos nas atividades | ⏳ Não implementado (use links nas instruções) |
| Materiais baixáveis e atividades **offline** | ⏳ Não implementado. A arquitetura permite (ver seção 8) |
| “Questões com mais erros” | ⏳ Não implementado: exige guardar respostas por questão. Hoje o painel mostra **conteúdos indicados para revisão** pelo professor |
| Notificações por e-mail | ⏳ Só notificações dentro do sistema |

---

## 2. O que você precisa

- [Node.js](https://nodejs.org) versão LTS (20 ou superior) e o [VS Code](https://code.visualstudio.com)
- Conta gratuita no [Supabase](https://supabase.com)
- (Opcional) conta na [Vercel](https://vercel.com) para publicar
- (Opcional) chave de API da [Anthropic](https://console.anthropic.com) ou da [OpenAI](https://platform.openai.com) para o assistente

---

## 3. Passo a passo para rodar no seu computador

### 3.1. Instalar
```bash
cd educatech
npm install
```

### 3.2. Criar o projeto no Supabase
1. Em supabase.com, clique em **New project**, escolha um nome e uma senha de banco (guarde-a).
2. Quando o projeto ficar pronto, abra **SQL Editor → New query**.
3. Abra o arquivo `supabase/migrations/0001_educatech.sql`, copie **todo o conteúdo**, cole no editor e clique em **Run**. Deve aparecer “Success”. Isso cria as tabelas, as regras de segurança (RLS) e os gatilhos.

### 3.3. Configurar a autenticação
No Supabase, em **Authentication**:
- **URL Configuration**: em *Site URL* coloque `http://localhost:3000`; em *Redirect URLs* adicione `http://localhost:3000/auth/callback`.
- **Sign In / Providers → Email**: deixe *Confirm email* **ligado** (recomendado) e defina o tamanho mínimo de senha como 8.
- Em **Rate Limits** você pode ajustar os limites de tentativas de login e envio de e-mails.

> O e-mail embutido do Supabase tem limite baixo de envios por hora. Para uma turma inteira, configure um SMTP próprio em *Authentication → SMTP Settings*.

### 3.4. Variáveis de ambiente
Copie `.env.example` para `.env.local` e preencha:

```
NEXT_PUBLIC_SUPABASE_URL=...        (Project Settings → API → Project URL)
NEXT_PUBLIC_SUPABASE_ANON_KEY=...   (Project Settings → API → anon public key)
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

Este projeto **não usa** a chave `service_role` do Supabase. Não a coloque em lugar nenhum.

### 3.5. Rodar
```bash
npm run dev
```
Abra <http://localhost:3000>. Se aparecer “Falta configurar o Supabase”, revise o `.env.local` e reinicie o comando.

### 3.6. Criar o primeiro administrador
Por segurança, **não existe cadastro público de administrador**. O acesso é concedido pelo banco:

1. Crie uma conta normalmente em `/criar-conta` (como aluno) e confirme o e-mail.
2. No Supabase → SQL Editor, rode (trocando o e-mail):
```sql
update public.profiles set role = 'admin', status = 'active' where email = 'seu-email@exemplo.com';
```
3. Saia e entre de novo: você será levado ao painel `/admin`.

### 3.7. Roteiro para montar uma demonstração
1. **Admin**: em *Escolas e turmas*, crie uma escola e uma turma (anote o *código da turma*).
2. **Professor**: crie outra conta com tipo “Professor”. Ela ficará “Aguardando aprovação”.
3. **Admin**: em *Visão geral*, aprove o professor. Depois, em *Escolas e turmas*, vincule o professor à turma.
4. **Aluno**: crie uma conta de aluno informando o código da turma (ou informe o código na tela inicial do aluno).
5. **Professor**: publique uma atividade. O aluno recebe uma notificação.
6. **Aluno**: abra a atividade e envie a resposta.
7. **Professor**: corrija, comente e preencha “Sugestão de revisão”. O aluno é notificado e vê a sugestão em *Meus estudos*.
8. **Aluno**: abra o Assistente (modo demonstrativo ou real).

---

## 4. Ligar a inteligência artificial

No `.env.local`:
```
AI_PROVIDER=anthropic      # ou: openai
AI_API_KEY=sua-chave-aqui
AI_MODEL=                  # opcional; vazio usa um modelo padrão do provedor
```
Reinicie o servidor. Sem `AI_PROVIDER` e `AI_API_KEY`, o assistente funciona em **modo demonstrativo** e avisa isso na tela.

Como funciona (arquivos `src/app/api/assistente/route.ts` e `src/lib/ia/assistente.ts`):
- Só alunos logados e ativos podem usar. Há limite de 15 perguntas a cada 10 minutos e 60 por hora.
- O servidor valida o tamanho e o formato das mensagens.
- O provedor recebe **apenas** o histórico da conversa e o ano escolar (se o aluno informou). Nome, e-mail, atividades e notas **não** são enviados.
- O texto do “tutor” (regras de comportamento, incentivo ao raciocínio, honestidade, segurança para menores e orientação em situações de risco) está em `instrucoesDoSistema()`. Ajuste com a ajuda dos professores.
- As conversas **não são salvas** no banco; só o horário de cada pergunta, para o limite de uso.

---

## 5. Publicar na Vercel
1. Envie o projeto para um repositório no GitHub (o `.gitignore` já protege o `.env.local`).
2. Na Vercel: **Add New → Project**, escolha o repositório.
3. Em **Environment Variables**, cadastre as mesmas variáveis do `.env.local`, usando o endereço final em `NEXT_PUBLIC_SITE_URL` (ex.: `https://educatech.vercel.app`).
4. No Supabase → Authentication → URL Configuration, troque/adicione o *Site URL* e o *Redirect URL* (`https://SEU-SITE/auth/callback`).

---

## 6. Estrutura do projeto

```
supabase/
  migrations/0001_educatech.sql   tabelas, funções, gatilhos e políticas RLS
  tests/                          testes das regras de acesso (ver docs/PLANO_DE_TESTES.md)
src/
  proxy.ts                        exige login nas rotas privadas e renova a sessão
  app/
    page.tsx                      página inicial
    (auth)/                       entrar, criar-conta, recuperar-senha, redefinir-senha
    aluno/  professor/  admin/    painéis (cada layout confere o papel do usuário)
    configuracoes/  notificacoes/ compartilhadas
    api/assistente/               rota segura da IA
  components/                     formulários acessíveis, menu, calendário, chat
  lib/
    supabase/                     clientes do servidor
    acoes/                        Server Actions (toda gravação passa por aqui)
    validacao.ts                  regras de validação (Zod)
    ia/assistente.ts              prompt do tutor e chamada ao provedor
    auth.ts                       exigirSessao(): login + conta ativa + papel
docs/
  PLANO_DE_TESTES.md              Etapa 6
  APRESENTACAO.md                 Etapa 7
```

### Banco de dados
Colunas e tabelas em inglês (como no plano do projeto); a interface é toda em português.

`profiles` · `schools` · `classes` · `class_members` · `activities` · `submissions` · `feedback` · `notifications` · `study_goals`, mais `announcements` (avisos), `ai_requests` (limite de uso da IA) e `security_events` (eventos de segurança).

---

## 7. Segurança: o que cada controle faz de verdade

| Controle | Onde está |
|---|---|
| Senhas nunca em texto puro | Supabase Auth (hash bcrypt) |
| Papel de professor/admin **não** vem do formulário | Gatilho `handle_new_user`: cria “aluno ativo” ou “professor pendente”. Admin só por SQL |
| Usuário não consegue se promover | Gatilho `protect_profile_columns` bloqueia mudança de `role`, `status`, `email` |
| Aluno só vê o que é dele | Políticas RLS (`activities`, `submissions`, `feedback`…) |
| Alunos não veem perfis de outros alunos | Função `can_see_profile` |
| Professor só mexe nas próprias atividades/turmas | RLS + função `owns_activity` |
| Professor não altera a resposta do aluno | Gatilho `protect_submission_columns` |
| Professor suspenso/pendente perde acesso aos dados | As funções de RLS exigem conta `active` |
| Rotas privadas protegidas | `src/proxy.ts` (login) + `exigirSessao()` em cada página/ação (papel) |
| Validação no servidor | Zod em `lib/validacao.ts`, usado em todas as Server Actions |
| Chave da IA fora do navegador | Só lida em `lib/ia/assistente.ts` (servidor); sem `NEXT_PUBLIC_` |
| Sem chave `service_role` | O projeto não a usa |
| Limite de requisições da IA | Contagem em `ai_requests` por usuário |
| Limite de tentativas de login/cadastro | Limites do próprio Supabase Auth (ajustáveis no painel) |
| Mensagem de login que não revela se o e-mail existe | `entrar` e `recuperarSenha` |
| Erros sem detalhes técnicos | Mensagens genéricas nas ações; detalhes só no log do servidor |
| Eventos de segurança | Tabela `security_events` (login, logout, troca de senha, mudança de situação de conta). Sem senhas nem conteúdo |
| Cabeçalhos de segurança | `next.config.ts` |

**Limitações conhecidas:** o código de turma/escola é um “segredo compartilhado”: quem o conhece pode entrar na turma como aluno. Troque os códigos se vazarem (recriando a turma) e distribua-os apenas pela escola. Um limite de requisições por IP no cadastro/login exige um serviço externo; hoje se depende dos limites do Supabase.

---

## 8. Próximos passos (arquitetura já preparada)

- **Anexos e materiais baixáveis:** usar o Supabase Storage com um bucket privado e políticas RLS ligadas às turmas. Adicionar tabela `attachments (activity_id, path)`.
- **Atividades offline:** transformar o app em PWA (service worker) e guardar rascunhos no navegador, sincronizando ao voltar a internet. Hoje **nada disso existe**; só as páginas são leves e responsivas.
- **Questões com mais erros:** criar `questions` e `answers` por atividade com correção por questão.
- **Notificações por e-mail**, painel de **responsáveis**, **auditoria** mais detalhada.

---

## 9. Problemas comuns

| Sintoma | O que fazer |
|---|---|
| “Falta configurar o Supabase” | Preencha o `.env.local` e reinicie o `npm run dev` |
| Não chega e-mail de confirmação | Veja o spam; confira limites do e-mail embutido; configure SMTP |
| Link de recuperar senha dá erro | Abra o link **no mesmo navegador** em que pediu; confira as *Redirect URLs* |
| “Aguardando aprovação” para o professor | Entre como admin e aprove em *Visão geral* |
| Professor aprovado não vê turmas | Vincule-o em *Escolas e turmas* |
| Assistente diz “Modo demonstrativo” | Falta `AI_PROVIDER` e `AI_API_KEY` no ambiente (e reiniciar) |
| Erro 502 no assistente | Chave inválida, sem crédito ou nome de modelo inválido em `AI_MODEL`. Veja o terminal |
| Mudou variável `NEXT_PUBLIC_…` e nada mudou | Pare e rode de novo (`npm run dev`); na Vercel, faça novo deploy |
