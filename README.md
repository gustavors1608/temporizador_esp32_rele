# Temporizador ESP32-C3 com RTC e Relé

## Visão Geral do Projeto

Sistema embarcado baseado em **ESP32-C3 Super Mini** que controla um relé seguindo uma regra de agendamento: a cada **X minutos**, ligar o relé por **Y minutos**, dentro de uma janela horária **Z até A**. Toda configuração é feita via **captive portal Wi-Fi** (Access Point). As configurações são persistidas em **SPIFFS**. O horário é mantido por um **RTC DS1302** e pode ser ajustado manualmente pelo portal.

---

## Escopo Definido

| Parâmetro | Decisão |
|---|---|
| Interface de configuração | Captive Portal via Access Point Wi-Fi |
| Credenciais do AP | SSID e senha fixos no código |
| Persistência | SPIFFS (JSON) — editável pelo portal |
| Agendamentos simultâneos | 1 (único agendamento ativo) |
| Granularidade X e Y | Minutos |
| Manutenção de horário | RTC DS1302 — ajuste manual pelo portal |
| Portal exibe | Status do relé, horário atual, configuração do agendamento, controle manual |

---

## Hardware — Pinout de Referência

### RTC DS1302 → ESP32-C3
| Pino RTC | GPIO ESP32 | Função |
|---|---|---|
| CLK | GPIO4 | Clock serial |
| DAT | GPIO5 | Dados (bidirecional) |
| RST (CE) | GPIO6 | Chip Enable |
| VCC | 3V3 | Alimentação |
| GND | GND | Terra |

### Relé → ESP32-C3
| Pino Relé | GPIO ESP32 | Observação |
|---|---|---|
| IN | GPIO0 | ⚠️ Strapping pin — ver nota abaixo |
| VCC | 5V | Alimentação da bobina |
| GND | GND | Terra comum |

> ⚠️ **Atenção GPIO0:** No ESP32-C3, o GPIO0 é um *strapping pin* verificado durante o boot. Se o módulo de relé mantiver o pino em nível LOW no momento da inicialização, o ESP32 pode entrar em modo de gravação. Recomenda-se usar um relé com lógica ativa em HIGH (aciona com HIGH), ou migrar para **GPIO3** ou **GPIO7** caso ocorram problemas de boot.

---

## Estrutura do Projeto (PlatformIO)

```
temporizador-esp32/
├── platformio.ini
├── data/                        # Arquivos para SPIFFS
│   ├── index.html               # Interface do captive portal
│   ├── style.css
│   └── config.json              # Configurações persistidas (gerado em runtime)
└── src/
    ├── main.cpp                 # Setup, loop, orquestração
    ├── config.h                 # Constantes globais (SSID, senha, GPIOs)
    ├── rtc_manager.cpp/.h       # Leitura e escrita no DS1302
    ├── relay_manager.cpp/.h     # Controle do relé
    ├── scheduler.cpp/.h         # Lógica de agendamento (X, Y, Z–A)
    ├── storage_manager.cpp/.h   # Leitura/escrita no SPIFFS (JSON)
    └── web_server.cpp/.h        # AP, DNS (captive), rotas HTTP
```

---

## platformio.ini

```ini
[env:esp32-c3-super-mini]
platform  = espressif32
board     = esp32-c3-devkitm-1
framework = arduino

; Ajuste de flash layout para suportar SPIFFS
board_build.partitions = default.csv

lib_deps =
  ; RTC DS1302
  makuna/RTC @ ^2.4.2
  ; JSON (persistência)
  bblanchon/ArduinoJson @ ^7.0.0
  ; Servidor HTTP / Captive Portal
  esphome/ESPAsyncWebServer-esphome @ ^3.2.2
  esphome/AsyncTCP-esphome @ ^2.0.0

monitor_speed = 115200
upload_speed  = 921600

; Upload do sistema de arquivos SPIFFS
extra_scripts = pre:upload_fs.py  ; opcional — ou usar: pio run --target uploadfs
```

> **Nota:** Para fazer upload dos arquivos da pasta `data/` para a SPIFFS, use o comando:
> ```
> pio run --target uploadfs
> ```

---

## Bibliotecas e Justificativas

| Biblioteca | Função | Motivo da escolha |
|---|---|---|
| `makuna/RTC` | Comunicação com DS1302 | Suporte nativo ao DS1302, API simples, madura |
| `ArduinoJson` | Serialização das configs | Padrão Arduino para JSON, eficiente em memória |
| `ESPAsyncWebServer` | Servidor HTTP assíncrono | Não bloqueia o loop, ideal para ESP32 |
| `AsyncTCP` | Base do servidor async | Dependência obrigatória do ESPAsyncWebServer |
| `SPIFFS` (built-in) | Sistema de arquivos | Nativo no framework Arduino para ESP32 |

---

## Arquitetura de Módulos

