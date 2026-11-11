# Auditoria técnica do wire-os-victor

Data: 2026-11-11. Base: commit `505eb84c` do wire-os-victor e commit `a6e4c3a8` do wire-os (somente leitura).

Escopo: análise estática do código. Nada foi compilado nem executado, e não houve acesso ao robô. Todo número de custo (CPU, memória, tempo) é estimativa por leitura de código, não medição. Cada afirmação está marcada como **FATO** (visto no código, com `arquivo:linha`) ou **HIPÓTESE**.

Caminhos sem prefixo são do wire-os-victor. Caminhos com prefixo `wire-os:` são do repositório wire-os.

## Resumo

- **As quedas de hora em hora têm causa identificada.** O webserver fecha qualquer websocket que fique 1 hora sem receber dados do cliente (`webServerProcess/src/webService.cpp:979-980`). O mesmo código serve o engine (8888) e o anim (8889), por isso as duas conexões caem juntas.
- **A causa mais provável da morte do engine é um deadlock no webserver**, disparado pela rajada de assinaturas na reconexão (achado A1). O caminho de código do deadlock é fato; que ele tenha ocorrido naquela noite é hipótese.
- **"vic-engine crashed" não significa necessariamente crash.** O código 915 aparece com o mesmo texto quando o engine apenas trava por 5 s, e o engine tem três caminhos de saída limpa e deliberada que também terminam nessa tela (achado A4).
- **"Restarts exhausted" exige três falhas seguidas com menos de 60 s entre elas.** Um crash isolado por hora nunca esgota o limite (`wire-os:anki/fault-code/fault-code-handler:19-26, 248-279`).
- **A hipótese original (acúmulo de memória por horas) continua possível, mas não é a mais provável.** Existe uma fila de envio sem limite (A2), porém ela só cresce se o cliente parar de ler. O dado que decide está no roteiro da seção 4.

Plataforma confirmada no código:
- **FATO:** toolchain `-march=armv7-a -mcpu=cortex-a7` (`cmake/vicos.oelinux.toolchain.cmake:158-161`).
- **FATO:** máquina `apq8009-robot`, kernel msm-4.9 com `msm8909_defconfig`, `mem=511M` (`wire-os:poky/victor/meta-qcom/conf/machine/apq8009-robot.conf:15,20`).
- **FATO:** build de release usa `ANKI_PROFILING_ENABLED=0` e `REMOTE_CONSOLE_ENABLED=1` (`project/victor/scripts/victor_build_release.sh:5-13`). `ANKI_DEV_CHEATS` fica no padrão 1 (`lib/util/source/anki/util/global/globalDefinitions.h:23-32`). Consequência: webviz, console e PerfMetric estão ligados; o profiler de CPU e os avisos de tick lento estão compilados fora.

---

## 1. Achados priorizados

Escala: impacto (alto, médio, baixo), esforço (horas ou dias de trabalho de um desenvolvedor), risco da mudança (baixo, médio, alto).

| # | Achado | Impacto | Esforço | Risco |
|---|---|---|---|---|
| A1 | Inversão de ordem de locks no webserver (deadlock da thread principal) | Alto | 2 h | Baixo |
| A2 | Fila de envio do websocket sem limite e laço ocupado com cliente lento | Alto | 1 dia | Médio |
| A3 | Timeout de websocket de 1 h, sem ping, e só 4 threads | Alto | 1 h (cliente) / 3 h (servidor) | Baixo |
| A4 | Engine sai de propósito em um único `EAGAIN` no IPC | Alto | 1 dia | Médio |
| A5 | Dados do cliente websocket sem validação (exceção derruba o engine) | Alto | 2 h | Baixo |
| A6 | Índice e ponteiro de conexão obsoletos após fechamento | Médio | 4 h | Baixo |
| A7 | NavMap: mapa inteiro na thread principal, sem limite de taxa | Médio | 1 dia | Baixo |
| A8 | Datagrama do gateway sem checagem de tamanho (engine sai) | Médio | 2 h | Baixo |
| A9 | Tick lento não é registrado; não há telemetria de memória por processo | Médio | 3 h | Baixo |
| A10 | Controle SDK sem timeout e sem dono | Médio | 1 dia | Médio |
| A11 | Temperatura: aviso só a 90 °C, sem ação; trabalho contínuo na base | Médio | 2 dias | Médio |
| A12 | `vic-robot`: contador de timeout que nunca zera e outros 3 defeitos | Médio | 2 h | Baixo |
| A13 | BehaviorID ou classe desconhecidos viram comportamento indefinido | Médio | 2 h | Baixo |
| A14 | `RobotState` a 16,7 Hz sem limite, com alocação por tick | Baixo | 4 h | Baixo |
| A15 | Alocações por tick no sistema de comportamentos e no microfone | Baixo | 1 dia | Baixo |
| A16 | Portas de depuração sem autenticação; segredos versionados | Médio (segurança) | 4 h | Baixo |
| A17 | Texto de falha igual para 914 e 915; pouco é gravado ao cair | Médio (diagnóstico) | 3 h | Baixo |

### A1. Inversão de ordem de locks no webserver

**Local:** `webServerProcess/src/webService.cpp:1043`, `1231`, `1307-1311`, `1688-1725`.

**Comportamento atual (FATO):**
- Thread do civetweb, em `OnReceiveWebSocket`: trava `s_wsConnectionsMutex` (`:1690`) e, ainda com ele travado, chama `ProcessRequest`, que chama `AddRequest` e trava `_requestMutex` (`:1309`).
- Thread principal do engine, em `WebService::Update`: trava `_requestMutex` durante a função inteira (`:1043`) e, dentro dela, o `sendToClient` trava `s_wsConnectionsMutex` (`:1231`). O mesmo ocorre em qualquer `SendToWebViz` ou `IsWebVizClientSubscribed` chamado por um handler.
- `WebService::Update` é chamado a cada tick pelo engine (`engine/cozmoEngine.cpp:391`).

**Problema:**
- **FATO:** as duas threads pegam os mesmos dois mutexes em ordens opostas. Isso é um deadlock clássico (ABBA).
- **FATO:** o deadlock precisa de uma mensagem `subscribe` ou `data` chegando enquanto o `Update` processa uma anterior cujo handler responde na hora. Handlers que respondem na assinatura incluem behaviors, intents, mood, features, sleeping, heldinpalm, cloudintents e socialpresence.
- **HIPÓTESE:** a reconexão de um cliente com 21 módulos manda 21 `subscribe` em rajada, o que abre essa janela várias vezes seguidas. O engine trava para sempre, o anim deixa de receber, e após 5 s o anim levanta o código 915 (`animProcess/src/cozmoAnim/animProcessMessages.cpp:882-913`). O handler de falhas então manda `kill -ILL` no engine (`wire-os:anki/fault-code/fault-code-handler:144-148`) e reinicia tudo.
- **HIPÓTESE:** o mesmo mecanismo explica "muitos pedidos de mapa travam o robô": cada pedido é uma mensagem `data` que passa por esse caminho.
- **HIPÓTESE:** se o cliente reconecta e reassina logo que a porta volta, o deadlock se repete dentro da janela de 60 s e esgota o limite de reinícios. Isso explicaria "Restarts exhausted". Ponto em aberto: 24 s entre a queda das 08:03 e a morte final é pouco para três ciclos se `TIMEOUT_RESTART_SEC=30` (configuração oskr). É preciso o log do handler para fechar a conta.

**Mudança sugerida:**
```cpp
// OnReceiveWebSocket: decidir sob o lock, enfileirar fora dele
std::string moduleName; size_t idx = 0; RequestType type; bool enqueue = false;
{
  std::lock_guard<std::mutex> lock(s_wsConnectionsMutex);
  // ... localizar conexão, atualizar subscribedModules, preencher as variáveis ...
}
if (enqueue) { ProcessRequest(conn, type, moduleName, std::to_string(idx), payload, false); }

// Update: tirar a lista sob o lock e processar sem ele
std::vector<Request*> work;
{ std::lock_guard<std::mutex> lock(_requestMutex); work.swap(_requests); }
for (Request* r : work) { /* processar; devolver à lista os que ainda esperam resposta */ }
```

**Como medir e validar no robô:**
- Reproduzir: um script que conecta em `ws://<robô>:8888/socket`, assina 21 módulos, fecha e repete em laço. Com o defeito, o engine deve travar em minutos.
- Confirmar o travamento: `gdb -p $(pidof vic-engine) -batch -ex "thread apply all bt"`. O esperado é a thread principal em `pthread_mutex_lock` dentro de `sendToClient` e uma thread do civetweb em `AddRequest`.
- Sem gdb: o minidump gerado pelo `kill -ILL` (ver seção 4) deve mostrar a mesma pilha.
- Depois da correção: o mesmo script por 1 h sem travar.

### A2. Fila de envio sem limite e laço ocupado com cliente lento

