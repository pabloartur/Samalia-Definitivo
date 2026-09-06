# Samalia · Agendamento

Site de agendamento da **Samalia designer de unhas** — página pública para as clientes
marcarem horário + painel da profissional (botão **ADM**) para ver todos os agendamentos.

## Arquivos

| Arquivo | Para quê |
|---|---|
| **`index.html`** | O site inteiro num arquivo só (já com as fotos embutidas). É o que você publica. |
| `GUIA-SUPABASE.md` | Passo a passo para ligar o banco de dados + login seguro, e para publicar (GitHub / AWS / Netlify). |
| `catalogo-samalia-roxo-amarelo.html` | O catálogo antigo (referência de visual). Não é usado pelo site novo. |

## Publicar (resumo)

O `index.html` é estático e autossuficiente — sobe em qualquer lugar:

- **GitHub Pages:** novo repositório → upload do `index.html` → Settings → Pages → branch `main`.
- **AWS S3:** bucket com *Static website hosting*, upload do `index.html`, policy pública de leitura (HTTPS via CloudFront).
- **Netlify:** arraste o `index.html` em <https://app.netlify.com/drop>.

Detalhes de cada um no `GUIA-SUPABASE.md`, passo 6.

## Funciona sem configurar nada (modo demonstração)

Abrindo o `index.html` direto, ele roda em modo demonstração: os agendamentos ficam
salvos só no navegador e o painel abre com a senha **`samalia`**.

Para uso real (agendamentos de qualquer celular + login de verdade), siga o
`GUIA-SUPABASE.md` — é grátis e leva ~10 min.

## Onde editar preços, endereço, horários e contatos

No bloco `CONFIG`, logo no começo do `<script>` do `index.html` (procure por
`██ CONFIGURAÇÃO ██`). Tudo comentado.
