# teupadel: marca e identidade visual

!!! info "Fonte de verdade / atualizar aqui"
    Esta página junta e substitui `contexto-identidade-visual.md` e a skill `teupadel-brand-context`,
    corrigidas com o estado real da UI (`ui.ia.teupadel.com`: `src/index.css`, `public/brand/`,
    `src/fonts/`). Em caso de dúvida, o código da UI vence. Atualizar aqui (PT + EN) quando a marca mudar.
    Implementação: [teupadel UI](../apps/teupadel-ui.md).

## O que é

O teupadel usa IA para analisar o movimento de um jogador de padel a partir de um vídeo. O utilizador grava
uma jogada com o telemóvel, faz upload e recebe em segundos um relatório com pontos fortes, pontos a
melhorar e sugestões práticas por golpe: o feedback que normalmente só um treinador dava.

Frase-âncora (hero): "A tua análise de padel, feita por IA." Eyebrow: "1.ª análise grátis". O produto
**não se posiciona como substituto de um treinador**, mas como ponto de partida acessível entre treinos:
tom de confiança e competência técnica, sem soar clínico nem pretensioso.

**Modelo comercial atual:** login obrigatório para analisar; cada análise custa 1 bola; a conta nova recebe
3 bolas de boas-vindas e as indicações dão +1 a cada lado; `/pricing` mostra planos de bolas por região
(BR e EU), ainda sem checkout. A marca não deve prometer "grátis, sem limites".

## Público e mercados

Jogadores amadores de padel sem acesso regular a um treinador. Três mercados via prefixo de URL: `/pt-pt`,
`/pt-br` e `/en`. A identidade tem de funcionar nos três (evitar referências culturais só de PT ou só de BR).

## Como funciona (para o storytelling visual)

1. Upload de vídeo (telemóvel, sem equipamento).
2. Visão computacional extrai a pose frame a frame (17 pontos: ombros, cotovelos, pulsos, quadris, joelhos,
   tornozelos, etc.).
3. Um LLM transforma os dados num relatório em linguagem natural.
4. O utilizador vê o **seu vídeo com o esqueleto detetado desenhado ao vivo por cima** (overlay em canvas,
   `PoseCanvasOverlay`) e miniaturas dos momentos do golpe, com links "Ver no vídeo". Não há GIF.

O esqueleto (linhas e nós a ligar articulações) é o elemento mais distintivo do produto; a identidade
dialoga com ele (linhas, nós, trajetórias) em vez de ir só por bola e raquete. Golpes: serviço, direita,
esquerda, vôlei, remate e posição de espera.

## Nome e tom

- O nome escreve-se sempre **teupadel**, tudo em minúsculas, mesmo no início de frase ou em títulos.
- Tom: acessível, encorajador, tecnicamente credível, não intimidante. Linguagem de quadra: os relatórios
  não trazem graus, ângulos nem centímetros.
- **Privacidade como valor:** o vídeo nunca é guardado (só o JSON do relatório), há banner de cookies e
  página de privacidade. Transmitir "confiável" e "transparente", não "vigilância".
- **Disclaimer sempre visível** nos relatórios: é IA, pode errar, não substitui um treinador. Tom humilde.
- Site institucional e ferramenta na mesma app, no domínio `teupadel.com`.

## Identidade visual atual

Tema **único claro**; não há tema escuro nem toggle. Tokens em `src/index.css` (nomes em português).

| Token | Valor | Uso |
|---|---|---|
| `--quadra` | `#2B5BFF` | Azul principal: links, botões secundários, logo, foco |
| `--bola` | `#C8FF3D` | Verde-limão: CTA principal (`tp-btn-principal`), esqueleto (`--esqueleto`) |
| `--noite` / `--on-noite` | `#0E1220` / `#FFFFFF` | Faixas escuras fixas (hero, rodapé, banners) |
| `--tinta` / `--tinta-suave` | `#0E1220` / `#525B70` | Texto |
| `--superficie` / `--superficie-elevada` | `#F6F7FA` / `#FFFFFF` | Fundos |
| `--vidro` / `--linha` / `--borda` | `#EDF0FF` / `#DCE1EC` / `#7D869A` | Realces, divisórias, bordas |
| `--positivo` / `--melhorar` / `--erro` | `#0A7266` / `#A84B06` / `#BE2F28` | Pontos fortes, a melhorar, erros (cada um com variante `-suave`) |
| `--selo` / `--vidro-escura` | `#3ED0C9` / `#1A2338` | Acentos dentro de cartões escuros |

Espaçamento `--space-1..20` (4 a 80 px), raios `--radius-s/m/l` (6/12/24 px) e `--radius-bola` (círculo),
sombra `--sombra-cartao`. Classes partilhadas: `tp-btn` (`principal`, `secundario`, `quadra`, `noite`),
`tp-tag`, `tp-card`, `tp-rotulo`.

**Tipografia** (self-hosted, `next/font/local`, licença OFL): Unbounded 500-800 (display, logo, títulos),
Figtree 400/500/700 (corpo), JetBrains Mono 500/600 (dados e timestamps).

**Assets** em `public/brand/` (vêm do design system da marca): `logo-horizontal.svg` e
`logo-horizontal-negativo.svg`, `simbolo.svg` (barra vertical mais pontos em diagonal ascendente, a
trajetória), `selo-quadra.svg` (quadrado azul com linhas de quadra e bola-limão, é também o `icon.svg`),
`raquete.svg` / `raquete-branca.svg` / `raquete-limao.svg` e `favicon-180.png`. Não reintroduzir texto
"TeuPadel" nem o ícone antigo no lugar do logo real.

## Ainda em aberto

- Ilustrações e iconografia própria para cada golpe.
- Diretrizes de tom visual escritas (hoje implícitas no copy e nos tokens).
- Versão escura do design system: fica para quando o site tiver seletor de tema.

!!! warning "Histórico / a confirmar"
    Estas afirmações vinham dos documentos antigos e **não se verificam** no código atual; não usar sem
    confirmar com o dono do produto:

    - Que a telemetria é "opt-in e anonimizada" tal como descrita. Existe banner de consentimento
      (`CookieConsent`) e `/cookies` lista `tp_consent`/`tp_vid`/`tp_sid`, mas o comportamento exato antes
      do consentimento não foi reverificado para esta página.
    - Que o produto é "gratuito, sem paywall" (era verdade antes das bolas; hoje só a 1.ª análise é grátis).

    A paleta escura `#0a1018` com laranja `#ffa53e`, a tipografia Oswald + Inter, o ícone laranja com
    quatro pontos, a ausência de logotipo e o "GIF do vídeo com esqueleto" eram do protótipo inicial e
    foram substituídos; deixaram de ser referência.
