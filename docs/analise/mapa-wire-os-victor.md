# Mapa do wire-os-victor: voz, intents, behaviors e visão

Data: 2026-11-11. Base: commit `505eb84c`. Só leitura: nada foi compilado nem executado no robô.

**Marcação:** **FATO** = visto no código (com `arquivo:linha`). **HIPÓTESE** = dedução que precisa ser confirmada no robô.

## Comece por aqui (os 3 pontos mais fáceis)

| # | O que mudar | Precisa compilar? | Tempo |
|---|---|---|---|
| 1 | Sensibilidade da wake word: um arquivo de texto no robô | Não | 5 min |
| 2 | Intent nova que dispara uma animação: uma entrada em `simple_voice_responses` | Não (só copiar o JSON e reiniciar o engine) | 15 min |
| 3 | Behavior novo copiando o `WireTest` (animação + fala) | Sim | 1 a 2 h na primeira vez |

Detalhes de cada um estão nas seções 2.1, 3.3 e 4.4.

---

## 1. Visão geral da arquitetura

### 1.1 Processos

| Processo | Código | O que faz |
|---|---|---|
| `vic-robot` | `robot/` | Fala com o body board pela UART (spine): motores, sensores, IMU, áudio cru dos 4 microfones. Roda em prioridade de tempo real. |
| `vic-anim` | `animProcess/` | Animações, tela (rosto), som (Wwise), TTS, **microfones e wake word**. Webserver de depuração na porta 8889. |
| `vic-engine` | `engine/` | O cérebro: behaviors, intents, visão, mapa de navegação, planejador, emoções. Webserver de depuração na porta 8888. |
| `vic-cloud` | `cloud/` (Go) | Envia o áudio da voz para o servidor de voz (WirePod) e devolve a intent. Também hospeda o gateway gRPC do SDK (`cloud/cloud/gateway.go`, `message_handler.go`). |
| `vic-switchboard` | `platform/switchboard/` | Bluetooth e pareamento com o app. |
| Outros | `dasmgr/`, `platform/` | `vic-dasmgr` (telemetria), `vic-faultCodeDisplay` (tela de erro), `update-engine` (OTA), `vic-log-*`. |

### 1.2 Como se comunicam

**FATO:** todos usam sockets unix de datagrama em `/dev/socket/`, definidos em `coretech/messaging/shared/socketConstants.h:29-51`.

```
body board ──UART──> vic-robot ──_anim_robot_──> vic-anim ──_engine_anim_──> vic-engine
                                                    │                           ▲
                                               mic_sock (áudio)            ai_sock (intent)
                                                    ▼                           │
                                                 vic-cloud ──gRPC "chipper"──> WirePod
                                                    │
                                    _engine_gateway_proto_ (SDK) <──> vic-engine
```

- O formato das mensagens entre processos é CLAD (`clad/`, `robot/clad/`), gerado por `victor-clad/`.
- O SDK e o WirePod falam com o engine por protobuf (`tools/protobuf/gateway/public/*.proto`), passando pelo gateway dentro do `vic-cloud`.
- Quem inicia e reinicia os processos fica no repositório `wire-os` (units do systemd e `fault-code-handler`).

---

## 2. Pipeline de voz

### 2.1 Wake word ("Hey Vector")

**Onde (FATO):**
- Motor: Picovoice Porcupine 1.5.0 (`animProcess/src/cozmoAnim/speechRecognizer/speechRecognizerPicovoice.cpp:1`).
- Classe `SpeechRecognizerPicovoice`, funções `Init()` (`:83-205`) e `Update()` (`:207-235`).
- Orquestração: `SpeechRecognizerSystem::Update()` (`animProcess/src/cozmoAnim/speechRecognizer/speechRecognizerSystem.cpp:144-162`).

**Arquivos que o robô lê na inicialização do `vic-anim` (FATO, `speechRecognizerPicovoice.cpp:23-35`):**

| Arquivo | Para quê | Padrão |
|---|---|---|
| `/data/data/com.anki.victor/persistent/picovoice/sensitivity2` | Sensibilidade do "Hey Vector" (0.0 a 1.0) | 0.60 |
| `/data/data/com.anki.victor/persistent/picovoice/custom_1-5-0.ppn` | Modelo próprio da wake word (formato Porcupine 1.5.0) | `/anki/data/assets/cozmo_resources/assets/picovoice/hey_vector.ppn` |
| `/data/data/com.anki.victor/persistent/picovoice/sensitivity_alexa` | Sensibilidade do "Alexa" | 0.50 |