**Local:** `webServerProcess/src/webService.cpp:1567-1581`, `1661-1671`; `lib/util/source/anki/util/dispatchQueue/taskExecutor.cpp:185-190`; `lib/util/source/3rd/civetweb/src/civetweb.c:4533-4572`, `13839`.

**Comportamento atual (FATO):**
- Cada mensagem de cada módulo vira uma tarefa na fila "WebsocketSender", com cópia completa do `Json::Value` capturada na lambda (`webService.cpp:1664`).
- A fila é um `push_back` sem limite (`taskExecutor.cpp:185-190`).
- O `Json::Value` é copiado três vezes na thread de quem envia: em `payload["data"] = data` (`:1576`), na captura da lambda, e ao guardar a `std::function` na fila.
- A serialização usa `ss << data`, que é o escritor com indentação (tabs e quebras de linha).
- Os sockets são não bloqueantes (`civetweb.c:13839`). Quando `send()` devolve `EWOULDBLOCK`, o `push()` repete a chamada em laço, sem `poll` e sem dormir, até estourar `request_timeout_ms`, cujo padrão é 30 s (`civetweb.c:1655`, `4533-4572`).
- Existe uma única fila serial para todas as conexões.

**Problema:**
- **HIPÓTESE:** se o cliente para de ler (notebook suspenso, aba congelada, Wi-Fi ruim), a thread de envio queima um núcleo inteiro por até 30 s por mensagem. Enquanto isso a thread principal segue enfileirando cópias de JSON, e a memória do engine cresce sem limite.
- **FATO:** uma conexão parada bloqueia o envio para todas as outras.
- **FATO:** após o timeout o envio fica parcial, e o próximo quadro começa no meio do anterior, corrompendo o fluxo (`civetweb.c:4604-4613`).
- **FATO:** há um comentário da própria Anki dizendo "if we send this data every tick, we crash the robot" (`animProcess/src/cozmoAnim/faceDisplay/faceInfoScreenManager.cpp:696-697`).
- Isso liga três sintomas: calor (núcleo a 100 %), memória (fila) e travamento.

**Mudança sugerida:**
```cpp
void WebService::SendToWebSockets(const std::string& module, const Json::Value& data) const {
  // serializa uma vez, sem indentação, e compartilha a string
  Json::FastWriter w;
  auto str = std::make_shared<const std::string>(
      "{\"module\":\"" + module + "\",\"data\":" + w.write(data) + "}");
  std::lock_guard<std::mutex> lock(s_wsConnectionsMutex);
  for (auto& c : _webSocketConnections) {
    if (!c.subscribedModules.count(module)) continue;
    if (c.state->pendingBytes.load() > kMaxPendingBytes /* 256 KB */) { ++c.state->dropped; continue; }
    c.state->pendingBytes += str->size();
    Util::Dispatch::Async(_dispatchQueue, [st = c.state, str] {
      if (st->open.load()) { mg_websocket_write(st->conn, WebSocketsTypeText, str->data(), str->size()); }
      st->pendingBytes -= str->size();
    });
  }
}
```
Complementos:
- No `push()` do civetweb, trocar o laço por `poll(POLLOUT, 50 ms)`.
- Configurar `request_timeout_ms` em 2000.
- Fechar a conexão em escrita falha ou parcial.
- Para módulos de estado, guardar só a última mensagem por (conexão, módulo).

**Como medir e validar no robô:**
- `top -H -p $(pidof vic-engine)` e observar a thread "WebsocketSender".
- `grep VmRSS /proc/$(pidof vic-engine)/status` a cada minuto.
- Reproduzir: assinar os módulos e pausar o cliente com `kill -STOP`. Com o defeito, a thread vai a 100 % e o RSS sobe.
- Depois da correção: RSS estável e contador de mensagens descartadas subindo.

### A3. Timeout de websocket de 1 h, sem ping, e só 4 threads

**Local:** `webServerProcess/src/webService.cpp:963-987`; `lib/util/source/3rd/civetweb/src/civetweb.c:9466-9471`, `9584-9597`; `resources/webserver/webViz.html:448-478`.

**Comportamento atual (FATO):**
- `websocket_timeout_ms` é `"3600000"` (`webService.cpp:979-980`). O civetweb 1.9.1 usa esse valor como timeout de leitura e fecha o socket se nada chegar do cliente.
- O servidor não envia ping e ignora o opcode de ping do cliente (`webService.cpp:1644`). Ele responde a mensagens `{"keepalive":...}` no nível da aplicação (`:1728-1732`).
- A página `webViz.html` não envia keepalive e não reconecta sozinha.
- `num_threads` é `"4"` (`:968-969`), e cada websocket ocupa uma thread enquanto estiver aberto.
- Patch da Anki: `TCP_USER_TIMEOUT` de 1000 ms em todo socket aceito (`civetweb.c:245-246`, `13840`).

**Problema:**
- **FATO:** um cliente que assina e depois só escuta é derrubado exatamente 3600 s após sua última mensagem. O atraso de reconexão explica a deriva de 05:02 para 06:03.
- **FATO:** é esse o limite de "~4 clientes". Com 4 websockets abertos, todo pedido HTTP (páginas, consolevars, estatísticas) fica na fila de aceitação de 20 posições.
- **FATO:** cada pedido HTTP que depende da thread principal prende uma thread por até 10 s (`webService.cpp:253-259`).
- **HIPÓTESE:** o `TCP_USER_TIMEOUT` de 1 s derruba conexões em qualquer engasgo de Wi-Fi maior que 1 s com dados pendentes.

**Mudança sugerida:**
- No cliente (vector-bridge), já: enviar `{"keepalive":true}` a cada 30 s. Isso elimina as quedas horárias sem tocar no robô.
- No servidor: baixar o timeout para 120 s, para liberar a thread de um cliente morto; subir `num_threads` para 8; recusar a conexão em `HandleWebSocketsConnect` (`:1597-1600`) quando já houver N websockets, de modo que sempre sobrem threads para HTTP.

**Como medir e validar no robô:**
- Conectar um cliente mudo e cronometrar a queda (esperado: 3600 s). Repetir com keepalive.
- `ss -tnp | grep 888` para ver conexões e a thread dona.

### A4. Engine sai de propósito em um único `EAGAIN` no IPC

**Local:** `coretech/messaging/shared/LocalUdpClient.cpp:21-22`, `170-177`, `198-204`; `engine/comms/robotConnectionManager.cpp:67-69`, `151-156`; `engine/robotInterface/messageHandler.cpp:80-84`; `engine/cozmoEngine.cpp:447-469`, `516-520`; `engine/tools/engined/cozmoEngineMain.cpp:422-435`; `animProcess/src/cozmoAnim/animComms.cpp:206-213`, `240-244`.

**Comportamento atual (FATO):**
- Os links engine↔anim e anim↔robot são sockets `AF_LOCAL SOCK_DGRAM` não bloqueantes, com buffers de 256 KB.
- Qualquer `send()` que não envie tudo, inclusive `EAGAIN` com a fila do par cheia, chama `Disconnect()` e fecha o socket (`LocalUdpClient.cpp:170-177`).
- No engine, isso faz `Update()` devolver `RESULT_FAIL_IO_CONNECTION_CLOSED`, o laço principal termina e `main` retorna 0 após registrar "Engine has stopped".
- Não há reconexão depois de entrar em `Running`. A tentativa a cada 1 s só existe no estado `ConnectingToRobot`.
- No sentido inverso, o anim derruba o engine num envio falho (`animComms.cpp:206-213`) e mostra 915 após 5 s.
- O anim fecha o socket do robô num envio falho e só o inicializa uma vez (`animComms.cpp:160-166`, `240-244`), ficando com 917 permanente.

**Problema:**
- **FATO:** uma pausa curta de um processo (carga, CPU limitada a 200 MHz em power save, pressão de memória) vira saída do engine sem minidump e sem sinal.
- **FATO:** o anim manda cerca de 145 mensagens por segundo ao engine mesmo ocioso (ver A15), o que enche a fila rápido se o engine parar.
- **HIPÓTESE:** em kernels antigos a fila de datagramas unix é limitada por `net.unix.max_dgram_qlen` (padrão 10 pacotes), independente de `SO_RCVBUF`. Não achei nenhuma configuração desse valor nos dois repositórios.

**Mudança sugerida:**
```cpp
// LocalUdpClient::Send / LocalUdpServer::Send
if (n < 0 && (errno == EAGAIN || errno == ENOBUFS)) { ++_droppedCount; return 0; } // mantém o socket
// desconectar só em ECONNREFUSED / ENOTCONN, e após N segundos seguidos de falha
```
- No engine, em erro do link com o anim, voltar para `EngineState::ConnectingToRobot` em vez de sair.
- Usar código de saída distinto e uma linha de log por motivo de saída.