### `config.h` — Constantes globais
```cpp
#define AP_SSID       "Temporizador"
#define AP_PASSWORD   "12345678"
#define AP_IP         IPAddress(192, 168, 4, 1)

#define PIN_RTC_CLK   4
#define PIN_RTC_DAT   5
#define PIN_RTC_RST   6
#define PIN_RELAY     0   // Considere migrar para GPIO3 ou GPIO7

#define CONFIG_FILE   "/config.json"
```

---

### `storage_manager` — SPIFFS / JSON

Responsável por ler e salvar as configurações em `/config.json` na SPIFFS.

**Estrutura do JSON salvo:**
```json
{
  "interval_min": 60,
  "duration_min": 5,
  "start_hour": 6,
  "start_minute": 0,
  "end_hour": 22,
  "end_minute": 0,
  "relay_active_high": true
}
```

**Interface pública:**
```cpp
bool loadConfig(ScheduleConfig& cfg);
bool saveConfig(const ScheduleConfig& cfg);
```

---

### `rtc_manager` — DS1302

Encapsula a biblioteca `makuna/RTC` para leitura e escrita do horário.

**Interface pública:**
```cpp
void begin();
RtcDateTime now();
bool setTime(int hour, int min, int sec, int day, int month, int year);
String getFormattedTime();   // retorna "HH:MM:SS"
```

---

### `relay_manager` — Controle do Relé

Gerencia o estado do relé, suportando acionamento manual e automático.

**Interface pública:**
```cpp
void begin();
void turnOn();
void turnOff();
bool isOn();
void setManualOverride(bool active, bool state);  // override manual pelo portal
bool isManualOverride();
```

---

### `scheduler` — Lógica de Agendamento

Coração do sistema. Avalia a cada ciclo do `loop()` se o relé deve estar ligado ou desligado, com base nas configurações e no horário atual do RTC.

**Regras de negócio:**

1. Se o horário atual está **fora da janela Z–A** → relé desligado, nenhuma ação.
2. Se o horário está **dentro da janela Z–A**:
   - Calcula quantos minutos se passaram desde o início da janela.
   - Dentro de cada ciclo de X minutos, os primeiros Y minutos = relé **ligado**.
   - Os minutos restantes (X − Y) = relé **desligado**.
3. **Override manual** tem prioridade sobre o agendamento (enquanto ativo).

**Interface pública:**
```cpp
void begin(ScheduleConfig* cfg, RtcManager* rtc, RelayManager* relay);
void update();   // chamado no loop() — decide estado do relé
```

**Pseudocódigo da lógica:**
```
função update():
  se manualOverride ativo:
    aplicar estado manual
    retornar

  agora = rtc.now()
  minutos_do_dia = agora.hora * 60 + agora.minuto

  janela_inicio = cfg.start_hour * 60 + cfg.start_minute
  janela_fim    = cfg.end_hour   * 60 + cfg.end_minute

  se minutos_do_dia < janela_inicio ou minutos_do_dia >= janela_fim:
    relay.turnOff()
    retornar

  minutos_na_janela = minutos_do_dia - janela_inicio
  posicao_no_ciclo  = minutos_na_janela % cfg.interval_min

  se posicao_no_ciclo < cfg.duration_min:
    relay.turnOn()
  senão:
    relay.turnOff()
```

---

### `web_server` — AP + Captive Portal + Rotas HTTP

**Inicialização:**
- Configura o ESP32 em modo **AP** com IP fixo `192.168.4.1`.
- Inicia um servidor DNS que redireciona **todos os domínios** para `192.168.4.1` (comportamento de captive portal).
- Serve os arquivos estáticos da SPIFFS (`index.html`, `style.css`).

**Rotas da API REST:**

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Serve `index.html` da SPIFFS |
| GET | `/api/status` | Retorna horário atual, estado do relé, config ativa |
| POST | `/api/config` | Salva novo agendamento |
| POST | `/api/time` | Ajusta o horário no RTC |
| POST | `/api/relay` | Liga/desliga manualmente (override) |

**Exemplo de resposta `/api/status`:**
```json
{
  "time": "14:35:22",
  "relay": true,
  "manual_override": false,
  "config": {
    "interval_min": 60,
    "duration_min": 5,
    "start_hour": 6,
    "start_minute": 0,
    "end_hour": 22,
    "end_minute": 0
  }
}
```

---

## Interface Web (Captive Portal)

A página `index.html` servida pela SPIFFS terá as seguintes seções:

**1. Status do Sistema**
- Horário atual (lido do RTC via `/api/status`, atualizado a cada 5s por polling)
- Indicador visual do estado do relé (ligado/desligado)
- Indicador de modo (automático ou override manual)

**2. Configuração do Agendamento**
- Campo: Intervalo X (minutos)
- Campo: Duração Y (minutos)
- Campo: Início da janela Z (hora:minuto)
- Campo: Fim da janela A (hora:minuto)
- Botão: Salvar configuração