**Como ajustar a sensibilidade:**
1. `ssh root@<robô> 'mkdir -p /data/data/com.anki.victor/persistent/picovoice && echo 0.75 > /data/data/com.anki.victor/persistent/picovoice/sensitivity2'`
2. `ssh root@<robô> 'systemctl restart vic-anim'` (o arquivo só é lido no `Init`).
3. Confirme no log: `grep "SpeechRecognizerPicovoice.Init" /var/log/messages` deve mostrar `Using sensitivity: 0.7500`.

Valor maior = detecta mais vezes, com mais falsos positivos. Suba de 0.05 em 0.05.

**Como trocar o modelo:** gere um `.ppn` no formato do Porcupine **1.5.0** (outra versão não carrega) e copie para `custom_1-5-0.ppn`. Se ele falhar, o código volta para o modelo padrão (`:158-163`).

**Atenção, a wake word só roda quando o detector de voz (VAD) acha que há fala (FATO):**
- `SpeechRecognizerSystem::Update` só chama o Porcupine se `vadActive` (`speechRecognizerSystem.cpp:160`).
- O VAD usa limiar absoluto 250 (`micDataProcessor.cpp:159`) e fica ativo por mais 1000 ms depois da última fala (`kMicData_QuietTimeCooldown_ms`, `:61`).
- **HIPÓTESE:** voz baixa ou longe pode nunca abrir o VAD, e aí a wake word nem é avaliada.

**Defeito no carregamento (FATO):** com Alexa ligada, a lista de modelos começa pelo da Alexa (`:139-149`). Os dois fallbacks fazem `ppns.front() = default_ppn` (`:160-170`), então trocam o modelo **da Alexa** pelo "Hey Vector" em vez de trocar o modelo próprio quebrado. Só afeta quem usa Alexa e um `.ppn` próprio inválido.

### 2.2 Processamento dos 4 microfones

**Onde (FATO):** `animProcess/src/cozmoAnim/micData/micDataProcessor.cpp`, classe `MicDataProcessor`.

**Fluxo:**
1. O `vic-robot` entrega blocos de 10 ms: 4 canais, 160 amostras cada, 16 kHz (`lib/micData/micDataTypes.h:28-31`).
2. Thread `MicProcRaw` (fixa na CPU 2) chama `ProcessRawAudio` e depois `ProcessMicrophonesSE` (`:419`).
3. Escolha do modo de processamento (`:568-576`):

| Situação | Modo | O que faz |
|---|---|---|
| Robô parado, sem calm mode | `SigEsBeamformingOn` | Beamforming e redução de ruído da Signal Essence (`MMIfProcessMicrophones`, `:646-650`) |
| Robô se mexendo | `SigEsBeamformingOff` | Mesma biblioteca, sem beamforming; direção é descartada (`:661-667`) |
| Calm mode (power save, dormindo) | `NoProcessingSingleMic` | Um microfone, só tira o nível DC e multiplica por 8 (`:618-634`) |

4. VAD (`DoSVad`, `:533-541`). Com o robô se mexendo, o VAD é forçado para "fala" (`:545-548`).
5. Direção da voz: 12 direções (`micDataTypes.h:42-43`), calculadas pela Signal Essence e guardadas em `MicImmediateDirection` (`micImmediateDirection.cpp`). Vão para o engine na mensagem `MicDirection` (`micDataProcessor.cpp:449-465`) e entram em `engine/components/mics/micDirectionHistory.cpp`.
6. Thread `MicProcTrigger` (CPU 1) passa o áudio processado para a wake word.

**Pontos que afetam a qualidade (FATO):**
- **Sem cancelamento de eco real:** a referência do alto-falante passada à Signal Essence é um buffer de zeros (`dummySpeakerOut`, `:643-650`). O robô não "subtrai" o próprio som.
- `kMicData_SpeakerNoiseDisablesMics` é `false` (`:55`): com o alto-falante tocando, o beamforming continua ligado.
- O travamento na direção da wake word está comentado de propósito (`:241-251`).

### 2.3 Do áudio ao WirePod e de volta

