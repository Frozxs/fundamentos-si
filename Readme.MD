# Sistema de Mensageria e Tratamento de Exceções — Supermercado Autônomo

> **Desenvolvimento da Parte 4 — Mensageria e Exceções**  
> **Área:** Sistemas de Informação  
> **Abordagem:** Pseudocódigo estruturado  
> **Nível:** Técnico-acadêmico

---

## 1. Introdução

Este documento apresenta o desenvolvimento da **Parte 4 — Mensageria e Exceções** da arquitetura de um sistema de supermercado autônomo.

O sistema utiliza uma arquitetura **Edge-first**, na qual o processamento principal do terminal de autoatendimento ocorre localmente, reduzindo a dependência de conectividade externa para as operações que não exigem autorização síncrona.

A comunicação entre os terminais e os dispositivos dos funcionários é realizada por uma arquitetura de mensageria assíncrona e bidirecional, utilizando comunicação local de baixa latência e mecanismos de fallback pela infraestrutura de nuvem.

O desenvolvimento apresentado neste documento tem como objetivo transformar a arquitetura técnica fornecida em um **algoritmo-mãe**, capaz de representar:

- funcionamento normal do terminal;
- comunicação local;
- detecção de exceções;
- geração de alertas;
- comunicação com funcionários;
- execução de comandos de resolução;
- fallback pela nuvem;
- recuperação da comunicação;
- tratamento das cinco principais exceções especificadas;
- auditoria e rastreabilidade das ações.

O algoritmo é apresentado em **pseudocódigo estruturado**, não estando vinculado a uma linguagem de programação específica.

---

# 2. Objetivo

O objetivo principal é modelar o comportamento do sistema de mensageria e tratamento de exceções de maneira sistemática.

O algoritmo deve representar o ciclo completo:

```text
EVENTO
   ↓
DETECÇÃO
   ↓
CLASSIFICAÇÃO
   ↓
DECISÃO
   ↓
ALTERAÇÃO DE ESTADO
   ↓
GERAÇÃO DE MENSAGEM
   ↓
ROTEAMENTO
   ↓
FUNCIONÁRIO
   ↓
AÇÃO
   ↓
COMANDO
   ↓
TERMINAL
   ↓
VALIDAÇÃO
   ↓
RESOLUÇÃO
   ↓
AUDITORIA
```

A finalidade é garantir que uma exceção não seja tratada apenas como uma mensagem de alerta, mas como um **fluxo completo de controle do sistema**.

---

# 3. Escopo

O desenvolvimento está concentrado na **Parte 4 — Mensageria e Exceções**.

São contemplados:

1. Comunicação normal entre terminal e funcionário;
2. Comunicação local via Edge Gateway;
3. Fallback de comunicação pela nuvem;
4. Retorno da comunicação ao fluxo local;
5. Tratamento de exceções;
6. Comandos de resolução;
7. Controle de estados;
8. Auditoria;
9. Os cinco cenários de exceção definidos na arquitetura.

Os cinco cenários são:

1. **Divergência de Peso / Sweethearting**
2. **Restrição Etária**
3. **Queda de WAN Durante Transação PIX**
4. **Falha Crítica de Hardware**
5. **Abandono de Terminal com Itens**

---

# 4. Arquitetura Geral

## 4.1 Arquitetura Edge-first

Cada terminal de autoatendimento possui capacidade local de processamento.

A arquitetura é composta, entre outros elementos, por:

- Scanner;
- Balança;
- Câmera;
- Hardware Abstraction Layer — HAL;
- PDV Core;
- SQLite local com WAL;
- AWS IoT Greengrass v2;
- Edge MQTT Broker;
- Edge Gateway;
- Rede Wi-Fi corporativa;
- Dispositivo do funcionário;
- AWS IoT Core;
- Amazon EventBridge;
- Amazon SQS FIFO;
- Amazon Aurora PostgreSQL;
- Amazon DynamoDB.

A lógica principal do terminal não depende continuamente da internet.

Quando existe conectividade local, o terminal pode se comunicar diretamente com o dispositivo do funcionário.

Quando a comunicação local não está disponível, o sistema utiliza a infraestrutura de nuvem como mecanismo de fallback.

---

# 5. Componentes Envolvidos na Mensageria

## 5.1 Terminal de Autoatendimento

É responsável por:

- detectar eventos;
- processar sensores;
- executar regras de negócio;
- alterar seus estados;
- gerar alertas;
- publicar mensagens;
- receber comandos;
- executar comandos;
- registrar eventos de auditoria.

## 5.2 Edge MQTT Broker

Responsável pelo transporte das mensagens MQTT dentro da infraestrutura de borda.

Recebe mensagens publicadas pelo terminal e permite o encaminhamento para:

- Edge Gateway;
- infraestrutura de sincronização;
- demais componentes autorizados.

## 5.3 Local WebSocket Gateway

É o gateway responsável pela comunicação de baixa latência com os dispositivos dos funcionários.

A comunicação local utiliza:

```text
WebSocket + TLS + mTLS
```

O objetivo é permitir que alertas cheguem ao funcionário mesmo quando a conexão com a internet estiver indisponível.

## 5.4 Dispositivo do Funcionário

Pode ser:

- tablet industrial;
- smartwatch;
- outro dispositivo corporativo autorizado.

O dispositivo recebe:

- alertas;
- informações da exceção;
- dados do terminal;
- informações necessárias para tomada de decisão.

Também pode enviar comandos de resolução.

## 5.5 AWS IoT Core

É utilizado como broker MQTT gerenciado na nuvem.

Também participa do mecanismo de fallback e da sincronização entre a infraestrutura local e a nuvem.

## 5.6 Cloud Bridge

Representa a ponte de comunicação entre a infraestrutura de borda e a infraestrutura de nuvem.

No fallback, permite que os eventos do terminal cheguem à infraestrutura de nuvem.

Na consolidação deste algoritmo, também é utilizado como caminho de retorno para comandos destinados ao terminal quando o funcionário estiver fora da rede local.

## 5.7 FCM e APNs

São utilizados para notificações push quando o dispositivo do funcionário não está conectado diretamente à infraestrutura local.

```text
AWS IoT Core
      ↓
Amazon SNS
      ↓
FCM / APNs
      ↓
Dispositivo do Funcionário
```

---

# 6. Modelo de Estados

Para evitar ambiguidades, são considerados separadamente:

1. **Estado do terminal**
2. **Estado da comunicação**
3. **Estado da exceção**

Esses estados não devem ser tratados como se fossem a mesma coisa.

## 6.1 Estados Principais do Terminal

```text
IDLE
EVALUATING
ITEM_ACCEPTED
READY_FOR_NEXT
LOCKED_EXCEPTION
WAITING_PIX_SETTLEMENT
PAYMENT_AMBIGUOUS_STATE
OUT_OF_SERVICE
```

## 6.2 Estados de Comunicação

```text
LOCAL
OFFLINE
FALLBACK_CLOUD
RECOVERING
```

Esses estados representam a disponibilidade do caminho de comunicação, e não o funcionamento do terminal.

Por exemplo:

```text
Terminal = LOCKED_EXCEPTION
Comunicação = LOCAL
```

ou:

```text
Terminal = WAITING_PIX_SETTLEMENT
Comunicação = FALLBACK_CLOUD
```

## 6.3 Estados de Exceção

Uma exceção pode estar:

```text
DETECTADA
ALERTADA
EM_ATENDIMENTO
COMANDO_ENVIADO
COMANDO_EXECUTADO
RESOLVIDA
REGISTRADA
```

A existência de uma mensagem enviada não significa que a exceção já foi resolvida.

---

# 7. Máquina de Estados do Terminal

O fluxo básico de operação é:

```text
              ItemScanned
[IDLE] ─────────────────────────> [EVALUATING]
                                      │
                       ┌──────────────┴──────────────┐
                       │                             │
                  Validação OK               Divergência
                       │                             │
                       ▼                             ▼
                [ITEM_ACCEPTED]             [LOCKED_EXCEPTION]
                       │                             │
                       ▼                             │
                [READY_FOR_NEXT] <──────────────────┘
```

Uma exceção pode interromper o fluxo normal e colocar o terminal em estado de bloqueio.

---

# 8. Comunicação Normal

## 8.1 Fluxo Terminal → Funcionário

O fluxo primário é:

```text
Terminal
   ↓
MQTT Publish
   ↓
Edge MQTT Broker
   ↓
Local WebSocket Gateway
   ↓
WebSocket + TLS
   ↓
Tablet / Smartwatch
```

Exemplo de tópico:

```text
store/{id}/terminal/{id}/alerts
```

## 8.2 Fluxo Funcionário → Terminal

Após analisar o alerta, o funcionário executa uma ação no aplicativo.

No fluxo local:

```text
Funcionário
   ↓
Aplicativo
   ↓
HTTP/3 + mTLS
   ↓
Local WebSocket Gateway
   ↓
MQTT
   ↓
Edge MQTT Broker
   ↓
Terminal
```

O comando é então processado pelo terminal.

---

# 9. Fallback pela Nuvem

Quando o funcionário estiver fora do alcance da rede Wi-Fi local, o alerta pode seguir o caminho:

```text
Terminal
   ↓
Edge MQTT Broker
   ↓
AWS IoT Core
   ↓
Amazon SNS
   ↓
FCM / APNs
   ↓
Rede 4G/5G
   ↓
Dispositivo do Funcionário
```

