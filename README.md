# APS 3 - Despacho de Missão com Checagem de Bateria

**Aluno:** Bladimir José Villarroel Guevara
**Disciplina:** Modelagem de Software - Prof. Sidney L. da Silva
**Projeto do grupo:** Sistema de Logística de Entregas Autônomas (AGVs)

## Sobre a funcionalidade

Escolhi o **despacho de missão**. É quando o operador pede para uma entrega ser feita e o sistema precisa escolher qual AGV vai fazer ela.

O ponto principal é a bateria: o AGV só pode receber a missão se a bateria dele for maior ou igual ao consumo estimado da rota + 20% de margem. Se não tiver bateria, ele é mandado para recarregar e o sistema tenta o próximo AGV. Se nenhum servir, a missão fica aguardando.

Depois que a missão é enviada, o AGV tem 10 segundos para confirmar. Se não confirmar, o AGV é liberado e a missão volta a aguardar.

Escolhi essa parte porque se ela der errado o AGV pode ficar sem bateria no meio do caminho ou dois AGVs podem ser mandados para a mesma entrega.

## Operações usadas no diagrama de sequência

Cada mensagem do diagrama de sequência é uma operação de quem recebe:

- **Aplicação Web:** `despacharEntrega(entregaId)`, `notificarStatus(MissaoResponseDTO)`
- **API de Orquestração:** `despachar(DespachoDTO)`, `onConfirmacao(ConfirmacaoDTO)`
- **Planejador de Rotas:** `calcularRota(RotaRequestDTO): RotaDTO`
- **Banco de Dados (repositório):** `criarMissao(entregaId)`, `buscarAgvsDisponiveis()`, `reservarAgv(agvId, missaoId, versao)`, `atualizarStatus(missaoId, status)`
- **Broker MQTT:** `publicarMissao(MissaoDTO)`, `enviarParaRecarga(agvId)`, `confirmarMissao(ConfirmacaoDTO)`
- **AGV:** `receberMissao(MissaoDTO)`

## 1. Diagrama de Sequência

`->>` é chamada síncrona, `-)` é mensagem assíncrona (pelo broker MQTT ou WebSocket) e `-->>` é retorno com DTO.

```mermaid
sequenceDiagram
    actor Operador
    participant Web as Aplicação Web
    participant API as API de Orquestração
    participant Planejador as Planejador de Rotas
    participant BD as Banco de Dados
    participant Broker as Broker MQTT
    participant AGV

    Operador->>Web: despacharEntrega(entregaId)
    Web->>API: despachar(DespachoDTO)
    API->>BD: criarMissao(entregaId)
    BD-->>API: MissaoDTO (status = CRIADA)

    API->>Planejador: calcularRota(RotaRequestDTO)
    Planejador-->>API: RotaDTO (distancia, consumoBateria)

    API->>BD: buscarAgvsDisponiveis()
    BD-->>API: lista de AgvDTO

    loop até reservar um AGV ou acabar a lista
        alt bateria >= consumoBateria + 20
            API->>BD: reservarAgv(agvId, missaoId, versao)
            BD-->>API: ReservaDTO (ok)
        else bateria < consumoBateria + 20
            API-)Broker: enviarParaRecarga(agvId)
        end
    end

    alt AGV reservado
        API->>BD: atualizarStatus(missaoId, DESPACHADA)
        API-)Broker: publicarMissao(MissaoDTO)
        API-->>Web: MissaoResponseDTO (status = DESPACHADA)
        Broker-)AGV: receberMissao(MissaoDTO)
        AGV-)Broker: confirmarMissao(ConfirmacaoDTO)
        Broker-)API: onConfirmacao(ConfirmacaoDTO)
        API->>BD: atualizarStatus(missaoId, EM_EXECUCAO)
        API-)Web: notificarStatus(MissaoResponseDTO)
    else nenhum AGV com bateria
        API->>BD: atualizarStatus(missaoId, AGUARDANDO_AGV)
        API-->>Web: MissaoResponseDTO (status = AGUARDANDO_AGV)
    end

    Web-->>Operador: mostra o status da missão
```

## 2. Diagrama de Atividades

Raias: Operador, API de Orquestração, Planejador de Rotas e AGV.