**Ida (FATO):**
1. O Porcupine detecta e chama `MicDataProcessor::TriggerWordDetectCallback` (`micDataProcessor.cpp:188-253`).
2. O anim manda `TriggerWordDetected` ao engine (com direção) e toca a animação de "escuta" (get-in) e o som (earcon).
3. **Só depois do earcon** o callback cria o job de streaming (`CreateTriggerWordDetectedJobs`, `:203-210`), copiando apenas 50 ms de áudio anterior (`kTriggerOverlapSize_ms`, `micDataTypes.h:34`).
4. `MicDataSystem::Update` (`animProcess/src/cozmoAnim/micData/micDataSystem.cpp:478-560`) manda `hotword` e depois blocos `audio` pelo socket `mic_sock` ao `vic-cloud`.
5. O streaming para quando chega `stopSignal` do vic-cloud ou após 6050 ms (`kStreamingTimeout_ms`, `micDataTypes.h:37`; `micDataSystem.cpp:525-535`).
6. No `vic-cloud`: `voice.Process.Run` (`cloud/internal/voice/process.go:142-350`) abre um stream gRPC "chipper" para o endereço `chipper` de `/data/data/server_config.json` (que o WirePod escreve; senão usa `/anki/data/assets/cozmo_resources/config/server_config.json`) (`cloud/internal/config/urls.go:48-59`).
7. Áudio comprimido em Opus (`WithCompression(true)`, `cloud/cloud/main.go:202`; parâmetros em `process.go:367-376`), em pacotes de 120 ms (`main.go:199`). Timeout total do stream: 9 s (`process.go:33`).

**Volta (FATO):**
1. O WirePod responde com a intent. O `vic-cloud` manda `stopSignal` ao anim e `result` ao engine pelo socket `ai_sock` (`process.go:278-292`).
2. No engine, a thread `BehaviorServer` recebe (`engine/aiComponent/behaviorComponent/behaviorComponentCloudServer.cpp:59-130`) e chama `UserIntentComponent::OnCloudData` (`engine/aiComponent/behaviorComponent/userIntentComponent.cpp:1143-1149`).
3. No tick seguinte, `UserIntentComponent::UpdateDependent` (`:792-830`) chama `SetIntentPendingFromCloudJSONValue` (`:584-661`).
4. Essa função procura o nome da intent em `user_intent_map.json` via `UserIntentMap::GetUserIntentFromCloudIntent` (`engine/aiComponent/behaviorComponent/userIntentMap.cpp:297-322`) e marca a user intent como pendente.
5. Um behavior pega a intent (seção 3). Se ninguém pegar em 3 ticks (`kMaxTicksToClear`, `userIntentComponent.cpp:58`), ela é descartada.

### 2.4 Causas prováveis, do lado do robô, de intents que não disparam

Ordenadas pela chance de ser o seu problema.

**Do momento da fala:**
1. **Falar logo após "Hey Vector" perde o começo da frase (FATO no mecanismo).** O áudio só começa a ser enviado depois do earcon (`micDataProcessor.cpp:203-210`), com 50 ms de sobra. O que você diz durante a animação de escuta não chega ao WirePod.
2. **O robô se vira enquanto você fala (FATO no mecanismo; efeito é HIPÓTESE).** Depois da wake word ele roda `ReactToTriggerDirectionAwake` (`resources/config/engine/behaviorComponent/behaviors/victorBehaviorTree/triggerWordDetected.json`). Movimento desliga o beamforming (`micDataProcessor.cpp:568-571`) e o motor faz ruído.
3. **Sem cancelamento de eco (FATO):** o earcon e qualquer som do robô entram no áudio enviado.

**Do caminho entre processos:**

