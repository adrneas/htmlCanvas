# Tech Base — HTML-in-Canvas para Desenvolvimento com IA Agêntica

> Versão otimizada para uso como contexto técnico em Codex/agentes.  
> Estado da tecnologia: experimental, em proposta WICG e Origin Trial/flags Chromium em maio de 2026. Não tratar como API estável.

---

## 0. Objetivo deste arquivo

Este documento deve servir como **fonte de verdade operacional** para agentes que irão desenvolver, revisar ou refatorar código usando **HTML-in-Canvas**.

O agente deve usar este arquivo para:

- entender a arquitetura correta da API;
- evitar APIs inventadas, nomes incorretos ou padrões frágeis;
- gerar código com fallback;
- preservar acessibilidade e interatividade;
- separar claramente protótipo experimental de código de produção;
- validar o resultado com critérios objetivos.

Este arquivo **não** deve ser tratado como documentação oficial da especificação. Sempre que houver conflito com a documentação oficial/WICG/Chromium, a fonte oficial prevalece.

---


## 1. Contratos TypeScript de Interface

Use estes contratos como âncora de geração. Eles são parciais e representam a superfície mínima esperada para POCs. Não inventar métodos fora destes contratos sem validação explícita em fonte oficial ou adapter isolado.

```ts
declare interface HTMLCanvasElement extends HTMLElement {
  layoutsubtree?: boolean;
  onpaint: ((this: HTMLCanvasElement, ev: Event) => any) | null;
  requestPaint?: () => void;
  captureElementImage?: (element: Element) => ImageBitmap | CanvasImageSource;
}

declare interface CanvasRenderingContext2D {
  drawElementImage(element: Element, x: number, y: number): DOMMatrix;
}

type HtmlInCanvasSupport = {
  canvas: boolean;
  context2d: boolean;
  drawElementImage: boolean;
  paintEvent: boolean;
  requestPaint: boolean;
};

type HtmlInCanvasMode = "native" | "fallback";

type HtmlInCanvasRenderTarget = {
  canvas: HTMLCanvasElement;
  ctx: CanvasRenderingContext2D;
  element: Element;
  x: number;
  y: number;
};

type HtmlInCanvasCleanup = () => void;
```

Agentes devem usar estes contratos para:

- guiar feature detection;
- evitar helpers inventados, como `transform.toCSS()`;
- separar modo nativo e fallback;
- garantir cleanup explícito;
- isolar APIs WebGL/WebGPU/engine-specific em adapters tipados.

---

## 2. Resumo executivo

HTML-in-Canvas é uma proposta experimental que permite renderizar elementos reais do DOM dentro de um `<canvas>`, mantendo capacidades nativas do navegador que seriam perdidas em uma renderização puramente pixel-based.

A proposta combina:

1. **DOM real**
   - layout HTML/CSS;
   - formulários;
   - seleção de texto;
   - acessibilidade;
   - busca nativa na página;
   - integração com DevTools e extensões.

2. **Canvas/WebGL/WebGPU**
   - composição gráfica;
   - efeitos visuais;
   - texturas;
   - cenas 2D/3D;
   - pipelines gráficos customizados.

A lógica central é:  
**o DOM continua existindo e participando de layout/hit testing, mas sua aparência visual é copiada para o canvas quando o navegador dispara um evento de paint.**

---

## 3. Status tecnológico

### 2.1 Maturidade

A tecnologia está em estágio experimental.

Assuma:

- API sujeita a mudanças;
- suporte limitado a Chromium/Chrome Canary/Origin Trial/flags;
- ausência de suporte universal entre navegadores;
- necessidade obrigatória de fallback;
- incompatibilidade possível com frameworks e bibliotecas atuais;
- risco alto para uso direto em produto final sem camada de abstração.

### 2.2 Decisão recomendada

Use HTML-in-Canvas em:

- protótipos;
- pesquisa técnica;
- demos;
- experiências internas;
- POCs;
- features progressivamente aprimoradas com fallback DOM tradicional.

Evite depender exclusivamente da API em:

- fluxos críticos de produto;
- interfaces sem fallback;
- features que precisam funcionar em todos os navegadores;
- entregas onde acessibilidade precisa ser garantida imediatamente em produção.

---