**Como medir e validar no robô:**
- `sysctl net.unix.max_dgram_qlen`.
- `kill -STOP $(pidof vic-anim); sleep 2; kill -CONT $(pidof vic-anim)` e procurar no log `RobotConnectionManager.SendData.Error` seguido de `Engine has stopped`.
- Nos logs das noites de falha, procurar: `CozmoEngineMain.main: Engine has stopped`, `LocalUdpClient.Send.Fail`, `MessageHandler.ProcessMessages`, `AnimProcessMessages.MonitorConnectionState.DisconnectedFromEngine`.

### A5. Dados do cliente websocket sem validação

**Local:** `webServerProcess/src/webService.cpp:1610-1629`, `1699-1716`; `lib/util/source/3rd/civetweb/src/civetweb.c:9499-9565`.

**Comportamento atual (FATO):**
- Exceções estão habilitadas (`cmake/vicos.oelinux.toolchain.cmake:36`) e o jsoncpp é compilado com `JSON_USE_EXCEPTION 1`.
- `data["module"].asString()` e `data["type"].asString()` lançam `Json::LogicError` se o valor for objeto ou array. Não há `try/catch`.
- `reader.parse(data, payload)` recebe um `char*` que não termina em NUL (`:1629`).
- O tamanho do quadro vem do cliente e vai direto para `mg_malloc(data_len)`, sem limite.

**Problema:**
- **FATO:** uma única mensagem malformada, vinda de qualquer máquina da rede, chama `std::terminate` e mata o vic-engine. Exemplo: `{"type":{},"module":"x"}`.
- **FATO:** o parse lê além do buffer.

**Mudança sugerida:**
```cpp
if (dataLen > kMaxWsMessageBytes /* 4096 */) { return; }
Json::Value payload; Json::Reader reader;
try {
  if (!reader.parse(data, data + dataLen, payload)) { return; }
  if (!payload["type"].isString() || !payload["module"].isString()) { return; }
  OnReceiveWebSocket(conn, payload);
} catch (const std::exception& e) {
  LOG_WARNING("WebService.HandleWebSocketsData.BadJson", "%s", e.what());
}
```

**Como medir e validar no robô:** enviar a mensagem do exemplo com `websocat`. Antes da correção o engine morre; depois, só aparece o aviso no log.

### A6. Índice e ponteiro de conexão obsoletos

**Local:** `webServerProcess/src/webService.cpp:1227-1241`, `1664`, `1740-1752`.

**Comportamento atual (FATO):**
- Os pedidos guardam um índice no vetor `_webSocketConnections`. `OnCloseWebSocket` remove com troca e `pop`, mudando os índices.
- As lambdas enfileiradas guardam o `mg_connection*` cru. O civetweb reaproveita essa estrutura por thread (`civetweb.c:13626`).
- `OnCloseWebSocket` desreferencia `it` sem checar `end()` (`:1744-1751`).
- A fila de envio não é parada em `Stop()` (`:1274-1304`).

**Problema:**
- **FATO:** depois de um fechamento, uma resposta pode ir para outro cliente ou para a resposta HTTP de outra conexão.
- **FATO:** a desreferência de `end()` é comportamento indefinido. **HIPÓTESE:** é alcançável só se chegar um fechamento de conexão não registrada.
- A janela se abre a cada queda horária do A3.

**Mudança sugerida:** dar a cada conexão um `std::shared_ptr<ConnState>` com identificador crescente e um `std::atomic<bool> open`. Capturar o `shared_ptr` nas lambdas, marcar `open=false` no fechamento e checar `it != end()`.

**Como medir e validar no robô:** script de conexão e desconexão em laço com dois clientes; conferir que nenhum recebe dados do outro. Em simulador, build com ASan.

### A7. NavMap: mapa inteiro na thread principal

**Local:** `engine/navMap/mapComponent.cpp:62`, `70`, `311-314`, `346-371`, `375-384`, `755-803`; `engine/navMap/memoryMap/memoryMap.cpp:249-287`; `engine/navMap/quadTree/quadTree.cpp:32-34`.

**Comportamento atual (FATO):**
- O handler do pedido só faz `_webMessageDirty = true` (`:311-314`); o conteúdo do pedido é ignorado.
- No tick seguinte, na thread principal: `GetBroadcastInfo` percorre a quadtree inteira com um `std::function` por nó e enche dois vetores; `BroadcastMapToWeb` cria um `Json::Value` por quadrado (`:782-784`).
- O pedido web também dispara `BroadcastMapToViz`, porque `shouldSendViz` é verdadeiro quando `_webMessageDirty` é (`:346`, `:355-360`). O envio é descartado adiante se não há cliente viz, mas os vetores são montados.
- Vários pedidos no mesmo tick viram um só. Um cliente insistente recebe um mapa por tick (60 ms).
- Não existe delta em nenhum canal (web, viz ou SDK).
- Sem pedido, não há trabalho de mapa: o envio periódico só existe se a renderização ou o SDK ativarem.
- Estrutura: quadtree com folhas de 8 mm e raiz de até 2048 mm. Derivado: no máximo 65.536 folhas e 87.381 nós. Há um único mapa por vez (`kMergeOldMaps = false`, `:70`).
- A cada 5 s roda `TimeoutObjects`, uma varredura da árvore inteira na thread principal (`:492-528`).

**Problema:**
- **HIPÓTESE:** cada pedido custa dezenas de milhares de alocações pequenas na thread principal. A 200 ou 400 MHz isso estoura o tick de 60 ms.
- Cada pedido passa também pelo caminho do A1.
- **FATO:** os temporizadores `static` em `:357` e `:377` são inicializados uma única vez, e `:382` divide por `_broadcastRate_sec`. **HIPÓTESE:** uma frequência 0 vinda do SDK passa no teste `>= 0` e divide por zero.

**Mudança sugerida:**
- Limite de taxa no handler: no máximo um mapa a cada 2 s, e nenhum se o mapa não mudou desde o último envio (reaproveitar a flag `wasChanged`, `:394`).
- Separar a condição: pedido web não deve acionar `BroadcastMapToViz`.
- Montar a string direto de `quadInfo` com `snprintf`, sem `Json::Value`.
- Pose leve e delta: ver seção 3.1 e 3.2.

**Como medir e validar no robô:**
- PerfMetric durante um script que pede o mapa a 10 Hz: `http://<robô>:8888/perfmetric?start`, depois `stop` e `dumplogall`. Comparar o tempo de tick antes e depois.
- Tamanho da resposta antes e depois de tirar a indentação (parte dos 146 KB é espaço em branco; a proporção não foi medida).

### A8. Datagrama do gateway sem checagem de tamanho

**Local:** `engine/cozmoAPI/comms/localUdpSocketComms.cpp:128-157`; `engine/cozmoAPI/comms/protoMessageHandler.cpp:84-121`, `188-192`, `215-219`; `engine/cozmoEngine.cpp:425-430`.

**Comportamento atual (FATO):**
- `RecvMessageInternal` recebe em um buffer de 2048 bytes criado com `reserve()` (não `resize()`), e confia no cabeçalho de tamanho `uint16` sem compará-lo com o que foi lido.
- `ProtoMessageHandler::Update` devolve `RESULT_FAIL` se qualquer datagrama falhar no parse, e o engine propaga isso até sair.
- `DeliverToExternal` ignora o resultado de `SerializeToArray` e usa `ByteSizeLong()` como tamanho, com buffer de 2048.

**Problema:**
- **FATO:** uma mensagem do gateway maior que 2046 bytes é truncada, causa leitura fora do buffer, falha de parse e saída do engine.
- **FATO:** uma mensagem de saída maior que 2048 bytes envia lixo da pilha.

**Mudança sugerida:** checar `msgSize + 2 <= dataLen`, descartar o pacote ruim com um aviso em vez de falhar o tick, e checar o retorno de `SerializeToArray`.

**Como medir e validar no robô:** procurar `ProtoMessageHandler.MessageBufferParseFailed` nos logs das noites de falha. Teste: enviar pelo SDK uma mensagem grande (por exemplo um `SayText` com texto de 3 KB).

### A9. Tick lento não é registrado; sem telemetria de memória

**Local:** `engine/tools/engined/cozmoEngineMain.cpp:381-409`; `platform/common/diagnosticDefines.h:15-19`; `animProcess/src/cozmoAnim/cozmoAnimMain.cpp:176-245`; `engine/components/robotHealthReporter.cpp:140-156`.

**Comportamento atual (FATO):**
- Tick do engine: 60 ms (`robot/include/anki/cozmo/shared/cozmoEngineConfig.h:77`). Tick do anim: 33 ms (`robot/include/anki/cozmo/shared/cozmoConfig.h:344`). Ambos dormem o tempo restante, sem espera ocupada.
- Os avisos de estouro (`CozmoEngineMain.main.overtime`, `CozmoAnimMain.overtime`) estão dentro de `#if ENABLE_TICK_TIME_WARNINGS`, que exige `ANKI_PROFILING_ENABLED`. No build de release não sai nada.
- Não há watchdog interno nem `abort` em tick lento.
- `DEV_ASSERT` é compilado fora em release; `ANKI_VERIFY` e `PRINT_NAMED_ERROR` só registram, não abortam (`lib/util/source/anki/util/logging/logging.h:416-420`, `474-505`).
- Não há leitura de `VmRSS` nem `mallinfo` no engine. O único controle de memória é do sistema inteiro, uma vez por segundo, com evento só na mudança de nível.

