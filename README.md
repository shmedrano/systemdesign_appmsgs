# System Design: Aplicativo de Mensagens Multiplataforma

25 de maio de 2026

---

## 1. Visão Geral

Este repositório contém a documentação completa do System Design para um Aplicativo de Mensagens Multiplataforma, desenvolvido como solução para o desafio de projetar um sistema capaz de enviar e receber mensagens com suporte a anexos, notificações push, status de presença (online/offline) e confirmação de leitura. O aplicativo deve funcionar em computadores, tablets e celulares, sendo acessado via navegador (Web/PWA).

A arquitetura proposta é **Event-Driven** (Baseada em Eventos) e baseada em microsserviços, garantindo escalabilidade horizontal, baixa latência, alta disponibilidade e segurança desde a concepção. O documento detalha cada componente técnico, desde a autenticação até a entrega confiável de mensagens e o gerenciamento de arquivos.

---

## 2. Requisitos do Sistema

### 2.1 Requisitos Funcionais

- Envio e recebimento de mensagens individuais e em grupos, com ordenação cronológica precisa.
- Suporte a anexos: imagens, vídeos, documentos e arquivos diversos, com upload seguro.
- Notificações push para dispositivos móveis e web quando o usuário estiver offline.
- Status de presença: exibir se um contato está online, offline ou ausente.
- Confirmação de leitura (*read receipts*): indicar se a mensagem foi entregue e lida.
- Histórico de mensagens completo e pesquisável.

### 2.2 Requisitos Não‑Funcionais

| Requisito | Descrição 
| Alta disponibilidade | SLA de 99,99%, sem ponto único de falha. |

---

## 3. Arquitetura Proposta

A arquitetura adota um modelo baseado em eventos com microsserviços desacoplados. Cada funcionalidade principal é isolada em seu próprio serviço, comunicando‑se através de um Message Broker (Apache Kafka) para garantir resiliência e escalabilidade.

### 3.1 Fluxo Geral

1. O usuário acessa o PWA que se conecta ao backend via WebSocket para comunicação em tempo real.
2. A autenticação é feita via JWT validado por um Gateway.
3. Mensagens, anexos, status de presença e confirmações de leitura fluem por filas assíncronas (Kafka), garantindo que o sistema não perca dados.
4. Dados estruturados são armazenados em DynamoDB (NoSQL) e arquivos em S3 com Presigned URLs.
5. Notificações push são disparadas via FCM/APNs quando o destinatário está offline.

---

## 4. Componentes Técnicos (Passo a Passo)

### Passo 1: Frontend & Autenticação

**Tecnologias:** React / Next.js (PWA), Service Workers, JWT.

O frontend é construído como uma Progressive Web App (PWA) para funcionar em qualquer dispositivo com navegador moderno. O Service Worker permite cache de assets e notificações push nativas.

- Autenticação: O usuário realiza login via formulário seguro (HTTPS). O servidor gera um JWT (JSON Web Token) contendo o ID do usuário e escopos de autorização. O token é armazenado no `localStorage` (web) ou `AsyncStorage` (mobile) e enviado em todo request como `Authorization: Bearer `.
- Sessão Stateless: O servidor não mantém estado de sessão; o JWT é autossuficiente, permitindo escalabilidade horizontal sem compartilhamento de sessão.

### Passo 2: Comunicação em Tempo Real

Tecnologias: WebSockets (Socket.IO), Load Balancer com Sticky Sessions (ou Kubernetes Ingress).

Para garantir latência ultrabaixa, a comunicação entre clientes e servidores é feita via WebSockets. O handshake inicial valida o JWT. Cada cliente mantém uma conexão persistente.

- O servidor de WebSocket (Gateway) roteia mensagens para o destinatário correto.
- Em caso de desconexão, o cliente tenta reconectar com backoff exponencial (evita sobrecarga).
- Para grupos com muitos participantes, o servidor utiliza fan-out otimizado: publica a mensagem em um tópico Kafka e cada servidor de WebSocket consome a fila para enviar apenas aos seus clientes conectados.

### Passo 3: Serviço de Presença

Tecnologias: Redis (com TTL), Heartbeats, Redis Pub/Sub.

O serviço de presença determina se um usuário está online sem necessidade de polling.

- Conexão aberta: Ao receber o evento `onopen` do WebSocket, o servidor executa `SET user:presence:{userID} "online" EX 30` no Redis.
- Heartbeat: O cliente envia um sinal leve a cada 20 segundos via WebSocket. O servidor apenas renova o TTL (`EXPIRE user:presence:{userID} 30`).
- Desconexão: No evento `onclose`, o servidor deleta a chave imediatamente. Se a conexão cair abruptamente, o Redis expira a chave após 30 segundos, marcando o usuário como offline sem custo computacional.
- Fan-out de status: Toda vez que o status de um usuário muda, o servidor publica um evento no Redis Pub/Sub. Os servidores WebSocket que mantêm conexões com os amigos daquele usuário consomem e enviam a atualização.

