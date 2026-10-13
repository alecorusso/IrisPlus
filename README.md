# 🌀 Iris+
 
Leitura na tela para quem tem **nistagmo** — o movimento involuntário e repetitivo dos olhos que faz o texto "fugir" enquanto a pessoa tenta ler.
 
O Iris+ rastreia as íris pela webcam e envia a posição dos olhos para uma extensão de navegador, que desloca o conteúdo da página na mesma direção do movimento ocular. A ideia é que o texto acompanhe o olho, em vez de o olho ter de perseguir o texto.
 
> Projeto desenvolvido em um hackathon. É um protótipo: funciona como prova de conceito, não como ferramenta clínica.
 
---
 
## Como funciona
 
```
 Webcam ──► Python (MediaPipe Face Mesh) ──► HTTP localhost:8765/data ──► Extensão Chrome ──► página desloca o texto
             • landmarks das íris                (JSON, último frame)        (polling a cada 50 ms)
             • distância rosto–câmera
```
 
1. **Rastreamento** — o [MediaPipe Face Mesh](https://developers.google.com/mediapipe) com `refine_landmarks=True` devolve 478 pontos do rosto, incluindo 4 pontos do contorno de cada íris. O centro de cada íris é o centróide desses pontos.
2. **Distância até a câmera** — mede, em pixels, a distância entre os cantos externos dos olhos (landmarks 33 e 263) e converte para centímetros com um modelo calibrado:
   $$d_{cm} = \frac{a}{p - b}$$
   onde $p$ é a distância entre os olhos em pixels. Os parâmetros $a$ e $b$ saem de uma calibração em dois pontos (o usuário se posiciona a 50 cm e a 30 cm, guiado por ovais na tela).
3. **Servidor local** — um servidor `aiohttp` roda em uma thread separada e expõe sempre o último frame processado em `GET http://localhost:8765/data`.
4. **Compensação no navegador** — a extensão consulta o servidor ~20 vezes por segundo, calcula quanto o ponto médio entre as duas íris se moveu desde a última leitura e aplica um `transform: translate(...)` nos elementos de texto da página, ajustado por um controle de sensibilidade.
---
 
## Estrutura do repositório
 
| Arquivo | O que faz |
|---|---|
| `iris_original.py` | Calibração de distância (dois pontos, 50 cm e 30 cm) + medição em tempo real pela webcam. Grava `calibracao_dados.txt` e `distancia_face.csv`. |
| `iris_demo.py` | **Script principal.** Rastreia as íris pela webcam e serve os dados em `http://localhost:8765/data` para a extensão. Ao sair, salva `eye_tracking_loop_<data>.csv`. |
| `iris_video.py` | Mesmo rastreamento, mas a partir de um arquivo de vídeo (`VIDEO_PATH`), útil para testar com gravações de pessoas com nistagmo. Salva `eye_tracking_completo_<data>.csv`. |
| `server.py` | Versão anterior do servidor, em WebSocket. Não é usada pelo fluxo atual (substituída pelo servidor HTTP dentro de `iris_demo.py`). |
| `IrisPlus-extension/` | Extensão Chrome (Manifest V3). `irisplus-http.js` é o painel de compensação conectado ao servidor. |
| `IrisPlus-extension/irisplus-widget.js` | Painel alternativo que **simula** um tremor assimétrico na página (amplitude, frequência de cada lado, direção), para entender o problema na prática. Não está registrado no `manifest.json`. |
| `nistagmo_reader.html` | Leitor independente (abre direto no navegador) que simula o movimento do texto e oferece ajustes de acessibilidade: frequência, amplitude, ângulo, tipo de movimento, ruído, suavização, tamanho de fonte, entrelinha, temas de contraste e modo "estabilizar". |
| `calibracao_dados.txt` | Exemplo de saída da calibração. |
| `distancia_face.csv` | Exemplo de saída da medição de distância. |
 
---
 
## Como rodar
 
### Requisitos
 
- Python 3.9 – 3.12 (a versão do MediaPipe fixada não suporta versões mais novas)
- Webcam
- Google Chrome (ou outro navegador baseado em Chromium)
### 1. Instalar dependências
 
```bash
python -m venv venv
# Windows
venv\Scripts\activate
# Linux / macOS
source venv/bin/activate
 
pip install -r requirements.txt
pip install aiohttp aiohttp-cors
```
 
> `aiohttp` e `aiohttp-cors` são usados pelo `iris_demo.py`, mas ainda não estão no `requirements.txt`.
 
### 2. Calibrar a distância
 
```bash
python iris_original.py
```
 
Siga as ovais na tela: posicione o rosto dentro de cada uma e pressione **ESPAÇO** quando o indicador de posicionamento estiver bom (primeiro a 50 cm, depois a 30 cm). Os parâmetros ficam em `calibracao_dados.txt`.
 
Copie os valores de `a` e `b` para o dicionário `CALIB_PARAMS` no topo de `iris_demo.py` (e de `iris_video.py`, se for usar).
 
### 3. Iniciar o rastreamento
 
```bash
python iris_demo.py
```
 
Uma janela mostra a câmera com o contorno e o centro das íris, a distância estimada e o status do servidor. Teste em outro terminal ou no navegador:
 
```
http://localhost:8765/data
```
 
Atalhos: **ESPAÇO** pausa, **ESC** ou **Q** sai e salva o CSV.
 
### 4. Carregar a extensão
 
1. Abra `chrome://extensions`
2. Ative o **Modo do desenvolvedor**
3. Clique em **Carregar sem compactação** e selecione a pasta `IrisPlus-extension/`
Em qualquer página aparece o painel **Iris+ HTTP** no canto inferior direito. Ele conecta sozinho ao servidor depois de 1 segundo. Escolha o eixo de compensação (horizontal, vertical ou ambos), ajuste a sensibilidade e clique em **Ativar Compensação**.
 
### Testar sem webcam
 
Abra `nistagmo_reader.html` direto no navegador para experimentar os ajustes de leitura e a simulação de movimento, sem precisar do Python.
 
---
 
## Formato dos dados
 
Resposta de `GET /data`:
 
```json
{
  "timestamp": 1730518400.12,
  "left_iris":  { "x": 312, "y": 241 },
  "right_iris": { "x": 398, "y": 239 },
  "frame_number": 1523,
  "eye_distance_px": 140.2,
  "distance_cm": 29.9,
  "loop_count": 0
}
```
 
Antes do primeiro rosto detectado, a resposta é `{"status": "no_data"}`.
 
Colunas do CSV salvo ao sair: `frame, distance_cm, eye_distance_px, left_iris_x, left_iris_y, right_iris_x, right_iris_y, loop_count`.
 
---
 
## Limitações conhecidas
 
- **Compensação simples:** usa a diferença entre duas leituras seguidas, sem filtro nem previsão. Entre a câmera, o processamento e o polling de 50 ms, a correção sempre chega com algum atraso em relação ao olho.
- **Coordenadas em pixels da câmera:** o deslocamento das íris não é convertido para ângulo do olhar nem para pixels da tela; o fator de escala é empírico (sensibilidade × 0,1).
- **Calibração de dois pontos:** o modelo de distância é exato só a 30 e 50 cm e assume a distância média entre os cantos dos olhos (`W_real = 6,5 cm`).
- **Submódulo vazio:** `MediaPipe-Iris/` aparece como submódulo, mas o repositório não tem `.gitmodules`, então a pasta vem vazia ao clonar. Ela não é necessária para rodar o projeto.
- O `iris_demo.py` ainda declara `VIDEO_PATH`, mas lê da webcam (`VideoCapture(0)`); o contador de progresso na tela não faz sentido nesse modo.
## Próximos passos possíveis
 
- Converter o movimento das íris em ângulo do olhar e mapear para pixels da tela
- Filtro (passa-banda ou Kalman) para separar o nistagmo do movimento voluntário de leitura
- Trocar o polling por WebSocket ou Server-Sent Events para reduzir a latência
- Ligar o `nistagmo_reader.html` aos dados reais de rastreamento, em vez da oscilação simulada
---
 
## Tecnologias
 
Python · OpenCV · MediaPipe Face Mesh · NumPy · aiohttp · JavaScript (Chrome Extension, Manifest V3) · HTML/CSS
 
## Autores
 
Alexander Oliveira — Engenharia Elétrica, Escola Politécnica da USP
Enzo Kendi Shinzato - Engenharia Elétrica, Escola Politécnica da USP
Giovanna Carvalho de Moraes - Estatística, Instituto de Matemática e Estatística da USP
Gustavo Esquetini - Física, Instituto de Física da USP
Leonardo - Física, Instituto de Física da USP
Marcos Vinicius Gomes - Engenharia Elétrica, Escola Politécnica da USP