Esse fluxo permite que o funcionário receba a notificação mesmo sem conexão direta com a rede local da loja.

---

# 10. Retorno do Comando pelo Fallback

Para completar o fluxo bidirecional quando o funcionário estiver fora da rede local, este desenvolvimento adota o seguinte caminho:

```text
Funcionário
   ↓
AWS IoT Core
   ↓
Cloud Bridge
   ↓
Infraestrutura Local
   ↓
Edge MQTT Broker
   ↓
Terminal
```

Essa é uma **consolidação arquitetural** utilizada para fechar a comunicação de retorno no cenário em que o alerta chega ao funcionário pela nuvem.

A ideia central é:

```text
ALERTA:
Terminal → Nuvem → Funcionário

COMANDO:
Funcionário → Nuvem → Edge → Terminal
```

---

# 11. Regra de Separação entre Alerta e Comando

O sistema deve distinguir:

### Alerta

Informa que algo aconteceu.

Exemplo:

```text
THEFT_PREVENTION_TRIGGER
```

### Comando

Representa uma ação solicitada pelo funcionário.

Exemplo:

```text
RELEASE_TERMINAL
```

Portanto:

```text
ALERTA ≠ COMANDO
```

Da mesma forma:

```text
COMANDO ENVIADO ≠ COMANDO EXECUTADO
```

e:

```text
COMANDO EXECUTADO ≠ EXCEÇÃO RESOLVIDA
```

---

# 12. Algoritmo-Mãe — Visão Geral

O algoritmo-mãe representa o comportamento geral do sistema.

```text
ALGORITMO SistemaMensageriaExcecoes

INÍCIO

    InicializarTerminal()
    InicializarSensores()
    InicializarBancoLocal()
    InicializarBroker()
    InicializarGateway()
    InicializarMonitoramentoDeConectividade()

    estadoTerminal ← IDLE
    estadoComunicacao ← LOCAL

    ENQUANTO sistema_ativo = VERDADEIRO FAÇA

        MonitorarConectividade()

        SE comunicacao_local_disponivel() ENTÃO
            estadoComunicacao ← LOCAL

        SENÃO SE comunicacao_cloud_disponivel() ENTÃO
            estadoComunicacao ← FALLBACK_CLOUD

        SENÃO
            estadoComunicacao ← OFFLINE
        FIM_SE

        evento ← AguardarEvento()

        SE evento = NENHUM_EVENTO ENTÃO
            ContinuarMonitoramento()
            CONTINUE
        FIM_SE

        excecao ← DetectarExcecao(evento)

        SE excecao = NENHUMA ENTÃO

            ProcessarOperacaoNormal(evento)

        SENÃO

            estadoTerminal ← DeterminarEstadoDeExcecao(excecao)

            mensagem ← GerarAlerta(excecao)

            RegistrarEventoLocal(excecao, mensagem)

            EntregarAlerta(mensagem, estadoComunicacao)

            resposta ← AguardarAcaoDoFuncionario()

            SE resposta = COMANDO_VALIDO ENTÃO

                ValidarPermissao(resposta)

                EnviarComando(resposta)

                resultado ← AguardarConfirmacaoDeExecucao()

                SE resultado = EXECUTADO ENTÃO

                    ValidarResolucao()

                    SE excecao_resolvida() ENTÃO

                        RegistrarResolucao()

                        RestaurarEstadoTerminal()

                    FIM_SE

                SENÃO

                    RegistrarFalhaDoComando()

                    ManterExcecaoAtiva()

                FIM_SE

            FIM_SE

        FIM_SE

        SE estadoComunicacao = OFFLINE ENTÃO
            ArmazenarEventoParaSincronizacao()
        FIM_SE

        SE conectividade_foi_restabelecida() ENTÃO

            estadoComunicacao ← RECOVERING

            ExecutarSincronizacao()

            ExecutarDeduplicacao()

            ExecutarReconciliação()

            ConfirmarSincronizacao()

            estadoComunicacao ← LOCAL

        FIM_SE

    FIM_ENQUANTO

FIM
```

---

# 13. Inicialização do Sistema

```text
PROCEDIMENTO InicializarTerminal()

    CarregarConfiguracao()
    InicializarHAL()
    InicializarScanner()
    InicializarBalanca()
    InicializarCamera()
    InicializarPDVCore()
    InicializarSQLite()
    InicializarGreengrassIPC()
    InicializarMQTT()
    InicializarPinpad()

    ValidarComponentes()

    SE componentes_validos = FALSO ENTÃO
        RegistrarFalhaDeInicializacao()
    FIM_SE

FIM_PROCEDIMENTO
```

---

# 14. Operação Normal

```text
PROCEDIMENTO ProcessarOperacaoNormal(evento)

    SE evento.tipo = ItemScanned ENTÃO

        estadoTerminal ← EVALUATING

        sku ← evento.SKU

        produto ← ConsultarCatalogoLocal(sku)

        SE produto possui AGE_VERIFICATION_REQUIRED ENTÃO

            TratarRestricaoEtaria(produto)

            RETORNAR

        FIM_SE

        AbrirJanelaDeValidacao()

        peso ← ObterPeso()

        imagem ← ObterInferenciaVisual()

        resultado ← ExecutarFusionArbiter(
            sku,
            peso,
            imagem
        )

        SE resultado = VALIDO ENTÃO

            AdicionarItemAoCarrinho()

            estadoTerminal ← ITEM_ACCEPTED

            estadoTerminal ← READY_FOR_NEXT

        SENÃO

            TratarDivergenciaPesoOuSweethearting(
                resultado
            )

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 15. Detecção Multissensorial

A arquitetura utiliza:

- código de barras;
- peso;
- visão computacional.

O processo geral é:

```text
Scanner
   ↓
ItemScanned
   ↓
Consulta SKU
   ↓
Peso esperado
   ↓
Peso medido
   ↓
Visão computacional
   ↓
Fusion Arbiter
   ↓
Validação
```

Para divergência de peso:

```text
Z = |ΔM - M_esperada| / σM
```

Quando:

```text
Z > 2.5
```

é gerada a condição:

```text
WEIGHT_ANOMALY
```

Também pode ocorrer bloqueio quando um objeto é transferido para a área de ensacamento sem um `ItemScanned` correspondente.

---

# 16. Núcleo de Tratamento de Exceções

```text
PROCEDIMENTO TratarExcecao(excecao)

    ClassificarExcecao(excecao)

    AlterarEstadoDoTerminal(excecao)

    mensagem ← CriarMensagemDeAlerta(excecao)

    RegistrarEvento(excecao)

    EntregarAlerta(mensagem)

    comando ← AguardarComandoDoFuncionario()

    SE comando = NENHUM ENTÃO
        ManterEstadoDeExcecao()
        RETORNAR
    FIM_SE

    ValidarComando(comando)

    resultado ← ExecutarComando(comando)

    RegistrarResultado(resultado)

    SE resultado = SUCESSO ENTÃO

        SE excecao_resolvida() ENTÃO
            FinalizarExcecao()
        FIM_SE

    SENÃO

        ManterEstadoDeExcecao()

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 17. Entrega de Alertas

```text
PROCEDIMENTO EntregarAlerta(mensagem)

    SE estadoComunicacao = LOCAL ENTÃO

        EnviarParaEdgeGateway(mensagem)

        EnviarWebSocket(mensagem)

        ConfirmarRecebimento()

    SENÃO SE estadoComunicacao = FALLBACK_CLOUD ENTÃO

        PublicarAWSIoT(mensagem)

        EnviarAmazonSNS(mensagem)

        EnviarFCM_APNs(mensagem)

        ConfirmarEntregaQuandoDisponivel()

    SENÃO

        ArmazenarAlertaLocalmente()

        MarcarComoPendente()

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 18. Recepção e Execução de Comandos

```text
PROCEDIMENTO ProcessarComando(comando)

    VerificarAutenticacao(comando)

    VerificarAutorizacao(comando)

    VerificarTerminalDestino(comando)

    VerificarEstadoAtual()

    SE comando = RELEASE_TERMINAL ENTÃO

        ExecutarReleaseTerminal()

    SENÃO SE comando = APPROVE_AGE ENTÃO

        ExecutarApproveAge()

    SENÃO SE comando = MANUAL_OVERRIDE_PAYMENT ENTÃO

        ExecutarManualOverridePayment()

    SENÃO SE comando = CLEAR_AND_RESET_SESSION ENTÃO

        ExecutarClearAndResetSession()

    SENÃO

        RegistrarComandoDesconhecido()

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 19. Idempotência dos Comandos

Para evitar execução duplicada de comandos durante reconexões, o desenvolvimento adota um identificador único para cada comando.

```text
CommandID
```

Antes de executar:

```text
SE CommandID já executado ENTÃO

    Não executar novamente

    Retornar confirmação da execução anterior

SENÃO

    Registrar CommandID

    Executar comando

FIM_SE
```

Essa regra é uma **consolidação de implementação** para evitar duplicidade durante retransmissões e reconexões.

---

# 20. Cenário 1 — Divergência de Peso / Sweethearting

## 20.1 Gatilhos

A exceção ocorre quando:

```text
Z > 2.5
```

ou:

```text
Objeto transferido para a área de ensacamento
sem ItemScanned correspondente
```