4. **Nome da intent sem entrada no mapa (FATO):** vira `unmatched_intent` e o robô faz "não entendi". Log: `UserIntentMap.NoCloudIntentMatch` (`userIntentMap.cpp:317`).
5. **Parâmetros diferentes do esperado (FATO):** a intent é descartada em silêncio, só com aviso no log: `BadParams` ou `MissingParams` (`userIntentComponent.cpp:628-648`). Comum quando o WirePod manda um nome de intent certo com parâmetros num formato diferente.
6. **Controle SDK ativo (FATO):** com um cliente SDK no controle, só intents da lista permitida passam; o resto vira `unmatched_intent` (`userIntentComponent.cpp:650-656`). Log: `UserIntentComponent.IgnoringNonWhitelist.Cloud`.
7. **Sem resposta à wake word configurada (FATO):** se nenhum behavior tiver empilhado uma resposta (`PushResponseToTriggerWord`, `userIntentComponent.cpp:989-1003`), o anim não abre o stream. Log: `MicDataProcessor.TWCallback` com "Don't have a wake word response setup" (`micDataProcessor.cpp:212-215`). Acontece em modos especiais (quieto, emergência, dormindo).
8. **`vic-cloud` desconectado (FATO):** sem cliente no `mic_sock`, o stream é ignorado. Log: `MicDataSystem.Update.StreamingStartIgnored` (`micDataSystem.cpp:509`).

**Defeitos de código encontrados:**

9. **Uma única vaga para a resposta (FATO):** `OnCloudData` sobrescreve `_pendingCloudIntent` (`userIntentComponent.cpp:1148`). Se duas mensagens chegam no mesmo tick (por exemplo `streamOpen` e `result`), só a última é tratada. **HIPÓTESE:** raro, mas uma intent pode se perder.
10. **A thread que recebe intents pode morrer (FATO):** em `select` interrompido por sinal (`EINTR`), a thread `BehaviorServer` faz `return` e não volta mais (`behaviorComponentCloudServer.cpp:93-96`). Daí em diante nenhuma intent chega até reiniciar o engine. **HIPÓTESE:** raro, só com sinal tratado entregue a essa thread.
11. **Fim do áudio cortado (HIPÓTESE):** o `bufferRoutine` só envia blocos completos de 120 ms (`cloud/internal/voice/stream/context.go:38-61`); o resto no buffer pode não ser enviado quando o stream fecha. Até 120 ms do fim da frase.

**Como diagnosticar no robô (build de desenvolvimento):**
1. Salvar o áudio exato que foi para o WirePod: `curl "http://<robô>:8889/consolevarset?key=kMicData_SaveRawFullIntent&value=1"` (variável em `micDataProcessor.cpp:65`). Os arquivos vão para `/data/data/com.anki.victor/cache/micdata/triggeredCapture/` (`:262`, até 100 arquivos). Ouça e veja se o começo da frase está lá.
2. Seguir uma frase no log: `grep -E "TWCallback|StreamingStart|StreamingEnd|NoCloudIntentMatch|BadParams|MissingParams|IgnoringNonWhitelist|GotError" /var/log/messages`.
3. Comparar com o log do WirePod: se o WirePod transcreveu certo e o robô não reagiu, o problema está nos itens 4 a 10. Se transcreveu errado, nos itens 1 a 3.

---

## 3. Intents

### 3.1 Onde ficam

| Item | Arquivo |
|---|---|
| Mapa intent da nuvem → user intent | `resources/config/engine/behaviorComponent/user_intent_map.json` |
| Lista de user intents (union CLAD, compilada) | `clad/src/clad/types/behaviorComponent/userIntent.clad` |
| Leitura do mapa | `engine/aiComponent/behaviorComponent/userIntentMap.cpp`, classe `UserIntentMap` |
| Intent pendente, ativação, timeout | `engine/aiComponent/behaviorComponent/userIntentComponent.cpp`, classe `UserIntentComponent` |
| Documentação da Anki | `docs/architecture/behaviors_intents.md` |

**Duas seções no `user_intent_map.json` (FATO):**
- `simple_voice_responses`: intent da nuvem → grupo de animação, sem C++. Exemplo: `intent_greeting_hello` → `ag_greeting_hello` (`user_intent_map.json:2-20`). Tratadas por `BehaviorSimpleVoiceResponse` (`engine/aiComponent/behaviorComponent/behaviors/behaviorSimpleVoiceResponse.cpp`).
- `user_intent_map`: cada entrada tem `cloud_intent` (o que o WirePod manda), `user_intent` (nome na union CLAD), e opcionalmente `app_intent` (o que o SDK manda), `feature_gate` e regras de troca de nomes de parâmetros.

### 3.2 Como uma intent liga a um behavior