**Problema:** um engine lento ou crescendo não deixa rastro no log. Foi por isso que as duas noites não deixaram evidência.

**Mudança sugerida:**
```cpp
// uma vez a cada 60 s, na thread principal do engine e do anim
static void LogHealth(int64_t maxTick_ms, size_t wsQueueDepth) {
  long rssPages = 0, dummy = 0;
  if (FILE* f = fopen("/proc/self/statm", "r")) { fscanf(f, "%ld %ld", &dummy, &rssPages); fclose(f); }
  LOG_INFO("Health.Periodic", "rss_kb=%ld max_tick_ms=%lld ws_queue=%zu",
           rssPages * 4, (long long)maxTick_ms, wsQueueDepth);
}
// e, sempre ligado, com limite de 1 linha a cada 10 s:
if (tickDuration_ms > 120) { LOG_WARNING("Engine.Tick.Slow", "%lld ms", (long long)tickDuration_ms); }
```

**Como medir e validar no robô:** `grep Health.Periodic /var/log/messages` depois de uma noite. O custo é uma leitura de arquivo por minuto.

### A10. Controle SDK sem timeout e sem dono

**Local:** `engine/components/sdkComponent.cpp:380`, `392-393`, `407-416`; `cloud/cloud/message_handler.go:1190-1194`, `1223`, `1247`, `1286-1303`; `cloud/cloud/gateway.go:207-211`.

**Comportamento atual (FATO):**
- `_sdkWantsControl` só é limpo por uma mensagem `ControlRelease`. Não há timeout nem keep-alive no componente.
- O RPC unário `AssumeBehaviorControl` nunca manda `ControlRelease` quando o fluxo termina. Só o RPC bidirecional `BehaviorControl` tem liberação adiada.
- O servidor gRPC não configura parâmetros de keepalive.
- Um segundo `ControlRequest` sobrescreve o dono sem avisar o primeiro. Um `ControlRelease` de qualquer conexão libera o controle, sem checar o identificador.
- SDK querendo controle é motivo de "sempre acordar" para o ciclo de sono (`engine/aiComponent/behaviorComponent/behaviors/sleeping/behaviorSleepCycle.cpp:143-151`).

**Problema:**
- **HIPÓTESE:** se o bridge some sem FIN nem RST, o robô fica parado em `SDKDefault` até o TCP desistir (minutos), ou até o engine reiniciar no caso do RPC unário.
- **FATO:** um lease mantido a noite toda impede o robô de dormir.
- **FATO:** qualquer outro cliente SDK quebra o lease do Redis em silêncio.

**Mudança sugerida:**
- No bridge, já: usar o RPC bidirecional `BehaviorControl` e soltar o controle à noite.
- No gateway: `grpc.KeepaliveParams` com `Time` de 5 s e `Timeout` de 5 s; liberação adiada em `AssumeBehaviorControl`.
- No engine: `if (id != _sdkControlConnId) return;` no release, e `ControlLost` para o dono anterior.

**Como medir e validar no robô:** pegar o controle, bloquear o bridge com `iptables -A INPUT -s <ip-do-bridge> -j DROP` e cronometrar até aparecer "Releasing SDK control" no log.

### A11. Temperatura: aviso só a 90 °C e trabalho contínuo na base

**Local:** `animProcess/src/cozmoAnim/animation/animationStreamer.cpp:185-197`, `1244-1255`; `osState/osState_vicos.cpp:108`, `362-381`; `engine/components/powerStateManager.cpp:38-64`, `79-91`, `173-205`, `380-414`; `cannedAnimLib/proceduralFace/proceduralFaceDrawer.cpp:900-912`; `animProcess/src/cozmoAnim/micData/micDataProcessor.cpp:565-573`, `646-657`.

**Comportamento atual (FATO):**
- O número na tela é um texto vermelho "<N>C" desenhado quando a zona térmica 3 chega a 90 °C. Só existe em builds com `ANKI_DEV_CHEATS`.
- Nenhum código deste repositório reduz carga ou desliga por temperatura da CPU. O único outro consumidor é a condição `ConditionHighTemperature`, também a 90 °C.
- O kernel tem pontos passivos em 75, 80 e 85 °C (`wire-os:kernel/msm-4.9/arch/arm64/boot/dts/qcom/msm8909-thermal.dtsi:123,163,185`). Não há thermal-engine no wire-os.
- O power save só é pedido por `BehaviorSleeping`, pelo modo quieto, por dois outros comportamentos e pelo módulo `power` do webviz. Não existe condição "na base".
- Em power save: calm mode, visão pausada, sensor da câmera em PWDN, LCD mais escuro, CPU a 200 ou 400 MHz, prox fora do mapa.
- Fora do power save, mesmo parado na base:
  - o rosto é renderizado e enviado por SPI a cada 33 ms, porque `ApplyNoise` sempre marca o quadro como alterado;
  - o beamforming de 4 microfones roda em todo bloco de 10 ms;
  - a visão roda com os modos que os comportamentos pedirem. `Faces` roda em todo quadro em qualquer nível de pedido (`resources/config/engine/visionScheduleMediator_config.json`).
- O mesmo trecho desenha um quadrado amarelo ou vermelho com os MB livres sob pressão de memória (`animationStreamer.cpp:1268-1282`).

**Problema:**
- **FATO:** ver o número significa que a CPU já chegou a 90 °C. É indicador, não proteção.
- **HIPÓTESE:** a 90 °C o kernel limita a frequência, os ticks estouram, as filas de IPC enchem e o A4 dispara.
- **HIPÓTESE:** o laço ocupado do A2 é uma fonte direta de calor com cliente webviz parado.

**Mudança sugerida:**
- Ação a 80 °C: forçar a frequência baixa com `SetDesiredCPUFrequency` (já existe), reduzir a visão e a taxa do rosto.
- No anim: não reenviar quadro idêntico.
```cpp
// AnimationStreamer::BufferFaceToSend
if (!_forceRedraw && memcmp(img.GetRawDataPointer(), _lastSent.GetRawDataPointer(), bytes) == 0) { return; }
```
- Modo noturno: ver seção 3.4.

**Como medir e validar no robô:**
```
while :; do date +%T; cat /sys/class/thermal/thermal_zone*/temp \
  /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq; sleep 10; done
top -H -b -d 5 -p $(pidof vic-anim),$(pidof vic-engine)
```
Experimento sem código: variável de console `ProcFace_NoiseNumFrames=0` (`proceduralFaceDrawer.cpp:77`) e comparar a CPU da thread "DrawFaceLoop".

### A12. `vic-robot`: quatro defeitos pontuais

**Local e comportamento atual (FATO):**
1. `robot/hal/src/hal.cpp:198-203`, `223`: `static u8 selectTimeoutCount` só é incrementado. Cinco timeouts de 1 s somados em toda a vida do processo, mesmo com dias de intervalo, encerram o `vic-robot` com o código 898.
2. `robot/supervisor/src/messages.cpp:273`: `while (RobotInterface::SendMessage(syncRobotAckMsg) == false);` é um laço sem saída em prioridade `SCHED_FIFO` máxima. Uma falha de envio desconecta o rádio (`robot/hal/src/radio.cpp:79-83`) e o laço nunca termina. **HIPÓTESE:** exige falha de envio no momento da sincronização.
3. `robot/hal/spine/spine.c:154-165`: `spine_receive_data` zera `rx_cursor` mas ainda copia para o deslocamento antigo, escrevendo além de `buf_rx`. Exige mais de 4 KB de dados pendentes sem parse.
4. `robot/hal/src/main.cpp:90`: `mlockall(MCL_FUTURE)` não trava as páginas já mapeadas, ao contrário do que diz o comentário.

Outros fatos do laço: não há espera ocupada no caminho normal; `HAL::Step` bloqueia em `select()` com timeout de 1 s (`hal.cpp:216`, `250`). O processo roda em `SCHED_FIFO` máximo sem checar o retorno (`main.cpp:176-178`), e a thread da IMU herda essa prioridade.

**Mudança sugerida:**
1. `selectTimeoutCount = 0;` antes do `return true` em `hal.cpp:235`.
2. Tentar de novo no próximo tick em vez de girar.
3. `next_offset = 0` depois de zerar o cursor.
4. `mlockall(MCL_CURRENT | MCL_FUTURE)`.

**Como medir e validar no robô:** `journalctl | grep check_spine_readable.timeout` e contar ocorrências desde o boot. `top -H -p $(pidof vic-robot)`.

### A13. BehaviorID ou classe desconhecidos

**Local:** `engine/aiComponent/behaviorComponent/behaviorTypesWrapper.cpp:27-36`; `engine/robotDataLoader.cpp:426-454`; `victor-clad/tools/message-buffers/emitters/CPP_emitter.py:345-358`; `engine/aiComponent/behaviorComponent/behaviorFactory.cpp:190-1241`.

