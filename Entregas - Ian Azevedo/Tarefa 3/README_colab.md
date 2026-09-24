# Linha de impedimento no Google Colab

Este notebook analisa lances de impedimento em vídeos de futebol. Você escolhe o frame do toque na bola, marca alguns pontos do campo e os dois jogadores envolvidos, e o notebook traça a linha de impedimento com a perspectiva correta da câmera e diz se o atacante estava impedido, com a margem em metros e a precisão da medição.

O notebook `offside_colab.ipynb` faz tudo no Google Colab, sem instalar nada no computador:

1. prepara o detector de jogadores (YOLO11 exportado para ONNX);
2. treina, em PyTorch e na GPU grátis do Colab, um YOLO que detecta a bola;
3. abre a interface de análise dentro de uma célula, com a detecção de jogadores e da bola rodando no Colab.

## Requisitos

- Uma conta Google. O Drive guarda os modelos e os treinos entre as sessões.
- O arquivo `offside-detector.zip` com o projeto. O notebook pede o upload na primeira vez.
- Uma conta grátis no [Roboflow](https://roboflow.com), para baixar o dataset da bola.
- O ambiente de execução com GPU: **Ambiente de execução → Alterar o tipo de ambiente de execução → GPU T4**.

### Chave do Roboflow

1. No Roboflow, copie a API key em *Settings → API Keys*.
2. No Colab, abra 🔑 **Secrets** na barra lateral esquerda e crie um segredo chamado `ROBOFLOW_API_KEY` com a chave.
3. Ative o acesso do notebook a esse segredo.

## Como abrir

Em [colab.research.google.com](https://colab.research.google.com), vá em **Arquivo → Fazer upload de notebook** e escolha `offside_colab.ipynb`. O Colab guarda uma cópia na pasta *Colab Notebooks* do Drive, e ela aparece nos recentes das próximas vezes.

## Etapas

| Etapa | O que faz | Quando rodar |
|---|---|---|
| 1. Preparar o projeto | Monta o Drive e descompacta o projeto | Toda sessão |
| 2. Dependências | Instala ultralytics, roboflow, ONNX e ONNX Runtime com GPU | Toda sessão |
| 3. Modelo de jogadores | Exporta o YOLO11s (COCO) para ONNX com entrada de 1280 px | Uma vez (pula sozinha se já existir) |
| 4a. Dataset da bola | Baixa o dataset do Roboflow e confere a resolução das imagens | Só para treinar |
| 4b. Fatiar | Corta as imagens em tiles de 640 px, na mesma escala da inferência com SAHI | Só para treinar |
| 4c. Treinar | Fine-tuning do YOLO na classe "bola" | Só para treinar |
| 4d. Continuar treino | Retoma um treino interrompido do último checkpoint | Só se a sessão cair |
| 4e. Exportar e validar | Exporta o melhor modelo para ONNX e mede as métricas no arquivo exportado | Logo depois do 4c |
| 5. Análise | Abre a interface de análise na célula | Sempre que for analisar |
| 6. Rastrear a bola | Rastreia a bola num vídeo inteiro e gera um vídeo com ela marcada | Opcional |

**As células da etapa 4 não verificam se o modelo já existe.** Depois de treinar e exportar, não rode a etapa 4 de novo: a 4a baixaria o dataset outra vez e a 4c começaria um treino novo (o anterior não é apagado).

Rode a **4e na mesma sessão do treino**. A exportação só precisa do `best.pt`, que fica no Drive, mas a validação precisa do dataset, que é apagado quando a sessão termina.

### Tempo de treino

Numa T4, com o dataset padrão (cerca de 2.300 tiles de treino), cada época leva por volta de 45 segundos, e as 60 épocas somam uns 45 minutos. O treino para antes se o modelo passar 25 épocas sem melhorar. A barra de progresso mostra só o tempo de cada época; multiplique pelas épocas restantes para estimar o total.

## O que fica salvo

Tudo o que precisa sobreviver entre sessões fica em `MyDrive/offside-detector-colab/`:

```
offside-detector-colab/
├── offside-detector.zip      o projeto (enviado na primeira vez)
├── models/
│   ├── players.onnx          detector de jogadores (etapa 3)
│   └── ball.onnx             detector de bola (etapa 4e)
├── runs/ball*/weights/       best.pt e last.pt de cada treino, com gráficos e métricas
├── outputs/                  imagens salvas pela interface e vídeos da etapa 6
└── videos/                   vídeos enviados na etapa 6 e os .ball.json
```

Isso vale em qualquer dispositivo em que você entrar com a mesma conta Google. O que **não** persiste: os pacotes instalados, o projeto descompactado em `/content` e o dataset. Por isso as etapas 1 e 2 rodam em toda sessão.

## Usando a interface (etapa 5)

Abra o vídeo ou a imagem do toque pelo botão da própria interface; o arquivo vem do seu computador.

1. **Frame do toque:** ande frame a frame (← →, Shift para 10) e clique em *Usar este frame*.
2. **Pontos do campo:** clique num ponto do campinho e depois no mesmo ponto da imagem. Use 6 ou mais, bem espalhados. O seletor acima do campinho mostra só uma metade, com o dobro da escala.
3. **Jogadores e bola:** *Detectar jogadores* e *Detectar a bola* rodam o YOLO no Colab. Com Defensor ou Atacante ativo, o 1º clique escolhe o jogador (na caixa dele, ou nos pés se ele não foi detectado) e o 2º marca a parte do corpo mais próxima da linha de fundo.
4. **Resultado:** o veredito aparece no pé da imagem. *Salvar imagem* grava o PNG em `outputs/` no Drive.

| Atalho | Ação |
|---|---|
| N | Mover a imagem |
| C | Marcar pontos do campo |
| D / A | Defensor / atacante (de novo: escolher outro jogador) |
| B | Bola |
| R | Descartar pessoa detectada (árbitro, reservas) |
| Tab | Alternar entre caixas sobrepostas |
| Shift+clique | Corrigir os pés do jogador ou marcar o chão sob a bola no ar |
| T | Voltar ao frame do toque |
| F | Enquadrar a imagem |

**Clique na imagem antes de usar os atalhos.** Assim eles vão para a interface, e não para o notebook, onde "D D" apaga uma célula.

## Problemas comuns

**Erro vermelho do pip sobre `protobuf` na etapa 2.** O conflito é com pacotes que já vêm no Colab (`google-ai-generativelanguage`, `grpcio-status`, `ydf`), que o projeto não usa. A instalação é concluída e pode ser ignorado.

**"Não achei o segredo ROBOFLOW_API_KEY".** Crie o segredo em 🔑 Secrets e ative o acesso do notebook a ele.

**"As imagens já vêm reduzidas para 640 px" na etapa 4a.** A versão escolhida do dataset tem pré-processamento de redimensionamento. Troque o número da versão por uma sem "Resize".

**A sessão caiu no meio do treino.** Rode as etapas 1, 2, 4a e 4b e depois a 4d, que continua do último checkpoint salvo no Drive.

**"Não achei models/ball.onnx" na interface.** Falta rodar a etapa 4e.

**O vídeo não abre na interface.** O navegador não suporta o codec. MP4 com H.264 e WebM são os mais compatíveis; outra saída é salvar o frame do toque como imagem e abri-la.

**A interface parou de responder aos botões de detecção.** A ligação com o Python vale só enquanto a sessão estiver ativa. Se a sessão reiniciou, rode as etapas 1, 2 e 5 de novo.

## Limitações

- O resultado depende da qualidade da calibração: confira sempre se as linhas calculadas coincidem com as do gramado.
- A estimativa da câmera supõe o frame inteiro da transmissão. Não use recortes nem vídeos com zoom digital.
- Com uma câmera só, projetar ombro ou cabeça no gramado supõe que eles estão na mesma posição lateral dos pés. A incerteza mostrada já inclui esse efeito, e ela diminui com câmeras alinhadas com o lance.
- A 25 ou 30 fps, um jogador em arrancada anda até uns 30 cm entre dois frames. Diferenças menores que isso não são decidíveis com vídeo de transmissão comum.