Duas formas, ambas FATO:
- **Em C++:** o behavior pergunta em `WantsToBeActivatedBehavior()` com `GetBehaviorComp<UserIntentComponent>().IsUserIntentPending(USER_INTENT(x))` e, ao ativar, chama `SmartActivateUserIntent(USER_INTENT(x))`. Exemplo completo: `engine/aiComponent/behaviorComponent/behaviors/victor/behaviorWireTest.cpp:47-52`, `:72-78`.
- **Em JSON:** a chave `respondToUserIntents` em qualquer behavior (`engine/aiComponent/behaviorComponent/behaviors/iCozmoBehavior.cpp:77-105`).

O behavior precisa estar dentro de um dispatcher que roda. Os de voz ficam em `resources/config/engine/behaviorComponent/behaviors/victorBehaviorTree/reactions/voiceFeatures.json`.

### 3.3 Passo a passo: intent nova

**Caminho A, só animação, sem compilar (o mais fácil):**
1. No WirePod, crie a intent personalizada com o nome `intent_meu_teste` (configuração do WirePod, fora deste repositório).
2. Em `resources/config/engine/behaviorComponent/user_intent_map.json`, dentro de `simple_voice_responses`, adicione:
   ```json
   {
     "cloud_intent": "intent_meu_teste",
     "response": {
       "anim_group": "ag_greeting_hello",
       "emotion_event": "RespondToShortVoiceCommand",
       "active_feature": "ReactToHello"
     }
   }
   ```
3. Copie o arquivo para o robô em `/anki/data/assets/cozmo_resources/config/engine/behaviorComponent/user_intent_map.json` (o sistema de arquivos pode precisar ser remontado como gravável; isso depende do wire-os).
4. `systemctl restart vic-engine` e diga "Hey Vector" + a frase.

Troque `anim_group` por um grupo que já exista no robô. Os arquivos de grupo não estão neste repositório (vêm com os assets); use nomes já citados em `user_intent_map.json` ou em `resources/assets/cladToFileMaps/AnimationTriggerMap.json`.

**Caminho B, intent com behavior próprio (exige compilar):**
1. Em `clad/src/clad/types/behaviorComponent/userIntent.clad`, adicione `UserIntent_Void meu_teste,` perto de `test_wire` (`:187`). Se a intent tiver parâmetros, crie uma `structure` antes.
2. Em `user_intent_map.json`, dentro de `user_intent_map`:
   ```json
   { "user_intent": "meu_teste", "cloud_intent": "intent_meu_teste", "app_intent": "intent_meu_teste" }
   ```
3. Crie o behavior que responde (seção 4.4) ou use `respondToUserIntents` num behavior existente.
4. Coloque o behavior em `voiceFeatures.json`.
5. Atualize os testes citados em `docs/architecture/behaviors_intents.md` (`completedUserIntents.json`, `testUserIntentTransitions.cpp`).
6. Compile e instale o engine.

Com o `app_intent`, você pode testar sem voz: o SDK manda `AppIntent{intent:"intent_meu_teste"}` (`engine/aiComponent/behaviorComponent/userIntentComponent.cpp:1151-1185`).

---

## 4. Behaviors

### 4.1 Onde ficam

| Item | Caminho |
|---|---|
| Classes C++ | `engine/aiComponent/behaviorComponent/behaviors/` (por pasta: `reactions/`, `sleeping/`, `exploring/`, `victor/`, `sdkBehaviors/`...) |
| Classe base | `engine/aiComponent/behaviorComponent/behaviors/iCozmoBehavior.h`, classe `ICozmoBehavior` |
| JSON de cada instância | `resources/config/engine/behaviorComponent/behaviors/` |
| Árvore principal | `resources/config/engine/behaviorComponent/behaviors/victorBehaviorTree/` |
| Condições reutilizáveis | `engine/aiComponent/beiConditions/conditions/` (ver `docs/architecture/beiConditions.md`) |
| Documentação da Anki | `docs/architecture/behaviors.md` |

### 4.2 Como um behavior é registrado

**FATO:** são três listas compiladas, geradas à mão por `tools/ai/generateBehaviorCode.py` (o script `tools/ai/createNewBehavior.py` cria o esqueleto e chama o gerador):
1. `clad/src/clad/types/behaviorComponent/behaviorClasses.clad`: nome da classe C++ (ex.: `WireTest`, `:263`).
2. `clad/src/clad/types/behaviorComponent/behaviorIDs.clad`: nome da instância, um por JSON (ex.: `WireTest`, `:492`).
3. `engine/aiComponent/behaviorComponent/behaviorFactory.cpp`: o `switch` que cria a classe (ex.: `:1212-1214`).

