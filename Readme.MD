Markdown# 🛒 Documento Técnico de Arquitetura (DTA) — Parte 4
> **Projeto:** Supermercado Autônomo (*Grab-and-Go*)  
> **Curso:** Bacharelado em Sistemas de Informação  
> **Módulo:** Mensageria Distribuída, Resiliência e Tratamento de Exceções  

![Edge Computing](https://img.shields.io/badge/Edge-MQTT%20Broker-orange?style=flat-for-the-badge)
![Cloud Pipeline](https://img.shields.io/badge/Cloud-Apache%20Kafka-blue?style=flat-for-the-badge)
![Security Standard](https://img.shields.io/badge/Security-mTLS%201.3%20%7C%20PCI--DSS-green?style=flat-for-the-badge)
![Resilience Protocol](https://img.shields.io/badge/Resilience-Store--and--Forward-red?style=flat-for-the-badge)

---

## 📋 Sumário
- [1. Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
- [2. Diagrama de Fluxo e Resiliência](#2-diagrama-de-fluxo-e-resiliência)
- [3. O Algoritmo-Mãe (Metodologia)](#3-o-algoritmo-mãe-metodologia)
  - [3.1 Entradas e Saídas](#31-entradas-e-saídas)
  - [3.2 Passos do Método](#32-passos-do-método)
  - [3.3 Matriz de Validação](#33-matriz-de-validação)
- [4. Especificação Técnica Completa em Pseudocódigo C](#4-especificação-técnica-completa-em-pseudocódigo-c)
  - [4.1 Cabeçalhos e Definições Globais](#41-cabeçalhos-e-definições-globais)
  - [4.2 Estruturas de Dados e Tipos (`structs` e `enums`)](#42-estruturas-de-dados-e-tipos-structs-e-enums)
  - [4.3 Assinaturas de Funções Auxiliares](#43-assinaturas-de-funções-auxiliares)
  - [4.4 Central de Tratamento das 5 Exceções](#44-central-de-tratamento-das-5-exceções)
  - [4.5 Core Engine de Mensageria e Resiliência](#45-core-engine-de-mensageria-e-resiliência)
  - [4.6 Worker de Sincronização em Segundo Plano](#46-worker-de-sincronização-em-segundo-plano)
  - [4.7 Ponto de Entrada do Sistema (`main`)](#47-ponto-de-entrada-do-sistema-main)
- [5. Matriz de Exceções e Ações Fail-Safe](#5-matriz-de-exceções-e-ações-fail-safe)

---

## 1. Visão Geral da Arquitetura

Para responder aos requisitos de **latência ultra-baixa** dentro da loja e **sincronização global** com os serviços centrais, a arquitetura de mensageria utiliza uma topologia híbrida e resiliente em duas camadas:

1. **Camada Local (Edge):** Broker MQTT rodando diretamente na infraestrutura física da loja. Recebe eventos das células de carga (prateleiras) e coordenadas das câmeras de visão computacional em submilissegundos.
2. **Camada Global (Cloud):** Cluster Apache Kafka que atua como barramento central para histórico de compras, sincronização de estoque, auditoria e comunicação com a adquirente de cartão.
3. **Mecanismo de Resiliência (Store-and-Forward):** Banco de dados SQLite criptografado via **SQLCipher (AES-256)** embutido na borda. Quando o link de internet com a nuvem cai, a loja continua operando offline. Todas as transações são enfileiradas localmente e descarregadas na nuvem assim que a conexão restabelecer.

> [!IMPORTANT]
> **Diretriz Fail-Safe:** O sistema prioriza a integridade das operações e a segurança do estabelecimento. Em qualquer dúvida lógica, falha de autenticação ou estouro de tempo (timeout), a ordem padrão do sistema é manter as catracas travadas e suspender a sessão até a validação.

---

## 2. Diagrama de Fluxo e Resiliência

```mermaid
flowchart TD
    subgraph EDGE[" 🏪 Infraestrutura Local (Loja / Edge) "]
        SENSORS["Câmeras & Células de Carga"] -->|Payload Leve| MQTT["Broker MQTT Local"]
        MQTT -->|Evento Capturado| ENGINE{"Link WAN Online?"}
        
        ENGINE -->|Sim| SYNC["Enviar para o Kafka"]
        ENGINE -->|Não / Instável| STORE["Salvar no SQLCipher (Store-and-Forward)"]
        
        STORE --> WORKER["Worker de Sincronização"]
        WORKER -->|Checa Conexão (polling 5s)| ENGINE
    end

    subgraph CLOUD[" ☁️ Nuvem Central (Cloud) "]
        SYNC --> KAFKA["Apache Kafka Cluster"]
        KAFKA --> PROCESSOR["Microsserviços de Processamento"]
        PROCESSOR --> DB[("Banco de Dados Central")]
    end

    subgraph HANDLER[" 🚨 Central de Exceções "]
        EX1["1. Queda WAN"] --> STORE
        EX2["2. Divergência Peso/Visão"] --> PAUSE["Pausa Carrinho + Alerta Operador"]
        EX3["3. Timeout Catraca"] --> LOCK1["Bloqueia Catraca de Entrada"]
        EX4["4. Falha mTLS"] --> DROP["Encerra Socket + Registro de Segurança"]
        EX5["5. Recusa no Pagamento"] --> LOCK2["Trava Catraca de Saída + Notificação App"]
    end
3. O Algoritmo-Mãe (Metodologia)O Algoritmo-Mãe é o método sequencial que orienta o projeto da camada de mensageria, garantindo padronização desde o modelo de dados até os cenários de falha.3.1 Entradas e SaídasTipoElementoDescriçãoEntradaTopologiaDefinição dos brokers (MQTT local e Kafka Cloud)EntradaContratosEstrutura dos payloads e tipos de eventosEntradaMatriz de FalhasMapeamento das exceções operacionais e de segurançaSaídaData ContractsTipos de dados (structs e enums) definidos em CSaídaCore EngineRotina de publicação com tratamento de fallbackSaídaHandlersFunções isoladas para tratamento dos 5 cenários de erro3.2 Passos do MétodoSnippet de códigogantt
    title Execução do Algoritmo-Mãe
    dateFormat X
    axisFormat %s
    section Etapas
    1. Modelagem de Dados         :active, p1, 0, 1
    2. Canal Local (MQTT)          :p2, 1, 2
    3. Engine Offline (SQLCipher)  :p3, 2, 3
    4. Trato de Exceções           :p4, 3, 4
    5. Sync Worker (Flush)         :p5, 4, 5
Mapeamento dos Contratos: Definir as estruturas de dados para as mensagens da borda e da nuvem.Desenvolvimento do Canal Principal: Implementar o envio no broker MQTT local para garantir baixíssima latência.Construção do Mecanismo de Fallback: Criar o desvio automático para o banco de dados SQLCipher caso a rede WAN falhe.Implementação da Central de Exceções: Desenvolver as regras de contenção para cada um dos 5 cenários críticos do projeto.Criação da Rotina de Sincronização: Implementar o worker em segundo plano para descarregar (flush) o banco local para a nuvem quando a internet reestabelecer.3.3 Matriz de Validação[x] Comunicação de Borda: Payload leve via MQTT especificado.[x] Fallback Transacional: Gravação offline em banco criptografado (SQLCipher) funcional.[x] Tratamento de Exceções: 100% dos 5 cenários mapeados e tratados individualmente.[x] Sintaxe do Código: Pseudocódigo em C limpo, estruturado, tipado e legível.[x] Política de Segurança: Modelo Fail-Safe aplicado em todas as falhas físicas e de software.4. Especificação Técnica Completa em Pseudocódigo CC/* ============================================================================
 * PROJETO: SUPERMERCADO AUTÔNOMO (DTA PARTE 4)
 * MÓDULO: MENSAGERIA, RESILIÊNCIA E TRATAMENTO DE EXCEÇÕES
 * LINGUAGEM: PSEUDOCÓDIGO ESTRUTURADO EM C
 * ============================================================================ */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <unistd.h>

/* ----------------------------------------------------------------------------
 * 4.1 CABEÇALHOS E DEFINIÇÕES GLOBAIS
 * ---------------------------------------------------------------------------- */
#define SUCESSO                  0
#define ERRO_GENERICO           -1
#define AVISO_MODO_OFFLINE       1

#define REDE_OFFLINE             0
#define REDE_ONLINE              1

#define CATRACA_LIBERADA         1
#define CATRACA_BLOQUEADA        0

#define MODO_OPERACIONAL_NORMAL  100
#define MODO_DEGRADADO_LOCAL     101

/* ----------------------------------------------------------------------------
 * 4.2 ESTRUTURAS DE DADOS E TIPOS (STRUCTS E ENUMS)
 * ---------------------------------------------------------------------------- */

/* Tipos de canais de comunicação disponíveis no sistema */
typedef enum {
    CANAL_LOCAL_MQTT = 1,
    CANAL_NUVEM_KAFKA = 2,
    ARMAZENAMENTO_LOCAL_SQLCIPHER = 3
} TipoCanal;

/* Origem do evento capturado pelos sensores da loja */
typedef enum {
    TIPO_EVENTO_PESO = 1,      /* Célula de Carga na prateleira */
    TIPO_EVENTO_VISAO = 2,     /* Smart Camera com IA */
    TIPO_EVENTO_CARRINHO = 3   /* Atualização do carrinho do cliente */
} TipoEvento;

/* Estrutura leve para tráfego local na borda (MQTT) */
typedef struct {
    char id_mensagem[36];
    char id_loja[16];
    TipoEvento tipo;
    char id_dispositivo[32];   
    float valor_delta_gramas;  
    float coordenadas_3d[3];   
    unsigned long timestamp_ms;
    char assinatura_hmac[65];  /* Assinatura HMAC-SHA256 de hardware */
} MensagemMQTTLocal;

/* Estrutura consolidada para o barramento na Nuvem (Kafka) */
typedef struct {
    char id_transacao[36];
    char id_sessao_usuario[36];
    char id_loja[16];
    char sku_produto[32];
    int quantidade;
    float preco_unitario;
    unsigned long timestamp_ms;
    char token_pagamento[128]; /* Token de Pagamento PCI-DSS */
} MensagemKafkaNuvem;

/* Catálogo das 5 exceções previstas no projeto */
typedef enum {
    EXCECAO_NENHUMA = 0,
    EXCECAO_FALHA_CONEXAO_WAN = 1,      /* 1. Queda de link com a Cloud */
    EXCECAO_DISCREPANCIA_PESO_VISAO = 2, /* 2. Divergência entre balança e câmera */
    EXCECAO_TIMEOUT_CATRACA = 3,         /* 3. Cliente não passou a tempo */
    EXCECAO_FALHA_AUTENTICACAO_MTLS = 4, /* 4. Falha no certificado X.509 */
    EXCECAO_ERRO_TOKEN_PAGAMENTO = 5    /* 5. Cartão recusado / Token inválido */
} CodigoExcecao;

/* Estruturas de contexto para os tratadores de erro */
typedef struct {
    char id_sessao[36];
    char id_usuario[36];
    int total_itens;
} SessaoCarrinho;

typedef struct {
    int socket_fd;
    char ip_origem[45];
    char id_dispositivo[32];
} DispositivoConexao;

/* ----------------------------------------------------------------------------
 * 4.3 ASSINATURAS E MOCKS DE FUNÇÕES AUXILIARES DE INFRAESTRUTURA
 * ---------------------------------------------------------------------------- */
void LogWarning(const char* mensagem) { printf("[WARNING] %s\n", mensagem); }
void LogError(const char* mensagem)   { printf("[ERROR]   %s\n", mensagem); }
void LogInfo(const char* mensagem)    { printf("[INFO]    %s\n", mensagem); }

int VerificarStatusConexaoWAN() {
    /* Simula verificação de conectividade com a nuvem (1 = Online, 0 = Offline) */
    return REDE_ONLINE; 
}

int PublicarMQTTLocal(const char* topico, MensagemMQTTLocal* msg) {
    /* Envio simulado para o broker MQTT da loja */
    return SUCESSO;
}

int EnviarParaKafkaCloud(MensagemMQTTLocal* msg) {
    /* Encaminhamento simulado do evento para o Kafka na nuvem */
    return SUCESSO;
}

void SalvarNoBancoLocalSQLCipher(MensagemMQTTLocal* msg) {
    LogInfo("Dados gravados com sucesso no SQLite/SQLCipher local (AES-256).");
}

int ObterQuantidadeMensagensPendentesSQLCipher() {
    return 0; /* Quantidade de mensagens acumuladas offline */
}

MensagemMQTTLocal LerProximaMensagemSQLCipher() {
    MensagemMQTTLocal msg;
    strcpy(msg.id_mensagem, "MSG-LOCAL-OFFLINE-001");
    return msg;
}

void RemoverMensagemSQLCipher(const char* id_mensagem) {
    /* Remove registro do banco offline após sincronização com a nuvem */
}

void AtualizarStatusOperacionalLoja(int modo) {
    if (modo == MODO_DEGRADADO_LOCAL) {
        LogWarning("Loja operando em MODO DEGRADADO LOCAL.");
    }
}

void PausarSessaoCarrinho(const char* id_sessao) {
    LogWarning("Sessão do carrinho pausada para análise.");
}

void NotificarPainelOperador(const char* motivo, const char* id_sessao) {
    LogInfo("Alerta enviado ao painel do operador humano.");
}

void CancelarSessaoEntradaCatraca(const char* id_sessao) {
    LogWarning("Sessão temporária de entrada cancelada.");
}

void ComandarCatraca(int estado) {
    if (estado == CATRACA_BLOQUEADA) {
        LogWarning("CATRACA DE ENTRADA: BLOQUEADA (Fail-Safe).");
    }
}

void ComandarCatracaSaida(const char* id_loja, int estado) {
    if (estado == CATRACA_BLOQUEADA) {
        LogError("CATRACA DE SAÍDA: TRAVADA (Aguardando Pagamento).");
    }
}

void EncerrarConexaoSocket(int socket_fd) {
    LogWarning("Socket TCP desconectado sumariamente.");
}

void RegistrarEventoAuditoria(const char* evento, const char* ip) {
    LogInfo("Evento registrado no log imutável de auditoria de segurança.");
}

void NotificarAppCliente(const char* id_usuario, const char* mensagem) {
    LogInfo("Notificação Push enviada para o smartphone do cliente.");
}

void PausarExecucao(int milissegundos) {
    usleep(milissegundos * 1000);
}

/* ----------------------------------------------------------------------------
 * 4.4 CENTRAL DE TRATAMENTO DAS 5 EXCEÇÕES MANDATÓRIAS
 * ---------------------------------------------------------------------------- */
void TratarExcecao(CodigoExcecao codigo, void* dados_contexto) {
    switch (codigo) {

        /* --------------------------------------------------------------------
         * EXCEÇÃO 1: Queda de Conexão WAN (Loja Offline)
         * -------------------------------------------------------------------- */
        case EXCECAO_FALHA_CONEXAO_WAN:
            LogWarning("EXCEÇÃO [1]: Conexão WAN perdida. Ativando Store-and-Forward.");
            MensagemMQTTLocal* msg_offline = (MensagemMQTTLocal*)dados_contexto;
            
            SalvarNoBancoLocalSQLCipher(msg_offline);
            AtualizarStatusOperacionalLoja(MODO_DEGRADADO_LOCAL);
            break;

        /* --------------------------------------------------------------------
         * EXCEÇÃO 2: Discrepância Peso vs. Visão Computacional
         * -------------------------------------------------------------------- */
        case EXCECAO_DISCREPANCIA_PESO_VISAO:
            LogError("EXCEÇÃO [2]: Divergência detectada entre Célula de Carga e Câmera.");
            SessaoCarrinho* sessao = (SessaoCarrinho*)dados_contexto;
            
            PausarSessaoCarrinho(sessao->id_sessao);
            NotificarPainelOperador("Divergência de peso/visão na prateleira", sessao->id_sessao);
            break;

        /* --------------------------------------------------------------------
         * EXCEÇÃO 3: Timeout na Passagem da Catraca (<100ms)
         * -------------------------------------------------------------------- */
        case EXCECAO_TIMEOUT_CATRACA:
            LogWarning("EXCEÇÃO [3]: Tempo limite de passagem na catraca excedido.");
            
            CancelarSessaoEntradaCatraca((char*)dados_contexto);
            ComandarCatraca(CATRACA_BLOQUEADA);
            break;

        /* --------------------------------------------------------------------
         * EXCEÇÃO 4: Falha no Handshake mTLS (Certificado X.509 Inválido)
         * -------------------------------------------------------------------- */
        case EXCECAO_FALHA_AUTENTICACAO_MTLS:
            LogError("EXCEÇÃO [4]: Certificado mTLS inválido! Bloqueando dispositivo.");
            DispositivoConexao* dev = (DispositivoConexao*)dados_contexto;
            
            EncerrarConexaoSocket(dev->socket_fd);
            RegistrarEventoAuditoria("FALHA_AUTENTICACAO_MTLS", dev->ip_origem);
            break;

        /* --------------------------------------------------------------------
         * EXCEÇÃO 5: Erro ou Recusa no Token de Pagamento (PCI-DSS)
         * -------------------------------------------------------------------- */
        case EXCECAO_ERRO_TOKEN_PAGAMENTO:
            LogError("EXCEÇÃO [5]: Pagamento recusado durante o encerramento da compra.");
            MensagemKafkaNuvem* msg_checkout = (MensagemKafkaNuvem*)dados_contexto;
            
            ComandarCatracaSaida(msg_checkout->id_loja, CATRACA_BLOQUEADA);
            NotificarAppCliente(msg_checkout->id_sessao_usuario, "Pagamento recusado. Atualize o cartão no app.");
            break;

        default:
            LogWarning("Código de exceção não catalogado recebido.");
            break;
    }
}

/* ----------------------------------------------------------------------------
 * 4.5 CORE ENGINE DE MENSAGERIA E RESILIÊNCIA
 * ---------------------------------------------------------------------------- */
int EnviarMensagemBorda(MensagemMQTTLocal msg) {
    int status = PublicarMQTTLocal("loja/sensores/eventos", &msg);

    if (status == SUCESSO) {
        /* Se a internet estiver ok, envia uma cópia pro Kafka na nuvem */
        if (VerificarStatusConexaoWAN() == REDE_ONLINE) {
            EnviarParaKafkaCloud(&msg);
        } else {
            /* Se caiu a WAN, aciona o salvamento offline via exceção */
            TratarExcecao(EXCECAO_FALHA_CONEXAO_WAN, &msg);
        }
        return SUCESSO;
    } else {
        /* Se até o broker local falhar, grava direto no banco interno */
        SalvarNoBancoLocalSQLCipher(&msg);
        return AVISO_MODO_OFFLINE;
    }
}

/* ----------------------------------------------------------------------------
 * 4.6 WORKER DE SINCRONIZAÇÃO EM SEGUNDO PLANO (FLUSH OFFLINE)
 * ---------------------------------------------------------------------------- */
void WorkerSincronizacaoOffline() {
    LogInfo("Iniciando Worker de Sincronização Store-and-Forward...");
    
    if (VerificarStatusConexaoWAN() == REDE_ONLINE) {
        int pendentes = ObterQuantidadeMensagensPendentesSQLCipher();
        
        while (pendentes > 0) {
            MensagemMQTTLocal msg = LerProximaMensagemSQLCipher();
            
            if (EnviarParaKafkaCloud(&msg) == SUCESSO) {
                RemoverMensagemSQLCipher(msg.id_mensagem);
                pendentes--;
            } else {
                /* Se a rede oscilar, interrompe e aguarda a próxima janela */
                break; 
            }
        }
    }
}

/* ----------------------------------------------------------------------------
 * 4.7 PONTO DE ENTRADA DO SISTEMA (MAIN)
 * ---------------------------------------------------------------------------- */
int main() {
    LogInfo("=================================================");
    LogInfo(" INICIALIZANDO SISTEMA DE MENSAGERIA E BORDA     ");
    LogInfo("=================================================");

    /* 1. Criando evento simulado de leitura de peso */
    MensagemMQTTLocal evento_peso;
    strcpy(evento_peso.id_mensagem, "MSG-88392-A");
    strcpy(evento_peso.id_loja, "LOJA-SP-001");
    evento_peso.tipo = TIPO_EVENTO_PESO;
    strcpy(evento_peso.id_dispositivo, "BALANCA-B3");
    evento_peso.valor_delta_gramas = 250.0f;
    evento_peso.timestamp_ms = 1700000000;
    strcpy(evento_peso.assinatura_hmac, "a3b1c8e9f...");

    /* 2. Processando o evento no fluxo principal */
    LogInfo("Processando envio do evento principal...");
    EnviarMensagemBorda(evento_peso);

    /* 3. Simulando a ocorrência da Exceção 2 (Divergência Peso/Visão) */
    LogInfo("Simulando gatilho da Exceção 2 (Inconsistência de Item)...");
    SessaoCarrinho sessao_alerta;
    strcpy(sessao_alerta.id_sessao, "SESSAO-USER-99");
    TratarExcecao(EXCECAO_DISCREPANCIA_PESO_VISAO, &sessao_alerta);

    /* 4. Simulando a ocorrência da Exceção 5 (Erro no Pagamento) */
    LogInfo("Simulando gatilho da Exceção 5 (Recusa de Cartão)...");
    MensagemKafkaNuvem checkout_recusado;
    strcpy(checkout_recusado.id_loja, "LOJA-SP-001");
    strcpy(checkout_recusado.id_sessao_usuario, "USER-7721");
    TratarExcecao(EXCECAO_ERRO_TOKEN_PAGAMENTO, &checkout_recusado);

    /* 5. Executa ciclo do worker de sincronização offline */
    WorkerSincronizacaoOffline();

    LogInfo("=================================================");
    LogInfo(" EXECUÇÃO CONCLUÍDA COM SUCESSO                 ");
    LogInfo("=================================================");

    return 0;
}
5. Matriz de Exceções e Ações Fail-SafeIDExceçãoGatilhoImpactoAção de Mitigação (Fail-Safe)1FALHA_CONEXAO_WANPerda de sinal com a nuvemPerda de sincronia globalEntra no modo Store-and-Forward e salva localmente no SQLCipher.2DISCREPANCIA_PESO_VISAO$\Delta \text{peso} \neq \text{item visto}$Risco de inconsistência/perdaPausa o carrinho e notifica o painel do operador para checagem.3TIMEOUT_CATRACATempo na catraca $> 100\text{ms}$Liberação pendenteCancela a sessão de entrada e mantém a catraca travada.4FALHA_AUTENTICACAO_MTLSCertificado X.509 rejeitadoAcesso não autorizadoDerruba o socket da conexão e gera alerta de auditoria de segurança.5ERRO_TOKEN_PAGAMENTOTransação recusada no gatewayRisco de inadimplênciaMantém a catraca de saída bloqueada e envia notificação no app do cliente.💻 Desenvolvido para compor a Parte 4 do Documento Técnico de Arquitetura (DTA).Sinta-se à vontade para clonar, adaptar e utilizar na sua apresentação acadêmica ou documentação de projeto.