```mermaid
flowchart TB
    subgraph R1[Operador]
        I((início))
        A1[Solicitar despacho da entrega]
        A2[Ver status da missão]
        F((fim))
    end

    subgraph R2[API de Orquestração]
        B1[Criar missão]
        FORK[ ]:::barra
        B2[Buscar AGVs disponíveis]
        JOIN[ ]:::barra
        B3[Pegar próximo AGV da lista]
        D1{Bateria suficiente?}
        B4[Reservar AGV]
        B5[Mandar AGV para recarga]
        D2{Tem mais AGV na lista?}
        B6[Marcar missão como AGUARDANDO_AGV]
        B7[Publicar missão no broker]
        D3{AGV confirmou?}
        B8[Missão EM_EXECUCAO]
        B9[Liberar AGV]
    end

    subgraph R3[Planejador de Rotas]
        C1[Calcular rota e consumo de bateria]
    end

    subgraph R4[AGV]
        G1[Receber missão e confirmar]
    end

    I --> A1 --> B1 --> FORK
    FORK --> C1 --> JOIN
    FORK --> B2 --> JOIN
    JOIN --> B3 --> D1
    D1 -->|"[bateria >= consumo + 20]"| B4 --> B7
    D1 -->|"[bateria < consumo + 20]"| B5 --> D2
    D2 -->|"[sim]"| B3
    D2 -->|"[não]"| B6 --> A2
    B7 --> G1 --> D3
    D3 -->|"[confirmou em até 10s]"| B8 --> A2
    D3 -->|"[passou de 10s]"| B9 --> B6
    A2 --> F

    classDef barra fill:#000,stroke:#000,color:#000
```

## 3. Diagrama de Estados (MissaoAGV)

```mermaid
stateDiagram-v2
    [*] --> Criada

    Criada --> Alocacao : despachar()
    Criada --> Cancelada : cancelar() [usuario == ADMIN]

    state Alocacao {
        [*] --> BuscandoAGV
        BuscandoAGV --> AGVReservado : agvEncontrado [bateria >= consumo + 20] / reservarAgv()
        BuscandoAGV --> AguardandoAGV : semAgvDisponivel / agendarNovaBusca()
        AguardandoAGV --> BuscandoAGV : timer [tentativas < 3]
    }

    AguardandoAGV --> Cancelada : timer [tentativas >= 3] / avisarOperador()
    AGVReservado --> Despachada : publicarMissao()

    state "Despachada" as Despachada
    Despachada : entry / iniciarTimer(10s)
    Despachada : exit / pararTimer()

    Despachada --> EmExecucao : confirmacaoRecebida / registrarInicio()
    Despachada --> AguardandoAGV : timeout / liberarAgv()

    EmExecucao --> Concluida : entregaFinalizada / liberarAgv()
    EmExecucao --> Falha : agvOffline

    Concluida --> [*]
    Cancelada --> [*]
    Falha --> [*]
```

Alguns cuidados para a missão não ficar em estado errado:
- uma missão em execução não pode ser cancelada direto, ela só termina como Concluída ou Falha;
- se a confirmação do AGV chegar depois do timeout, ela é ignorada porque a missão já voltou para AguardandoAGV;
- a reserva do AGV usa a coluna `versao` (ver ADR 0001).

## 4. C4 - Nível 3 (Componentes)

```mermaid
flowchart TB
    op["Operador Logístico<br/>[Pessoa]"]
    web["Aplicação Web<br/>[Container: React]"]

    subgraph api[API de Orquestração - Container: Node.js + NestJS]
        ctrl["DespachoController<br/>[Componente: REST Controller]"]
        svc["DespachoService<br/>[Componente: regra da bateria]"]
        repo["MissaoRepository<br/>[Componente: TypeORM]"]
        cliente["PlanejadorClient<br/>[Componente: cliente gRPC]"]
        mqttpub["MqttService<br/>[Componente: MQTT.js]"]
    end

    planejador["Planejador de Rotas<br/>[Container: Python]"]
    bd[("Banco de Dados<br/>[Container: PostgreSQL]")]
    broker["Broker MQTT<br/>[Container: Mosquitto]"]
    agv["AGVs<br/>[Sistema externo]"]

    op -->|usa| web
    web -->|"HTTPS / REST (JSON)"| ctrl
    ctrl --> svc
    svc --> repo
    svc --> cliente
    svc --> mqttpub
    repo -->|SQL| bd
    cliente -->|gRPC| planejador
    mqttpub -->|MQTT| broker
    broker -->|MQTT| agv
    svc -->|"WebSocket (status)"| web

    classDef pessoa fill:#08427b,color:#fff
    classDef container fill:#1168bd,color:#fff
    classDef comp fill:#85bbf0,color:#000
    classDef ext fill:#999,color:#fff
    class op pessoa
    class web,planejador,bd,broker container
    class ctrl,svc,repo,cliente,mqttpub comp
    class agv ext
```

## 5. ADR

[ADR 0001 - Usar bloqueio otimista para reservar o AGV](docs/architecture/adr-0001.md)

## Ligação entre os diagramas

| Atividade | Sequência | Estado |
|---|---|---|
| Criar missão | `criarMissao()` | Criada |
| Calcular rota / buscar AGVs (fork) | `calcularRota()` e `buscarAgvsDisponiveis()` | Alocacao |
| Reservar AGV | `reservarAgv()` | BuscandoAGV → AGVReservado |
| Publicar missão | `publicarMissao()` | AGVReservado → Despachada |
| AGV confirmou | `onConfirmacao()` | Despachada → EmExecucao |
| Marcar AGUARDANDO_AGV | `atualizarStatus(AGUARDANDO_AGV)` | AguardandoAGV |