**3. Ajuste de Horário**
- Campo: Data e hora (datetime-local input HTML)
- Botão: Sincronizar RTC

**4. Controle Manual**
- Botão: Ligar relé agora
- Botão: Desligar relé agora
- Botão: Retornar ao automático

---

## Fluxo de Funcionamento — Diagrama de Estados

```
[BOOT]
   │
   ▼
[Inicializa SPIFFS, RTC, Relé, Wi-Fi AP]
   │
   ▼
[Carrega config.json]──(falha)──► [Usa configuração padrão]
   │
   ▼
[Loop principal]
   │
   ├──► [web_server.handleClient()]   — processa requisições HTTP
   │
   └──► [scheduler.update()]          — avalia e controla relé
              │
              ├── Override manual ativo? ──► Aplica estado manual
              │
              ├── Dentro da janela Z–A? ──► Não ──► Relay OFF
              │
              └── Sim ──► posição no ciclo < Y? ──► Sim ──► Relay ON
                                                 └── Não ──► Relay OFF
```

---

## Ordem de Implementação — Etapas

### Etapa 1 — Setup do Projeto e Hardware
- Criar projeto no PlatformIO com o `platformio.ini` definido.
- Instalar bibliotecas.
- Testar comunicação com o DS1302 (leitura de horário via Serial).
- Testar acionamento do relé.
- Verificar comportamento do GPIO0 no boot.

### Etapa 2 — Módulo de Armazenamento (SPIFFS)
- Implementar `storage_manager` com leitura e escrita do `config.json`.
- Testar persistência: salvar, reiniciar, ler — verificar integridade.

### Etapa 3 — Módulo do Agendador
- Implementar `scheduler` com a lógica de janela horária e ciclo X/Y.
- Testar com horários simulados via Serial (sem portal ainda).
- Validar casos de borda: janela que cruza meia-noite, X < Y, janela = 0 min.

### Etapa 4 — Access Point e Captive Portal
- Configurar ESP32 em modo AP com IP fixo.
- Configurar servidor DNS para redirecionar todo tráfego.
- Servir uma página HTML mínima da SPIFFS para validar o captive portal.
- Testar conexão com smartphone (Android e iOS).

### Etapa 5 — API REST e Integração
- Implementar todas as rotas HTTP (`/api/status`, `/api/config`, `/api/time`, `/api/relay`).
- Integrar `web_server` com `scheduler`, `rtc_manager` e `storage_manager`.

### Etapa 6 — Interface Web Completa
- Desenvolver `index.html` com todas as seções definidas.
- Implementar polling do `/api/status` a cada 5 segundos.
- Testar todos os formulários e botões.

### Etapa 7 — Testes de Sistema e Ajustes Finais
- Teste de operação contínua (24h+).
- Verificar comportamento após queda de energia (RTC mantém horário, ESP32 recarrega config da SPIFFS).
- Ajustar lógica de manual override (timeout automático? — decisão a definir).
- Revisão de casos de borda do agendador.

---

## Considerações Técnicas Importantes

**Janela horária cruzando meia-noite** — se Z > A (ex: 22h às 06h), a lógica de comparação de minutos do dia precisa tratar o caso onde `fim < inicio`. Recomenda-se tratar essa situação na Etapa 3.

**Polling vs. WebSocket** — para simplicidade, o portal usa polling HTTP a cada 5s para atualizar o status. Para uma experiência mais fluida no futuro, pode-se migrar para WebSocket.

**Tamanho da SPIFFS** — o `config.json` ocupa poucos bytes. O maior consumo será o `index.html`. Manter o HTML + CSS abaixo de 100KB é recomendado para o layout de partição padrão.

**Thread safety** — o `ESPAsyncWebServer` executa callbacks em uma task separada (FreeRTOS). Se o `scheduler` modificar o estado do relé ao mesmo tempo que uma requisição HTTP tenta fazer o mesmo, pode haver condição de corrida. Usar `portMUX` ou flags atômicas para proteger o estado compartilhado do relé.

**Compatibilidade do captive portal** — iOS detecta captive portals via requisição a `captive.apple.com`. Android usa `connectivitycheck.gstatic.com`. O servidor DNS precisa interceptar **qualquer domínio** e redirecionar para `192.168.4.1`. A biblioteca `AsyncDNSServer` ou `DNSServer` (built-in) resolve isso.

---

## Resumo das Dependências

```ini
lib_deps =
  makuna/RTC @ ^2.4.2
  bblanchon/ArduinoJson @ ^7.0.0
  esphome/ESPAsyncWebServer-esphome @ ^3.2.2
  esphome/AsyncTCP-esphome @ ^2.0.0
```

Sistema de arquivos: **SPIFFS** (nativo no ESP32 Arduino framework)

---

*Plano gerado para: ESP32-C3 Super Mini + DS1302 + Relé 5V | PlatformIO + Arduino Framework*