No boot, `RobotDataLoader::LoadBehaviors` (`engine/robotDataLoader.cpp:426-454`) lê todos os JSON e o `BehaviorContainer` (`engine/aiComponent/behaviorComponent/behaviorContainer.cpp:35-60`) cria cada um pela fábrica.

**Por isso um behavior só por JSON falhou:** o `behaviorID` precisa existir em `behaviorIDs.clad`, que é compilado. **Exceção:** behaviors anônimos (`"anonymousBehaviors"` dentro do JSON de outro behavior) não precisam de ID (`iCozmoBehavior.cpp:502-567`). Exemplo: `victorBehaviorTree/reactions/simpleVoiceResponses.json`.

### 4.3 Como um behavior é escolhido

**FATO:**
- A raiz é o `ModeSelector` (`victorBehaviorTree/modeSelector.json`), que escolhe entre modos (desligando, Alexa, emergência, quieto, SDK, ciclo de sono).
- Dentro do modo acordado, `GlobalInterruptions` (`victorBehaviorTree/globalInterruptions.json`) é um `DispatcherStrictPriority`: percorre a lista na ordem e ativa o **primeiro** cuja `WantsToBeActivated()` for verdadeira. A ordem é: reações físicas → SDK → wake word → timers → ... → `VoiceFeatures` → ... → `HighLevelAI`.
- `HighLevelAI` (`engine/aiComponent/behaviorComponent/behaviors/behaviorHighLevelAI.cpp`) é a máquina de estados do tempo livre (explorar, socializar, brincar, ficar na base).
- Um behavior ativo delega para outros (`DelegateIfInControl`) ou executa ações (`engine/actions/`).
- Para ver ao vivo: módulo `behaviors` do webviz em `http://<robô>:8888/webViz.html` (build de desenvolvimento).

### 4.4 Passo a passo: behavior novo copiando o `WireTest`

O `WireTest` é o exemplo mais simples do código: toca uma animação e diz "Hello World" quando recebe a intent `test_wire`. Foi adicionado por este fork e já está ligado de ponta a ponta.

Arquivos de referência:
- `engine/aiComponent/behaviorComponent/behaviors/victor/behaviorWireTest.h`
- `engine/aiComponent/behaviorComponent/behaviors/victor/behaviorWireTest.cpp`
- `resources/config/engine/behaviorComponent/behaviors/victorBehaviorTree/wireTest.json`
- entrada em `user_intent_map.json:197-201` e em `reactions/voiceFeatures.json:21`

Passos para um `BehaviorMeuOla`:
1. Copie `behaviorWireTest.h/.cpp` para `behaviorMeuOla.h/.cpp` na mesma pasta e troque `WireTest` por `MeuOla` nos nomes.
2. No `.cpp`, troque `USER_INTENT(test_wire)` pela sua intent (seção 3.3, caminho B) e o texto do `SayTextAction`.
3. Crie `victorBehaviorTree/meuOla.json`:
   ```json
   { "behaviorID": "MeuOla", "behaviorClass": "MeuOla" }
   ```
4. Rode `python tools/ai/generateBehaviorCode.py` para atualizar `behaviorClasses.clad`, `behaviorIDs.clad` e `behaviorFactory.cpp`. Confira o diff dos três.
5. Adicione `"MeuOla"` na lista de `reactions/voiceFeatures.json`.
6. Compile o engine (build oficial por Docker), instale e reinicie o `vic-engine`.
7. Teste pelo SDK com `AppIntent` antes de testar por voz.

Peças úteis para o corpo do behavior (FATO):
- Animação: `TriggerLiftSafeAnimationAction(AnimationTrigger::...)` (`engine/actions/animActions.h`).
- Fala: `SayTextAction` (`engine/actions/sayTextAction.h`).
- Movimento: `DriveStraightAction`, `TurnInPlaceAction` (`engine/actions/basicActions.h`).
- Encadear: `DelegateIfInControl(acao, callback)`.

---

## 5. Câmera e visão

### 5.1 Onde o stream é processado