Também são considerados os critérios de bloqueio definidos pela arquitetura, como múltiplos objetos detectados em uma passagem ou retirada prematura de itens.

## 20.2 Estado

O terminal transiciona para:

```text
LOCKED_EXCEPTION
```

Nesse estado:

- display é congelado;
- leitores são desativados;
- usuário recebe mensagem amigável;
- evento de alarme é enviado.

## 20.3 Mensagem

Tópico:

```text
store/{id}/term/{id}/alert
```

Tipo:

```text
THEFT_PREVENTION_TRIGGER
```

Dados:

```text
SKU inferido
Delta de peso
URL do clipe de vídeo
TerminalID
Timestamp
```

## 20.4 Fluxo

```text
Divergência
    ↓
LOCKED_EXCEPTION
    ↓
Gerar THEFT_PREVENTION_TRIGGER
    ↓
Enviar alerta
    ↓
Funcionário recebe vibração
    ↓
Funcionário verifica comparação visual
    ↓
Funcionário corrige item
    ↓
Funcionário envia RELEASE_TERMINAL
    ↓
Terminal valida comando
    ↓
Terminal executa liberação
    ↓
Terminal retorna ao fluxo normal
    ↓
Registrar auditoria
```

## 20.5 Pseudocódigo

```text
PROCEDIMENTO TratarDivergenciaPesoOuSweethearting(resultado)

    estadoTerminal ← LOCKED_EXCEPTION

    CongelarInterface()

    DesativarLeitores()

    mensagem.tipo ← THEFT_PREVENTION_TRIGGER

    mensagem.topico ←
        "store/{id}/term/{id}/alert"

    mensagem.SKU ← resultado.SKU_inferido
    mensagem.deltaPeso ← resultado.deltaPeso
    mensagem.video ← resultado.video

    RegistrarEventoLocal(mensagem)

    EntregarAlerta(mensagem)

    funcionario ← AguardarAtendimento()

    SE funcionario realizou_correcao = VERDADEIRO ENTÃO

        comando ← RELEASE_TERMINAL

        EnviarComando(comando)

        resultadoComando ← ExecutarComando(comando)

        SE resultadoComando = SUCESSO ENTÃO

            estadoTerminal ← READY_FOR_NEXT

            RegistrarResolucao()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 21. Cenário 2 — Restrição Etária

## 21.1 Gatilho

O sistema identifica um SKU com:

```text
AGE_VERIFICATION_REQUIRED
```

## 21.2 Comportamento do Terminal

O fluxo de escaneamento é interrompido.

O terminal apresenta:

```text
"Item com restrição de idade.
Um atendente está a caminho."
```

## 21.3 Mensagem

Tópico:

```text
store/{id}/term/{id}/alert
```

Tipo:

```text
AGE_GATE_REQUIRED
```

Campos:

```text
SKU
Descrição
TerminalID
Timestamp
```

## 21.4 Fluxo

```text
SKU escaneado
    ↓
AGE_VERIFICATION_REQUIRED
    ↓
Interromper inclusão
    ↓
Gerar AGE_GATE_REQUIRED
    ↓
Notificar funcionário
    ↓
Funcionário verifica documento
    ↓
Funcionário envia APPROVE_AGE
    ↓
Terminal valida comando
    ↓
Terminal libera fluxo
    ↓
Item é incluído
```

## 21.5 Pseudocódigo

```text
PROCEDIMENTO TratarRestricaoEtaria(produto)

    mensagem.tipo ← AGE_GATE_REQUIRED

    mensagem.SKU ← produto.SKU
    mensagem.descricao ← produto.descricao
    mensagem.TerminalID ← IdentificarTerminal()
    mensagem.timestamp ← ObterTimestamp()

    InterromperInclusaoDoItem()

    ExibirMensagem(
        "Item com restrição de idade. Um atendente está a caminho."
    )

    RegistrarEventoLocal(mensagem)

    EntregarAlerta(mensagem)

    comando ← AguardarComandoDoFuncionario()

    SE comando = APPROVE_AGE ENTÃO

        ValidarAutorizacaoDoFuncionario()

        resultado ← ExecutarApproveAge()

        SE resultado = SUCESSO ENTÃO

            DesbloquearTerminal()

            ConcluirInclusaoDoItem()

            RegistrarResolucao()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 22. Cenário 3 — Queda de WAN Durante Transação PIX

## 22.1 Gatilho

A exceção ocorre quando:

```text
timeout > 15 segundos
```

ou quando o socket é encerrado enquanto o terminal está em:

```text
WAITING_PIX_SETTLEMENT
```

## 22.2 Regra Fundamental

O sistema **não deve interpretar automaticamente a perda da comunicação como pagamento negado ou aprovado**.

A condição deve ser tratada como:

```text
PAYMENT_AMBIGUOUS_STATE
```

## 22.3 Comportamento

O terminal:

- mantém o QR Code;
- mantém a contagem regressiva;
- executa sondagem;
- tenta o link secundário LTE;
- aguarda confirmação do status.

## 22.4 Mensagem

Tópico:

```text
store/{id}/term/{id}/pix_probe
```

Tipo:

```text
PAYMENT_AMBIGUOUS_STATE
```

Dados relevantes:

```text
txid
TerminalID
status
timestamp
```

## 22.5 Fluxo

```text
WAITING_PIX_SETTLEMENT
        ↓
Timeout > 15s
        ↓
PAYMENT_AMBIGUOUS_STATE
        ↓
Manter QR Code
        ↓
pix_probe
        ↓
Polling local
        ↓
Tentar LTE
        ↓
Pagamento confirmado?
     /          \
   SIM           NÃO
   ↓              ↓
Finalizar       Funcionário
transação       verifica adquirente
                  ↓
          MANUAL_OVERRIDE_PAYMENT
```

## 22.6 Pseudocódigo

```text
PROCEDIMENTO TratarQuedaWANPIX()

    estadoTerminal ← PAYMENT_AMBIGUOUS_STATE

    ManterQRCodeNaTela()

    ManterContagemRegressiva()

    mensagem.tipo ← PAYMENT_AMBIGUOUS_STATE

    mensagem.txid ← ObterTXID()

    Publicar(
        "store/{id}/term/{id}/pix_probe",
        mensagem
    )

    resultado ← ConsultarStatusPIX()

    SE resultado = CONFIRMADO ENTÃO

        FinalizarPagamento()

        RegistrarResolucao()

        RETORNAR

    FIM_SE

    AtivarLinkSecundarioLTE()

    resultado ← ConsultarStatusPIX()

    SE resultado = CONFIRMADO ENTÃO

        FinalizarPagamento()

        RegistrarResolucao()

    SENÃO

        EnviarAlertaFuncionario(
            "Terminal: Timeout PIX. Verificando liquidação..."
        )

        funcionario ← AguardarVerificacao()

        SE funcionario confirmou_pagamento = VERDADEIRO ENTÃO

            comando ← MANUAL_OVERRIDE_PAYMENT

            ExecutarComando(comando)

            RegistrarOverrideManual()

        SENÃO

            ManterEstadoAmbiguo()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 23. Cenário 4 — Falha Crítica de Hardware

## 23.1 Gatilhos

Pode ocorrer por:

- perda de keep-alive da balança;
- erro no driver de captura de vídeo V4L2.

Exemplos:

```text
SCALE_USB0
ERR_TIMEOUT
```

## 23.2 Comportamento

O terminal:

1. encerra a sessão de forma emergencial;
2. informa o cliente;
3. orienta transferência dos itens;
4. gera alerta crítico;
5. passa para `OUT_OF_SERVICE`.

## 23.3 Mensagem

Tópico:

```text
store/{id}/term/{id}/system
```

Tipo:

```text
HARDWARE_FAULT
```

Campos:

```text
device
error
TerminalID
timestamp
```

## 23.4 Fluxo

```text
Falha de hardware
       ↓
HARDWARE_FAULT
       ↓
Encerramento emergencial
       ↓
Notificar cliente
       ↓
Notificar manutenção
       ↓
OUT_OF_SERVICE
       ↓
Salvar transação pendente
       ↓
Gerar recuperação por token/QR
       ↓
Transferência para terminal adjacente
```

## 23.5 Pseudocódigo

```text
PROCEDIMENTO TratarFalhaHardware(falha)

    estadoTerminal ← OUT_OF_SERVICE

    ExecutarEncerramentoEmergencial()

    mensagem.tipo ← HARDWARE_FAULT

    mensagem.device ← falha.device
    mensagem.error ← falha.error
    mensagem.TerminalID ← IdentificarTerminal()

    ExibirMensagemAoCliente(
        "Transfira os itens para o terminal adjacente."
    )

    RegistrarEventoLocal(mensagem)

    EnviarAlertaManutencao(mensagem)

    SalvarTransacaoPendente()

    token ← GerarTokenDeRecuperacao()

    GerarQRCode(token)

    RegistrarFalhaHardware()

FIM_PROCEDIMENTO
```

---

# 24. Cenário 5 — Abandono de Terminal com Itens

## 24.1 Gatilho

A exceção ocorre quando:

```text
ausência de interação ≥ 60 segundos
```

e os sensores indicam:

```text
ausência do cliente
```

## 24.2 Primeiro estágio

O sistema emite aviso sonoro direcional por:

```text
15 segundos
```

## 24.3 Segundo estágio

Se o cliente não retornar:

```text
bloquear terminal
```

## 24.4 Mensagem

Tópico:

```text
store/{id}/term/{id}/alert
```

Tipo:

```text
SESSION_ABANDONED
```

Campos:

```text
valor total do carrinho
lista de itens
TerminalID
timestamp
```

## 24.5 Fluxo

```text
60s sem interação
       ↓