## 4. Vocabulário e nomes corretos

### 3.1 Nomes corretos

Use estes nomes:

```html
<canvas layoutsubtree>
```

```js
ctx.drawElementImage(element, x, y)
```

```js
canvas.onpaint = () => {}
canvas.addEventListener("paint", handler)
```

```js
canvas.requestPaint?.()
```

```js
gl.texElementImage2D(...)
```

```js
device.queue.copyElementImageToTexture(...)
```

```js
canvas.captureElementImage(element)
```

### 3.2 Nomes incorretos ou suspeitos

Não usar:

```html
<canvas layout subtree>
```

Motivo: o atributo correto é `layoutsubtree`, sem espaço.

Não assumir que existe:

```js
transform.toCSS()
```

Motivo: os exemplos oficiais usam `transform.toString()` ou aplicam diretamente a string retornada/serializada.

Não assumir como estável:

```js
new THREE.HTMLTexture(element)
```

Motivo: há demos/integrações experimentais, mas não tratar como API estável do Three.js sem verificação da versão/biblioteca usada.

Não inventar helpers como:

```js
gl.getElementTransform(element)
canvas.getElementTransform(element)
```

Só usar se estiverem confirmados no ambiente ou documentados na versão alvo. A proposta menciona helper de transformação para contextos 3D, mas o agente deve tratar isso como área experimental e validar a assinatura real.

---

## 5. Modelo mental da API

### 4.1 Estrutura

O conteúdo HTML fica dentro do próprio `<canvas>`:

```html
<canvas id="stage" layoutsubtree>
  <div id="panel">
    <label for="name">Nome</label>
    <input id="name" />
    <button type="button">Enviar</button>
  </div>
</canvas>
```

O elemento interno:

- participa de layout;
- participa de hit testing;
- pode ser acessível;
- pode receber foco;
- não aparece automaticamente na tela como DOM comum;
- precisa ser desenhado explicitamente no canvas.

### 4.2 Renderização

O navegador tira um snapshot interno da renderização dos filhos do canvas.

Durante o evento `paint`, o código chama:

```js
ctx.drawElementImage(panel, x, y)
```

Isso desenha o elemento no canvas e retorna uma transformação que deve ser aplicada ao elemento DOM para manter o hit testing, foco e acessibilidade alinhados com os pixels desenhados.

### 4.3 Sincronização

Regra crítica:

> Sempre que um elemento DOM for desenhado em uma posição/escala/transformação dentro do canvas, sua posição DOM invisível/interativa deve ser sincronizada com a posição visual desenhada.

Sem isso, o usuário pode ver o botão em um lugar e o navegador entender que ele está em outro.

---

## 6. Contrato estrutural obrigatório

Qualquer implementação gerada por agente deve respeitar este contrato.

### 5.1 HTML

```html
<canvas id="stage" layoutsubtree>
  <div id="ui-root">
    <!-- UI real e semântica aqui -->
  </div>
</canvas>
```

Regras:

- o atributo deve ser `layoutsubtree`;
- elementos desenhados devem ser filhos diretos do canvas quando exigido pela API;
- não usar `display: none` nos elementos que serão desenhados;
- preferir esconder visualmente via mecanismo da própria API, não via CSS que remova boxes;
- manter HTML semântico: `button`, `input`, `label`, `form`, `nav`, etc.

### 5.2 CSS

```css
#stage {
  width: 800px;
  height: 600px;
}

#ui-root {
  transform-origin: 0 0;
}
```

Regras:

- definir tamanho CSS do canvas;
- sincronizar o tamanho interno do canvas com o tamanho físico em device pixels;
- usar `transform-origin` explícito quando a posição precisa ser precisa;
- evitar depender de CSS transform no elemento-fonte para o desenho, pois CSS transforms do elemento podem ser ignorados pelo desenho e ainda afetar hit testing.

### 5.3 JavaScript 2D mínimo