| Etapa | Onde |
|---|---|
| Captura (daemon `mm-anki-camera`) | `platform/camera/vicos/camera_client/camera_client.c` (6 slots de quadro, `:55`) |
| Serviço de câmera no engine | `platform/camera/cameraService_vicos.cpp`, classe `CameraService` |
| Pega o quadro e manda para a thread | `engine/components/visionComponent.cpp`, classe `VisionComponent` |
| Processamento (thread "VisionSystem") | `engine/vision/visionSystem.cpp`, classe `VisionSystem` |
| Quais modos rodam e com que frequência | `engine/components/visionScheduleMediator/`; config em `resources/config/engine/visionScheduleMediator_config.json` |

**FATO:** sensor 1280x720, processamento em 640x360 (`robot/include/anki/cozmo/shared/cozmoConfig.h:185-189`). Um quadro por vez: se a thread está ocupada, o quadro novo é descartado.

### 5.2 Detectores existentes

Lista de modos em `clad/src/clad/types/visionModes.clad:13-32`.

| Modo | Implementação |
|---|---|
| `Markers` (marcadores dos cubos e da base) | `coretech/vision/engine/markerDetector.cpp` |
| `Faces` (rosto, identidade, olhar) | `coretech/vision/engine/faceTrackerImpl_yunet.cpp`, `faceRecognizer_yunet.cpp`, `faceNet.cpp`, `gazeDirection.cpp`, `eyeContact.cpp` |
| `People`, `Hands` (redes neurais TFLite) | `coretech/neuralnets/neuralNetModel_tflite.cpp`; modelos em `resources/config/engine/vision_config.json` (`NeuralNets`, `person_detector`, `hand_detector`) |
| `Motion` | `engine/vision/motionDetector.cpp` |
| `OverheadEdges`, `Obstacles` | `engine/vision/overheadEdgesDetector.cpp`, `groundPlaneClassifier.cpp` |
| `Lasers`, `BrightColors`, `Illumination` | `laserPointDetector.cpp`, `coretech/vision/engine/brightColorDetector.cpp`, `illuminationDetector.cpp` |
| `Offboard` (manda a imagem para fora) | `coretech/vision/engine/offboardProcessor.cpp`, `coretech/neuralnets/neuralNetModel_offboard.cpp` |

Resultados de redes neurais viram "pontos salientes" (`SalientPoint`) e vão para `engine/aiComponent/salientPointsComponent.cpp`.

### 5.3 Opções para processar os frames num PC

| Opção | Como | Prós e contras |
|---|---|---|
| **A. SDK `CameraFeed`** | gRPC `CameraFeed` / `EnableImageStreaming` (`tools/protobuf/gateway/public/external_interface.proto:206`, `:480`); o engine envia JPEG em `ImageChunk` (`engine/components/visionComponent.cpp:1565-1640`) | Funciona hoje com o SDK do WirePod. Só ida: o resultado não volta para o robô sozinho. |
| **B. Modo `Offboard` (já existe, ida e volta)** | O engine salva o quadro, o `vic-cloud` manda por gRPC `AnalyzeImage` (`tools/protobuf/vision/private/vision.proto:16-18`) ao endereço `offboard_vision` de `server_config.json` (`cloud/internal/config/urls.go:19`; cliente em `cloud/internal/offboard_vision/client_dev.go:71-145`). A resposta JSON vira pontos salientes no engine (`coretech/vision/engine/parseSalientPointsFromJson.cpp`). | Melhor para fechar o ciclo: os behaviors reagem ao que o PC detectar. Só em build de desenvolvimento (`ANKI_DEV_CHEATS`, `engine/vision/visionModesHelpers.cpp:32-35`; `server_shipping.go` desliga). Você escreve o servidor gRPC no PC. |
| **C. `CaptureSingleImage`** | gRPC (`external_interface.proto:488`) | Simples, foto sob demanda. Lento para fluxo contínuo. |

Recomendação: **A** para começar a ver imagens no PC hoje; **B** quando quiser que o robô reaja ao que o PC vê.

Para B, configurar no robô (HIPÓTESE no formato exato; os nomes das chaves são FATO):
```json
// /data/data/server_config.json (o WirePod já escreve este arquivo; acrescente a chave)
{ "chipper": "...", "offboard_vision": "IP_DO_PC:PORTA" }
```
E um behavior que peça o modo `Offboard`.