Verificar presença
       ↓
Cliente ausente
       ↓
Aviso sonoro por 15s
       ↓
Cliente retorna?
     /       \
   SIM        NÃO
   ↓           ↓
Continuar    Bloquear
              ↓
        SESSION_ABANDONED
              ↓
        Notificar funcionário
              ↓
      CLEAR_AND_RESET_SESSION
              ↓
        Imprimir etiqueta
              ↓
         Limpar sessão
              ↓
             IDLE
```

## 24.6 Pseudocódigo

```text
PROCEDIMENTO TratarAbandonoTerminal()

    IniciarTemporizador(60 segundos)

    SE sem_interacao_por_60s() E cliente_ausente() ENTÃO

        EmitirAvisoSonoro(15 segundos)

        SE cliente_ausente_apos_aviso() ENTÃO

            BloquearTerminal()

            mensagem.tipo ← SESSION_ABANDONED

            mensagem.valorCarrinho ← ObterValorCarrinho()

            mensagem.listaItens ← ObterItensCarrinho()

            RegistrarEventoLocal(mensagem)

            EntregarAlerta(mensagem)

            comando ← AguardarComandoDoFuncionario()

            SE comando = CLEAR_AND_RESET_SESSION ENTÃO

                ImprimirEtiquetaDeItensCancelados()

                CancelarSessao()

                LimparCarrinho()

                LiberarRecursos()

                estadoTerminal ← IDLE

                RegistrarResolucao()

            FIM_SE

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 25. Monitoramento da Comunicação

O sistema deve monitorar continuamente:

- disponibilidade da rede local;
- conexão MQTT;
- sessões WebSocket;
- disponibilidade da WAN;
- link LTE/5G;
- heartbeats;
- status de mensagens pendentes.

## 25.1 Pseudocódigo

```text
PROCEDIMENTO MonitorarConectividade()

    local ← VerificarRedeLocal()

    mqtt ← VerificarMQTT()

    wan ← VerificarWAN()

    websocket ← VerificarWebSocket()

    SE local = DISPONIVEL E mqtt = DISPONIVEL ENTÃO

        estadoComunicacao ← LOCAL

    SENÃO SE wan = DISPONIVEL ENTÃO

        estadoComunicacao ← FALLBACK_CLOUD

    SENÃO

        estadoComunicacao ← OFFLINE

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 26. Operação Offline

A perda da WAN não significa necessariamente que o terminal deixou de funcionar.

A arquitetura foi projetada para permitir operação local e armazenamento das informações que precisam ser sincronizadas posteriormente.

Quando uma operação puder ser concluída localmente:

```text
Terminal
   ↓
SQLite WAL
   ↓
Outbox
   ↓
PENDING_SYNC
```

Posteriormente:

```text
Conexão restaurada
      ↓
Sync Coordinator
      ↓
Greengrass Stream Manager
      ↓
SQS FIFO
      ↓
Deduplicação
      ↓
Aurora / DynamoDB
      ↓
ACK
      ↓
SYNCED
```

---

# 27. Recuperação da WAN

A recuperação da conexão deve ser tratada como uma transição própria.

```text
OFFLINE
   ↓
Conectividade detectada
   ↓
RECOVERING
   ↓
Sincronização
   ↓
Deduplicação
   ↓
Reconciliação
   ↓
Confirmação
   ↓
LOCAL
```

## 27.1 Pseudocódigo

```text
PROCEDIMENTO RecuperarConectividade()

    estadoComunicacao ← RECOVERING

    transacoes ← LerOutbox()

    PARA cada transacao EM transacoes FAÇA

        EnviarParaStreamManager(transacao)

    FIM_PARA

    EnviarParaSQS_FIFO()

    ExecutarDeduplicacao()

    ExecutarReconciliação()

    SE sincronizacao_confirmada() ENTÃO

        AtualizarOutboxParaSYNCED()

        EnviarACK()

        estadoComunicacao ← LOCAL

    SENÃO

        ManterPendencias()

        estadoComunicacao ← RECOVERING

    FIM_SE

FIM_PROCEDIMENTO
```

---

# 28. Controle de Mensagens Pendentes

Quando uma mensagem não puder ser entregue imediatamente:

```text
mensagem.status ← PENDING
```

A mensagem permanece armazenada até que exista um caminho válido para sua entrega.

Quando entregue:

```text
PENDING
  ↓
SENT
  ↓
ACKNOWLEDGED
```

Caso ocorra falha:

```text
PENDING
  ↓
RETRY
```

---

# 29. Reenvio e Deduplicação

Mensagens e comandos podem ser retransmitidos durante uma falha de comunicação.

Por isso, o sistema deve verificar identificadores antes de processar novamente uma operação.

```text
Receber mensagem
      ↓
Verificar identificador
      ↓
Já processada?
   /        \
 SIM         NÃO
 ↓            ↓
Ignorar      Processar
             ↓
        Registrar ID
```

No fluxo transacional da arquitetura, o `UUIDv7` é utilizado para identificar operações e permitir deduplicação.

Para comandos de resolução, este desenvolvimento utiliza `CommandID` como mecanismo de idempotência.

---

# 30. Auditoria

Todas as exceções relevantes devem gerar registros de auditoria.

Um registro deve permitir reconstruir:

```text
O que aconteceu?
Quando aconteceu?
Em qual terminal?
Qual foi a exceção?
Qual mensagem foi enviada?
Qual funcionário recebeu?
Qual comando foi enviado?
Qual comando foi executado?
Qual foi o resultado?
Quando a exceção foi encerrada?
```

## 30.1 Fluxo de Auditoria

```text
Evento
 ↓
Detecção
 ↓
Alerta
 ↓
Ação do funcionário
 ↓
Comando
 ↓
Execução
 ↓
Resultado
 ↓
Resolução
 ↓
Registro de auditoria
```

A arquitetura utiliza, entre outros mecanismos, o **Amazon Aurora PostgreSQL** para registros transacionais e o **Amazon DynamoDB** para eventos e logs de auditoria.

---

# 31. Controle de Autorização do Funcionário

Comandos sensíveis devem ser executados somente após validação da identidade e autorização do funcionário.

No cenário de divergência de peso, por exemplo, a arquitetura especifica:

```text
Biometria/PIN
```

como mecanismo de autorização para:

```text
RELEASE_TERMINAL
```

No cenário de restrição etária, o funcionário deve realizar a verificação do documento e então executar:

```text
APPROVE_AGE
```

---

# 32. Regra de Segurança para Exceções

Uma exceção não deve ser encerrada simplesmente porque o funcionário recebeu a notificação.

O encerramento ocorre somente quando:

```text
Detecção
   ↓
Ação humana
   ↓
Comando
   ↓
Execução confirmada
   ↓
Condição resolvida
```

Somente então:

```text
Exceção = RESOLVIDA
```

---

# 33. Matriz Consolidada dos Cinco Cenários

| # | Exceção | Gatilho | Estado/ação do terminal | Mensagem | Ação do funcionário | Resolução |
|---|---|---|---|---|---|---|
| 1 | Divergência de Peso / Sweethearting | `Z > 2.5` ou item sem scan | `LOCKED_EXCEPTION` | `THEFT_PREVENTION_TRIGGER` | Conferir/corrigir item | `RELEASE_TERMINAL` |
| 2 | Restrição Etária | `AGE_VERIFICATION_REQUIRED` | Interrompe inclusão | `AGE_GATE_REQUIRED` | Verificar documento | `APPROVE_AGE` |
| 3 | Queda WAN durante PIX | Timeout > 15s / socket fechado | `PAYMENT_AMBIGUOUS_STATE` | `PAYMENT_AMBIGUOUS_STATE` | Verificar liquidação | `MANUAL_OVERRIDE_PAYMENT` |
| 4 | Falha de Hardware | Keep-alive/driver perdido | `OUT_OF_SERVICE` | `HARDWARE_FAULT` | Acionar manutenção/transferência | Recuperação em terminal adjacente |
| 5 | Abandono com Itens | 60s + cliente ausente | Bloqueio após aviso | `SESSION_ABANDONED` | Limpar sessão | `CLEAR_AND_RESET_SESSION` |

---

# 34. Matriz de Comunicação

| Situação | Origem | Destino | Canal |
|---|---|---|---|
| Alerta local | Terminal | Funcionário | MQTT → WebSocket |
| Comando local | Funcionário | Terminal | HTTP/3 mTLS → Gateway → MQTT |
| Alerta externo | Terminal | Funcionário | AWS IoT → SNS → FCM/APNs |
| Comando externo | Funcionário | Terminal | AWS IoT → Cloud Bridge → Edge Broker |
| Sincronização | Edge | Cloud | Greengrass Stream Manager → SQS FIFO |
| Confirmação | Cloud | Edge | MQTT |
| Pagamento PIX | AWS/PSP | Terminal | Webhook → AWS → MQTT/WebSocket |

---

# 35. Algoritmo-Mãe Consolidado

O algoritmo abaixo representa a integração de todos os fluxos desenvolvidos.

```text
ALGORITMO SUPERMERCADO_AUTONOMO_MENSAGERIA