```js
const canvas = document.querySelector("#stage");
const ctx = canvas.getContext("2d");
const uiRoot = document.querySelector("#ui-root");

function supportsHtmlInCanvas2D() {
  return Boolean(
    canvas &&
    ctx &&
    "drawElementImage" in CanvasRenderingContext2D.prototype &&
    "onpaint" in canvas
  );
}

function syncCanvasResolution() {
  const rect = canvas.getBoundingClientRect();
  const dpr = window.devicePixelRatio || 1;

  canvas.width = Math.max(1, Math.round(rect.width * dpr));
  canvas.height = Math.max(1, Math.round(rect.height * dpr));

  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
}

const resizeObserver = new ResizeObserver(() => {
  syncCanvasResolution();
  canvas.requestPaint?.();
});

resizeObserver.observe(canvas);

canvas.addEventListener("paint", () => {
  ctx.reset?.();
  syncCanvasResolution();

  const transform = ctx.drawElementImage(uiRoot, 100, 80);

  uiRoot.style.transformOrigin = "0 0";
  uiRoot.style.transform = transform.toString();
});
```

Observação: `ctx.reset()` ainda pode não estar disponível em todos os ambientes. O agente pode usar fallback com `ctx.setTransform(1, 0, 0, 1, 0, 0)` + `clearRect`.

---

## 7. Feature detection e fallback

### 6.1 Regra obrigatória

Todo código gerado deve conter fallback.

Nunca gerar uma implementação que dependa exclusivamente de HTML-in-Canvas.

### 6.2 Feature detection mínimo

```js
function supportsHtmlInCanvas2D(canvas) {
  const ctx = canvas.getContext("2d");

  return Boolean(
    ctx &&
    "drawElementImage" in CanvasRenderingContext2D.prototype &&
    "onpaint" in canvas
  );
}
```

### 6.3 Estratégias de fallback

#### Opção A — DOM overlay

Mais segura para interfaces interativas.

Renderiza o canvas normalmente e posiciona uma camada HTML sobreposta.

```html
<div class="stage-shell">
  <canvas id="stage"></canvas>
  <div id="fallback-ui">
    <!-- mesma UI semântica -->
  </div>
</div>
```

```css
.stage-shell {
  position: relative;
}

#stage,
#fallback-ui {
  position: absolute;
  inset: 0;
}

#fallback-ui {
  pointer-events: auto;
}
```

Use quando:

- existem inputs, botões e formulários;
- acessibilidade é prioridade;
- a UI precisa funcionar em qualquer navegador.

#### Opção B — Canvas-only simplificado

Mais frágil. Só usar para conteúdo não interativo ou decorativo.

Use quando:

- labels simples;
- gráficos estáticos;
- nenhum input/foco/copy-paste é necessário.

#### Opção C — Desabilitar feature experimental

Use quando a experiência depende fortemente da API.

Exibir aviso claro:

```txt
Esta visualização experimental requer navegador Chromium com HTML-in-Canvas habilitado.
```

---

## 8. Resize, escala e anti-blur

### 7.1 Problema

Canvas possui dois tamanhos:

- tamanho CSS;
- tamanho interno em pixels.

Se o canvas interno não acompanha `devicePixelRatio`, a UI renderizada pode ficar borrada.

### 7.2 Solução recomendada

Preferir `ResizeObserver` com `device-pixel-content-box` quando disponível.

```js
const observer = new ResizeObserver(([entry]) => {
  const box = entry.devicePixelContentBoxSize?.[0];

  if (box) {
    canvas.width = box.inlineSize;
    canvas.height = box.blockSize;
  } else {
    const rect = canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    canvas.width = Math.round(rect.width * dpr);
    canvas.height = Math.round(rect.height * dpr);
  }

  canvas.requestPaint?.();
});

observer.observe(canvas, { box: "device-pixel-content-box" });
```

---

## 9. Ciclo de paint

### 8.1 Regra

O evento `paint` deve ser tratado como o ponto principal para redesenhar HTML no canvas.

```js
canvas.addEventListener("paint", (event) => {
  for (const element of event.changedElements ?? []) {
    // opcional: otimizar redraw parcial
  }

  render();
});
```

### 8.2 Quando chamar `requestPaint`

Use `canvas.requestPaint?.()` quando:

- a posição do elemento muda por lógica externa;
- a câmera 3D muda;
- uma animação do canvas precisa redesenhar HTML;
- houve resize;
- o canvas atualiza todo frame.

### 8.3 Cuidado

Mudanças de DOM feitas dentro do handler de `paint` podem só aparecer no frame seguinte. Não criar loops de mutação sem controle.