---

## 6. Visão futura: onde encaixar cada capacidade

### 6.1 Reconhecer mais objetos

- **No robô (TFLite):** três mudanças:
  1. adicionar o modelo em `resources/config/engine/vision_config.json`, bloco `NeuralNets`;
  2. criar um `VisionMode` em `clad/src/clad/types/visionModes.clad`;
  3. ligar modo e rede em `sNetModeLUT` (`engine/vision/visionModesHelpers.cpp:27-36`).
- **Cuidado (FATO):** se um modo pedir uma rede que não existe, o engine sai com `exit(-1)` (`engine/vision/visionSystem.cpp:1836-1844`). Há um comentário "this made robo crash" na própria tabela (`visionModesHelpers.cpp:30`). A CPU é pequena: prefira o PC.
- **No PC:** use o modo `Offboard` (5.3 B). Cada objeto volta como `SalientPoint` com tipo e retângulo.

### 6.2 Memória temporal

| Peça existente | Onde | Uso |
|---|---|---|
| Variáveis persistentes | `engine/components/variableSnapshot/variableSnapshotComponent.cpp` | Guardar valores entre reinícios |
| Rostos conhecidos e "visto por último" | `engine/faceWorld.h`, classe `FaceWorld` | Lembrar pessoas |
| Quadro-negro da IA | `engine/aiComponent/aiWhiteboard.cpp`, classe `AIWhiteboard` | Fatos de curto prazo compartilhados entre behaviors |
| Histórico de direção da voz | `engine/components/mics/micDirectionHistory.cpp` | 10 s de direções |
| Estatísticas de longo prazo | `engine/components/robotStatsTracker.cpp` | Contadores salvos |

Sugestão: uma memória de eventos ("vi X às HH:MM em tal lugar") cabe como um componente novo no padrão do `AIWhiteboard`, salvo pelo `VariableSnapshotComponent`.

### 6.3 Memória espacial

- **Mapa de navegação:** `engine/navMap/mapComponent.cpp`, classe `MapComponent`, sobre uma quadtree (`engine/navMap/quadTree/`) com tipos de conteúdo (livre, obstáculo, penhasco, borda). Ver `docs/architecture/map.md`. É de curto alcance (2 m) e é apagado quando o robô perde a localização.
- **Objetos com pose:** `engine/blockWorld/blockWorld.cpp`, classe `BlockWorld`.
- **Injetar objetos de fora (FATO, já existe):** `CreateFixedCustomObject` e `DefineCustomBox` pelo SDK (`external_interface.proto:444-452`; `blockWorld.cpp:233-243`). Entram no mapa como obstáculos.
- Sugestão: guardar o mapa de longo prazo no PC e mandar ao robô só os objetos relevantes.

### 6.4 Curiosidade

| Peça existente | Onde |
|---|---|
| Emoções (`Happy`, `Confident`, `Social`, `Stimulated`, `Trust`) | `clad/src/clad/types/emotionTypes.clad:10-18`; `engine/moodSystem/` |
| Exploração | `engine/aiComponent/behaviorComponent/behaviors/exploring/behaviorExploring.cpp` |
| Máquina de estados do tempo livre | `behaviorHighLevelAI.cpp`; config em `victorBehaviorTree/highLevelAI.json` |
| Reação a ponto saliente | `engine/aiComponent/beiConditions/conditions/conditionSalientPointDetected.cpp` |
| Árvore definida pelo usuário (não auditada) | `engine/aiComponent/behaviorComponent/userDefinedBehaviorTreeComponent/` |

Sugestão: "curiosidade" = subir `Stimulated` quando algo novo aparece (objeto nunca visto, som em direção nova) e deixar o `HighLevelAI` escolher explorar. O ponto de entrada é o `MoodManager` (`engine/moodSystem/moodManager.cpp`) com eventos de emoção definidos em JSON.

---

## Não verificado

- Configuração do WirePod (fora deste repositório): formato exato da intent e dos parâmetros que ele envia.
- Se o seu build tem `ANKI_DEV_CHEATS` ligado (necessário para webviz, variáveis de console e modo `Offboard`).
- Nomes exatos de grupos de animação disponíveis no seu robô.
- Se `/anki` é gravável no seu robô (depende do wire-os).
- Internos da biblioteca Signal Essence (binária).
