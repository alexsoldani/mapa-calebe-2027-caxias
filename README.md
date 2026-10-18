# Missão Calebe 2027 — Mapa territorial 3D

Site estático e auto-contido para explorar o território **Itatiaia · Vila São Luís · Sarapuí (Duque de Caxias – RJ)** em 2.5D/3D: bairros-foco, pontos de interesse, ruas, rios, terreno, prédios 3D e **10 oportunidades de missão** clicáveis com painel lateral, deep links e interface PT/EN.

- **Página:** [map/index.html](index.html) — HTML, CSS e JS inline, sem build.
- **Dados:** map/data/*.json, carregados com fetch().
- **Screenshots de verificação:** [map/screenshots/](screenshots/)
- **Sem chaves de API, sem backend, sem bundler.** Pode ser publicado como está no GitHub Pages.

---

## 1. Rodar localmente

O uso de fetch() exige HTTP (não abra o arquivo com file://).

~~~bash
# a partir da raiz deste repositório
python3 -m http.server 8765 --directory map
~~~

Abra **http://localhost:8765/**.

Teste rápido dos arquivos:

~~~bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8765/
for f in bairros.geojson pois.json streets.json water.json oportunidades.json; do
  curl -fsS "http://localhost:8765/data/$f" | python3 -m json.tool > /dev/null && echo "OK $f"
done
~~~

## 2. Publicar no GitHub Pages

1. Envie o repositório para o GitHub.
2. **Settings → Pages → Build and deployment → Deploy from a branch**, escolha main e a pasta onde o map/ está (por exemplo / (root)).
3. Acesse https://<usuario>.github.io/<repositorio>/map/.

Todos os caminhos são **relativos** (data/bairros.geojson, CDN HTTPS), então funciona tanto na raiz de um domínio quanto em um subdiretório (/repo/map/). Não é necessário servidor, build ou variável de ambiente.

> Se preferir publicar o mapa como site principal, copie o **conteúdo** de map/ para a raiz do branch de Pages (mantendo data/ ao lado de index.html).

## 3. Estrutura

~~~text
map/
├── index.html                  # página única (CSS + JS inline, MapLibre via CDN)
├── README.md                   # este arquivo
├── data/
│   ├── bairros.geojson         # polígonos aproximados (foco + contexto)
│   ├── pois.json               # pontos de interesse (OSM)
│   ├── streets.json            # vias principais como LineStrings
│   ├── water.json              # rios/canais/córregos como LineStrings
│   └── oportunidades.json      # 10 oportunidades (PT + EN)
└── screenshots/                # evidências de verificação headless
~~~

## 4. Como o mapa funciona

### Tecnologia

| Item | Implementação |
|---|---|
| Mapas | **MapLibre GL JS v5** via CDN (unpkg.com/maplibre-gl@^5) |
| Basemap | Estilo **OpenFreeMap "liberty"** (tiles.openfreemap.org/styles/liberty) |
| Terreno | setTerrain + DEM **AWS Terrarium** (elevation-tiles-prod/terrarium/{z}/{x}/{y}.png, encoding terrarium, tileSize 256, maxzoom 15, exagero 1.12) |
| Céu/atmosfera | map.setSky(...) com névoa azul-noite + overlay CSS de vinheta |
| Prédios 3D | fill-extrusion da source openmaptiles, source-layer building, com render_height/render_min_height e fallback de ~8 m (mínimo 4 m) |
| Bairros | fill suave + fill-extrusion de laje (~7 m; foco 0.42 de opacidade, contexto 0.10) + contorno com glow + rótulos grandes |
| POIs | circle colorido por categoria + symbol com ícones desenhados em canvas em runtime (sem sprite externo) |
| Oportunidades | Marcadores HTML numerados 1–10; prioridade 1 pulsa suavemente (respeitando prefers-reduced-motion) |
| Tour | map.flyTo() encadeado pelos 4 bairros-foco com easing, pitch e bearing próprios |

Nenhuma chave de API é usada. A atribuição obrigatória aparece no canto inferior direito (OpenFreeMap / OpenStreetMap / OpenMapTiles / AWS Terrain Tiles) e começa recolhida no modo compacto.

### Interações

- **Clique em bairro** → painel compacto com nome, tipo (foco/contexto) e descrição (PT/EN).
- **Clique em POI** → popup com nome, categoria, endereço e tags.
- **Clique em pino numerado** → painel lateral (desktop) ou bottom sheet (celular) com público-alvo, descrição, atividades, justificativa, impacto e parcerias.
- **Tour pelos bairros** → sobrevoo animado Itatiaia → Vila São Luís → Parque Sarapuí → Sarapuí, com legenda e botão de parada.
- **Visão geral** → retorna ao enquadramento inicial.
- **Camadas** → Bairros, POIs, Oportunidades, Ruas, Rios, Prédios 3D e Terreno 3D.
- **Categorias de POIs** → filtros combináveis (8 categorias) + "Mostrar todas".
- **PT/EN** → traduz a interface, descrições de bairro e todo o conteúdo das oportunidades (armazenado nos dois idiomas).
- **Deep link** → #op-3 (também aceita #op=3 e ?op=3) abre a oportunidade e centraliza a câmera. O botão **Copiar link** copia a URL direta.
- **Acessibilidade** → rótulos ARIA, foco visível, fechamento com **Esc**, retorno de foco ao pino, armadilha de foco no painel, alvos ≥ 44 px, link de salto, redução de movimento.

### Visão inicial

~~~js
center: [-43.2940, -22.7690], zoom: 13.2, pitch: 55, bearing: -22
~~~

Em telas estreitas o zoom inicial é reduzido (~12.75) para manter os quatro bairros-foco visíveis; Sarapuí fica ~3,3 km ao norte. O tour aproxima (~z15) de cada área.

### Fallback do basemap

Antes de criar o mapa, a página faz um preflight do estilo. Se o estilo falhar (rede/HTTP) ou demorar além de ~26 s, aparece um overlay de erro com **Tentar novamente**, que refaz o boot sem recarregar a página. Depois do carregamento, um watchdog de 32 s evita uma tela de splash travada.

## 5. Formato dos dados

> Todos os coords usam **[lon, lat]**. Já pois.json e oportunidades.json usam campos separados lat e lon (a página converte para [lon, lat]).

### 5.1 data/bairros.geojson

FeatureCollection com 15 polígonos aproximados.

| Propriedade | Tipo | Descrição |
|---|---|---|
| key | string | chave estável (usada nas descrições e no tour) |
| nome | string | nome exibido no mapa e nos painéis |
| tipo | "foco" ou "contexto" | foco recebe cor/volume/rótulo forte; contexto é esmaecido |
| cor | string hex | cor do preenchimento, contorno e rótulo |

Bairros-foco atuais: **Itatiaia** #ff7a45, **Vila São Luiz** #4ea8ff, **Parque Sarapuí** #38d39f, **Sarapuí** #c084fc.

Para adicionar um bairro-foco: adicione o Feature, use tipo "foco" e inclua uma entrada em BAIRRO_INFO no index.html (busque por "var BAIRRO_INFO"). O tour usa TOUR_STOPS e precisa ser ajustado se você quiser incluí-lo no sobrevoo.

### 5.2 data/pois.json

Array de pontos:

| Campo | Tipo | Descrição |
|---|---|---|
| id | string | identificador único (ex.: n331873876) |
| nome | string | nome do local (pode ser vazio) |
| cat | string | educacao, saude, fe, lazer, comercio, transporte, social, seguranca |
| lat, lon | number | coordenadas WGS84 |
| sub | string | subtipo OSM opcional (ex.: pharmacy) |
| tags | object | tags OSM relevantes (addr:street, operator, religion, ...) |

Cores e rótulos das categorias ficam em "var CATS" no index.html; os ícones são gerados em canvas (funções em ICON_DRAW), sem sprite externo.

### 5.3 data/streets.json

Array de vias: **{ "nome": string, "classe": "motorway" | "trunk" | "primary" | "secondary" | "tertiary", "coords": [[lon,lat], ...] }**.

Larguras e cores são expressões por classe e zoom; vias motorway/trunk recebem destaque âmbar.

### 5.4 data/water.json

Array de cursos d'água: **{ "nome": string, "tipo": "river" | "canal" | "stream" | "water" | "reservoir", "coords": [[lon,lat], ...] }**.

Rios recebem linha mais larga e rótulo de linha; o **Rio Sarapuí** está entre eles. Nomes vazios simplesmente não geram rótulo.

### 5.5 data/oportunidades.json — schema estável

Array de **10 objetos**. Todos os campos abaixo são obrigatórios (o Lead pode substituir os textos PT/EN mantendo o schema):

| Campo | Tipo | Descrição |
|---|---|---|
| id | string | estável, no formato op-1 … op-10 |
| num | number | 1–10; define o número do pino e o deep link #op-N |
| nome_pt, nome_en | string | nomes nos dois idiomas |
| bairro | string | bairro de referência (texto livre, ex.: "Vila São Luiz / Itatiaia") |
| prioridade | number 1–3 | **1 = mais alta**; os três itens com prioridade 1 pulsam no mapa |
| lat, lon | number | coordenadas WGS84 |
| publico_pt, publico_en | string | público-alvo |
| descricao_pt, descricao_en | string | 2–3 frases de descrição |
| atividades_pt[], atividades_en[] | string[] | 3–4 atividades sugeridas |
| justificativa_pt, justificativa_en | string | 1–2 frases ligadas ao diagnóstico territorial |
| impacto_pt, impacto_en | string | impacto esperado |
| parcerias_pt, parcerias_en | string | parcerias e notas de logística |

Prioridades atuais (placeholders): **1, 3, 7** com prioridade 1; os demais com 2 ou 3.

Exemplo mínimo de um item:

~~~json
{
  "id": "op-11",
  "num": 11,
  "nome_pt": "Nova ação",
  "nome_en": "New action",
  "bairro": "Itatiaia",
  "prioridade": 2,
  "lat": -22.7779,
  "lon": -43.3018,
  "publico_pt": "Famílias do bairro.",
  "publico_en": "Neighbourhood families.",
  "descricao_pt": "Descrição curta.",
  "descricao_en": "Short description.",
  "atividades_pt": ["Atividade 1", "Atividade 2", "Atividade 3"],
  "atividades_en": ["Activity 1", "Activity 2", "Activity 3"],
  "justificativa_pt": "Justificativa territorial.",
  "justificativa_en": "Territorial rationale.",
  "impacto_pt": "Impacto esperado.",
  "impacto_en": "Expected impact.",
  "parcerias_pt": "Parcerias e logística.",
  "parcerias_en": "Partnerships and logistics."
}
~~~

**Para adicionar uma 11ª oportunidade:** acrescente o objeto com num: 11 e id: "op-11". O pino, o painel e o deep link #op-11 são criados automaticamente; não é preciso mexer no JS. Mantenha num e id únicos e ordenados.

**Para editar textos:** altere apenas os campos *_pt / *_en em data/oportunidades.json. O painel é montado em runtime e aceita texto com acentos, quebras e pontuação normalmente (o conteúdo é escapado antes de entrar no DOM).

## 6. Personalização rápida (index.html)

Procure por estas âncoras dentro do script inline:

| Âncora | O que controla |
|---|---|
| var UI = { pt: ..., en: ... } | todos os rótulos da interface nos dois idiomas |
| var BAIRRO_INFO / CONTEXT_BAIRRO_INFO | descrições dos bairros-foco e de contexto (PT/EN) |
| var CATS | cores, rótulos e desenho dos ícones de POIs |
| var TOUR_STOPS | paradas do tour (centro, zoom, pitch, bearing, nome) |
| var OVERVIEW, TABLET_OVERVIEW, MOBILE_OVERVIEW | enquadramentos iniciais por largura de tela |
| var MAP_STYLE_URL, DEM_URL | basemap e DEM |
| PRIORITY_COLORS | cores dos selos de prioridade |
| CSS :root | paleta, tipografia, raios, sombras |

Para trocar o idioma padrão, ajuste state.lang (padrão pt). A escolha do usuário é lembrada em localStorage (mc-lang).

## 7. Performance, acessibilidade e UX

- pixelRatio limitado a 2, pitch inicial 55°, camadas moderadas e ícones desenhados em canvas.
- prefers-reduced-motion: desativa pulso dos pinos, animações de interface e transforma o tour em saltos de câmera com pausas.
- Alvos de toque ≥ 44 px; controles colapsam em bottom sheet no celular (testado em 390×844 e layout previsto para 360 px).
- Foco visível, título e rótulos ARIA, anúncio em região aria-live, fechamento por **Esc** e retorno de foco ao pino de origem.
- Atribuição obrigatória de mapa/dados sempre visível (modo compacto recolhível).
- Vinheta CSS e céu azul-noite dão atmosfera sem prejudicar a leitura dos overlays coloridos.

## 8. Verificação realizada

- Servidor local python3 -m http.server 8765 --directory map; index.html e os 5 JSONs responderam **HTTP 200** e todos os JSONs foram parseados.
- node --check no conteúdo do script inline (extraído para arquivo temporário) → sem erros de sintaxe.
- Chrome headless (SwiftShader/WebGL2) via CDP, com captura de console/exceções:
  - desktop 1440×900 e celular 390×844;
  - terreno ativo, 10 marcadores, 3 com pulso, camadas presentes (mc-*), prédios 3D e rótulos renderizados;
  - abertura do painel por clique, PT/EN, deep link #op-7, **Esc** fechando e devolvendo o foco;
  - toggles de camada, filtros de categoria e tour iniciado/parado;
  - **overlay de erro** ao bloquear tiles.openfreemap.org e **recuperação** após "Tentar novamente";
  - **0 erros de console / exceções** nas páginas testadas.
- Screenshots em [screenshots/](screenshots/) (desktop, painel, inglês, mobile, controles, deep link, fallback e recuperação).

Comandos úteis de manutenção:

~~~bash
# checar sintaxe do JS inline (macOS/Linux, Python 3 + Node)
python3 - <<'PY'
import re
html = open('map/index.html', encoding='utf-8').read()
js = re.findall(r'<script>(.*?)</script>', html, re.S)[0]
open('/tmp/map-inline.js', 'w', encoding='utf-8').write(js)
PY
node --check /tmp/map-inline.js && echo "JS OK"
~~~

## 9. Limitações e dependências

- **Polígonos aproximados**, não limites oficiais de bairro; os POIs vêm de uma amostra OSM e podem estar incompletos/desatualizados. O mapa é de diagnóstico e comunicação, não de georreferenciamento legal.
- **Precisa de internet**: MapLibre (unpkg), estilo/tiles (OpenFreeMap), DEM (AWS) e fonte Inter (Google Fonts, com fallback para fontes do sistema). Não há modo offline.
- **WebGL 2** é necessário; navegadores antigos exibem o overlay explicativo.
- O fallback de erro cobre a falha do **basemap/estilo**. Falhas isoladas de tiles de terreno/relevo não derrubam o mapa (podem gerar avisos no console do navegador).
- O DEM público da AWS é um serviço de demonstração; disponibilidade e precisão não são garantidas.
- CDN usa @^5 para acompanhar correções do MapLibre. Para produção de longo prazo, **fixe uma versão exata** (ex.: maplibre-gl@5.14.0) no link e no script.
- O estilo "liberty" é de tema claro; a interface é escura de propósito, garantindo contraste e legibilidade dos overlays coloridos. Se quiser um basemap escuro, troque MAP_STYLE_URL por um estilo escuro compatível (a atribuição e o fallback continuam válidos).
- Screenshots foram gerados com Chrome headless + SwiftShader; a renderização em GPU real pode ter pequenas diferenças de antialiasing.
- Os textos de oportunidades.json são **placeholders de trabalho**; o Lead substituirá por conteúdo final baseado em evidências, mantendo schema, ids, números e coordenadas.

## 10. Créditos e licenças

- Dados de mapa: © OpenStreetMap contributors (ODbL).
- Basemap/estilo: © OpenFreeMap, © OpenMapTiles.
- Elevação: AWS Terrain Tiles (Terrarium).
- Biblioteca: MapLibre GL JS (BSD-3-Clause).
- Tipografia: Inter (SIL Open Font License), via Google Fonts.

O código desta página e os textos das oportunidades são do projeto **Missão Calebe 2027**.