---

## 10. Segurança e privacidade

A API possui restrições para evitar vazamento de dados sensíveis via leitura de pixels, timing ou invalidação.

O agente deve evitar depender de pintura de:

- conteúdo cross-origin;
- imagens externas sem CORS adequado;
- iframes cross-origin;
- SVGs com referências externas;
- canvases contaminados por dados cross-origin;
- estados de links visitados;
- marcações de spellcheck/grammar;
- informações pendentes de autofill;
- temas/sistema quando tratados como dados sensíveis.

Regra prática:

> Não use HTML-in-Canvas para capturar, exportar ou inspecionar visualmente conteúdo que o JavaScript normal não deveria conseguir observar.

---

## 11. Acessibilidade

HTML-in-Canvas não elimina a responsabilidade de acessibilidade.

Checklist mínimo:

- usar HTML semântico;
- associar `label` e `input`;
- preservar ordem lógica de foco;
- testar navegação por teclado;
- testar leitor de tela quando possível;
- aplicar transformação sincronizada após desenho;
- não deixar elementos invisíveis interceptando eventos fora da área desenhada;
- remover elementos obsoletos do DOM;
- não duplicar controles interativos entre canvas e fallback ativo ao mesmo tempo.

---

## 12. Gerenciamento de elementos e memória

### 11.1 Remoção correta

Se um elemento não será mais usado:

```js
element.remove();
canvas.requestPaint?.();
```

Não basta parar de desenhar o elemento.

Motivo: ele pode continuar participando de acessibilidade, foco e hit testing.

### 11.2 Pools

Para muitas UIs dinâmicas:

- reutilizar nós DOM quando possível;
- remover nós não utilizados;
- evitar criar/remover centenas de elementos por frame;
- separar estado de renderização de estado DOM;
- usar virtualização quando houver listas grandes.

---

## 13. Integração 2D

### 12.1 Caso recomendado

Use 2D quando:

- precisa de labels ricos em gráficos;
- quer painéis, tooltips ou controles em canvas;
- precisa de texto internacionalizado;
- quer inputs reais sobre composição canvas;
- quer manter acessibilidade.

### 12.2 Skeleton 2D recomendado

```js
export function mountHtmlInCanvas2D({ canvas, uiRoot, draw }) {
  const ctx = canvas.getContext("2d");

  if (!ctx || !("drawElementImage" in CanvasRenderingContext2D.prototype)) {
    return { supported: false };
  }

  function resize() {
    const rect = canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    canvas.width = Math.max(1, Math.round(rect.width * dpr));
    canvas.height = Math.max(1, Math.round(rect.height * dpr));

    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }

  function render() {
    resize();

    if (ctx.reset) {
      ctx.reset();
      ctx.setTransform(window.devicePixelRatio || 1, 0, 0, window.devicePixelRatio || 1, 0, 0);
    } else {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    }

    draw?.(ctx);

    const transform = ctx.drawElementImage(uiRoot, 24, 24);
    uiRoot.style.transformOrigin = "0 0";
    uiRoot.style.transform = transform.toString();
  }

  canvas.addEventListener("paint", render);

  const ro = new ResizeObserver(() => {
    resize();
    canvas.requestPaint?.();
  });

  ro.observe(canvas);

  resize();
  canvas.requestPaint?.();

  return {
    supported: true,
    destroy() {
      ro.disconnect();
      canvas.removeEventListener("paint", render);
    }
  };
}
```

---

## 14. Integração WebGL/WebGPU

### 13.1 WebGL

Conceito:

```js
gl.texElementImage2D(/* target, level, internalformat, format, type, element */);
```

Uso:

- renderizar UI HTML como textura;
- aplicar a textura em plano, cubo, mesh ou superfície 3D;
- sincronizar posição DOM com a projeção/câmera para manter interação alinhada.

Risco:

- assinatura pode mudar;
- helper de transformação pode variar;
- suporte em biblioteca 3D pode ser experimental;
- câmera/matriz/projeção exigem sincronização rigorosa.

### 13.2 WebGPU

Conceito:

```js
device.queue.copyElementImageToTexture(source, destination, size);
```

Uso:

- copiar o elemento/snapshot para uma textura WebGPU;
- renderizar essa textura no pipeline 3D/2D;
- sincronizar transformação DOM separadamente.

Risco:

- assinatura e suporte ainda experimentais;
- necessidade de validar comportamento no runtime alvo;
- cuidado com snapshots, workers e `ElementImage`.

### 13.3 Regra para agentes

Quando gerar código WebGL/WebGPU:

- preferir wrappers pequenos e isolados;
- não misturar lógica da aplicação com chamadas experimentais;
- documentar assinatura assumida;
- criar fallback;
- escrever teste manual de alinhamento entre clique e pixels;
- não afirmar compatibilidade com Three.js/PlayCanvas sem validação da versão.

---

## 15. Integração com frameworks

### 14.1 React

Padrão recomendado:

- renderizar a UI dentro do canvas como DOM real;
- usar `ref` para canvas e elemento;
- inicializar HTML-in-Canvas em `useEffect`;
- limpar listeners/observers no cleanup;
- manter fallback como branch separado.

Restrição obrigatória para agentes:

- isolar o ciclo de pintura do canvas (`paint`, `drawElementImage`, `requestPaint`) do estado declarativo do framework (`useState`, stores reativas, signals);
- não acionar `requestPaint` diretamente a partir de mudanças de estado que também são causadas pelo próprio ciclo de pintura;
- manter dados mutáveis de renderização em `useRef` ou módulo isolado, não em `useState`;
- evitar loops `state -> render React -> requestPaint -> paint -> setState -> render React`;
- usar estado React apenas para modo de suporte, fallback, dados de UI e controles do usuário;
- se for necessário reagir a mudanças de layout/conteúdo, usar throttling/debouncing ou fila explícita de renderização.

Exemplo conceitual:

```jsx
function HtmlCanvasDemo() {
  const canvasRef = useRef(null);
  const uiRef = useRef(null);
  const [mode, setMode] = useState("native");

  useEffect(() => {
    const canvas = canvasRef.current;
    const uiRoot = uiRef.current;

    if (!canvas || !uiRoot) return;

    const ctx = canvas.getContext("2d");

    if (!ctx || !("drawElementImage" in CanvasRenderingContext2D.prototype)) {
      setMode("fallback");
      return;
    }

    function render() {
      ctx.reset?.();
      const transform = ctx.drawElementImage(uiRoot, 32, 32);
      uiRoot.style.transformOrigin = "0 0";
      uiRoot.style.transform = transform.toString();
    }

    canvas.addEventListener("paint", render);
    canvas.requestPaint?.();

    return () => {
      canvas.removeEventListener("paint", render);
    };
  }, []);

  if (mode === "fallback") {
    return <FallbackDomUI />;
  }

  return (
    <canvas ref={canvasRef} layoutsubtree="true">
      <div ref={uiRef}>
        <button type="button">Ação</button>
      </div>
    </canvas>
  );
}
```

Observação: em JSX, atributos booleanos não padronizados podem exigir string ou propagação explícita. Validar o HTML final renderizado.

### 14.2 Three.js

Não assumir API oficial estável.

Estratégia segura:

- criar uma camada `HtmlCanvasTextureAdapter`;
- validar se a extensão/biblioteca usada oferece suporte;
- caso contrário, usar DOM overlay ou textura tradicional;
- isolar chamadas experimentais em um arquivo único.

### 14.3 PlayCanvas

Mesma regra:

- tratar suporte como experimental;
- validar APIs reais do engine;
- encapsular upload/cópia de elemento HTML;
- manter fallback.

---

## 16. Padrão de arquitetura para projeto com agentes

### 15.1 Organização sugerida

```txt
src/
  html-in-canvas/
    support.ts
    resizeCanvas.ts
    mount2D.ts
    fallbackOverlay.ts
    syncTransform.ts
    types.ts
  components/
    CanvasStage.tsx
    CanvasFallbackUI.tsx
  tests/
    htmlInCanvas.support.test.ts
    htmlInCanvas.fallback.test.ts
```

### 15.2 Responsabilidades

`support.ts`

- feature detection;
- identificação do modo disponível: `native-2d`, `webgl`, `webgpu`, `fallback`.

`resizeCanvas.ts`

