# Guia de instalação — Samalia · Agendamento

O arquivo **`index.html`** já funciona sozinho em **modo demonstração**
(os agendamentos ficam salvos só no navegador de quem abre). Para usar de verdade —
com banco de dados na nuvem, agendamentos vindos de qualquer celular e um painel de
admin com **login seguro** — siga os passos abaixo. Leva ~10 minutos e é de graça.

---

## 1. Criar o projeto no Supabase

1. Entre em <https://supabase.com> e crie uma conta (pode usar login do GitHub/Google).
2. Clique em **New project**.
   - **Name:** `samalia` (ou o que quiser)
   - **Database Password:** gere uma senha forte e **guarde** (não é a senha do admin do site, é a do banco).
   - **Region:** escolha **South America (São Paulo)**.
3. Espere ~2 minutos enquanto o projeto é criado.

---

## 2. Pegar as 2 chaves e colar no site

1. No projeto, menu lateral → **Project Settings** (engrenagem) → **API**.
2. Copie:
   - **Project URL** → algo como `https://abcdefgh.supabase.co`
   - **Project API keys → `anon` `public`** → um texto longo começando com `eyJ...`
3. Abra `index.html` num editor de texto e ache o bloco `CONFIG`.
   Troque só estas duas linhas:

   ```js
   supabase: {
     url: 'https://abcdefgh.supabase.co',      // <- sua Project URL
     anonKey: 'eyJhbGciOiJIUzI1NiIsInR5cCI6...' // <- sua chave anon public
   },
   ```

   > A chave `anon` é **pública por natureza** — pode ficar no arquivo e no ar sem
   > problema. Quem protege os dados são as regras (RLS) do passo 3.

Assim que essas chaves forem válidas, o aviso de "modo demonstração" some sozinho.

---

## 3. Criar a tabela e as regras de segurança (SQL)

1. No Supabase, menu lateral → **SQL Editor** → **New query**.
2. Cole **todo** o bloco abaixo e clique em **Run**.

```sql
-- ========== SAMALIA · AGENDAMENTO — ESQUEMA + SEGURANÇA ==========

create table if not exists public.agendamentos (
  id           uuid primary key default gen_random_uuid(),
  criado_em    timestamptz not null default now(),
  nome         text not null check (char_length(trim(nome)) between 2 and 80),
  telefone     text not null check (char_length(telefone) between 8 and 20),
  servico      text not null check (char_length(servico) between 2 and 60),
  servico_id   text,
  preco        numeric(10,2) check (preco is null or (preco >= 0 and preco < 100000)),
  data         date not null,
  hora         text not null check (hora ~ '^[0-2][0-9]:[0-5][0-9]$'),
  observacao   text check (observacao is null or char_length(observacao) <= 300),
  status       text not null default 'pendente'
               check (status in ('pendente','confirmado','concluido','cancelado')),
  unique (data, hora)            -- impede dois agendamentos no mesmo horário
);

-- Liga o "Row Level Security": nada entra/sai sem uma regra explícita.
alter table public.agendamentos enable row level security;

-- PÚBLICO (visitante não logado): SÓ pode INSERIR, e com regras mínimas.
-- Não pode ler, editar nem apagar nada.
drop policy if exists "anon insere agendamento" on public.agendamentos;
create policy "anon insere agendamento" on public.agendamentos
  for insert to anon
  with check (
    data >= current_date
    and data <= current_date + 120
    and char_length(trim(nome)) between 2 and 80
    and char_length(telefone) between 8 and 20
    and status = 'pendente'
  );

-- ADMIN (logado por e-mail/senha): lê, atualiza e apaga tudo.
drop policy if exists "auth le agendamentos" on public.agendamentos;
create policy "auth le agendamentos" on public.agendamentos
  for select to authenticated using (true);

drop policy if exists "auth atualiza agendamentos" on public.agendamentos;
create policy "auth atualiza agendamentos" on public.agendamentos
  for update to authenticated using (true) with check (true);

drop policy if exists "auth apaga agendamentos" on public.agendamentos;
create policy "auth apaga agendamentos" on public.agendamentos
  for delete to authenticated using (true);

-- HORÁRIOS OCUPADOS para a página pública:
-- função só-leitura que devolve APENAS data + hora (nunca nome/telefone),
-- para o site poder "apagar" os horários já tomados sem expor cliente nenhum.
create or replace function public.horarios_ocupados(de date, ate date)
returns table (data date, hora text)
language sql
stable
security definer
set search_path = public
as $$
  select a.data, a.hora
  from public.agendamentos a
  where a.status <> 'cancelado'
    and a.data between de and ate
$$;

revoke all on function public.horarios_ocupados(date, date) from public;
grant execute on function public.horarios_ocupados(date, date) to anon, authenticated;
```

Deve aparecer **Success. No rows returned**. Pronto — banco e regras criados.

---

## 4. Fechar o cadastro e criar SÓ o seu login de admin

Assim ninguém consegue se registrar sozinho — só existe a conta que você criar.