**Comportamento atual (FATO):**
- `BehaviorIDFromString` devolve um `BehaviorID` não inicializado quando a busca falha; `ANKI_VERIFY` só registra.
- Esse valor é usado como chave em `RobotDataLoader::_behaviors`; a colisão só é pega por `DEV_ASSERT`, que não existe em release.
- Uma `behaviorClass` desconhecida passa pelo `FromString` do CLAD, que em `NDEBUG` devolve o primeiro valor do enum.
- `BehaviorID` e `BehaviorClass` são arquivos CLAD versionados, gerados à mão por `tools/ai/generateBehaviorCode.py`. Não há geração no build.
- Os JSON são lidos uma única vez no boot, só da pasta de recursos. Não existe caminho de carga em tempo de execução nem pasta de sobrescrita.
- Comportamentos anônimos não precisam de ID: qualquer JSON pode declarar `"anonymousBehaviors"` (`engine/aiComponent/behaviorComponent/behaviors/iCozmoBehavior.cpp:502-567`).

**Problema:**
- **HIPÓTESE:** foi isso que a tentativa de "reação a pessoa só por JSON" encontrou. O arquivo novo ou virou alias de um ID existente, ou gerou um ID que nenhum dispatcher referencia.
- Um JSON errado não produz um erro claro; produz um robô com árvore de comportamentos alterada.

**Mudança sugerida:** inicializar `BehaviorID id = BEHAVIOR_ID(Anonymous)` e, em `LoadBehaviors`, pular o arquivo com um erro explícito quando a conversão falhar.

**Como medir e validar no robô:** instalar um JSON com ID inexistente e conferir que sai uma linha de erro e a árvore segue normal.

### A14. `RobotState` a 16,7 Hz sem limite

**Local:** `engine/robotManager.cpp:192-195`; `engine/robot.cpp:2291-2301`; `engine/cozmoAPI/comms/protoMessageHandler.cpp:84-101`; `robot/include/anki/cozmo/shared/cozmoConfig.h:393-396`.

**Comportamento atual (FATO):**
- A cada tick o engine aloca um `RobotState` protobuf com submensagens e o envia ao gateway, além do `RobotState` CLAD.
- Não há limite de taxa nem teste de "alguém está ouvindo" antes de serializar. O descarte só ocorre no envio, se não houver cliente do gateway.
- `DeliverToExternal` faz cópia profunda de cada mensagem e monta um pacote de 2 KB.
- Em calm mode o processo do robô manda estado a cada 50 ticks, então o engine reenvia o mesmo estado a 16,7 Hz.

**Problema:** trabalho constante e alocação por tick, sem relação com mudança de estado. É o canal de pose que o bridge deveria usar, mas é mais caro que o necessário.

**Mudança sugerida:** enviar quando o estado mudar ou a cada 100 ms acordado e 1 s em calm mode ou na base; reutilizar um objeto membro em vez de `new`.

**Como medir e validar no robô:** contador de mensagens ao gateway no PerfMetric antes e depois.

### A15. Alocações por tick em comportamentos e microfone

**Local e comportamento atual (FATO):**
- `engine/aiComponent/behaviorComponent/behaviorSystemManager.cpp:187-207` e `behaviorStack.cpp:230-250`: a cada tick é reconstruído um `std::set<IBehavior*>` com tudo que está em escopo ativável. O próprio comentário diz "This is inefficient and should be replaced".
- `animProcess/src/cozmoAnim/micData/micDataProcessor.cpp:449-465`: cada bloco de 10 ms faz `make_unique` de uma mensagem `MicDirection`. Somando `MicDataState` e `AnimationState`, são cerca de 145 mensagens por segundo ao engine, mesmo ocioso.
- `engine/vision/visionSystem.cpp:281`: `OverheadMap` de 1000x1000 é alocado no início; o único uso visto está comentado como experimental (`engine/components/visionComponent.cpp:610`). Estimativa: cerca de 4 MB.
- `engine/aiComponent/behaviorComponent/behaviorComponentMessageHandler.cpp:329-345`: cada `ExecuteBehaviorByID` cria um comportamento novo que não é removido.

**Problema:** custo fixo pequeno, mas permanente, em um processador limitado. Não explica o crash.

**Mudança sugerida:** guardar o conjunto em cache usando a flag `_behaviorStackDirty` (`behaviorStack.cpp:226`); publicar `MicDirection` a 10 Hz a partir de um slot pré-alocado; remover o `OverheadMap` se confirmado sem uso.

**Como medir e validar no robô:** `perf top -p $(pidof vic-engine)` ou PerfMetric antes e depois.

### A16. Portas de depuração sem autenticação; segredos versionados

**Comportamento atual (FATO):**
- As portas 8888 e 8889 escutam em todas as interfaces, sem autenticação e com CORS `*`. As opções de senha estão comentadas (`webServerProcess/src/webService.cpp:960-985`).
- `/consolefunccall` e `/consolevarset` executam qualquer função ou variável de console (`:1006-1010`).
- `/systemctl?proc=X&<ação>` executa `/bin/systemctl` com argumentos não validados, em todo build que não seja simulador (`:765-798`).
- A listagem de diretórios está ligada para `persistent`, `cache` e `resources`.
- O firewall abre 8887-8890, 22, 5555 e outras em builds eng e engoskr (`wire-os:poky/victor/meta-vicos/recipes-connectivity/iptables-persistent/files/iptables-eng.rules:53`, `iptables-engoskr.rules:70`). Em builds user só 443 e 8080.
- Arquivos de chave versionados (somente caminhos; conteúdo não exibido):
  - `robot_sshkey` (chave privada RSA; listada no `.gitignore:421` mas rastreada; referenciada em `project/victor/scripts/backtracewatcher.py:12`);
  - `robot/syscon/tools/victorSysConSigningKeyPvt.pem`;
  - `robot/syscon/tools/development.pem`;
  - `project/victor/ota_test.key`;
  - `platform/config/etc/ota.pas`;
  - um token de upload embutido em URL em `platform/config/bin/vic-crashuploader` e `platform/config/*/etc/vic-crashuploader.env`.

**Problema:** qualquer máquina da rede local pode derrubar o engine (A5), parar serviços ou executar funções de console.

**Mudança sugerida:** escutar em `127.0.0.1` por padrão e expor por túnel SSH; ou restringir no firewall ao IP do bridge. Condicionar `/systemctl` e os endpoints de console a build de desenvolvimento. Tratar as chaves versionadas como comprometidas.

**Como medir e validar no robô:** `curl http://<robô>:8888/consolevarlist` de outra máquina, antes e depois.

### A17. Texto igual para 914 e 915; pouco é gravado ao cair

**Local:** `platform/faultCodeDisplay/faultCodeDisplay.cpp:27-36`, `66-76`; `platform/config/bin/vic-on-exit:36-64`; `platform/victorCrashReports/google_breakpad.cpp:81-136`, `205-217`; `platform/config/bin/vic-crashuploader:15`.

**Comportamento atual (FATO):**
- 914 (o systemd viu o engine sair mal) e 915 (o anim ficou 5 s sem o engine) mostram o mesmo texto, "vic-engine crashed.".
- "Restarts exhausted." é só um sufixo de texto quando o programa roda sem `-r`.
- Os minidumps vão para `/data/data/com.anki.victor/cache/crashDumps/`. O nome leva a hora de início do processo, não a do crash.
- O `.dmp.log` só é gravado se `/run/anki-crash-log` existir.
- `SIGKILL` (morte por falta de memória) e saída limpa não deixam minidump.
- O `vic-crashuploader` está desativado por um `exit 0` na linha 15, então nada limpa a pasta de dumps.
- Os logs vão para `/var/log/messages`, que é perdido no reboot.

**Problema:** depois de uma noite ruim sobra só a tela, que não distingue crash real, travamento, saída deliberada e morte por memória.

**Mudança sugerida:** textos distintos ("engine parou de responder" para 915); o handler de falhas deve gravar código, contagem, `/proc/meminfo` e o fim do log em `/data` antes de desenhar (ver seção 5).

**Como medir e validar no robô:** provocar cada caso (`kill -SEGV`, `kill -STOP`, `kill -KILL` no engine) e conferir tela e arquivos gerados.

---

## 2. Comece por aqui

1. **Hoje, sem tocar no robô (30 min): keepalive e reconexão educada no vector-bridge.**
   - Enviar `{"keepalive":true}` a cada 30 s em cada websocket (A3).
   - Ao reconectar, esperar o engine estabilizar (por exemplo 30 s após a porta voltar) e assinar os módulos com 100 ms entre eles, não em rajada (reduz a janela do A1).
   - À noite, fechar as conexões e soltar o controle SDK (A10).