INÍCIO

    ==================================================
    1. INICIALIZAÇÃO
    ==================================================

    InicializarTerminal()
    InicializarSensores()
    InicializarSQLite()
    InicializarMQTT()
    InicializarEdgeGateway()
    InicializarMonitoramento()

    estadoTerminal ← IDLE
    estadoComunicacao ← LOCAL


    ==================================================
    2. LOOP PRINCIPAL
    ==================================================

    ENQUANTO sistema_ativo = VERDADEIRO FAÇA


        ==================================================
        2.1 MONITORAMENTO DE COMUNICAÇÃO
        ==================================================

        MonitorarConectividade()


        ==================================================
        2.2 RECUPERAÇÃO DE COMUNICAÇÃO
        ==================================================

        SE comunicacao_restaurada() ENTÃO

            estadoComunicacao ← RECOVERING

            RecuperarConectividade()

        FIM_SE


        ==================================================
        2.3 OBTENÇÃO DE EVENTO
        ==================================================

        evento ← AguardarEvento()


        SE evento = NENHUM_EVENTO ENTÃO

            VerificarAbandono()
            VerificarHardware()
            VerificarPIX()

            CONTINUE

        FIM_SE


        ==================================================
        2.4 DETECÇÃO DE EXCEÇÕES
        ==================================================

        SE detectar_divergencia_peso(evento) ENTÃO

            TratarDivergenciaPesoOuSweethearting(evento)

            CONTINUE

        FIM_SE


        SE detectar_sweethearting(evento) ENTÃO

            TratarDivergenciaPesoOuSweethearting(evento)

            CONTINUE

        FIM_SE


        SE detectar_restricao_etaria(evento) ENTÃO

            TratarRestricaoEtaria(evento.produto)

            CONTINUE

        FIM_SE


        SE detectar_queda_WAN_PIX(evento) ENTÃO

            TratarQuedaWANPIX()

            CONTINUE

        FIM_SE


        SE detectar_falha_hardware(evento) ENTÃO

            TratarFalhaHardware(evento.falha)

            CONTINUE

        FIM_SE


        SE detectar_abandono(evento) ENTÃO

            TratarAbandonoTerminal()

            CONTINUE

        FIM_SE


        ==================================================
        2.5 OPERAÇÃO NORMAL
        ==================================================

        ProcessarOperacaoNormal(evento)


        ==================================================
        2.6 RECEBIMENTO DE COMANDOS
        ==================================================

        comando ← VerificarComandosPendentes()

        SE comando ≠ NENHUM ENTÃO

            ProcessarComando(comando)

        FIM_SE


        ==================================================
        2.7 AUDITORIA
        ==================================================

        RegistrarEventosPendentes()

    FIM_ENQUANTO

FIM
```

---

# 36. Fluxo Geral Consolidado

```text
                         ┌─────────────────────┐
                         │      TERMINAL       │
                         └──────────┬──────────┘
                                    │
                              Evento detectado
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Existe exceção?     │
                         └──────────┬──────────┘
                              NÃO   │   SIM
                              │     │
                              ▼     ▼
                     Operação normal  Classificar
                                      exceção
                                         │
                                         ▼
                                Alterar estado
                                         │
                                         ▼
                                Gerar mensagem
                                         │
                                         ▼
                            ┌─────────────────────┐
                            │ Comunicação local?  │
                            └──────────┬──────────┘
                                 SIM  │  NÃO
                                 │    │
                                 ▼    ▼
                              Gateway  AWS
                                 │    │
                                 ▼    ▼
                              WebSocket
                                 │
                                 ▼
                         Dispositivo funcionário
                                 │
                                 ▼
                            Ação humana
                                 │
                                 ▼
                              Comando
                                 │
                                 ▼
                         ┌────────────────────┐
                         │ Comunicação local? │
                         └─────────┬──────────┘
                              SIM  │  NÃO
                              │    │
                              ▼    ▼
                           Gateway AWS IoT
                              │    │
                              └─┬──┘
                                ▼
                         Edge MQTT Broker
                                │
                                ▼
                              Terminal
                                │
                                ▼
                       Executar comando
                                │
                                ▼
                       Validar resolução
                                │
                         ┌──────┴──────┐
                         │             │
                       SIM            NÃO
                         │             │
                         ▼             ▼
                    Finalizar      Manter exceção
                         │
                         ▼
                      Auditoria
```

---

# 37. Fluxo de Recuperação da Comunicação

```text
              ┌───────────────┐
              │    ONLINE     │
              └───────┬───────┘
                      │
                Falha de WAN
                      │
                      ▼
              ┌───────────────┐
              │    OFFLINE    │
              └───────┬───────┘
                      │
             Tentar comunicação
                      │
              ┌───────┴───────┐
              │               │
           Disponível       Indisponível
              │               │
              ▼               ▼
         RECOVERING         OFFLINE
              │
              ▼
        Sincronização
              │
              ▼
         Deduplicação
              │
              ▼
        Reconciliação
              │
              ▼
          Confirmação
              │
              ▼
             LOCAL
```

---

# 38. Fluxo de Exceção Genérico

Todos os cenários podem ser representados pelo seguinte modelo:

```text
[EVENTO]
    ↓
[DETECÇÃO]
    ↓
[CLASSIFICAÇÃO]
    ↓
[ALTERAÇÃO DE ESTADO]
    ↓
[GERAÇÃO DE ALERTA]
    ↓
[ROTEAMENTO]
    ↓
[FUNCIONÁRIO]
    ↓
[AÇÃO]
    ↓
[COMANDO]
    ↓
[VALIDAÇÃO]
    ↓
[EXECUÇÃO]
    ↓
[CONFIRMAÇÃO]
    ↓
[RESOLUÇÃO]
    ↓
[AUDITORIA]
```

Esse fluxo constitui o núcleo conceitual do algoritmo-mãe.

---

# 39. Decisões de Consolidação Arquitetural

Durante a transformação do documento técnico em um algoritmo único, algumas decisões foram necessárias para representar fluxos que estavam parcialmente descritos.

Essas decisões não devem ser interpretadas como novos requisitos originais do documento, mas como **consolidações necessárias para fechar o algoritmo**.

## 39.1 Retorno de comandos pelo fallback

O documento descreve explicitamente o caminho de alerta pela nuvem:

```text
Terminal
→ AWS IoT Core
→ Amazon SNS
→ FCM/APNs
→ Funcionário
```

Porém, o retorno do comando quando o funcionário está fora da rede local não é detalhado com o mesmo nível de especificidade.

Para fechar o fluxo bidirecional, foi adotado:

```text
Funcionário
→ AWS IoT Core
→ Cloud Bridge
→ Edge MQTT Broker
→ Terminal
```

## 39.2 Normalização de `ReleaseTerminal`

O documento utiliza:

```text
ReleaseTerminal
```

em um ponto da arquitetura e:

```text
RELEASE_TERMINAL
```

na matriz de exceções.

Para o algoritmo, foi adotado:

```text
RELEASE_TERMINAL
```

como identificador padronizado do comando.

## 39.3 Separação dos estados

Foi adotada a separação entre:

```text
Estado do Terminal
```

e:

```text
Estado da Comunicação
```

Isso evita interpretar:

```text
WAN OFFLINE
```

como:

```text
TERMINAL OFFLINE
```

O terminal pode continuar executando determinadas operações localmente mesmo quando a WAN estiver indisponível.

## 39.4 Idempotência

Foi introduzido `CommandID` como mecanismo de controle de duplicidade dos comandos.

Essa decisão é necessária para que retransmissões provocadas por falhas de comunicação não causem execução duplicada.

## 39.5 Pagamento PIX ambíguo

A perda da WAN durante o PIX não é interpretada automaticamente como:

```text
PAGAMENTO NEGADO
```

nem como:

```text
PAGAMENTO APROVADO
```

O estado permanece:

```text
PAYMENT_AMBIGUOUS_STATE
```

até que exista confirmação suficiente da liquidação.

---

# 40. Regras Fundamentais do Sistema

### Regra 1 — Edge-first

Sempre que possível, o processamento deve ocorrer localmente.

### Regra 2 — Comunicação local é prioritária

Quando o funcionário estiver na rede local, utiliza-se:

```text
MQTT + WebSocket + mTLS
```

### Regra 3 — Nuvem funciona como fallback

Quando a rede local não estiver disponível para o funcionário, utiliza-se a infraestrutura de nuvem.

### Regra 4 — Exceção bloqueia quando necessário

Situações de risco podem colocar o terminal em:

```text
LOCKED_EXCEPTION
```

### Regra 5 — Alerta não resolve exceção

O alerta apenas comunica o evento.

### Regra 6 — Comando precisa ser autorizado

A ação do funcionário deve ser validada antes da execução.

### Regra 7 — Execução precisa ser confirmada

O envio de um comando não significa que o terminal o executou.

### Regra 8 — PIX ambíguo permanece ambíguo

Sem confirmação da liquidação, não se assume sucesso ou falha.

### Regra 9 — Falha crítica de hardware gera indisponibilidade

O terminal passa para:

```text
OUT_OF_SERVICE
```

### Regra 10 — Toda exceção deve ser auditável

O histórico deve permitir reconstruir o evento completo.

---

# 41. Tecnologias Relacionadas

A arquitetura técnica utiliza diferentes tecnologias conforme a camada.

## Nuvem

```text
AWS IoT Core
Amazon EventBridge
Amazon SQS FIFO
Amazon EKS
AWS Lambda
Amazon Aurora PostgreSQL
Amazon DynamoDB
Amazon ElastiCache Redis
Amazon S3
Amazon SNS
```

## Edge

```text
Rust
Tauri
TypeScript
SQLite
AWS IoT Greengrass v2
TensorRT
```

## Aplicação do Funcionário

```text
Flutter
Dart
Bloc
WebSocket
gRPC-Web
FCM
APNs
SQLCipher
```

---

# 42. Segurança da Comunicação

A comunicação deve utilizar mecanismos de autenticação e criptografia compatíveis com a arquitetura.

Entre os mecanismos especificados estão:

```text
mTLS
TLS
Certificados X.509
MQTT
WebSocket
HTTP/3
```

A arquitetura utiliza certificados individuais para os terminais no AWS IoT Core.

A comunicação local entre o dispositivo do funcionário e o Edge Gateway utiliza mTLS.

---

# 43. Rastreabilidade

Cada operação relevante deve possuir identificação suficiente para rastreamento.

Exemplos:

```text
TerminalID
SKU
txid
UUIDv7
CommandID
Timestamp
Tipo da exceção
Estado
Resultado
```

Isso permite relacionar:

```text
Evento
    +
