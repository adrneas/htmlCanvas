# Contexto Tecnológico: HTML-in-Canvas API (Maio 2026)

## 1. Visão Geral e Arquitetura
A **HTML-in-Canvas API** é um padrão aberto (atualmente em Origin Trial no Google Chrome a partir de maio de 2026) que resolve a quebra de acessibilidade e interatividade ao renderizar interfaces dentro de contextos gráficos bidimensionais ou tridimensionais (`canvas`, `WebGL`, `WebGPU`). 

Anteriormente, renderizar UI dentro de um canvas transformava o conteúdo em uma grade estática de pixels. Esta API atua como uma ponte bidirecional: ela permite pintar elementos do DOM diretamente no canvas/texturas mantendo o ciclo de vida do DOM ativo. O navegador continua processando testes de colisão (hit testing), acessibilidade, seleção de texto, tradução nativa e ferramentas de busca (`Ctrl+F`).

### Requisitos Estruturais Básicos
1. **Atributo `layout subtree`**: Deve ser declarado explicitamente na tag `<canvas>`. Ele instrui o motor do navegador a processar a subárvore do DOM aninhada para fins de renderização e acessibilidade.
2. **HTML Aninhado**: Os elementos de interface (botões, inputs, forms, SVGs) devem ser declarados diretamente dentro das tags `<canvas></canvas>`.
3. **Sincronização de Transformação CSS**: O desenvolvedor deve obrigatoriamente capturar a matriz de transformação gerada pelo método de renderização da API e aplicá-la de volta ao estilo CSS do elemento DOM correspondente. Isso garante que a camada invisível de interação do DOM coincida exatamente com os pixels desenhados no canvas.

---

## 2. Fluxo de Implementação e Código Nativo

### 2.1 Renderização em Contexto 2D
Para contextos `2d`, utiliza-se o método `drawElementImage` dentro do evento de repintura do elemento.

```javascript
// 1. Configuração do Canvas e Captura do Elemento
const canvas = document.querySelector('canvas');
const ctx = canvas.getContext('2d');
const uiElement = document.getElementById('my-ui-component');

// 2. Loop de Renderização / Evento de Paint
uiElement.addEventListener('paint', (event) => {
  // Limpa o canvas antes de redesenhar
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Desenha o elemento DOM nas coordenadas X, Y desejadas
  const transform = ctx.drawElementImage(uiElement, 10, 10);

  // CRÍTICO: Atualiza o transform do DOM para manter a interatividade alinhada
  uiElement.style.transform = transform.toCSS(); 
});

### 2.2 Renderização em Contexto WebGL (Texturas)
Em ambientes 3D com WebGL, mapeia-se o elemento DOM diretamente como uma textura utilizando texElementImage2D.

```javascript
// Equivalente ao texImage2D tradicional, mas aceita um elemento DOM
function updateWebGLTexture() {
  gl.bindTexture(gl.TEXTURE_2D, texture);
  
  // Passa o elemento DOM diretamente como fonte da textura
  gl.texElementImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, uiElement);
  
  // Projeta a coordenada 3D de volta para o espaço CSS para atualizar o DOM
  const transform = gl.getElementTransform(uiElement);
  uiElement.style.transform = transform;
}```

###2.3 Renderização em Contexto WebGPU
Em WebGPU, utiliza-se o método de cópia de imagem externa adaptado para elementos do DOM: copyElementImageToTexture.

```javascript
function updateWebGPUTexture() {
  device.queue.copyElementImageToTexture(
    { element: uiElement },
    { texture: webgpuTexture },
    [uiElement.offsetWidth, uiElement.offsetHeight]
  );
  
  // Atualização obrigatória do CSS transform pós-cópia
  const transform = canvas.getElementTransform(uiElement);
  uiElement.style.transform = transform;
}

## 3. Integração com Frameworks 3D (Camada de Abstração)

### 3.1 Three.js
O Three.js implementou suporte experimental nativo via 3.htmlTexture.

```javascript
import * as THREE from 'three';

// Captura o elemento DOM interno ao canvas
const element = document.getElementById('ui-card');

// Cria a textura vinculada ao HTML
const htmlTexture = new THREE.HTMLTexture(element);

// Aplica a textura ao material do objeto 3D
const material = new THREE.MeshBasicMaterial({ map: htmlTexture });
const geometry = new THREE.BoxGeometry(2, 2, 2);
const mesh = new THREE.Mesh(geometry, material);

scene.add(mesh);

### 3.2 PlayCanvas
O PlayCanvas utiliza listeners de evento para atualizar a propriedade de mapa difuso (diffuseMap).

// Criação da textura baseada em elemento HTML
const htmlElement = document.getElementById('ui-container');
const texture = new pc.Texture(device);

htmlElement.addEventListener('paint', () => {
    // Sincroniza os pixels do DOM com a GPU
    texture.uploadElement(htmlElement);
});

const material = new pc.StandardMaterial();
material.diffuseMap = texture;
material.update();

## 4. Diretrizes Críticas para Geração de Código por IA
Ao instruir o Codex ou agentes autônomos a desenvolver componentes baseados nesta tecnologia, certifique-se de que as seguintes regras sejam estritamente validadas na saída do código:

1. Estrutura HTML Obrigatória:
<canvas layout subtree width="800" height="600">
    <div id="ui-component">
        <input type="text" placeholder="Digite aqui..." />
        <button>Enviar</button>
    </div>
</canvas>

2. Gerenciamento de Escala (Anti-Blur): Sempre implemente um ResizeObserver no canvas para ajustar os fatores de escala de pixels (window.devicePixelRatio) antes de invocar as rotinas de desenho, evitando artefatos borrados na UI renderizada.

3. Limpeza de Memória (Garbage Collection): Se um elemento HTML for removido da lógica de renderização do canvas, ele deve ser explicitamente removido da subárvore do elemento <canvas> no DOM. Caso contrário, o motor do navegador continuará expondo o elemento invisível para a árvore de acessibilidade e interceptando eventos fantasmas de mouse/teclado.

4. Fallback Obrigatório: Como a API está em estágio de Origin Trial (maio de 2026), o código gerado deve conter uma validação de feature detection ('drawElementImage' in CanvasRenderingContext2D.prototype) e um fallback funcional baseado em renderização DOM tradicional sobreposta, caso a API não esteja ativa no navegador cliente.