2. **Na próxima falha, antes de reiniciar (10 min): rodar o roteiro de coleta da seção 4.2.** Ele decide entre deadlock, saída deliberada, morte por memória e crash real.
3. **Primeira correção no robô (2 h): A1, a inversão de locks.** É a causa mais provável e a correção é pequena e isolada em `webService.cpp`.
4. **Segunda correção (meio dia): A5 e A3 no servidor.** Validação da entrada, limite de conexões e timeout curto, no mesmo arquivo.
5. **Terceira correção (1 a 2 dias): A2 e A4.** Fila com limite e fim do laço ocupado; `EAGAIN` deixa de derrubar o engine. Junto, a linha de saúde periódica do A9 para ter dados nas noites seguintes.

---

## 3. Extensões para a arquitetura do dono

### 3.1 Canal leve de pose e estado

Três opções, da mais barata para a mais completa.

**Opção A: usar o que já existe (sem código no robô).** O engine já envia `RobotState` protobuf ao gateway a cada tick, com pose e origem (`engine/robotManager.cpp:192-195`). O bridge pode consumir o fluxo de eventos do SDK e reduzir a taxa do lado dele. Custo no robô: o que já é gasto hoje.

**Opção B: módulo webviz `pose` (cerca de 30 linhas).** Ao lado de `MapComponent::UpdateRobotPose()` (`engine/navMap/mapComponent.cpp:389`), com os mesmos sete campos que hoje vão no fim do mapa (`:793-800`):
```cpp
static const float kPosePeriod_s = 0.2f;   // 5 Hz
if (now_s - _lastPoseWeb_s >= kPosePeriod_s) {
  if (auto sender = WebService::WebVizSender::CreateWebVizSender("pose", webService)) {
    Json::Value& d = sender->Data();
    d["x"] = pose.x; d["y"] = pose.y; d["z"] = pose.z;
    d["qW"] = q.w; d["qX"] = q.x; d["qY"] = q.y; d["qZ"] = q.z;
    d["originId"] = originId; d["onCharger"] = onCharger; d["t"] = now_s;
  }
  _lastPoseWeb_s = now_s;
}
```
Cada mensagem tem cerca de 150 bytes. Enviar só quando a pose mudar além de 8 mm ou 2 graus reduz ainda mais.

**Opção C: endpoint HTTP `/getpose` sem a thread principal.** A thread principal grava um snapshot uma vez por tick; o handler HTTP lê a cópia sem passar por `ProcessRequest`, como já faz `GetPerfStats` (`webServerProcess/src/webService.cpp:609`). Assim um cliente abusivo nunca bloqueia o tick.
```cpp
struct PoseSnapshot { float x, y, z, qW, qX, qY, qZ; uint32_t originId; uint8_t flags; double t; };
std::mutex _poseMutex; PoseSnapshot _pose;           // escrito no tick, lido no handler
```

Recomendação: A agora, C como canal oficial.

### 3.2 Delta do mapa

- **FATO:** `QuadTreeProcessor::OnNodeContentChanged` já é chamado em toda mudança de conteúdo (`engine/navMap/memoryMap/memoryMap.cpp:151-157`), e já existe um `Fold` restrito a uma região (`engine/navMap/quadTree/quadTreeNode.cpp:303`).
- Esboço:
  1. Acumular ali uma caixa envolvente "suja" (4 floats).
  2. No envio, percorrer só essa região e emitir `{x, y, tamanho, conteúdo}` das folhas dentro dela.
  3. Enviar o mapa inteiro só na assinatura ou quando a origem mudar.
  4. Zerar a caixa após cada envio.
- Protocolo sugerido: `{"type":"data","module":"navmap","data":{"update":true,"mode":"delta","since":<seq>}}`, com resposta `{"seq":N,"full":false,"quads":[...],"pose":{...}}`. Se `since` não bater com o último `seq`, o servidor responde com `full:true`.

### 3.3 Proteção do engine contra clientes abusivos

Tudo em `webServerProcess/src/webService.cpp`:

| Limite | Onde | Valor sugerido |
|---|---|---|
| Websockets simultâneos | `HandleWebSocketsConnect` (`:1597-1600`) | 3, com `num_threads` 8 |
| Tamanho de mensagem de entrada | `HandleWebSocketsData` (`:1610`) | 4 KB |
| Mensagens de entrada por conexão | `OnReceiveWebSocket` | 20 por segundo |
| Bytes pendentes de saída por conexão | `SendToWebSocket` (`:1661`) | 256 KB, descartando o excesso |
| Pedidos pendentes | `AddRequest` (`:1307`) | 64 |
| Intervalo mínimo entre mapas | handler do navmap (`mapComponent.cpp:311`) | 2 s |
| Timeout de websocket ocioso | opção do civetweb (`:979`) | 120 s |

Complemento: um módulo `stats` que publique os contadores de descarte, para o bridge saber que está sendo limitado.

### 3.4 Modo noturno

**Hoje, sem código no robô:**
1. `AppIntent{intent:"intent_system_charger"}` e esperar o robô chegar à base.
2. `AppIntent{intent:"intent_system_sleep"}`.
3. Soltar o controle SDK e fechar as conexões de webviz.
4. De manhã, um `ControlRequest` acorda o robô (motivo de "sempre acordar" do SDK).

Fatos que sustentam o roteiro:
- `AppIntent` não exige controle de comportamento e vira intenção de usuário pendente (`engine/aiComponent/behaviorComponent/userIntentComponent.cpp:1151-1185`).
- `system_sleep` leva ao sono profundo (`engine/aiComponent/behaviorComponent/behaviors/sleeping/behaviorSleepCycle.cpp:558-563`).
- `BehaviorSleeping` pede power save (`behaviors/sleeping/behaviorSleeping.cpp:86-88`), que pausa a visão e a câmera, escurece o LCD e baixa a CPU.
- O microfone não é desligado; a palavra de ativação continua funcionando.

**Alternativa em build de desenvolvimento:** `{"type":"data","module":"power","data":{"enablePowerSave":"true"}}` no websocket do engine força o power save (`engine/components/powerStateManager.cpp:79-91`). Qualquer outro valor remove o pedido.

**Versão de produto (cerca de 60 linhas):** um componente que pede `PowerStateManager::RequestPowerSaveMode("NightMode")` quando o robô está na base, sem controle SDK e parado há N minutos ou com um sinalizador de noite vindo de fora; e remove o pedido ao acordar. Ressalva: power save também limita a CPU e liga o calm mode. Se o robô precisar responder rápido, usar só `TogglePowerSaveSetting(Camera)` e `ProxSensorNavMap`.

**Ganhos adicionais no anim:** não reenviar quadro idêntico (A11); reduzir `MicDirection`, `MicDataState` e `AnimationState` para 1 a 5 Hz em calm mode (A15).

**Atenção:** o wire-os tem um reinício de manutenção diário entre 01:00 e 05:00 (ver seção 5). Ele pode coincidir com o modo noturno.

### 3.5 Comportamentos acionados por fora

**Estágio 0, sem código.**
- Com o lease: `PlayAnimationTrigger` e `SayText` pelo SDK.
- Sem o lease: `AppIntent` com uma intenção existente, por exemplo `intent_imperative_lookatme` ou `intent_imperative_come`.
- Prova de ponta a ponta já no repositório: `intent_test_wire` aciona `BehaviorWireTest`, que fala "Hello World" (`resources/config/engine/behaviorComponent/behaviors/victorBehaviorTree/reactions/voiceFeatures.json:21`).
- **FATO:** `ExecuteBehaviorByID` pelo webviz substitui a árvore inteira e não volta sozinho (`engine/aiComponent/behaviorComponent/behaviorComponentMessageHandler.cpp:102-122`). É ferramenta de depuração, não gatilho.

**Estágio 1, uma recompilação pequena; depois, reações só por JSON (recomendado).**
1. Em `clad/src/clad/types/behaviorComponent/userIntent.clad`, adicionar `structure UserIntent_ExternalTrigger { string name }` e a entrada `UserIntent_ExternalTrigger external_trigger,` na união, antes de `test_SEPARATOR`.
2. Em `resources/config/engine/behaviorComponent/user_intent_map.json`:
   ```json
   {"user_intent":"external_trigger","app_intent":"intent_external_trigger","app_substitutions":{"param":"name"}}
   ```
3. Em `.../victorBehaviorTree/reactions/voiceFeatures.json`, comportamentos anônimos, que não precisam de BehaviorID novo:
   ```json
   "behaviors":["...", "ExtPersonSeen"],
   "anonymousBehaviors":[{
     "behaviorName":"ExtPersonSeen", "behaviorClass":"AnimSequence",
     "animTriggers":["GreetAfterLongTime"],
     "respondToUserIntents":[{"type":"external_trigger","name":"person_seen"}]
   }]
   ```
4. O bridge envia `AppIntent{intent:"intent_external_trigger", param:"person_seen"}`.

Cada reação nova passa a ser mais uma entrada anônima com outro `name`: cópia de recursos e reinício do engine, sem compilar.