1. Menu lateral → **Authentication** → **Providers** (ou **Sign In / Providers**).
2. Em **Email**, **desligue** a opção **"Allow new users to sign up"** (Enable Sign-ups) e salve.
3. Menu lateral → **Authentication** → **Users** → **Add user** → **Create new user**:
   - **Email:** o seu e-mail
   - **Password:** uma senha forte (essa é a que você vai usar no painel)
   - Marque **Auto Confirm User** (para não precisar confirmar por e-mail).
   - **Create user**.

Esse é o único acesso ao painel. Se um dia esquecer a senha, volte aqui em
**Users → (seus três pontinhos) → Reset password / Send magic link**.

---

## 5. Testar

1. Abra o `index.html` (duplo clique já serve; veja o passo 6 para deixar online).
2. Faça um agendamento de teste na página pública.
3. Clique no botão **ADM** no topo (ou adicione `#/admin` no fim do endereço).
4. Entre com o **e-mail e senha** do passo 4.
5. O agendamento de teste tem que aparecer na lista, com nome e telefone do cliente.

---

## 6. (Recomendado) Deixar o site no ar com HTTPS

Abrir por `file://` funciona, mas o ideal é ter um link `https://` para mandar às clientes.
O site é **um arquivo só** (`index.html`), já com as fotos embutidas — é só subir.

**GitHub Pages**
1. Crie um repositório novo em <https://github.com/new> (ex.: `agenda-samalia`), público.
2. **Add file → Upload files** → arraste o `index.html` → **Commit changes**.
3. **Settings → Pages → Branch:** `main` / `/root` → **Save**.
4. Em ~1 min o site fica em `https://SEU-USUARIO.github.io/agenda-samalia/`.

**AWS S3 (site estático)**
1. Console S3 → **Create bucket** (nome único, ex.: `agenda-samalia`), desmarque "Block all public access".
2. **Upload** o `index.html`.
3. **Properties → Static website hosting → Enable**, index document = `index.html`.
4. **Permissions → Bucket policy**: libere `s3:GetObject` público para `arn:aws:s3:::agenda-samalia/*`.
5. Use o "Bucket website endpoint" que aparece em Static website hosting.
   (Para HTTPS + domínio próprio, ponha um CloudFront na frente.)

**Mais rápido ainda:** <https://app.netlify.com/drop> — arraste o `index.html` e o link sai na hora.

Depois é só divulgar o link. O painel fica no mesmo link + `#/admin` (ou botão **ADM**).

---

## Onde mudar preços, serviços, endereço e contatos

Tudo fica no **bloco `CONFIG`** no topo do `<script>` dentro do `index.html`
(procure por `██ CONFIGURAÇÃO ██`). Você edita direto ali:

| O que | Onde no CONFIG |
|---|---|
| Nome do studio, rodapé | `marca` |
| WhatsApp e Instagram | `contato` |
| Endereço, cidade, referência, ponto no mapa | `local` |
| Dias e horários de atendimento, duração dos encaixes, almoço | `horario` |
| Lista de serviços com **preço** e duração | `servicos` |
| Chaves do Supabase | `supabase` |
| Trava de inatividade, tentativas de login, limite anti-spam | `seguranca` |

> Se você mudar `horario.diasParaFrente` para mais de **120**, aumente também o
> `current_date + 120` na política de INSERT do SQL (passo 3).

---

## Como a "segurança reforçada" funciona aqui

**Painel do admin**
- Login por **e-mail + senha reais** validados no servidor do Supabase (senha guardada
  com hash bcrypt, tokens JWT com expiração) — não há senha escrita no código.
- **Cadastro público desligado:** só existe a conta que você criou.
- **Trava por tentativas:** após 5 erros, o login trava por 5 minutos naquele aparelho.
- **Logout automático** após 20 minutos parado.
- Sessão isolada por origem; nada dos agendamentos fica acessível sem login.

**Formulário público**
- **RLS (Row Level Security):** o visitante só consegue *inserir* um agendamento.
  Ler, editar ou apagar é **impossível** sem estar logado como admin.
- Os horários ocupados aparecem no site por uma função que devolve **só data e hora** —
  nome e telefone de outras clientes nunca saem do banco.
- **`UNIQUE(data, hora)`** no banco: dois pedidos para o mesmo horário — o segundo é
  recusado pelo próprio banco, sem depender do navegador.
- **Honeypot** (campo escondido) barra robôs de spam.
- **Limite por aparelho:** no máx. 3 agendamentos/hora e 20s entre envios.
- **Sanitização + validação** de nome, telefone e observação (remove HTML, caracteres
  de controle e marcas invisíveis; exige DDD; limita tamanho).
- Datas no passado e "em cima da hora" são bloqueadas no site **e** na regra do banco.

**Valores que ajudam a fortalecer ainda mais (opcional)**
- Ativar **"Confirm email"** e usar um e-mail de admin de verdade.
- No Supabase, **Authentication → Rate limits**, reduzir tentativas por hora.
- Hospedar em HTTPS (passo 6) — evita qualquer tráfego em texto puro.