### Passo 4: Fluxo de Mensagens

Tecnologias: WebSockets, Apache Kafka, DynamoDB / Cassandra.

O fluxo de mensagens é dividido em duas etapas: ingestão/persistência (caminho crítico) e distribuição (caminho assíncrono).

4.1 Ingestão e Persistência
1. O Usuário A envia uma mensagem via WebSocket.
2. O Serviço de Grava Msg e Anexos gera um Snowflake ID (ordenável e único).
3. A mensagem é persistida no DynamoDB (tabela `messages` com partition key = `chatID`, sort key = `messageID`).
4. O servidor responde ao User 1 com um ACK confirmando o recebimento e salvamento.

4.2 Distribuição via Message Broker
1. Ao mesmo tempo da persistência, a mensagem é publicada no tópico Kafka `chat-messages`.
2. O Serviço Envia msgs consome do Kafka e consulta o Redis (presença):
   - Se online: envia a mensagem diretamente via WebSocket para o destinatário. O app do destinatário responde com um ACK de entrega.
   - Se offline: publica em outro tópico para disparar notificação push.

### Passo 5: Gerenciamento de Anexos

Tecnologias: AWS S3, Presigned URLs, VPC Endpoints, IAM.

- Upload: O cliente solicita ao backend uma **Presigned URL** com permissão de escrita e tempo de expiração curto (5 minutos). O backend, rodando dentro de uma VPC, gera a URL via SDK da AWS. O cliente faz upload diretamente para o bucket S3.
- Download: O cliente solicita uma Presigned URL de leitura para exibir o anexo na conversa.
- Segurança: O bucket S3 é privado, acessível apenas via VPC Endpoint. URLs assinadas garantem que apenas usuários autenticados possam acessar o arquivo.
- Integridade: Apenas o link do arquivo (URL) é enviado na mensagem, mantendo a mensagem leve.

### Passo 6: Notificações e Confirmação de Leitura

Tecnologias: FCM, APNs, Kafka (tópicos separados), Redis (buffer de eventos).

6.1 Notificações Push
Quando o Serviço de Entrega detecta que o destinatário está offline, ele publica um evento em um tópico Kafka `push-notifications`. Um consumidor dedicado consulta o Firebase Cloud Messaging (FCM) para Android/Web e Apple Push Notification Service (APNs) para iOS, enviando a notificação.

6.2 Confirmação de Leitura (Read Receipts)
Para evitar contenção na fila principal de mensagens, as confirmações de leitura são tratadas em um fluxo separado:
- Quando o usuário abre a conversa e as mensagens aparecem na tela, seu app envia um evento `READ_RECEIPT` via WebSocket.
- Este evento é publicado em um tópico Kafka `read-receipts`.
- Um serviço consumidor agrega as leituras em **Redis** (usando hashes ou sorted sets) e periodicamente faz um **batch update** no DynamoDB, atualizando o campo `is_read` de várias mensagens de uma vez, minimizando write amplification.

Exemplo de registro no Redis: `HSET read:chatID:userID messageID timestamp`

---

## 5. Stack Tecnológica

| **Backend** | Node.js (para I/O intensivo) + Go (para módulos de alta performance) | Node.js agiliza desenvolvimento de APIs; Go é usado para serviços críticos (presença, ingestão). |
| **Cache / Presença** | Redis (clusters) | Latência sub‑milissegundo, TTL nativo, Pub/Sub para fan-out de status. |
| **Banco de Dados** | Amazon DynamoDB (ou Cassandra) | Escrita massiva, baixa latência, esquema flexível para mensagens. |
| **CDN / Edge** | CloudFront / Cloudflare | Redução de latência global, cache de assets estáticos e anexos. |

---

## 6. Considerações de Segurança

- **TLS 1.3:** Toda comunicação (HTTP, WebSocket) é criptografada com TLS 1.3, garantindo confidencialidade e integridade.
- **Presigned URLs:** Anexos nunca são expostos publicamente; cada acesso exige uma URL temporária gerada com permissões mínimas.
- **VPC Endpoints:** O tráfego para S3 e DynamoDB trafega dentro da VPC da AWS, nunca pela internet pública.
- **Autenticação via JWT:** Tokens com assinatura HMAC‑SHA256, renovação periódica e revogação via blacklist no Redis.

---

Elaborado por Sandro Medrano.