Ressalvas (todas FATO):
- Um nome sem dono faz o robô dizer "I can't do that" depois de 3 ticks (`userIntentComponent.cpp:58`). Incluir um comportamento anônimo de menor prioridade que aceite só o tipo, sem fazer nada.
- A intenção substitui uma intenção de voz pendente e conta como comando de voz para acordar.
- Enquanto o comportamento SDK está ativo, intenções fora da lista permitida não são encaminhadas (`behaviors/sdkBehaviors/behaviorSDKInterface.cpp:312-316`). Enviar com o lease solto.
- Os nomes exatos das chaves de `behaviorModifiers` não foram verificados.

**Estágio 2, gatilho próprio sem efeitos de intenção (cerca de 150 linhas).**
- Transporte: reaproveitar `AppIntent`. Em `OnAppIntent`, antes da busca no mapa, interceptar o prefixo `"ext:"` e guardar `{nome, parâmetro, tick}` em um `ExternalTriggerComponent`.
- Condição nova `ExternalTrigger` em `clad/src/clad/types/behaviorComponent/beiConditionTypes.clad`, com caso em `engine/aiComponent/beiConditions/beiConditionFactory.cpp`:
  ```cpp
  bool ConditionExternalTrigger::AreConditionsMetInternal(BehaviorExternalInterface& bei) const {
    return bei.GetAIComponent().GetComponent<ExternalTriggerComponent>().IsPending(_name, _maxAge_s);
  }
  ```
- Árvore: um dispatcher novo `ExternalReactions` (`DispatcherStrictPriorityWithCooldown`, um BehaviorID novo) com filhos anônimos, cada um com `"wantsToBeActivatedCondition":{"conditionType":"ExternalTrigger","name":"person_seen","maxAge_s":2}`.
- Posição: em `.../victorBehaviorTree/globalInterruptions.json`, logo antes de `"OptionalReactions"`. Fica abaixo das reações físicas obrigatórias, do SDK, da palavra de ativação e dos comandos de voz, e acima do tempo livre.
- Resultado: nunca passa na frente de segurança ou voz, interrompe o tempo livre, fica inerte dormindo e com o SDK no controle, e não gera "intenção sem dono".

### 3.6 Ganchos para a visão offboard

- **FATO, já existe:** injeção de obstáculos pelo SDK com `CreateFixedCustomObject`, `DefineCustomBox` e `DeleteCustomObjects` (`tools/protobuf/gateway/public/external_interface.proto:444-452`; `engine/blockWorld/blockWorld.cpp:233-243`, `1231`). Eles entram no mapa como obstáculo (`engine/navMap/mapComponent.cpp:996-1052`). É o caminho pronto para obstáculos vistos pela câmera de teto.
- **FATO, não existe:** nenhuma mensagem externa injeta pessoa, rosto ou ponto saliente. O único chamador de `AddSalientPoints` é a visão embarcada (`engine/components/visionComponent.cpp:1211`). O modelo "offboard" existente envia a imagem do robô para fora; ele não aceita detecções de terceiros.
- Esboço de mensagem nova (cerca de 40 linhas no engine, mais proto e conversão no gateway):
  ```
  // clad/src/clad/externalInterface/messageGameToEngine.clad
  message InjectSalientPoint {
    uint_8  salientType,      // Person, Hand, ...
    float_32 x_img, float_32 y_img, float_32 score,
    uint_32 timestamp_ms
  }
  ```
  O handler chama `SalientPointsComponent::AddSalientPoints` (`engine/aiComponent/salientPointsComponent.cpp:46`). Comportamentos que usam `ConditionSalientPointDetected` passam a reagir sem a detecção de pessoas embarcada.
- Com isso, o modo `People` da visão pode ficar desligado, o que tira carga do robô.

### 3.7 Telemetria interna barata

- **Já existe:** PerfMetric por HTTP na porta 8888: `/perfmetric?start`, `stop`, `status`, `dumplog`, `dumplogall`, `dumpfiles` (`lib/util/source/anki/util/perfMetric/perfMetricImpl.h:129`, `602-647`). Cada quadro traz tempo de execução e de sono do tick, contagem de mensagens, frequência da CPU, bateria e comportamento ativo. Anel de 1000 quadros no engine e 2000 no anim.
- **Já existe, desligado:** módulo webviz `cpu` (variável de console `kWebvizUpdatePeriod`, `osState/osState_vicos.cpp:63`).
- **Compilado fora:** profiler de CPU. Exige recompilar com `-DANKI_PROFILING_ENABLED=1`, o que também liga os avisos de tick.
- **A adicionar:** a linha `Health.Periodic` do A9, e um endpoint `/gethealth` que devolva em JSON, sem a thread principal: RSS, pico de tick dos últimos 60 s, profundidade e descartes da fila de websocket, temperatura, frequência da CPU e contadores de descarte de IPC. O bridge publica isso no Redis uma vez por minuto.

---

## 4. Hipóteses abertas e roteiro de coleta

### 4.1 Hipóteses e o dado que confirma cada uma

| # | Hipótese | Dado que confirma | Dado que refuta |
|---|---|---|---|
| H1 | Deadlock do A1 na reconexão; o anim levanta 915 e o handler mata o engine com `SIGILL` | Log com "Handle fault code 915" e "kill -ILL vic-engine"; minidump com a thread principal em `pthread_mutex_lock` sob `WebService::Update` | Código 914 no log; minidump com falha de segmentação |
| H2 | Saída deliberada do engine por `EAGAIN` no IPC (A4) | `Engine has stopped` e `RobotConnectionManager.SendData.Error` no log; nenhum minidump | Minidump presente |
| H3 | Morte por falta de memória (fila do A2 ou logs em RAM) | `dmesg` com `lowmemorykiller` ou `Killed process` do vic-engine; RSS crescente no log por minuto | RSS estável até a queda |
| H4 | Exceção por mensagem malformada (A5) | Minidump com `std::terminate` e `Json::` na pilha | Sem `SIGABRT` |
| H5 | As quedas das 05:02, 06:03 e 07:03 foram só o timeout de 1 h, com os processos vivos | PIDs do vic-engine e do vic-anim iguais antes e depois de cada queda | PIDs diferentes; "Restart anki-robot.target" a cada hora |
| H6 | O evento da 01:38 foi o reinício de manutenção do wire-os, não um crash | Arquivo `/data/maintenance_reboot` com data daquela noite; tempo de atividade zerado à 01:38 | Tempo de atividade contínuo; "Handle fault code" à 01:38 |
| H7 | O robô roda configuração com `ON_FAULT_RESTART=0` (o primeiro crash já mostra "Restarts exhausted") | Conteúdo de `/anki/etc/fault-code-handler.env` | Valor 1 no arquivo |
| H8 | Calor vem do laço ocupado do A2 ou do trabalho contínuo na base (A11) | Thread "WebsocketSender" a 100 % em `top -H`; temperatura caindo sem cliente | Temperatura igual com e sem cliente |
| H9 | Renovação de DHCP ou varredura de Wi-Fi contribui para as quedas | `journalctl -u connman -u wpa_supplicant` com eventos nos mesmos minutos | Sem eventos de rede nos horários |

Não foi encontrado nenhum trabalho periódico de hora em hora nos processos do robô além do timeout do websocket. No wire-os, o único temporizador horário é o `update-engine.timer`, que dorme um tempo aleatório de até 3600 s antes de agir (`wire-os:poky/victor/meta-anki/recipes-anki-robot/files/update-engine.timer:9`), portanto não cai em minuto fixo. Não foi possível ler o código do daemon `wired` (diretório vazio no clone), que roda como root e mexe em frequências.

### 4.2 Na próxima falha, antes de reiniciar (10 min)

```sh
ssh root@<robô> '
  date; uptime
  cat /anki/etc/fault-code-handler.env
  cat /run/fault_restart_count /run/fault_restart_uptime 2>/dev/null
  ls -la /run/fault_code* /data/maintenance_reboot 2>/dev/null
  systemctl status vic-engine vic-anim vic-robot --no-pager -l
  journalctl -b --no-pager > /data/coleta-journal.txt
  dmesg > /data/coleta-dmesg.txt
  cp /var/log/messages /data/coleta-messages.txt
  grep -iE "oom|killed process|lowmemorykiller|thermal" /data/coleta-dmesg.txt
  cat /proc/meminfo; df /run /var/volatile
  cat /sys/class/thermal/thermal_zone*/temp
  sysctl net.unix.max_dgram_qlen
  ls -la /data/data/com.anki.victor/cache/crashDumps /data/fault-reports /data/panics
'
scp -r root@<robô>:/data/coleta-*.txt \
       root@<robô>:/data/data/com.anki.victor/cache/crashDumps .
```

Termos a procurar em `coleta-messages.txt`:
- `fault-code-handler`: "Handle fault code N", "Restart count x/2", "No more restarts allowed", "kill -ILL".
- `Engine has stopped`, `RobotConnectionManager.SendData.Error`, `LocalUdpClient.Send.Fail`.
- `DisconnectedFromEngine`, `ProtoMessageHandler.MessageBufferParseFailed`.
- `tombstone: caught signal`, `BOOM`, `GoogleBreakpad.DumpCallback`.
- `check_spine_readable.timeout`.