Terminal
    +
Transação
    +
Funcionário
    +
Comando
    +
Resultado
```

---

# 44. Conclusão

O algoritmo desenvolvido transforma a arquitetura técnica do supermercado autônomo em um modelo operacional estruturado para **mensageria, comunicação e tratamento de exceções**.

O ponto central do desenvolvimento é que uma exceção não representa apenas um alerta.

Ela representa um ciclo completo:

```text
Detectar
   ↓
Bloquear ou alterar o fluxo quando necessário
   ↓
Comunicar
   ↓
Aguardar ação humana
   ↓
Receber comando
   ↓
Validar
   ↓
Executar
   ↓
Confirmar
   ↓
Resolver
   ↓
Auditar
```

A arquitetura mantém a abordagem **Edge-first**, priorizando comunicação local de baixa latência e utilizando a nuvem como mecanismo de fallback quando necessário.

Os cinco cenários especificados foram incorporados ao algoritmo-mãe:

```text
1. Divergência de Peso / Sweethearting
2. Restrição Etária
3. Queda de WAN Durante Transação PIX
4. Falha Crítica de Hardware
5. Abandono de Terminal com Itens
```

Dessa forma, o algoritmo-mãe funciona como uma representação geral do comportamento do sistema, enquanto cada cenário de exceção constitui uma especialização desse mesmo fluxo.

---

# 45. Resumo Final do Algoritmo

```text
┌───────────────────────────────────────────────┐
│             SUPERMERCADO AUTÔNOMO             │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
                ┌───────────────┐
                │ Evento ocorre │
                └───────┬───────┘
                        │
                        ▼
                ┌───────────────┐
                │ Detectar      │
                │ exceção       │
                └───────┬───────┘
                        │
               ┌────────┴────────┐
               │                 │
              NÃO               SIM
               │                 │
               ▼                 ▼
          Fluxo normal     Alterar estado
                                 │
                                 ▼
                           Gerar alerta
                                 │
                                 ▼
                         Comunicação local?
                           /            \
                         SIM            NÃO
                         │               │
                         ▼               ▼
                    WebSocket        AWS IoT
                         │               │
                         │          SNS/FCM/APNs
                         │               │
                         └───────┬───────┘
                                 ▼
                           Funcionário
                                 │
                                 ▼
                             Ação
                                 │
                                 ▼
                            Comando
                                 │
                                 ▼
                           Validação
                                 │
                                 ▼
                            Execução
                                 │
                                 ▼
                           Confirmação
                                 │
                         ┌───────┴───────┐
                         │               │
                       RESOLVE         FALHA
                         │               │
                         ▼               ▼
                     Auditoria      Manter exceção
                         │
                         ▼
                  Retorno ao fluxo
```


---

# 46. Metodologia de Desenvolvimento — Prompts Utilizados

Esta seção registra os dois prompts utilizados na metodologia de desenvolvimento. O primeiro funciona como prompt-base para orientar a construção da tarefa; o segundo é a especificação operacional detalhada utilizada para conduzir o desenvolvimento técnico. Eles complementam o conteúdo técnico apresentado nas seções anteriores.

## 46.1 Prompt 1 — Prompt-base

```text
# PROMPT 1 — PROMPT-BASE

Sou aluno de Sistemas de Informação e estou desenvolvendo, junto com meu grupo, um projeto acadêmico sobre a arquitetura de um supermercado autônomo. Vou fornecer um documento técnico que descreve toda a arquitetura do sistema. Quero que você leia o documento inteiro antes de começar a desenvolver qualquer coisa, porque a solução precisa ser baseada no que está realmente definido nele.

Meu grupo ficou responsável pela Parte 4 — Mensageria e Exceções. Porém, não quero que você desenvolva essa parte de forma isolada. Primeiro quero que você entenda como o supermercado inteiro funciona, quais são seus componentes, como eles se comunicam, quais estados existem, quais eventos acontecem, como funcionam os sensores, o pagamento, a operação offline, a sincronização e o tratamento de falhas. Depois dessa compreensão geral, quero que você construa um algoritmo-mãe que represente o funcionamento geral do sistema. Esse algoritmo-mãe será a base para desenvolver o algoritmo específico da nossa Parte 4.

Quero que você atue como um professor de Sistemas de Informação e também como um profissional experiente em arquitetura de software, mas explicando de uma forma que alunos de graduação consigam compreender e defender academicamente. Não quero simplesmente um código pronto. Quero entender a lógica que está sendo construída.

O resultado deve ser desenvolvido em pseudocódigo estruturado e ter um nível técnico-acadêmico. Preserve os nomes dos componentes, tecnologias, estados, eventos, mensagens e fluxos utilizados no documento.

É muito importante que você não invente informações e não altere silenciosamente o que está no documento. Se encontrar alguma lacuna, ambiguidade ou inconsistência, identifique claramente o problema, explique por que ele afeta o algoritmo e apresente uma solução proposta. Diferencie aquilo que está no documento daquilo que foi necessário definir durante o desenvolvimento.

Depois de construir o algoritmo-mãe, utilize-o para desenvolver a Parte 4 — Mensageria e Exceções, respeitando a ordem e os cinco cenários apresentados no documento.

Quero que o desenvolvimento considere a comunicação entre terminal e funcionário, comunicação local, fallback pela nuvem, comandos de resolução, estados, eventos, tratamento de exceções, recuperação, operação offline, sincronização, segurança, idempotência, deduplicação e auditoria, sempre respeitando o que o documento realmente estabelece.

No final, quero ter uma solução suficientemente organizada para posteriormente transformar todo o trabalho em um único README.md para o GitHub, onde o professor poderá avaliar o desenvolvimento.

O mais importante é: primeiro compreender, depois abstrair no algoritmo-mãe, depois desenvolver a Parte 4 e somente então consolidar tudo em uma documentação final.

```

## 46.2 Prompt 2 — Prompt de desenvolvimento

```text
# PROMPT 2 — PROMPT DE DESENVOLVIMENTO

## PROJETO — SUPERMERCADO AUTÔNOMO
### Desenvolvimento do Algoritmo-Mãe e da Parte 4 — Mensageria e Exceções

Sou aluno de Sistemas de Informação e estou desenvolvendo, junto com meu grupo, um projeto acadêmico de arquitetura de software para um supermercado autônomo.

O documento técnico fornecido nesta conversa é a fonte principal deste trabalho.

Meu grupo é responsável pela:

**PARTE 4 — MENSAGERIA E EXCEÇÕES**

Quero que você desenvolva essa parte a partir da arquitetura completa apresentada no documento, e não como um módulo isolado.

O desenvolvimento deve seguir obrigatoriamente esta ordem:

1. Leitura completa do documento;
2. Compreensão da arquitetura;
3. Identificação dos componentes envolvidos;
4. Identificação dos estados e eventos;
5. Identificação dos fluxos de comunicação;
6. Construção do modelo geral de funcionamento;
7. Construção do ALGORITMO-MÃE;
8. Derivação do algoritmo específico da Parte 4;
9. Aplicação aos cinco cenários de exceção;
10. Tratamento de recuperação;
11. Tratamento de operação offline;
12. Tratamento de idempotência e deduplicação;
13. Auditoria;
14. Validação;
15. Consolidação da documentação.

## 1. SEU PAPEL

Atue como professor universitário de Sistemas de Informação e arquiteto de software sênior.

Você deve possuir domínio sobre:

- arquitetura de software;
- sistemas distribuídos;
- Edge Computing;
- IoT;
- sistemas orientados a eventos;
- mensageria assíncrona;
- MQTT;
- WebSocket;
- máquinas de estados;
- tratamento de exceções;
- tolerância a falhas;
- operação offline;
- sincronização;
- segurança;
- auditoria.

Porém, explique o raciocínio de maneira compatível com alunos de graduação em Sistemas de Informação.

Não quero uma resposta excessivamente abstrata.

Cada decisão importante deve ser explicada em termos de:

**O QUE acontece?**  
**POR QUE acontece?**  
**QUANDO acontece?**  
**QUEM executa?**  
**QUAL mensagem/evento é produzido?**  
**QUAL estado muda?**  
**COMO o sistema se recupera?**

## 2. FONTE PRINCIPAL

Leia o documento técnico inteiro antes de desenvolver os algoritmos.

Não trabalhe somente com a seção 4.

Identifique no documento:

- hardware;
- sensores;
- scanner;
- balança;
- câmera;
- PDV Core;
- banco local;
- Edge Computing;
- AWS IoT Greengrass;
- MQTT;
- WebSocket;
- AWS IoT Core;
- EventBridge;
- SQS;
- Aurora;
- DynamoDB;
- dispositivos dos funcionários;
- pagamento;
- máquina de estados;
- prevenção de perdas;
- operação offline;
- sincronização;
- segurança;
- auditoria.

Preserve a nomenclatura original sempre que possível.

## 3. REGRA CONTRA INVENÇÃO

Não invente componentes, tecnologias, mensagens, estados ou regras que não estejam sustentados pelo documento.

Quando algo não estiver suficientemente definido, classifique como:

**[DOCUMENTO]**  
Informação explicitamente apresentada no documento.

**[INFERÊNCIA]**  
Conclusão necessária para conectar partes que o documento já apresenta, sem alterar sua arquitetura.

**[PROPOSTA]**  
Decisão criada para preencher uma lacuna necessária ao algoritmo.

Sempre deixe essa diferença explícita.

## 4. IDENTIFICAÇÃO DE LACUNAS

Se encontrar uma lacuna, apresente:

- localização do problema;
- informação existente;
- informação ausente;
- impacto no algoritmo;
- solução proposta;
- justificativa;
- classificação como DOCUMENTO, INFERÊNCIA ou PROPOSTA.

Não corrija silenciosamente o documento.

## 5. MODELO GERAL DO SISTEMA

Antes do algoritmo-mãe, descreva conceitualmente o funcionamento geral.

Mostre:

```text
Cliente
↓
Terminal
↓
Sensores
↓
Processamento Edge
↓
Validação
↓
Carrinho/Sessão
↓
Pagamento
↓
Confirmação
↓
Finalização
↓
Sincronização
↓
Auditoria
```

Inclua também os caminhos de exceção.

O algoritmo-mãe deve representar o comportamento geral e não somente a mensageria.

## 6. ALGORITMO-MÃE

Construa um algoritmo-mãe em pseudocódigo estruturado.

Ele deve representar pelo menos:

1. Inicialização do terminal;
2. Verificação dos componentes;
3. Verificação da comunicação;
4. Preparação da sessão;
5. Operação normal;
6. Leitura dos produtos;
7. Processamento dos sensores;
8. Fusão das informações;
9. Validação do item;
10. Inclusão do item;
11. Verificação de exceções;
12. Bloqueio quando necessário;
13. Mensageria;
14. Atendimento pelo funcionário;
15. Pagamento;
16. Tratamento de falha de pagamento;
17. Finalização;
18. Persistência;
19. Sincronização;
20. Auditoria;
21. Encerramento ou nova sessão.

O algoritmo deve possuir:

- SE;
- SENÃO;
- CASO;
- ENQUANTO;
- estados;
- eventos;
- chamadas de procedimentos;
- retornos;
- condições de erro;
- recuperação.

## 7. MÁQUINA DE ESTADOS

Considere a máquina de estados definida no documento.

Quando aplicável, trabalhe com estados como:

```text
IDLE
EVALUATING
ITEM_ACCEPTED
READY_FOR_NEXT
LOCKED_EXCEPTION
WAITING_PIX_SETTLEMENT
OUT_OF_SERVICE
RECOVERING
```

Não adicione estados apenas por conveniência.

Quando um estado adicional for necessário para organizar o algoritmo, classifique-o como **PROPOSTA DE CONSOLIDAÇÃO**.

## 8. SEPARAÇÃO DE ESTADOS

Para evitar confusão, diferencie três dimensões:

### Estado do terminal

Exemplo:

```text
IDLE
EVALUATING
LOCKED_EXCEPTION
WAITING_PIX_SETTLEMENT
OUT_OF_SERVICE
```

### Estado da comunicação

Exemplo:

```text
ONLINE
OFFLINE
DEGRADED
RECONNECTING
```

### Estado da exceção

Exemplo:

```text
DETECTED
ALERT_SENT
AWAITING_OPERATOR
COMMAND_RECEIVED
COMMAND_EXECUTED
RESOLVED
```

Não misture esses estados.

Explique quando uma dimensão altera outra.

## 9. PARTE 4 — MENSAGERIA

Depois do algoritmo-mãe, desenvolva o algoritmo específico da Parte 4.

A lógica deve contemplar:

```text
EVENTO
↓
DETECÇÃO
↓
VALIDAÇÃO
↓
CRIAÇÃO DA MENSAGEM
↓
PUBLICAÇÃO
↓
ROTEAMENTO
↓
ENTREGA
↓
NOTIFICAÇÃO DO FUNCIONÁRIO
↓
AÇÃO DO FUNCIONÁRIO
↓
COMANDO
↓
VALIDAÇÃO
↓
EXECUÇÃO
↓
CONFIRMAÇÃO
↓
AUDITORIA
↓
RESOLUÇÃO
```

## 10. ALERTA NÃO É RESOLUÇÃO

O algoritmo deve diferenciar claramente:

1. exceção detectada;
2. alerta criado;
3. alerta publicado;
4. alerta entregue;
5. funcionário notificado;
6. funcionário autenticado;
7. comando criado;
8. comando enviado;
9. comando recebido;
10. comando validado;
11. comando executado;
12. estado alterado;
13. exceção resolvida;
14. operação retomada.

Não considere que publicar uma mensagem significa que o problema foi resolvido.

## 11. COMUNICAÇÃO LOCAL

Utilize o fluxo definido no documento:

```text
TERMINAL
↓
EDGE MQTT BROKER / AWS IoT CORE
↓
LOCAL WEBSOCKET GATEWAY
↓
TABLET / SMARTWATCH DO FUNCIONÁRIO
```

Considere:

- MQTT;
- WebSocket;
- TLS/mTLS;
- baixa latência;
- operação local mesmo quando a internet estiver indisponível.

## 12. FALLBACK DE COMUNICAÇÃO

Considere o fallback descrito no documento:

```text
TERMINAL
↓
AWS IoT CORE
↓
CLOUD BRIDGE
↓
AMAZON SNS
↓
FCM/APNs
↓
DISPOSITIVO DO FUNCIONÁRIO
```

O fallback deve ser acionado quando a comunicação local não estiver disponível ou quando o funcionário estiver fora do alcance da rede local, conforme a arquitetura.

## 13. COMUNICAÇÃO DE RETORNO

O funcionário não apenas recebe mensagens.

Ele também envia comandos.

Fluxo local:

```text
FUNCIONÁRIO
↓
HTTP/3 + mTLS
↓
LOCAL WEBSOCKET GATEWAY
↓
MQTT
↓
EDGE BROKER
↓
TERMINAL
```

Caso seja necessária comunicação fora da rede local, represente o caminho de retorno correspondente pela infraestrutura de nuvem.

Se esse retorno não estiver completamente especificado no documento, marque como **PROPOSTA DE CONSOLIDAÇÃO** e explique a decisão.

## 14. IDEMPOTÊNCIA

Comandos críticos devem possuir identificador único.

Utilize:

```text
CommandID
```

Antes de executar um comando:

1. verificar autenticidade;
2. verificar autorização;
3. verificar terminal;
4. verificar exceção;
5. verificar CommandID;
6. verificar estado atual;
7. verificar se o comando já foi executado;
8. executar somente se for novo;
9. registrar resultado.

Se o mesmo comando chegar duas vezes, ele não deve provocar duas execuções.

## 15. DEDUPLICAÇÃO

Mensagens e eventos podem ser retransmitidos.

O algoritmo deve impedir que uma mensagem duplicada provoque:

- dois desbloqueios;
- duas liberações;
- duas aprovações;
- duas finalizações;
- duas alterações financeiras;
- duas alterações de estoque.

Explique como o identificador da mensagem/transação permite detectar duplicidade.

## 16. OPERAÇÃO OFFLINE

O sistema deve respeitar a arquitetura Edge-first.

Quando a WAN estiver indisponível:

- o terminal deve continuar funcionando localmente quando permitido;
- utilizar SQLite;
- utilizar WAL;
- armazenar eventos/transações pendentes;
- utilizar Outbox;
- utilizar UUIDv7;
- aguardar sincronização.

Quando a conexão retornar:

```text
OFFLINE
↓
DETECÇÃO DE ONLINE
↓
SYNC COORDINATOR
↓
OUTBOX
↓
STREAM MANAGER
↓
SQS FIFO
↓
PROCESSAMENTO NA NUVEM
↓
DEDUPLICAÇÃO
↓
PERSISTÊNCIA
↓
ACK
↓
ATUALIZAÇÃO LOCAL
```

## 17. CENÁRIO 1 — DIVERGÊNCIA DE PESO / SWEETHEARTING

Utilize o cenário exatamente como definido no documento.

Gatilhos principais:

- Z > 2.5;
- objeto transferido sem scan;
- Optical Bypass;
- múltiplos itens;
- retirada prematura quando aplicável.