- sincronização CSS pixels/device pixels;
- ResizeObserver;
- requestPaint após resize.

`mount2D.ts`

- listeners de paint;
- chamada `drawElementImage`;
- aplicação do transform;
- cleanup.

`fallbackOverlay.ts`

- DOM overlay;
- ativação/desativação sem duplicar foco.

`syncTransform.ts`

- funções utilitárias para aplicar transform;
- comentários sobre origem, escala e CTM.

---

## 17. Critérios de aceite

### Contrato de penalização para agentes

Falhas que invalidam a entrega:

| Falha | Severidade | Ação esperada |
|---|---:|---|
| `layout subtree` em vez de `layoutsubtree` | crítica | rejeitar/patch imediato |
| chamada a `drawElementImage` sem feature detection | crítica | adicionar guarda e fallback |
| fallback ausente ou apenas visual | crítica | implementar fallback DOM funcional |
| API inventada sem adapter | crítica | remover ou isolar atrás de adapter tipado |
| falta de sincronização `DOMMatrix -> CSS transform` | crítica | aplicar transform e `transform-origin` |
| ausência de cleanup | alta | retornar função de cleanup |
| `devicePixelRatio` ignorado | alta | adicionar `ResizeObserver` e escala correta |
| duplicação de elementos focáveis no fallback | alta | garantir um único modo interativo ativo |
| loop React causado por `paint -> setState -> requestPaint` | alta | mover estado de renderização para refs/adapters |
 para agentes

Uma entrega só é aceitável se cumprir todos os pontos abaixo.

### 16.1 API

- [ ] Usa `layoutsubtree`, não `layout subtree`.
- [ ] Usa `drawElementImage` apenas após suporte detectado.
- [ ] Usa evento `paint` ou `requestPaint` quando adequado.
- [ ] Não inventa helpers não confirmados.
- [ ] Isola chamadas experimentais.

### 16.2 Fallback

- [ ] Existe fallback funcional.
- [ ] Fallback preserva a interação principal.
- [ ] Não há dois controles interativos ativos simultaneamente causando foco duplicado.
- [ ] Mensagem de limitação aparece quando fallback não é possível.

### 16.3 Interação

- [ ] Clique ocorre onde o elemento aparece visualmente.
- [ ] Foco via teclado funciona.
- [ ] Inputs aceitam digitação.
- [ ] Botões disparam eventos.
- [ ] Seleção de texto funciona quando aplicável.

### 16.4 Acessibilidade

- [ ] HTML semântico.
- [ ] Labels associados.
- [ ] Ordem de foco coerente.
- [ ] Elementos removidos não continuam acessíveis.
- [ ] Fallback acessível.

### 16.5 Visual

- [ ] Canvas não fica borrado em telas HiDPI.
- [ ] Resize mantém proporção correta.
- [ ] UI desenhada e DOM interativo permanecem alinhados.
- [ ] Não há elementos fantasmas interceptando eventos.

### 16.6 Manutenção

- [ ] Cleanup de listeners.
- [ ] Cleanup de ResizeObserver.
- [ ] Remoção de elementos obsoletos.
- [ ] Comentários indicam o caráter experimental da API.
- [ ] Código permite troca por fallback sem reescrever a aplicação.

---

## 18. Anti-padrões

Evitar:

```html
<canvas layout subtree>
```

Evitar:

```js
uiElement.style.transform = transform.toCSS();
```

Evitar:

```js
document.body.appendChild(uiElement);
ctx.drawElementImage(uiElement, 0, 0);
```

Motivo: o elemento precisa estar no contexto correto do canvas, conforme restrições da proposta.

Evitar:

```js
uiRoot.style.display = "none";
ctx.drawElementImage(uiRoot, 0, 0);
```

Motivo: elemento sem boxes gerados não deve ser desenhado.

Evitar:

```js
if (!supported) {
  alert("Atualize seu navegador");
}
```

Motivo: precisa haver fallback funcional quando possível.

Evitar:

```js
// Criar 500 inputs novos por frame
```

Motivo: custo alto de layout, acessibilidade e garbage collection.

---

## 19. Implementation prompt for Codex/agent

Use this prompt when requesting implementation:

```txt
You are developing a POC using the experimental HTML-in-Canvas API. 
Adhere strictly to the technical constraints outlined in this document.

Constraints:
1. Use the `layoutsubtree` attribute (no spaces).
2. Perform explicit feature detection before invoking `drawElementImage`.
3. Render actual HTML within the `<canvas>` tags.
4. Use the `paint` event listener to draw elements.
5. Apply the `DOMMatrix` transform returned by `drawElementImage` to the DOM element to synchronize hit testing, focus, and accessibility.
6. Implement `ResizeObserver` handling `devicePixelRatio` to prevent blurring.
7. Provide a functional DOM overlay fallback. Do not invent non-documented WebGL/WebGPU/Three.js APIs; if using experimental integrations, isolate them in adapters.
8. Include cleanup for all observers and listeners.
9. Output manual testing criteria covering clicks, focus, resizing, accessibility, and fallback behavior.

Output verified, isolated, and highly cohesive code.
```

---

## 20. Review prompt for agent

Use this prompt when requesting architectural review:

```txt
Conduct a strict architectural and security review of the provided HTML-in-Canvas implementation. 

Flag the following critical failures:
- Incorrect attribute usage (e.g., `layout subtree` instead of `layoutsubtree`).
- Missing feature detection before API calls.
- Lack of a functional DOM overlay fallback.
- Usage of fabricated or unverified APIs (e.g., `transform.toCSS()`).
- Missing CSS transform synchronization.
- Blurry canvas rendering due to unhandled `devicePixelRatio`.
- Invisible ghost elements intercepting pointer events.
- Memory leaks (missing listener/observer cleanups).
- Broken accessibility structures or duplicate focusable elements in fallbacks.

Output format:
1. Critical Issues
2. Moderate Issues
3. Suggested Optimizations
4. Refactored Code Patch
```

---

## 21. Manual testing prompt

Use this prompt when requesting manual test coverage:

```txt
Create a manual testing checklist for this HTML-in-Canvas POC covering:

- supported browser path;
- unsupported browser path;
- mouse/pointer click alignment;
- keyboard focus order;
- input typing;
- text selection;
- native Ctrl/Cmd+F page search;
- browser zoom;
- window resize;
- HiDPI display behavior;
- screen reader accessibility;
- dynamic element removal;
- DOM overlay fallback behavior;
- absence of duplicate focusable elements between native and fallback modes;
- cleanup after component unmount or route change.
```

---

## 22. Exemplo de fallback completo simplificado

```html
<div class="stage-shell" data-mode="auto">
  <canvas id="stage" layoutsubtree>
    <form id="native-ui">
      <label for="email">Email</label>
      <input id="email" type="email" />
      <button type="submit">Enviar</button>
    </form>
  </canvas>

  <form id="fallback-ui" hidden>
    <label for="fallback-email">Email</label>
    <input id="fallback-email" type="email" />
    <button type="submit">Enviar</button>
  </form>
</div>
```

```js
const canvas = document.querySelector("#stage");
const nativeUi = document.querySelector("#native-ui");
const fallbackUi = document.querySelector("#fallback-ui");
const ctx = canvas.getContext("2d");

const supported =
  ctx &&
  "drawElementImage" in CanvasRenderingContext2D.prototype &&
  "onpaint" in canvas;

if (!supported) {
  fallbackUi.hidden = false;
  canvas.hidden = true;
} else {
  fallbackUi.hidden = true;
  canvas.hidden = false;

  canvas.addEventListener("paint", () => {
    ctx.reset?.();
    const transform = ctx.drawElementImage(nativeUi, 40, 40);
    nativeUi.style.transformOrigin = "0 0";
    nativeUi.style.transform = transform.toString();
  });

  canvas.requestPaint?.();
}
```

---

## 23. Fontes técnicas para validação

Consultar antes de consolidar qualquer implementação de produção:

- Chrome Developers — Introducing the HTML-in-Canvas API origin trial  
  https://developer.chrome.com/blog/html-in-canvas-origin-trial

- WICG — HTML-in-Canvas explainer  
  https://github.com/WICG/html-in-canvas

- Chrome Platform Status — HTML-in-canvas  
  https://chromestatus.com/feature/5172548013916160

- Blink Dev — Intent/Developer Trial threads  
  https://groups.google.com/a/chromium.org/g/blink-dev