Os minidumps se abrem com `minidump_stackwalk` e os binários sem strip do mesmo build. Para os rastros "BOOM" existe `project/victor/scripts/backtracewatcher.py`.

### 4.3 Coleta contínua durante a noite (deixar rodando)

```sh
# no robô, grava em /data para sobreviver a reinícios
nohup sh -c 'while :; do
  ts=$(date +%F_%T)
  for p in vic-engine vic-anim vic-robot; do
    pid=$(pidof $p) || continue
    rss=$(grep VmRSS /proc/$pid/status | tr -s " " | cut -d" " -f2)
    thr=$(grep Threads /proc/$pid/status | cut -f2)
    echo "$ts $p pid=$pid rss_kb=$rss threads=$thr"
  done
  echo "$ts mem $(grep -E "MemAvailable|MemFree" /proc/meminfo | tr -s " " | tr "\n" " ")"
  echo "$ts temp $(cat /sys/class/thermal/thermal_zone3/temp) freq $(cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq)"
  echo "$ts ws $(ss -tn | grep -cE ":888[89]")"
  sleep 60
done >> /data/coleta-noite.log 2>&1' &
```

O que olhar:
- `pid=` muda em uma queda horária: o processo reiniciou (refuta H5).
- `rss_kb` do vic-engine sobe continuamente: confirma H3.
- `temp` e `freq` antes da queda: liga ou não ao calor (H8).

### 4.4 Experimentos controlados

| Noite | Configuração | O que decide |
|---|---|---|
| 1 | Zero conexões, robô na base | Linha de base de RSS, temperatura e estabilidade |
| 2 | Um cliente com 21 módulos e keepalive a cada 30 s | Se o keepalive elimina as quedas horárias; se o RSS cresce com assinatura longa |
| 3 | Mesmo cliente, sem keepalive, reconexão em rajada | Reproduz o cenário original com coleta ligada |

Durante o dia, três testes de 10 minutos cada:
1. Script de assinar 21 módulos, fechar e repetir (reproduz H1).
2. `kill -STOP` no cliente com os módulos assinados, observando `top -H` e RSS (reproduz H3 e H8).
3. `kill -STOP` de 2 s no vic-anim (reproduz H2).

---

## 5. Pontos de contato com o wire-os

### 5.1 Como a tela de erro é produzida (FATO)

1. Um serviço termina e roda `ExecStopPost=/anki/bin/vic-on-exit`.
2. Se o resultado não for `success`, o script grava o código do serviço em `/run/fault_code` (`platform/config/bin/vic-on-exit:36-64`). Códigos: engine 914, robot 916, anim 800, switchboard 913, cloud 923 (`platform/config/etc/vic-*.env`).
3. Um socket do systemd ativa `fault-code.service`, que roda `wire-os:anki/fault-code/fault-code-handler`.
4. O handler manda `SIGTERM` a todos os serviços de `anki-robot.target`, espera 3 s e manda `SIGQUIT`. Para 915 manda `kill -ILL` no vic-engine; para 917, no vic-anim.
5. Se o reinício for permitido: dorme `TIMEOUT_RESTART_SEC`, incrementa `/run/fault_restart_count`, grava a hora e roda `systemctl restart anki-robot.target`.
6. Limite: `FAULT_RESTART_LIMIT=2` e `FAULT_RESTART_LIMIT_SEC=60`. A contagem zera se passaram 60 s ou mais desde o último reinício. Com contagem 2, não reinicia mais e a tela mostra "Restarts exhausted.".

Consequências:
- **FATO:** todas as units `vic-*` têm `Restart=no`. O systemd nunca reinicia um serviço sozinho; quem reinicia é o handler, e ele derruba todos os processos juntos. Isso explica 8888 e 8889 caírem no mesmo instante em uma falha real.
- **FATO:** a configuração `epdev` tem `ON_FAULT_RESTART=0`; as demais têm 1 (`platform/config/*/etc/fault-code-handler.env`).
- **FATO:** depois de esgotar, nada tenta de novo. O robô fica parado até alguém intervir.

### 5.2 O que depende de mudança no wire-os

| # | Mudança | Onde | Motivo |
|---|---|---|---|
| W1 | Handler grava sempre, em `/data/fault-reports`, o fim do log, `dmesg`, `/proc/meminfo`, `ps` e o resultado do serviço, com limite de N relatórios | `anki/fault-code/fault-code-handler:296-325` | Hoje só grava com `ON_FAULT_UPLOAD_LOG=1` e `/run/das_allow_upload`; em oskr não grava nada |
| W2 | Após esgotar os reinícios, tentar de novo com espera longa (por exemplo 10 min) ou reiniciar o sistema | `fault-code-handler:248-279`, `370-388` | O robô não deveria ficar morto até de manhã |
| W3 | Histórico de contagem de reinícios em `/data` | `fault-code-handler` | `/run` é tmpfs e some no reboot |
| W4 | `OOMScoreAdjust` por unit: vic-robot e vic-anim negativos, serviços auxiliares positivos | units em `poky/victor/meta-anki/recipes/anki-robot/files/` | Nenhuma unit define hoje; todos os processos têm o mesmo peso |
| W5 | Habilitar `CONFIG_MEMCG` no kernel e então `MemoryHigh` e `MemoryMax` nas units | `defconfig:142` (`# CONFIG_MEMCG is not set`) | Sem MEMCG, limites de memória por unit não funcionam |
| W6 | Ativar swap em zram no boot (128 a 192 MB) | unit nova; `CONFIG_ZRAM=y` já está no `defconfig:1281` | Nada ativa swap hoje |
| W7 | Limitar os logs em RAM: `RuntimeMaxUse=16M` no journald, limite ou rotação de `/var/log/messages`, `size=` nos tmpfs | `meta-vicos/recipes-core/systemd/systemd-conf/journald.conf`; `syslog-ng.conf.systemd:34,39` | Journal de até 64 MB em `/run` e `messages` sem limite. **HIPÓTESE:** `/var/log` também é RAM |
| W8 | Definir `net.unix.max_dgram_qlen` (por exemplo 512) em um `sysctl.d` | arquivo novo | Reduz a chance do `EAGAIN` do A4. Não achei `sysctl.d` para este alvo |
| W9 | `MALLOC_ARENA_MAX=2` no ambiente do vic-engine | `platform/config/etc/vic-engine.env` (este repositório) e unit | Menos fragmentação com muitas threads |
| W10 | Core dumps em builds de desenvolvimento: `LimitCORE=infinity`, `core_pattern` em `/data/cores` com limpeza | units; `set_core_pattern.sh:3` | Hoje, na prática, não há core |
| W11 | Carregar `ramoops` para guardar o log do kernel entre reinícios | `ankiinit:68-69` (modprobe comentado) | Pós-morte de pânico do kernel |
| W12 | Firewall: tirar 8887-8890 e 5555 do `iptables-engoskr.rules`, ou restringir ao IP do bridge | `iptables-persistent/files/iptables-engoskr.rules:70` | Portas sem autenticação abertas para toda a rede. O arquivo IPv6 já as fecha |
| W13 | Reinício de manutenção: registrar a decisão no journal e respeitar um arquivo de inibição criado pelo bridge | `anki/rebooter/rebooter.sh:52-66`, `88-99` | Roda todo dia entre 01:00 e 05:00 se o tempo de atividade for de 4 h ou mais |
| W14 | Política térmica explícita abaixo de 85 °C | `ankiinit:6-17`; `msm8909-thermal.dtsi` | Não há thermal-engine; só os pontos passivos do kernel |
| W15 | `Nice` e classe de E/S ociosa na unit do `update-engine` | `update-engine.service` | Roda de hora em hora em horário aleatório |

Dependências entre os dois repositórios:
- W8 e a correção do A4 se complementam; o A4 é a correção real.
- W5 é pré-requisito para qualquer limite de memória por processo.
- W1 e o A17 juntos tornam cada falha diagnosticável sem acesso ao robô na hora.

### 5.3 Não verificado

- Código do daemon `wired` e dos diretórios `anki/victor`, `anki/wired` e `anki/vic-cloudless`, vazios no clone raso.
- Configuração efetiva de chrony, logrotate e sshd (camadas upstream ausentes no clone).
- Qual configuração (`oskr`, `development`, `epdev`) está instalada no robô do dono.
- Pontos críticos de desligamento térmico no device tree (só as primeiras 200 linhas foram lidas).
- Taxa de captura da câmera; limites de memória do álbum de rostos, do `imageSaver` e do `CameraFeed` do SDK.
- Internos do Alexa, da biblioteca Signal Essence e dos plugins do Wwise.
- Lado Go do gateway e do switchboard além dos trechos citados; internos do wire-pod (fora destes repositórios).
- Volume de DAS e de log gerado pelo engine.