Fluxo:

```text
detectar
→ validar
→ LOCKED_EXCEPTION
→ congelar interface
→ desativar leitores
→ criar THEFT_PREVENTION_TRIGGER
→ publicar alerta
→ notificar funcionário
→ funcionário verificar item
→ corrigir
→ enviar RELEASE_TERMINAL
→ autenticar
→ autorizar
→ validar CommandID
→ executar
→ registrar auditoria
→ retornar ao estado operacional adequado.
```

## 18. CENÁRIO 2 — RESTRIÇÃO ETÁRIA

Gatilho:

```text
AGE_VERIFICATION_REQUIRED
```

Fluxo:

```text
SKU identificado
→ verificar restrição
→ interromper inclusão
→ informar cliente
→ gerar AGE_GATE_REQUIRED
→ enviar alerta
→ funcionário recebe
→ verifica documento
→ envia APPROVE_AGE
→ autenticação
→ autorização
→ validação
→ execução
→ desbloqueio
→ inclusão do item
→ auditoria.
```

## 19. CENÁRIO 3 — QUEDA DE WAN DURANTE PIX

Gatilhos:

- timeout superior a 15 segundos;
- socket fechado;
- estado WAITING_PIX_SETTLEMENT.

REGRA:

Não transformar automaticamente a transação em pagamento recusado.

Utilizar:

```text
PAYMENT_AMBIGUOUS_STATE
```

Fluxo:

```text
detectar perda
→ manter QR
→ manter countdown
→ gerar pix_probe
→ consultar status
→ polling
→ tentar LTE
→ notificar funcionário
→ apresentar txid
→ verificar liquidação
→ confirmar resultado.
```

Se a comunicação secundária falhar:

permitir `MANUAL_OVERRIDE_PAYMENT` somente mediante validação adequada e registrar auditoria.

## 20. CENÁRIO 4 — FALHA CRÍTICA DE HARDWARE

Gatilhos:

- perda de keep-alive da balança;
- erro do driver de vídeo;
- falha crítica de periférico.

Fluxo:

```text
detectar
→ classificar
→ encerrar sessão de maneira segura
→ informar cliente
→ gerar HARDWARE_FAULT
→ informar manutenção
→ OUT_OF_SERVICE
→ preservar transação pendente
→ gerar mecanismo de recuperação
→ permitir continuidade em terminal adjacente
→ registrar auditoria.
```

## 21. CENÁRIO 5 — ABANDONO DE TERMINAL

Gatilhos:

- 60 segundos sem interação;
- ausência de cliente detectada.

Fluxo:

```text
detectar ausência
→ aviso sonoro por 15 segundos
→ verificar retorno
→ se retornar, continuar
→ se não retornar, bloquear
→ gerar SESSION_ABANDONED
→ incluir carrinho e valor
→ notificar funcionário
→ receber CLEAR_AND_RESET_SESSION
→ validar
→ limpar sessão
→ emitir etiqueta quando aplicável
→ retornar IDLE
→ auditoria.
```

## 22. RECUPERAÇÃO

Toda exceção deve possuir caminho de recuperação.

Para cada cenário, informe:

- estado inicial;
- gatilho;
- estado durante exceção;
- mensagem;
- funcionário responsável;
- comando;
- validação;
- estado após comando;
- condição de sucesso;
- condição de falha;
- recuperação;
- auditoria.

## 23. SEGURANÇA

Para comandos sensíveis:

1. autenticar funcionário;
2. verificar autorização;
3. verificar origem;
4. verificar terminal;
5. verificar exceção;
6. verificar estado;
7. verificar CommandID;
8. executar;
9. registrar auditoria.

Não permita que uma mensagem recebida diretamente altere um estado crítico sem validação.

## 24. AUDITORIA

Registre pelo menos quando aplicável:

- TerminalID;
- SessionID;
- EventID;
- CommandID;
- timestamp;
- tipo da exceção;
- estado anterior;
- estado posterior;
- funcionário;
- comando;
- resultado;
- evidência;
- erro.

A auditoria deve permitir reconstruir o que aconteceu.

## 25. FORMATO DOS ALGORITMOS

Utilize pseudocódigo estruturado.

Exemplo de estrutura:

```text
ALGORITMO NomeDoAlgoritmo

ENTRADA:
    ...

SAÍDA:
    ...

VARIÁVEIS:
    ...

INÍCIO

    ...

    SE condição ENTÃO
        ...
    SENÃO
        ...
    FIM-SE

FIM
```

Não utilize código Java, C, Python ou outra linguagem de programação.

O objetivo é especificar o comportamento do sistema antes da implementação.

## 26. EXPLICAÇÃO DO PSEUDOCÓDIGO

Após cada algoritmo importante, explique:

- objetivo;
- entradas;
- saídas;
- variáveis;
- decisões;
- estados;
- eventos;
- mensagens;
- recuperação.

Explique também como o algoritmo se conecta à arquitetura.

## 27. MATRIZ DE RASTREABILIDADE

Ao final, produza uma matriz:

| Elemento | Origem no documento | Implementação no algoritmo | Observação |
|----------|---------------------|----------------------------|------------|

Inclua:

- estados;
- mensagens;
- protocolos;
- componentes;
- exceções;
- comandos;
- mecanismos de recuperação.

## 28. MATRIZ DOS CINCO CENÁRIOS

Produza:

| Cenário | Gatilho | Estado | Mensagem | Funcionário | Comando | Recuperação | Auditoria |
|----------|---------|--------|----------|-------------|---------|------------|-----------|

## 29. DECISÕES DE CONSOLIDAÇÃO

No final, crie uma seção:

**DECISÕES ARQUITETURAIS DE CONSOLIDAÇÃO**

Nela, registre somente aquilo que precisou ser definido para fechar lacunas do documento.

Exemplos:

- normalização de `ReleaseTerminal` para `RELEASE_TERMINAL`;
- uso de `CommandID`;
- separação entre estado do terminal, comunicação e exceção;
- caminho de retorno quando o funcionário estiver fora da rede local;
- estado `RECOVERING`, se necessário.

Não apresente essas decisões como se fossem originalmente escritas no documento.

## 30. VALIDAÇÃO FINAL

Antes de concluir, faça uma verificação:

```text
[ ] Documento foi considerado integralmente.
[ ] Algoritmo-mãe foi desenvolvido antes da Parte 4.
[ ] Parte 4 deriva do algoritmo-mãe.
[ ] Os cinco cenários foram implementados.
[ ] Comunicação terminal → funcionário foi contemplada.
[ ] Comunicação funcionário → terminal foi contemplada.
[ ] Fallback foi contemplado.
[ ] Operação offline foi contemplada.
[ ] Recuperação foi contemplada.
[ ] Idempotência foi contemplada.
[ ] Deduplicação foi contemplada.
[ ] Auditoria foi contemplada.
[ ] Estados foram diferenciados.
[ ] Alertas foram diferenciados de comandos.
[ ] Comandos foram diferenciados de resolução.
[ ] Lacunas foram identificadas.
[ ] Propostas foram separadas do conteúdo original.
[ ] Não foram inventados componentes sem identificação.
[ ] O pseudocódigo está estruturado.
[ ] O material pode ser posteriormente consolidado em README.md.
```

## 31. RESULTADO FINAL

A resposta final deverá demonstrar uma cadeia lógica completa:

```text
ARQUITETURA
↓
FUNCIONAMENTO GERAL
↓
ALGORITMO-MÃE
↓
MENSAGERIA
↓
EXCEÇÕES
↓
COMANDOS
↓
RECUPERAÇÃO
↓
AUDITORIA
↓
VALIDAÇÃO
```

Não pule diretamente para o resultado final.

Construa o raciocínio progressivamente.

O objetivo não é somente produzir um algoritmo que pareça correto.

O objetivo é produzir uma especificação acadêmica que possa ser compreendida, explicada, defendida e posteriormente implementada por alunos de Sistemas de Informação.

```

## 46.3 Relação entre os dois prompts

O **Prompt 1** estabelece o contexto, o objetivo acadêmico e a metodologia geral. Ele é propositalmente mais corrido e próximo da forma como um aluno de Sistemas de Informação descreveria a necessidade do projeto.

O **Prompt 2** transforma essa necessidade em uma especificação de desenvolvimento mais precisa. Ele define a sequência de análise, as regras de fidelidade ao documento, o tratamento de lacunas, o algoritmo-mãe, a mensageria, os estados, os cinco cenários de exceção, recuperação, segurança, idempotência, deduplicação, auditoria, rastreabilidade e validação final.

Dessa forma, a metodologia segue a cadeia:

```text
PROMPT-BASE
    ↓
COMPREENSÃO DO PROBLEMA
    ↓
PROMPT DE DESENVOLVIMENTO
    ↓
LEITURA DA ARQUITETURA
    ↓
ALGORITMO-MÃE
    ↓
PARTE 4 — MENSAGERIA E EXCEÇÕES
    ↓
VALIDAÇÃO E CONSOLIDAÇÃO
    ↓
README.md
```

---

## Fim do Desenvolvimento

**Parte 4 — Mensageria e Exceções**

**Paradigma:** Edge-first  
**Modelo:** Mensageria assíncrona e full-duplex  
**Representação:** Pseudocódigo estruturado  
**Escopo:** Comunicação, exceções, fallback, recuperação e auditoria