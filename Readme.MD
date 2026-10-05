<div align="center">

# Sistema de Mensageria e Tratamento de Exceções — Supermercado Autônomo

[![Área: Sistemas de Informacao](https://img.shields.io/badge/Area-Sistemas%20de%20Informacao-blue?style=for-the-badge&logo=bookstack)](https://github.com)
[![Abordagem: Pseudocodigo](https://img.shields.io/badge/Abordagem-Pseudocodigo%20Estruturado-orange?style=for-the-badge&logo=codeforces)](https://github.com)
[![Nivel: Tecnico-Academico](https://img.shields.io/badge/Nivel-Tecnico%20Academico-success?style=for-the-badge&logo=academia)](https://github.com)
[![Arquitetura: Edge-first](https://img.shields.io/badge/Arquitetura-Edge--first-purple?style=for-the-badge&logo=cloud-flare)](https://github.com)

</div>

---

## 1. Introducao

Este documento apresenta o desenvolvimento da **Parte 4 — Mensageria e Excecoes** da arquitetura de um sistema de supermercado autonomo.

O sistema utiliza uma arquitetura **Edge-first**, na qual o processamento principal do terminal de autoatendimento ocorre localmente, reduzindo a dependencia de conectividade externa para as operacoes que nao exigem autorizacao sincrona.

A comunicacao entre os terminais e os dispositivos dos funcionarios e realizada por uma arquitetura de mensageria assincrona e bidirecional, utilizando comunicacao local de baixa latencia e mecanismos de fallback pela infraestrutura de nuvem.

O desenvolvimento apresentado neste documento tem como objetivo transformar a arquitetura tecnica fornecida em um **algoritmo-mae**, capaz de representar:

- funcionamento normal do terminal;
- comunicacao local;
- deteccao de excecoes;
- geracao de alertas;
- comunicacao com funcionarios;
- execucao de comandos de resolucao;
- fallback pela nuvem;
- recuperacao da comunicacao;
- tratamento das cinco principais excecoes especificadas;
- auditoria e rastreabilidade das acoes.

O algoritmo e apresentado em **pseudocodigo estruturado**, nao estando vinculado a uma linguagem de programacao especifica.

---

## 2. Objetivo

O objetivo principal e modelar o comportamento do sistema de mensageria e tratamento de excecoes de maneira sistematica.

O algoritmo deve representar o ciclo completo:

```text
EVENTO
   ↓
DETECCAO
   ↓
CLASSIFICACAO
   ↓
DECISAO
   ↓
ALTERACAO DE ESTADO
   ↓
GERACAO DE MENSAGEM
   ↓
ROTEAMENTO
   ↓
FUNCIONARIO
   ↓
ACAO
   ↓
COMANDO
   ↓
TERMINAL
   ↓
VALIDACAO
   ↓
RESOLUCAO
   ↓
AUDITORIA
```

A finalidade e garantir que uma excecao nao seja tratada apenas como uma mensagem de alerta, mas como um **fluxo completo de controle do sistema**.

---

## 3. Escopo

O desenvolvimento esta concentrado na **Parte 4 — Mensageria e Excecoes**.

 Sao contemplados:

1. Comunicacao normal entre terminal e funcionario;
2. Comunicacao local via Edge Gateway;
3. Fallback de comunicacao pela nuvem;
4. Retorno da comunicacao ao fluxo local;
5. Tratamento de excecoes;
6. Comandos de resolucao;
7. Controle de estados;
8. Auditoria;
9. Os cinco cenarios de excecao definidos na arquitetura.

Os cinco cenarios sao:

1. **Divergencia de Peso / Sweethearting**
2. **Restricao Etaria**
3. **Queda de WAN Durante Transacao PIX**
4. **Falha Critica de Hardware**
5. **Abandono de Terminal com Itens**

---

## 4. Arquitetura Geral

### 4.1 Arquitetura Edge-first

Cada terminal de autoatendimento possui capacidade local de processamento.

A arquitetura e composta, entre outros elementos, por:

- Scanner;
- Balanca;
- Camera;
- Hardware Abstraction Layer — HAL;
- PDV Core;
- SQLite local com WAL;
- AWS IoT Greengrass v2;
- Edge MQTT Broker;
- Edge Gateway;
- Rede Wi-Fi corporativa;
- Dispositivo do funcionario;
- AWS IoT Core;
- Amazon EventBridge;
- Amazon SQS FIFO;
- Amazon Aurora PostgreSQL;
- Amazon DynamoDB.

A logica principal do terminal nao depende continuamente da internet.

Quando existe conectividade local, o terminal pode se comunicar diretamente com o dispositivo do funcionario.

Quando a comunicacao local nao esta disponivel, o sistema utiliza a infraestrutura de nuvem como mecanismo de fallback.

---

## 5. Componentes Envolvidos na Mensageria

### 5.1 Terminal de Autoatendimento

E responsavel por:

- detectar eventos;
- processar sensores;
- executar regras de negocio;
- alterar seus estados;
- gerar alertas;
- publicar mensagens;
- receber comandos;
- executar comandos;
- registrar eventos de auditoria.

### 5.2 Edge MQTT Broker

Responsavel pelo transporte das mensagens MQTT dentro da infraestrutura de borda.

Recebe mensagens publicadas pelo terminal e permite o encaminhamento para:

- Edge Gateway;
- infraestrutura de sincronizacao;
- demais componentes autorizados.

### 5.3 Local WebSocket Gateway

E o gateway responsavel pela comunicacao de baixa latencia com os dispositivos dos funcionarios.

A comunicacao local utiliza:

```text
WebSocket + TLS + mTLS
```

O objetivo e permitir que alertas cheguem ao funcionario mesmo quando a conexao com a internet estiver indisponivel.

### 5.4 Dispositivo do Funcionario

Pode ser:

- tablet industrial;
- smartwatch;
- outro dispositivo corporativo autorizado.

O dispositivo recebe:

- alertas;
- informacoes da excecao;
- dados do terminal;
- informacoes necessarias para tomada de decisao.

Também pode enviar comandos de resolucao.

### 5.5 AWS IoT Core

E utilizado como broker MQTT gerenciado na nuvem.

Também participa do mecanismo de fallback e da sincronizacao entre a infraestrutura local e a nuvem.

### 5.6 Cloud Bridge

Representa a ponte de comunicacao entre a infraestrutura de borda e a infraestrutura de nuvem.

No fallback, permite que os eventos do terminal cheguem à infraestrutura de nuvem.

Na consolidacao deste algoritmo, também e utilizado como caminho de retorno para comandos destinados ao terminal quando o funcionario estiver fora da rede local.

### 5.7 FCM e APNs

Sao utilizados para notificacoes push quando o dispositivo do funcionario nao esta conectado diretamente à infraestrutura local.

```text
AWS IoT Core
      ↓
Amazon SNS
      ↓
FCM / APNs
      ↓
Dispositivo do Funcionario
```

---

## 6. Modelo de Estados

Para evitar ambiguidades, sao considerados separadamente:

1. **Estado do terminal**
2. **Estado da comunicacao**
3. **Estado da excecao**

Esses estados nao devem ser tratados como se fossem a mesma coisa.

### 6.1 Estados Principais do Terminal

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

### 6.2 Estados de Comunicacao

```text
LOCAL
OFFLINE
FALLBACK_CLOUD
RECOVERING
```

Esses estados representam a disponibilidade do caminho de comunicacao, e nao o funcionamento do terminal.

Por exemplo:

```text
Terminal = LOCKED_EXCEPTION
Comunicacao = LOCAL
```

ou:

```text
Terminal = WAITING_PIX_SETTLEMENT
Comunicacao = FALLBACK_CLOUD
```

### 6.3 Estados de Excecao

Uma excecao pode estar:

```text
DETECTADA
ALERTADA
EM_ATENDIMENTO
COMANDO_ENVIADO
COMANDO_EXECUTADO
RESOLVIDA
REGISTRADA
```

A existencia de uma mensagem enviada nao significa que a excecao ja foi resolvida.

---

## 7. Máquina de Estados do Terminal

O fluxo basico de operacao e:

```text
              ItemScanned
[IDLE] ─────────────────────────> [EVALUATING]
                                      │
                       ┌──────────────┴──────────────┐
                       │                             │
                  Validacao OK               Divergencia
                       │                             │
                       ▼                             ▼
                [ITEM_ACCEPTED]             [LOCKED_EXCEPTION]
                       │                             │
                       ▼                             │
                [READY_FOR_NEXT] <──────────────────┘
```

Uma excecao pode interromper o fluxo normal e colocar o terminal em estado de bloqueio.

---

## 8. Comunicacao Normal

### 8.1 Fluxo Terminal → Funcionario

O fluxo primario e:

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

Exemplo de topico:

```text
store/{id}/terminal/{id}/alerts
```

### 8.2 Fluxo Funcionario → Terminal

Apos analisar o alerta, o funcionario executa uma acao no aplicativo.

No fluxo local:

```text
Funcionario
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

O comando e então processado pelo terminal.

---

## 9. Fallback pela Nuvem

Quando o funcionario estiver fora do alcance da rede Wi-Fi local, o alerta pode seguir o caminho:

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
Dispositivo do Funcionario
```

Esse fluxo permite que o funcionario receba a notificacao mesmo sem conexao direta com a rede local da loja.

---

## 10. Retorno do Comando pelo Fallback

Para completar o fluxo bidirecional quando o funcionario estiver fora da rede local, este desenvolvimento adota o seguinte caminho:

```text
Funcionario
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

Essa e uma **consolidacao arquitetural** utilizada para fechar a comunicacao de retorno no cenario em que o alerta chega ao funcionario pela nuvem.

A ideia central e:

```text
ALERTA:
Terminal → Nuvem → Funcionario

COMANDO:
Funcionario → Nuvem → Edge → Terminal
```

---

## 11. Regra de Separacao entre Alerta e Comando

O sistema deve distinguir:

### Alerta

Informa que algo aconteceu.

Exemplo:

```text
THEFT_PREVENTION_TRIGGER
```

### Comando

Representa uma acao solicitada pelo funcionario.

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
COMANDO EXECUTADO ≠ EXCECAO RESOLVIDA
```

---

## 12. Algoritmo-Mãe — Visão Geral

O algoritmo-mae representa o comportamento geral do sistema.

```text
ALGORITMO SistemaMensageriaExcecoes

INICIO

    InicializarTerminal()
    InicializarSensores()
    InicializarBancoLocal()
    InicializarBroker()
    InicializarGateway()
    InicializarMonitoramentoDeConectividade()

    estadoTerminal ← IDLE
    estadoComunicacao ← LOCAL

    ENQUANTO sistema_ativo = VERDADEIRO FACA

        MonitorarConectividade()

        SE comunicacao_local_disponivel() ENTAO
            estadoComunicacao ← LOCAL

        SENAO SE comunicacao_cloud_disponivel() ENTAO
            estadoComunicacao ← FALLBACK_CLOUD

        SENAO
            estadoComunicacao ← OFFLINE
        FIM_SE

        evento ← AguardarEvento()

        SE evento = NENHUM_EVENTO ENTAO
            ContinuarMonitoramento()
            CONTINUE
        FIM_SE

        excecao ← DetectarExcecao(evento)

        SE excecao = NENHUMA ENTAO

            ProcessarOperacaoNormal(evento)

        SENAO

            estadoTerminal ← DeterminarEstadoDeExcecao(excecao)

            mensagem ← GerarAlerta(excecao)

            RegistrarEventoLocal(excecao, mensagem)

            EntregarAlerta(mensagem, estadoComunicacao)

            resposta ← AguardarAcaoDoFuncionario()

            SE resposta = COMANDO_VALIDO ENTAO

                ValidarPermissao(resposta)

                EnviarComando(resposta)

                resultado ← AguardarConfirmacaoDeExecucao()

                SE resultado = EXECUTADO ENTAO

                    ValidarResolucao()

                    SE excecao_resolvida() ENTAO

                        RegistrarResolucao()

                        RestaurarEstadoTerminal()

                    FIM_SE

                SENAO

                    RegistrarFalhaDoComando()

                    ManterExcecaoAtiva()

                FIM_SE

            FIM_SE

        FIM_SE

        SE estadoComunicacao = OFFLINE ENTAO
            ArmazenarEventoParaSincronizacao()
        FIM_SE

        SE conectividade_foi_restabelecida() ENTAO

            estadoComunicacao ← RECOVERING

            ExecutarSincronizacao()

            ExecutarDeduplicacao()

            ExecutarReconciliacao()

            ConfirmarSincronizacao()

            estadoComunicacao ← LOCAL

        FIM_SE

    FIM_ENQUANTO

FIM
```

---

## 13. Inicialização do Sistema

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

    SE componentes_validos = FALSO ENTAO
        RegistrarFalhaDeInicializacao()
    FIM_SE

FIM_PROCEDIMENTO
```

---

## 14. Operação Normal

```text
PROCEDIMENTO ProcessarOperacaoNormal(evento)

    SE evento.tipo = ItemScanned ENTAO

        estadoTerminal ← EVALUATING

        sku ← evento.SKU

        produto ← ConsultarCatalogoLocal(sku)

        SE produto possui AGE_VERIFICATION_REQUIRED ENTAO

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

        SE resultado = VALIDO ENTAO

            AdicionarItemAoCarrinho()

            estadoTerminal ← ITEM_ACCEPTED

            estadoTerminal ← READY_FOR_NEXT

        SENAO

            TratarDivergenciaPesoOuSweethearting(
                resultado
            )

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 15. Deteccao Multissensorial

A arquitetura utiliza:

- codigo de barras;
- peso;
- visao computacional.

O processo geral e:

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
Visao computacional
   ↓
Fusion Arbiter
   ↓
Validacao
```

Para divergencia de peso:

```text
Z = |ΔM - M_esperada| / σM
```

Quando:

```text
Z > 2.5
```

e gerada a condicao:

```text
WEIGHT_ANOMALY
```

Também pode ocorrer bloqueio quando um objeto e transferido para a área de ensacamento sem um `ItemScanned` correspondente.

---

## 16. Núcleo de Tratamento de Exceções

```text
PROCEDIMENTO TratarExcecao(excecao)

    ClassificarExcecao(excecao)

    AlterarEstadoDoTerminal(excecao)

    mensagem ← CriarMensagemDeAlerta(excecao)

    RegistrarEvento(excecao)

    EntregarAlerta(mensagem)

    comando ← AguardarComandoDoFuncionario()

    SE comando = NENHUM ENTAO
        ManterEstadoDeExcecao()
        RETORNAR
    FIM_SE

    ValidarComando(comando)

    resultado ← ExecutarComando(comando)

    RegistrarResultado(resultado)

    SE resultado = SUCESSO ENTAO

        SE excecao_resolvida() ENTAO
            FinalizarExcecao()
        FIM_SE

    SENAO

        ManterEstadoDeExcecao()

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 17. Entrega de Alertas

```text
PROCEDIMENTO EntregarAlerta(mensagem)

    SE estadoComunicacao = LOCAL ENTAO

        EnviarParaEdgeGateway(mensagem)

        EnviarWebSocket(mensagem)

        ConfirmarRecebimento()

    SENÃO SE estadoComunicacao = FALLBACK_CLOUD ENTAO

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

## 18. Recepção e Execução de Comandos

```text
PROCEDIMENTO ProcessarComando(comando)

    VerificarAutenticacao(comando)

    VerificarAutorizacao(comando)

    VerificarTerminalDestino(comando)

    VerificarEstadoAtual()

    SE comando = RELEASE_TERMINAL ENTAO

        ExecutarReleaseTerminal()

    SENÃO SE comando = APPROVE_AGE ENTAO

        ExecutarApproveAge()

    SENÃO SE comando = MANUAL_OVERRIDE_PAYMENT ENTAO

        ExecutarManualOverridePayment()

    SENÃO SE comando = CLEAR_AND_RESET_SESSION ENTAO

        ExecutarClearAndResetSession()

    SENÃO

        RegistrarComandoDesconhecido()

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 19. Idempotência dos Comandos

Para evitar execucao duplicada de comandos durante reconexoes, o desenvolvimento adota um identificador unico para cada comando.

```text
CommandID
```

Antes de executar:

```text
SE CommandID ja executado ENTAO

    Nao executar novamente

    Retornar confirmacao da execucao anterior

SENAO

    Registrar CommandID

    Executar comando

FIM_SE
```

Essa regra e uma **consolidacao de implementacao** para evitar duplicidade durante retransmissoes e reconexoes.

---

## 20. Cenário 1 — Divergência de Peso / Sweethearting

### 20.1 Gatilhos

A excecao ocorre quando:

```text
Z > 2.5
```

ou:

```text
Objeto transferido para a area de ensacamento
sem ItemScanned correspondente
```

Também sao considerados os criterios de bloqueio definidos pela arquitetura, como multiplos objetos detectados em uma passagem ou retirada prematura de itens.

### 20.2 Estado

O terminal transiciona para:

```text
LOCKED_EXCEPTION
```

Nesse estado:

- display e congelado;
- leitores sao desativados;
- usuario recebe mensagem amigavel;
- evento de alarme e enviado.

### 20.3 Mensagem

Topico:

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
URL do clipe de video
TerminalID
Timestamp
```

### 20.4 Fluxo

```text
Divergencia
    ↓
LOCKED_EXCEPTION
    ↓
Gerar THEFT_PREVENTION_TRIGGER
    ↓
Enviar alerta
    ↓
Funcionario recebe vibracao
    ↓
Funcionario verifica comparacao visual
    ↓
Funcionario corrige item
    ↓
Funcionario envia RELEASE_TERMINAL
    ↓
Terminal valida comando
    ↓
Terminal executa liberacao
    ↓
Terminal retorna ao fluxo normal
    ↓
Registrar auditoria
```

### 20.5 Pseudocódigo

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

    SE funcionario realizou_correcao = VERDADEIRO ENTAO

        comando ← RELEASE_TERMINAL

        EnviarComando(comando)

        resultadoComando ← ExecutarComando(comando)

        SE resultadoComando = SUCESSO ENTAO

            estadoTerminal ← READY_FOR_NEXT

            RegistrarResolucao()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 21. Cenário 2 — Restrição Etária

### 21.1 Gatilho

O sistema identifica um SKU com:

```text
AGE_VERIFICATION_REQUIRED
```

### 21.2 Comportamento do Terminal

O fluxo de escaneamento e interrompido.

O terminal apresenta:

```text
"Item com restricao de idade.
Um atendente esta a caminho."
```

### 21.3 Mensagem

Topico:

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
Descricao
TerminalID
Timestamp
```

### 21.4 Fluxo

```text
SKU escaneado
    ↓
AGE_VERIFICATION_REQUIRED
    ↓
Interromper inclusao
    ↓
Gerar AGE_GATE_REQUIRED
    ↓
Notificar funcionario
    ↓
Funcionario verifica documento
    ↓
Funcionario envia APPROVE_AGE
    ↓
Terminal valida comando
    ↓
Terminal libera fluxo
    ↓
Item e incluido
```

### 21.5 Pseudocódigo

```text
PROCEDIMENTO TratarRestricaoEtaria(produto)

    mensagem.tipo ← AGE_GATE_REQUIRED

    mensagem.SKU ← produto.SKU
    mensagem.descricao ← produto.descricao
    mensagem.TerminalID ← IdentificarTerminal()
    mensagem.timestamp ← ObterTimestamp()

    InterromperInclusaoDoItem()

    ExibirMensagem(
        "Item com restricao de idade. Um atendente esta a caminho."
    )

    RegistrarEventoLocal(mensagem)

    EntregarAlerta(mensagem)

    comando ← AguardarComandoDoFuncionario()

    SE comando = APPROVE_AGE ENTAO

        ValidarAutorizacaoDoFuncionario()

        resultado ← ExecutarApproveAge()

        SE resultado = SUCESSO ENTAO

            DesbloquearTerminal()

            ConcluirInclusaoDoItem()

            RegistrarResolucao()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 22. Cenário 3 — Queda de WAN Durante Transação PIX

### 22.1 Gatilho

A excecao ocorre quando:

```text
timeout > 15 segundos
```

ou quando o socket e encerrado enquanto o terminal esta em:

```text
WAITING_PIX_SETTLEMENT
```

### 22.2 Regra Fundamental

O sistema **nao deve interpretar automaticamente a perda da comunicacao como pagamento negado ou aprovado**.

A condicao deve ser tratada como:

```text
PAYMENT_AMBIGUOUS_STATE
```

### 22.3 Comportamento

O terminal:

- mantem o QR Code;
- mantem a contagem regressiva;
- executa sondagem;
- tenta o link secundario LTE;
- aguarda confirmacao do status.

### 22.4 Mensagem

Topico:

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

### 22.5 Fluxo

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
   SIM           NAO
   ↓              ↓
Finalizar       Funcionario
transacao       verifica adquirente
                  ↓
          MANUAL_OVERRIDE_PAYMENT
```

### 22.6 Pseudocódigo

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

    SE resultado = CONFIRMADO ENTAO

        FinalizarPagamento()

        RegistrarResolucao()

        RETORNAR

    FIM_SE

    AtivarLinkSecundarioLTE()

    resultado ← ConsultarStatusPIX()

    SE resultado = CONFIRMADO ENTAO

        FinalizarPagamento()

        RegistrarResolucao()

    SENAO

        EnviarAlertaFuncionario(
            "Terminal: Timeout PIX. Verificando liquidacao..."
        )

        funcionario ← AguardarVerificacao()

        SE funcionario confirmou_pagamento = VERDADEIRO ENTAO

            comando ← MANUAL_OVERRIDE_PAYMENT

            ExecutarComando(comando)

            RegistrarOverrideManual()

        SENAO

            ManterEstadoAmbiguo()

        FIM_SE

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 23. Cenário 4 — Falha Crítica de Hardware

### 23.1 Gatilhos

Pode ocorrer por:

- perda de keep-alive da balanca;
- erro no driver de captura de video V4L2.

Exemplos:

```text
SCALE_USB0
ERR_TIMEOUT
```

### 23.2 Comportamento

O terminal:

1. encerra a sessao de forma emergencial;
2. informa o cliente;
3. orienta transferencia dos itens;
4. gera alerta critico;
5. passa para `OUT_OF_SERVICE`.

### 23.3 Mensagem

Topico:

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

### 23.4 Fluxo

```text
Falha de hardware
       ↓
HARDWARE_FAULT
       ↓
Encerramento emergencial
       ↓
Notificar cliente
       ↓
Notificar manutencao
       ↓
OUT_OF_SERVICE
       ↓
Salvar transacao pendente
       ↓
Gerar recuperacao por token/QR
       ↓
Transferencia para terminal adjacente
```

### 23.5 Pseudocódigo

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

## 24. Cenário 5 — Abandono de Terminal com Itens

### 24.1 Gatilho

A excecao ocorre quando:

```text
ausencia de interacao ≥ 60 segundos
```

e os sensores indicam:

```text
ausencia do cliente
```

### 24.2 Primeiro estágio

O sistema emite aviso sonoro direcional por:

```text
15 segundos
```

### 24.3 Segundo estágio

Se o cliente nao retornar:

```text
bloquear terminal
```

### 24.4 Mensagem

Topico:

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

### 24.5 Fluxo

```text
60s sem interacao
       ↓
Verificar presenca
       ↓
Cliente ausente
       ↓
Aviso sonoro por 15s
       ↓
Cliente retorna?
     /       \
   SIM        NAO
   ↓           ↓
Continuar    Bloquear
              ↓
        SESSION_ABANDONED
              ↓
        Notificar funcionario
              ↓
      CLEAR_AND_RESET_SESSION
              ↓
        Imprimir etiqueta
              ↓
         Limpar sessao
              ↓
             IDLE
```

### 24.6 Pseudocódigo

```text
PROCEDIMENTO TratarAbandonoTerminal()

    IniciarTemporizador(60 segundos)

    SE sem_interacao_por_60s() E cliente_ausente() ENTAO

        EmitirAvisoSonoro(15 segundos)

        SE cliente_ausente_apos_aviso() ENTAO

            BloquearTerminal()

            mensagem.tipo ← SESSION_ABANDONED

            mensagem.valorCarrinho ← ObterValorCarrinho()

            mensagem.listaItens ← ObterItensCarrinho()

            RegistrarEventoLocal(mensagem)

            EntregarAlerta(mensagem)

            comando ← AguardarComandoDoFuncionario()

            SE comando = CLEAR_AND_RESET_SESSION ENTAO

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

## 25. Monitoramento da Comunicação

O sistema deve monitorar continuamente:

- disponibilidade da rede local;
- conexao MQTT;
- sessoes WebSocket;
- disponibilidade da WAN;
- link LTE/5G;
- heartbeats;
- status de mensagens pendentes.

### 25.1 Pseudocódigo

```text
PROCEDIMENTO MonitorarConectividade()

    local ← VerificarRedeLocal()

    mqtt ← VerificarMQTT()

    wan ← VerificarWAN()

    websocket ← VerificarWebSocket()

    SE local = DISPONIVEL E mqtt = DISPONIVEL ENTAO

        estadoComunicacao ← LOCAL

    SENÃO SE wan = DISPONIVEL ENTAO

        estadoComunicacao ← FALLBACK_CLOUD

    SENÃO

        estadoComunicacao ← OFFLINE

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 26. Operação Offline

A perda da WAN nao significa necessariamente que o terminal deixou de funcionar.

A arquitetura foi projetada para permitir operacao local e armazenamento das informacoes que precisam ser sincronizadas posteriormente.

Quando uma operacao puder ser concluida localmente:

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
Conexao restaurada
      ↓
Sync Coordinator
      ↓
Greengrass Stream Manager
      ↓
SQS FIFO
      ↓
Deduplicacao
      ↓
Aurora / DynamoDB
      ↓
ACK
      ↓
SYNCED
```

---

## 27. Recuperação da WAN

A recuperacao da conexao deve ser tratada como uma transicao propria.

```text
OFFLINE
   ↓
Conectividade detectada
   ↓
RECOVERING
   ↓
Sincronizacao
   ↓
Deduplicacao
   ↓
Reconciliacao
   ↓
Confirmacao
   ↓
LOCAL
```

### 27.1 Pseudocódigo

```text
PROCEDIMENTO RecuperarConectividade()

    estadoComunicacao ← RECOVERING

    transacoes ← LerOutbox()

    PARA cada transacao EM transacoes FACA

        EnviarParaStreamManager(transacao)

    FIM_PARA

    EnviarParaSQS_FIFO()

    ExecutarDeduplicacao()

    ExecutarReconciliacao()

    SE sincronizacao_confirmada() ENTAO

        AtualizarOutboxParaSYNCED()

        EnviarACK()

        estadoComunicacao ← LOCAL

    SENAO

        ManterPendencias()

        estadoComunicacao ← RECOVERING

    FIM_SE

FIM_PROCEDIMENTO
```

---

## 28. Controle de Mensagens Pendentes

Quando uma mensagem nao puder ser entregue imediatamente:

```text
mensagem.status ← PENDING
```

A mensagem permanece armazenada até que exista um caminho valido para sua entrega.

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

## 29. Reenvio e Deduplicação

Mensagens e comandos podem ser retransmitidos durante uma falha de comunicacao.

Por isso, o sistema deve verificar identificadores antes de processar novamente uma operacao.

```text
Receber mensagem
      ↓
Verificar identificador
      ↓
Ja processada?
   /        \
 SIM         NAO
 ↓            ↓
Ignorar      Processar
             ↓
        Registrar ID
```

No fluxo transacional da arquitetura, o `UUIDv7` e utilizado para identificar operacoes e permitir deduplicação.

Para comandos de resolucao, este desenvolvimento utiliza `CommandID` como mecanismo de idempotencia.

---

## 30. Auditoria

Todas as excecoes relevantes devem gerar registros de auditoria.

Um registro deve permitir reconstruir:

```text
O que aconteceu?
Quando aconteceu?
Em qual terminal?
Qual foi a excecao?
Qual mensagem foi enviada?
Qual funcionario recebeu?
Qual comando foi enviado?
Qual comando foi executado?
Qual foi o resultado?
Quando a excecao foi encerrada?
```

### 30.1 Fluxo de Auditoria

```text
Evento
 ↓
Deteccao
 ↓
Alerta
 ↓
Acao do funcionario
 ↓
Comando
 ↓
Execucao
 ↓
Resultado
 ↓
Resolucao
 ↓
Registro de auditoria
```

A arquitetura utiliza, entre outros mecanismos, o **Amazon Aurora PostgreSQL** para registros transacionais e o **Amazon DynamoDB** para eventos e logs de auditoria.

---

## 31. Controle de Autorização do Funcionário

Comandos sensíveis devem ser executados somente apos validacao da identidade e autorizacao do funcionario.

No cenario de divergencia de peso, por exemplo, a arquitetura especifica:

```text
Biometria/PIN
```

como mecanismo de autorizacao para:

```text
RELEASE_TERMINAL
```

No cenario de restricao etaria, o funcionario deve realizar a verificacao do documento e então executar:

```text
APPROVE_AGE
```

---

## 32. Regra de Segurança para Exceções

Uma excecao nao deve ser encerrada simplesmente porque o funcionario recebeu a notificacao.

O encerramento ocorre somente quando:

```text
Deteccao
   ↓
Acao humana
   ↓
Comando
   ↓
Execucao confirmada
   ↓
Condicao resolvida
```

Somente então:

```text
Excecao = RESOLVIDA
```

---

## 33. Matriz Consolidada dos Cinco Cenários

| # | Excecao | Gatilho | Estado/acao do terminal | Mensagem | Acao do funcionario | Resolucao |
|---|---|---|---|---|---|---|
| 1 | Divergencia de Peso / Sweethearting | `Z > 2.5` ou item sem scan | `LOCKED_EXCEPTION` | `THEFT_PREVENTION_TRIGGER` | Conferir/corrigir item | `RELEASE_TERMINAL` |
| 2 | Restricao Etaria | `AGE_VERIFICATION_REQUIRED` | Interrompe inclusao | `AGE_GATE_REQUIRED` | Verificar documento | `APPROVE_AGE` |
| 3 | Queda WAN durante PIX | Timeout > 15s / socket fechado | `PAYMENT_AMBIGUOUS_STATE` | `PAYMENT_AMBIGUOUS_STATE` | Verificar liquidacao | `MANUAL_OVERRIDE_PAYMENT` |
| 4 | Falha de Hardware | Keep-alive/driver perdido | `OUT_OF_SERVICE` | `HARDWARE_FAULT` | Acionar manutencao/transferencia | Recuperacao em terminal adjacente |
| 5 | Abandono com Itens | 60s + cliente ausente | Bloqueio pós-aviso | `SESSION_ABANDONED` | Limpar sessao | `CLEAR_AND_RESET_SESSION` |

---

## 34. Matriz de Comunicação

| Situacao | Origem | Destino | Canal |
|---|---|---|---|
| Alerta local | Terminal | Funcionario | MQTT → WebSocket |
| Comando local | Funcionario | Terminal | HTTP/3 mTLS → Gateway → MQTT |
| Alerta externo | Terminal | Funcionario | AWS IoT → SNS → FCM/APNs |
| Comando externo | Funcionario | Terminal | AWS IoT → Cloud Bridge → Edge Broker |
| Sincronizacao | Edge | Cloud | Greengrass Stream Manager → SQS FIFO |
| Confirmacao | Cloud | Edge | MQTT |
| Pagamento PIX | AWS/PSP | Terminal | Webhook → AWS → MQTT/WebSocket |

---

## 35. Algoritmo-Mãe Consolidado

O algoritmo abaixo representa a integracao de todos os fluxos desenvolvidos.

```text
ALGORITMO SUPERMERCADO_AUTONOMO_MENSAGERIA

INICIO

    ==================================================
    1. INICIALIZACAO
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

    ENQUANTO sistema_ativo = VERDADEIRO FACA


        ==================================================
        2.1 MONITORAMENTO DE COMUNICACAO
        ==================================================

        MonitorarConectividade()


        ==================================================
        2.2 RECUPERACAO DE COMUNICACAO
        ==================================================

        SE comunicacao_restaurada() ENTAO

            estadoComunicacao ← RECOVERING

            RecuperarConectividade()

        FIM_SE


        ==================================================
        2.3 OBTENCAO DE EVENTO
        ==================================================

        evento ← AguardarEvento()


        SE evento = NENHUM_EVENTO ENTAO

            VerificarAbandono()
            VerificarHardware()
            VerificarPIX()

            CONTINUE

        FIM_SE


        ==================================================
        2.4 DETECCAO DE EXCECOES
        ==================================================

        SE detectar_divergencia_peso(evento) ENTAO

            TratarDivergenciaPesoOuSweethearting(evento)

            CONTINUE

        FIM_SE


        SE detectar_sweethearting(evento) ENTAO

            TratarDivergenciaPesoOuSweethearting(evento)

            CONTINUE

        FIM_SE


        SE detectar_restricao_etaria(evento) ENTAO

            TratarRestricaoEtaria(evento.produto)

            CONTINUE

        FIM_SE


        SE detectar_queda_WAN_PIX(evento) ENTAO

            TratarQuedaWANPIX()

            CONTINUE

        FIM_SE


        SE detectar_falha_hardware(evento) ENTAO

            TratarFalhaHardware(evento.falha)

            CONTINUE

        FIM_SE


        SE detectar_abandono(evento) ENTAO

            TratarAbandonoTerminal()

            CONTINUE

        FIM_SE


        ==================================================
        2.5 OPERACAO NORMAL
        ==================================================

        ProcessarOperacaoNormal(evento)


        ==================================================
        2.6 RECEBIMENTO DE COMANDOS
        ==================================================

        comando ← VerificarComandosPendentes()

        SE comando ≠ NENHUM ENTAO

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

## 36. Fluxo Geral Consolidado

```text
                         ┌─────────────────────┐
                         │      TERMINAL       │
                         └──────────┬──────────┘
                                    │
                              Evento detectado
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Existe excecao?     │
                         └──────────┬──────────┘
                              NAO   │   SIM
                              │     │
                              ▼     ▼
                     Operacao normal  Classificar
                                      excecao
                                         │
                                         ▼
                                Alterar estado
                                         │
                                         ▼
                                Gerar mensagem
                                         │
                                         ▼
                            ┌─────────────────────┐
                            │ Comunicacao local?  │
                            └──────────┬──────────┘
                                 SIM  │  NAO
                                 │    │
                                 ▼    ▼
                              Gateway  AWS
                                 │    │
                                 ▼    ▼
                              WebSocket
                                 │
                                 ▼
                         Dispositivo funcionario
                                 │
                                 ▼
                            Acao humana
                                 │
                                 ▼
                              Comando
                                 │
                                 ▼
                         ┌────────────────────┐
                         │ Comunicacao local? │
                         └─────────┬──────────┘
                              SIM  │  NAO
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
                       Validar resolucao
                                │
                         ┌──────┴──────┐
                         │             │
                       SIM            NAO
                         │             │
                         ▼             ▼
                    Finalizar      Manter excecao
                         │
                         ▼
                      Auditoria
```

---

## 37. Fluxo de Recuperação da Comunicação

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
             Tentar comunicacao
                      │
              ┌───────┴───────┐
              │               │
           Disponivel       Indisponivel
              │               │
              ▼               ▼
         RECOVERING         OFFLINE
              │
              ▼
        Sincronizacao
              │
              ▼
         Deduplicacao
              │
              ▼
        Reconciliacao
              │
              ▼
          Confirmacao
              │
              ▼
             LOCAL
```

---

## 38. Fluxo de Exceção Genérico

Todos os cenarios podem ser representados pelo seguinte modelo:

```text
[EVENTO]
    ↓
[DETECCAO]
    ↓
[CLASSIFICACAO]
    ↓
[ALTERACAO DE ESTADO]
    ↓
[GERACAO DE ALERTA]
    ↓
[ROTEAMENTO]
    ↓
[FUNCIONARIO]
    ↓
[ACAO]
    ↓
[COMANDO]
    ↓
[VALIDACAO]
    ↓
[EXECUCAO]
    ↓
[CONFIRMACAO]
    ↓
[RESOLUCAO]
    ↓
[AUDITORIA]
```

Esse fluxo constitui o nucleo conceitual do algoritmo-mae.

---

## 39. Decisões de Consolidação Arquitetural

Durante a transformacao do documento tecnico em um algoritmo unico, algumas decisoes foram necessarias para representar fluxos que estavam parcialmente descritos.

Essas decisoes nao devem ser interpretadas como novos requisitos originais do documento, mas como **consolidacoes necessarias para fechar o algoritmo**.

### 39.1 Retorno de comandos pelo fallback

O documento descreve explicitamente o caminho de alerta pela nuvem:

```text
Terminal
→ AWS IoT Core
→ Amazon SNS
→ FCM/APNs
→ Funcionario
```

Porém, o retorno do comando quando o funcionario esta fora da rede local nao e detalhado com o mesmo nivel de especificidade.

Para fechar o fluxo bidirecional, foi adotado:

```text
Funcionario
→ AWS IoT Core
→ Cloud Bridge
→ Edge MQTT Broker
→ Terminal
```

### 39.2 Normalização de `ReleaseTerminal`

O documento utiliza:

```text
ReleaseTerminal
```

em um ponto da arquitetura e:

```text
RELEASE_TERMINAL
```

na matriz de excecoes.

Para o algoritmo, foi adotado:

```text
RELEASE_TERMINAL
```

como identificador padronizado do comando.

### 39.3 Separação dos estados

Foi adotada a separacao entre:

```text
Estado do Terminal
```

e:

```text
Estado da Comunicacao
```

Isso evita interpretar:

```text
WAN OFFLINE
```

como:

```text
TERMINAL OFFLINE
```

O terminal pode continuar executando determinadas operacoes localmente mesmo quando a WAN estiver indisponível.

### 39.4 Idempotência

Foi introduzido `CommandID` como mecanismo de controle de duplicidade dos comandos.

Essa decisao e necessaria para que retransmissoes provocadas por falhas de comunicacao nao causem execucao duplicada.

### 39.5 Pagamento PIX ambíguo

A perda da WAN durante o PIX nao e interpretada automaticamente como:

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

ate que exista confirmacao suficiente da liquidacao.

---

## 40. Regras Fundamentais do Sistema

### Regra 1 — Edge-first

Sempre que possivel, o processamento deve ocorrer localmente.

### Regra 2 — Comunicação local é prioritária

Quando o funcionario estiver na rede local, utiliza-se:

```text
MQTT + WebSocket + mTLS
```

### Regra 3 — Nuvem funciona como fallback

Quando a rede local nao estiver disponivel para o funcionario, utiliza-se a infraestrutura de nuvem.

### Regra 4 — Exceção bloqueia quando necessário

Situacoes de risco podem colocar o terminal em:

```text
LOCKED_EXCEPTION
```

### Regra 5 — Alerta não resolve exceção

O alerta apenas comunica o evento.

### Regra 6 — Comando precisa ser autorizado

A acao do funcionario deve ser validada antes da execucao.

### Regra 7 — Execução precisa ser confirmada

O envio de um comando nao significa que o terminal o executou.

### Regra 8 — PIX ambíguo permanece ambíguo

Sem confirmacao da liquidacao, nao se assume sucesso ou falha.

### Regra 9 — Falha crítica de hardware gera indisponibilidade

O terminal passa para:

```text
OUT_OF_SERVICE
```

### Regra 10 — Toda exceção deve ser auditável

O historico deve permitir reconstruir o evento completo.

---

## 41. Tecnologias Relacionadas

A arquitetura tecnica utiliza diferentes tecnologias conforme a camada.

### Nuvem

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

### Edge

```text
Rust
Tauri
TypeScript
SQLite
AWS IoT Greengrass v2
TensorRT
```

### Aplicação do Funcionário

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

## 42. Segurança da Comunicação

A comunicacao deve utilizar mecanismos de autenticacao e criptografia compativeis com a arquitetura.

Entre os mecanismos especificados estao:

```text
mTLS
TLS
Certificados X.509
MQTT
WebSocket
HTTP/3
```

A arquitetura utiliza certificados individuais para os terminais no AWS IoT Core.

A comunicacao local entre o dispositivo do funcionario e o Edge Gateway utiliza mTLS.

---

## 43. Rastreabilidade

Cada operacao relevante deve possuir identificacao suficiente para rastreamento.

Exemplos:

```text
TerminalID
SKU
txid
UUIDv7
CommandID
Timestamp
Tipo da excecao
Estado
Resultado
```

Isso permite relacionar:

```text
Evento
    +
Terminal
    +
Transacao
    +
Funcionario
    +
Comando
    +
Resultado
```

---

## 44. Conclusão

O algoritmo desenvolvido transforma a arquitetura tecnica do supermercado autonomo em um modelo operacional estruturado para **mensageria, comunicacao e tratamento de excecoes**.

O ponto central do desenvolvimento e que uma excecao nao representa apenas um alerta.

Ela representa um ciclo completo:

```text
Detectar
   ↓
Bloquear ou alterar o fluxo quando necessario
   ↓
Comunicar
   ↓
Aguardar acao humana
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

A arquitetura mantem a abordagem **Edge-first**, priorizando comunicacao local de baixa latencia e utilizando a nuvem como mecanismo de fallback quando necessario.

Os cinco cenarios especificados foram incorporados ao algoritmo-mae:

```text
1. Divergencia de Peso / Sweethearting
2. Restricao Etaria
3. Queda de WAN Durante Transacao PIX
4. Falha Critica de Hardware
5. Abandono de Terminal com Itens
```

Dessa forma, o algoritmo-mae funciona como uma representacao geral do comportamento do sistema, enquanto cada cenario de excecao constitui uma especializacao desse mesmo fluxo.

---

## 45. Resumo Final do Algoritmo

```text
┌───────────────────────────────────────────────┐
│             SUPERMERCADO AUTONOMO             │
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
                │ excecao       │
                └───────┬───────┘
                        │
               ┌────────┴────────┐
               │                 │
              NAO               SIM
               │                 │
               ▼                 ▼
          Fluxo normal     Alterar estado
                                 │
                                 ▼
                           Gerar alerta
                                 │
                                 ▼
                         Comunicacao local?
                           /            \
                         SIM            NAO
                         │               │
                         ▼               ▼
                    WebSocket        AWS IoT
                         │               │
                         │          SNS/FCM/APNs
                         │               │
                         └───────┬───────┘
                                 ▼
                           Funcionario
                                 │
                                 ▼
                             Acao
                                 │
                                 ▼
                            Comando
                                 │
                                 ▼
                           Validacao
                                 │
                                 ▼
                            Execucao
                                 │
                                 ▼
                           Confirmacao
                                 │
                         ┌───────┴───────┐
                         │               │
                       RESOLVE         FALHA
                         │               │
                         ▼               ▼
                     Auditoria      Manter excecao
                         │
                         ▼
                  Retorno ao fluxo
```

---

## 46. Metodologia de Desenvolvimento — Prompts Utilizados

Esta secao registra os dois prompts utilizados na metodologia de desenvolvimento. O primeiro funciona como prompt-base para orientar a construcao da tarefa; o segundo e a especificacao operacional detalhada utilizada para conduzir o desenvolvimento tecnico. Eles complementam o conteudo tecnico apresentado nas secoes anteriores.

### 46.1 Prompt 1 — Prompt-base

```text
# PROMPT 1 — PROMPT-BASE

Sou aluno de Sistemas de Informacao e estou desenvolvendo, junto com meu grupo, um projeto academico sobre a arquitetura de um supermercado autonomo. Vou fornecer um documento tecnico que descreve toda a arquitetura do sistema. Quero que você leia o documento inteiro antes de comecar a desenvolver qualquer coisa, porque a solucao precisa ser baseada no que esta realmente definido nele.

Meu grupo ficou responsavel pela Parte 4 — Mensageria e Excecoes. Porém, nao quero que você desenvolva essa parte de forma isolada. Primeiro quero que você entenda como o supermercado inteiro funciona, quais sao seus componentes, como eles se comunicam, quais estados existem, quais eventos acontecem, como funcionam os sensores, o pagamento, a operacao offline, a sincronizacao e o tratamento de falhas. Depois dessa compreensao geral, quero que você construa um algoritmo-mae que represente o funcionamento geral do sistema. Esse algoritmo-mae sera a base para desenvolver o algoritmo especifico da nossa Parte 4.

Quero que você atue como um professor de Sistemas de Informacao e também como um profissional experiente em arquitetura de software, mas explicando de uma forma que alunos de graduacao consigam compreender e defender academicamente. Nao quero simplesmente um codigo pronto. Quero entender a logica que esta sendo construida.

O resultado deve ser desenvolvido em pseudocodigo estruturado e ter um nivel tecnico-academico. Preserve os nomes dos componentes, tecnologias, estados, eventos, mensagens e fluxos utilizados no documento.

E muito importante que você nao invente informacoes e nao altere silenciosamente o que esta no documento. Se encontrar alguma lacuna, ambiguidade ou inconsistencia, identifique claramente o problema, explique por que ele afeta o algoritmo e apresente uma solucao proposta. Diferencie aquilo que esta no documento daquilo que foi necessario definir durante o desenvolvimento.

Depois de construir o algoritmo-mae, utilize-o para desenvolver a Parte 4 — Mensageria e Excecoes, respeitando a ordem e os cinco cenarios apresentados no documento.

Quero que o desenvolvimento considere a comunicacao entre terminal e funcionario, comunicacao local, fallback pela nuvem, comandos de resolucao, estados, eventos, tratamento de excecoes, recuperacao, operacao offline, sincronizacao, seguranca, idempotencia, deduplicacao e auditoria, sempre respeitando o que o documento realmente estabelece.

No final, quero ter uma solucao suficientemente organizada para posteriormente transformar todo o trabalho em um unico README.md para o GitHub, onde o professor podera avaliar o desenvolvimento.

O mais importante e: primeiro compreender, depois abstrair no algoritmo-mae, depois desenvolver a Parte 4 e somente então consolidar tudo em uma documentacao final.
```

### 46.2 Prompt 2 — Prompt de desenvolvimento

```text
# PROMPT 2 — PROMPT DE DESENVOLVIMENTO

## PROJETO — SUPERMERCADO AUTONOMO
### Desenvolvimento do Algoritmo-Mae e da Parte 4 — Mensageria e Excecoes

Sou aluno de Sistemas de Informacao e estou desenvolvendo, junto com meu grupo, um projeto academico de arquitetura de software para um supermercado autonomo.

O documento tecnico fornecido nesta conversa e a fonte principal deste trabalho.

Meu grupo e responsavel pela:

**PARTE 4 — MENSAGERIA E EXCECOES**

Quero que você desenvolva essa parte a partir da arquitetura completa apresentada no documento, e nao como um modulo isolado.

O desenvolvimento deve seguir obrigatoriamente esta ordem:

1. Leitura completa do documento;
2. Compreensao da arquitetura;
3. Identificacao dos componentes envolvidos;
4. Identificacao dos estados e eventos;
5. Identificacao dos fluxos de comunicacao;
6. Construcao do modelo geral de funcionamento;
7. Construcao do ALGORITMO-MAE;
8. Derivacao do algoritmo especifico da Parte 4;
9. Aplicacao aos cinco cenarios de excecao;
10. Tratamento de recuperacao;
11. Tratamento de operacao offline;
12. Tratamento de idempotencia e deduplicacao;
13. Auditoria;
14. Validacao;
15. Consolidacao da documentacao.

## 1. SEU PAPEL

Atue como professor universitario de Sistemas de Informacao e arquiteto de software senior.

Você deve possuir dominio sobre:

- arquitetura de software;
- sistemas distribuidos;
- Edge Computing;
- IoT;
- sistemas orientados a eventos;
- mensageria assincrona;
- MQTT;
- WebSocket;
- maquinas de estados;
- tratamento de excecoes;
- tolerancia a falhas;
- operacao offline;
- sincronizacao;
- seguranca;
- auditoria.

Porém, explique o raciocinio de maneira compativel com alunos de graduacao em Sistemas de Informacao.

Nao quero uma resposta excessivamente abstrata.

Cada decisao importante deve ser explicada em termos de:

**O QUE acontece?**  
**POR QUE acontece?**  
**QUANDO acontece?**  
**QUEM executa?**  
**QUAL mensagem/evento e produzido?**  
**QUAL estado muda?**  
**COMO o sistema se recupera?**

## 2. FONTE PRINCIPAL

Leia o documento tecnico inteiro antes de desenvolver os algoritmos.

Nao trabalhe somente com a secao 4.

Identifique no documento:

- hardware;
- sensores;
- scanner;
- balanca;
- camera;
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
- dispositivos dos funcionarios;
- pagamento;
- maquina de estados;
- prevencao de perdas;
- operacao offline;
- sincronizacao;
- seguranca;
- auditoria.

Preserve a nomenclatura original sempre que possivel.

## 3. REGRA CONTRA INVENCAO

Nao invente componentes, tecnologias, mensagens, estados ou regras que nao estejam sustentados pelo documento.

Quando algo nao estiver suficientemente definido, classifique como:

**[DOCUMENTO]**  
Informacao explicitamente apresentada no documento.

**[INFERENCIA]**  
Conclusao necessaria para conectar partes que o documento ja apresenta, sem alterar sua arquitetura.

**[PROPOSTA]**  
Decisao criada para preencher uma lacuna necessaria ao algoritmo.

Sempre deixe essa diferenca explicita.

## 4. IDENTIFICACAO DE LACUNAS

Se encontrar uma lacuna, apresente:

- localizacao do problema;
- informacao existente;
- informacao ausente;
- impacto no algoritmo;
- solucao proposta;
- justificativa;
- classificacao como DOCUMENTO, INFERENCIA ou PROPOSTA.

Nao corrija silenciosamente o documento.

## 5. MODELO GERAL DO SISTEMA

Antes do algoritmo-mae, descreva conceitualmente o funcionamento geral.

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
Validacao
↓
Carrinho/Sessao
↓
Pagamento
↓
Confirmacao
↓
Finalizacao
↓
Sincronizacao
↓
Auditoria
```

Inclua também os caminhos de excecao.

O algoritmo-mae deve representar o comportamento geral e nao somente a mensageria.

## 6. ALGORITMO-MAE

Construa um algoritmo-mae em pseudocodigo estruturado.

Ele deve representar pelo menos:

1. Inicializacao do terminal;
2. Verificacao dos componentes;
3. Verificacao da comunicacao;
4. Preparacao da sessao;
5. Operacao normal;
6. Leitura dos produtos;
7. Processamento dos sensores;
8. Fusao das informacoes;
9. Validacao do item;
10. Inclusao do item;
11. Verificacao de excecoes;
12. Bloqueio quando necessario;
13. Mensageria;
14. Atendimento pelo funcionario;
15. Pagamento;
16. Tratamento de falha de pagamento;
17. Finalizacao;
18. Persistencia;
19. Sincronizacao;
20. Auditoria;
21. Encerramento ou nova sessao.

O algoritmo deve possuir:

- SE;
- SENAO;
- CASO;
- ENQUANTO;
- estados;
- eventos;
- chamadas de procedimentos;
- retornos;
- condicoes de erro;
- recuperacao.

## 7. MAQUINA DE ESTADOS

Considere a maquina de estados definida no documento.

Quando aplicavel, trabalhe com estados como:

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

Nao adicione estados apenas por conveniencia.

Quando um estado adicional for necessario para organizar o algoritmo, classifique-o como **PROPOSTA DE CONSOLIDACAO**.

## 8. SEPARACAO DE ESTADOS

Para evitar confusao, diferencie tres dimensoes:

### Estado do terminal

Exemplo:

```text
IDLE
EVALUATING
LOCKED_EXCEPTION
WAITING_PIX_SETTLEMENT
OUT_OF_SERVICE
```

### Estado da comunicacao

Exemplo:

```text
ONLINE
OFFLINE
DEGRADED
RECONNECTING
```

### Estado da excecao

Exemplo:

```text
DETECTED
ALERT_SENT
AWAITING_OPERATOR
COMMAND_RECEIVED
COMMAND_EXECUTED
RESOLVED
```

Nao misture esses estados.

Explique quando uma dimensao altera outra.

## 9. PARTE 4 — MENSAGERIA

Depois do algoritmo-mae, desenvolva o algoritmo especifico da Parte 4.

A logica deve contemplar:

```text
EVENTO
↓
DETECCAO
↓
VALIDACAO
↓
CRIACAO DA MENSAGEM
↓
PUBLICACAO
↓
ROTEAMENTO
↓
ENTREGA
↓
NOTIFICACAO DO FUNCIONARIO
↓
ACAO DO FUNCIONARIO
↓
COMANDO
↓
VALIDACAO
↓
EXECUCAO
↓
CONFIRMACAO
↓
AUDITORIA
↓
RESOLUCAO
```

## 10. ALERTA NAO E RESOLUCAO

O algoritmo deve diferenciar claramente:

1. excecao detectada;
2. alerta criado;
3. alerta publicado;
4. alerta entregue;
5. funcionario notificado;
6. funcionario autenticado;
7. comando criado;
8. comando enviado;
9. comando recebido;
10. comando validado;
11. comando executado;
12. estado alterado;
13. excecao resolvida;
14. operacao retomada.

Nao considere que publicar uma mensagem significa que o problema foi resolvido.

## 11. COMUNICACAO LOCAL

Utilize o fluxo definido no documento:

```text
TERMINAL
↓
EDGE MQTT BROKER / AWS IoT CORE
↓
LOCAL WEBSOCKET GATEWAY
↓
TABLET / SMARTWATCH DO FUNCIONARIO
```

Considere:

- MQTT;
- WebSocket;
- TLS/mTLS;
- baixa latencia;
- operacao local mesmo quando a internet estiver indisponivel.

## 12. FALLBACK DE COMUNICACAO

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
DISPOSITIVO DO FUNCIONARIO
```

O fallback deve ser acionado quando a comunicacao local nao estiver disponivel ou quando o funcionario estiver fora do alcance da rede local, conforme a arquitetura.

## 13. COMUNICACAO DE RETORNO

O funcionario nao apenas recebe mensagens.

Ele também envia comandos.

Fluxo local:

```text
FUNCIONARIO
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

Caso seja necessaria comunicacao fora da rede local, represente o caminho de retorno correspondente pela infraestrutura de nuvem.

Se esse retorno nao estiver completamente especificado no documento, marque como **PROPOSTA DE CONSOLIDACAO** e explique a decisao.

## 14. IDEMPOTENCIA

Comandos criticos devem possuir identificador unico.

Utilize:

```text
CommandID
```

Antes de executar um comando:

1. verificar autenticidade;
2. verificar autorizacao;
3. verificar terminal;
4. verificar excecao;
5. verificar CommandID;
6. verificar estado atual;
7. verificar se o comando ja foi executado;
8. executar somente se for novo;
9. registrar resultado.

Se o mesmo comando chegar duas vezes, ele nao deve provocar duas execucoes.

## 15. DEDUPLICACAO

Mensagens e eventos podem ser retransmitidos.

O algoritmo deve impedir que uma mensagem duplicada provoque:

- dois desbloqueios;
- duas liberacoes;
- duas aprovacoes;
- duas finalizacoes;
- duas alteracoes financeiras;
- duas alteracoes de estoque.

Explique como o identificador da mensagem/transacao permite detectar duplicidade.

## 16. OPERACAO OFFLINE

O sistema deve respeitar a arquitetura Edge-first.

Quando a WAN estiver indisponivel:

- o terminal deve continuar funcionando localmente quando permitido;
- utilizar SQLite;
- utilizar WAL;
- armazenar eventos/transacoes pendentes;
- utilizar Outbox;
- utilizar UUIDv7;
- aguardar sincronizacao.

Quando a conexao retornar:

```text
OFFLINE
↓
DETECCAO DE ONLINE
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
DEDUPLICACAO
↓
PERSISTENCIA
↓
ACK
↓
ATUALIZACAO LOCAL
```

## 17. CENARIO 1 — DIVERGENCIA DE PESO / SWEETHEARTING

Utilize o cenario exatamente como definido no documento.

Gatilhos principais:

- Z > 2.5;
- objeto transferido sem scan;
- Optical Bypass;
- multiplos itens;
- retirada prematura quando aplicavel.

Fluxo:

```text
detectar
→ validar
→ LOCKED_EXCEPTION
→ congelar interface
→ desativar leitores
→ criar THEFT_PREVENTION_TRIGGER
→ publicar alerta
→ notificar funcionario
→ funcionario verificar item
→ corrigir
→ enviar RELEASE_TERMINAL
→ autenticar
→ autorizar
→ validar CommandID
→ executar
→ registrar auditoria
→ retornar ao estado operacional adequado.
```

## 18. CENARIO 2 — RESTRICAO ETARIA

Gatilho:

```text
AGE_VERIFICATION_REQUIRED
```

Fluxo:

```text
SKU identificado
→ verificar restricao
→ interromper inclusao
→ informar cliente
→ gerar AGE_GATE_REQUIRED
→ enviar alerta
→ funcionario recebe
→ verifica documento
→ envia APPROVE_AGE
→ autenticacao
→ autorizacao
→ validacao
→ execucao
→ desbloqueio
→ inclusao do item
→ auditoria.
```

## 19. CENARIO 3 — QUEDA DE WAN DURANTE PIX

Gatilhos:

- timeout superior a 15 segundos;
- socket fechado;
- estado WAITING_PIX_SETTLEMENT.

REGRA:

Nao transformar automaticamente a transacao em pagamento recusado.

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
→ notificar funcionario
→ apresentar txid
→ verificar liquidacao
→ confirmar resultado.
```

Se a comunicacao secundaria falhar:

permitir `MANUAL_OVERRIDE_PAYMENT` somente mediante validacao adequada e registrar auditoria.

## 20. CENARIO 4 — FALHA CRITICA DE HARDWARE

Gatilhos:

- perda de keep-alive da balanca;
- erro do driver de video;
- falha critica de periferico.

Fluxo:

```text
detectar
→ classificar
→ encerrar sessao de maneira segura
→ informar cliente
→ gerar HARDWARE_FAULT
→ informar manutencao
→ OUT_OF_SERVICE
→ preservar transacao pendente
→ gerar mecanismo de recuperacao
→ permitir continuidade em terminal adjacente
→ registrar auditoria.
```

## 21. CENARIO 5 — ABANDONO DE TERMINAL

Gatilhos:

- 60 segundos sem interacao;
- ausencia de cliente detectada.

Fluxo:

```text
detectar ausencia
→ aviso sonoro por 15 segundos
→ verificar retorno
→ se retornar, continuar
→ se nao retornar, bloquear
→ gerar SESSION_ABANDONED
→ incluir carrinho e valor
→ notificar funcionario
→ receber CLEAR_AND_RESET_SESSION
→ validar
→ limpar sessao
→ emitir etiqueta quando aplicavel
→ retornar IDLE
→ auditoria.
```

## 22. RECUPERACAO

Toda excecao deve possuir caminho de recuperacao.

Para cada cenario, informe:

- estado inicial;
- gatilho;
- estado durante excecao;
- mensagem;
- funcionario responsavel;
- comando;
- validacao;
- estado apos comando;
- condicao de sucesso;
- condicao de falha;
- recuperacao;
- auditoria.

## 23. SEGURANCA

Para comandos sensíveis:

1. autenticar funcionario;
2. verificar autorizacao;
3. verificar origem;
4. verificar terminal;
5. verificar excecao;
6. verificar estado;
7. verificar CommandID;
8. executar;
9. registrar auditoria.

Nao permita que uma mensagem recebida diretamente altere um estado critico sem validacao.

## 24. AUDITORIA

Registre pelo menos quando aplicável:

- TerminalID;
- SessionID;
- EventID;
- CommandID;
- timestamp;
- tipo da excecao;
- estado anterior;
- estado posterior;
- funcionario;
- comando;
- resultado;
- evidencia;
- erro.

A auditoria deve permitir reconstruir o que aconteceu.

## 25. FORMATO DOS ALGORITMOS

Utilize pseudocodigo estruturado.

Exemplo de estrutura:

```text
ALGORITMO NomeDoAlgoritmo

ENTRADA:
    ...

SAIDA:
    ...

VARIAVEIS:
    ...

INICIO

    ...

    SE condicao ENTAO
        ...
    SENAO
        ...
    FIM-SE

FIM
```

Nao utilize codigo Java, C, Python ou outra linguagem de programacao.

O objetivo e especificar o comportamento do sistema antes da implementacao.

## 26. EXPLICACAO DO PSEUDOCODIGO

Apos cada algoritmo importante, explique:

- objetivo;
- entradas;
- saidas;
- variaveis;
- decisoes;
- estados;
- eventos;
- mensagens;
- recuperacao.

Explique também como o algoritmo se conecta à arquitetura.

## 27. MATRIZ DE RASTREABILIDADE

Ao final, produza uma matriz:

| Elemento | Origem no documento | Implementacao no algoritmo | Observacao |
|----------|---------------------|----------------------------|------------|

Inclua:

- estados;
- mensagens;
- protocolos;
- componentes;
- excecoes;
- comandos;
- mecanismos de recuperacao.

## 28. MATRIZ DOS CINCO CENARIOS

Produza:

| Cenario | Gatilho | Estado | Mensagem | Funcionario | Comando | Recuperacao | Auditoria |
|----------|---------|--------|----------|-------------|---------|------------|-----------|

## 29. DECISOES DE CONSOLIDACAO

No final, crie uma secao:

**DECISOES ARQUITETURAIS DE CONSOLIDACAO**

Nela, registre somente aquilo que precisou ser definido para fechar lacunas do documento.

Exemplos:

- normalizacao de `ReleaseTerminal` para `RELEASE_TERMINAL`;
- uso de `CommandID`;
- separacao entre estado do terminal, comunicacao e excecao;
- caminho de retorno quando o funcionario estiver fora da rede local;
- estado `RECOVERING`, se necessario.

Nao apresente essas decisoes como se fossem originalmente escritas no documento.

## 30. VALIDACAO FINAL

Antes de concluir, faca uma verificacao:

```text
[ ] Documento foi considerado integralmente.
[ ] Algoritmo-mae foi desenvolvido antes da Parte 4.
[ ] Parte 4 deriva do algoritmo-mae.
[ ] Os cinco cenarios foram implementados.
[ ] Comunicacao terminal → funcionario foi contemplada.
[ ] Comunicacao funcionario → terminal foi contemplada.
[ ] Fallback foi contemplado.
[ ] Operacao offline foi contemplada.
[ ] Recuperacao foi contemplada.
[ ] Idempotencia foi contemplada.
[ ] Deduplicacao foi contemplada.
[ ] Auditoria foi contemplada.
[ ] Estados foram diferenciados.
[ ] Alertas foram diferenciados de comandos.
[ ] Comandos foram diferenciados de resolucao.
[ ] Lacunas foram identificadas.
[ ] Propostas foram separadas do conteudo original.
[ ] Nao foram inventados componentes sem identificacao.
[ ] O pseudocodigo esta estruturado.
[ ] O material pode ser posteriormente consolidado em README.md.
```

## 31. RESULTADO FINAL

A resposta final devera demonstrar uma cadeia logica completa:

```text
ARQUITETURA
↓
FUNCIONAMENTO GERAL
↓
ALGORITMO-MAE
↓
MENSAGERIA
↓
EXCECOES
↓
COMANDOS
↓
RECUPERACAO
↓
AUDITORIA
↓
VALIDACAO
```

Nao pule diretamente para o resultado final.

Construa o raciocinio progressivamente.

O objetivo nao e somente produzir um algoritmo que pareca correto.

O objetivo e produzir uma especificacao academica que possa ser compreendida, explicada, defendida e posteriormente implementada por alunos de Sistemas de Informacao.
```

---

## Fim do Desenvolvimento

**Parte 4 — Mensageria e Excecoes**

**Paradigma:** Edge-first  
**Modelo:** Mensageria assincrona e full-duplex  
**Representacao:** Pseudocodigo estruturado  
**Escopo:** Comunicacao, excecoes, fallback, recuperacao e auditoria