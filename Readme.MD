# 🛒 Documento Técnico de Arquitetura (DTA) — Parte 4
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
- [4. Especificação Técnica em Pseudocódigo C](#4-especificação-técnica-em-pseudocódigo-c)
  - [4.1 Estruturas de Dados (`structs` e `enums`)](#41-estruturas-de-dados-structs-e-enums)
  - [4.2 Engine Principal de Mensageria e Resiliência](#42-engine-principal-de-mensageria-e-resiliência)
  - [4.3 Central de Tratamento das 5 Exceções Mandatórias](#43-central-de-tratamento-das-5-exceções-mandatórias)
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

    subgraph CLOUD[" ☁️️ Nuvem Central (Cloud) "]
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
