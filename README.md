# 🌡️ Cooler Monitor ESP32

Sistema automático de monitoramento e controle de temperatura para equipamentos de rede (Modem + TP-Link Deco X50) instalados em móvel de MDF, com controle de coolers via PWM, alertas e comandos pelo Telegram, atualizações de firmware via GitHub (OTA) e relatórios automáticos com data e hora.

---

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Hardware](#hardware)
- [Esquema de Ligação](#esquema-de-ligação)
- [Software e Tecnologias](#software-e-tecnologias)
- [Funcionalidades](#funcionalidades)
- [Comandos Telegram](#comandos-telegram)
- [OTA via GitHub](#ota-via-github)
- [Instalação e Configuração](#instalação-e-configuração)
- [Histórico de Versões](#histórico-de-versões)

---

## 🎯 Visão Geral

O projeto nasceu da necessidade de resfriar um espaço fechado em MDF onde ficam instalados um **Modem Claro** e um **TP-Link Deco X50**, evitando superaquecimento e possíveis danos ao mobiliário e equipamentos.

O sistema foi construído em duas fases:

**Parte 1 — Coolers sempre ligados**
Dois coolers de PC conectados diretamente em uma fonte 12V via barrel jack, criando um fluxo de ar forçado (entrada na base + exaustor no topo).

**Parte 2 — Controle inteligente**
ESP32 com sensor de temperatura DS18B20 controlando os coolers via PWM (MOSFET LR7843), com integração ao Telegram para alertas, comandos e atualizações remotas de firmware.

---

## 🔧 Hardware

### Componentes principais

| Componente | Modelo | Especificações |
|---|---|---|
| Microcontrolador | DOIT ESP32 WROOM-32D | Dual Core 240MHz, Wi-Fi 802.11 b/g/n, BT 4.2, 4MB Flash, USB-C |
| Sensor temperatura | DS18B20 à prova d'água | -55°C a +125°C, precisão ±0.5°C, interface 1-Wire |
| MOSFET | LR7843 (módulo opto-acoplado) | 30V/161A, gate 3.3V compatível com ESP32 |
| Conversor tensão | Mini-360 (HW-187) | Step Down 12V→5V, 1.8A nominal, calibrado para 5.02V |
| Cooler 1 | DC Brushless M1202512M | 120mm, 12V, 0.12A (entrada de ar) |
| Cooler 2 | DC Brushless M802512M | 80mm, 12V, 0.16A (exaustor) |
| Fonte | APD WA-36A12R | 12V 3A, conector barrel jack |
| Resistor | CR25 4.7kΩ 1/4W | Pull-up do sensor DS18B20 |

### Consumo total dos coolers
```
Cooler 1: 0.12A + Cooler 2: 0.16A = 0.28A total
Fonte suporta 3A → uso de apenas 9.3% da capacidade
```

### Onde comprar
- **Curto Circuito** (curtocircuito.com.br) — ESP32, sensor, MOSFET, Mini-360, resistores
- **Mercado Livre** — coolers, fonte

---

## ⚡ Esquema de Ligação

### Fluxo de energia
```
Tomada 110/220V
    ↓
Fonte APD WA-36A12R (12V 3A)
    ↓                    ↓
Mini-360            MOSFET LR7843
(12V → 5V)         (chave eletrônica)
    ↓                    ↓
ESP32 VIN          Coolers 12V
(alimentação)      (controlados por PWM)
```

### Mapa de pinos ESP32

| Pino ESP32 | Componente | Função |
|---|---|---|
| GPIO 4 (D4) | DS18B20 DATA | Leitura de temperatura |
| GPIO 16 (RX2) | LR7843 PWM | Controle de velocidade PWM |
| 3V3 | DS18B20 VCC + resistor 4.7kΩ | Alimentação do sensor |
| GND | DS18B20 GND + LR7843 GND | Terra geral |
| VIN | Mini-360 OUT+ | Alimentação 5V |

### Ligação do Mini-360
```
Fonte 12V (+) → IN+  Mini-360
Fonte 12V (-) → IN-  Mini-360
Mini-360 OUT+ → VIN  ESP32  (5V calibrados)
Mini-360 OUT- → GND  ESP32
```

### Ligação do LR7843
```
Fonte 12V (+) → Borne + (azul)
Coolers (+)   → Borne LOAD (azul)
Coolers (-) + Fonte (-) → Borne - (azul)
ESP32 GPIO16  → PWM (verde)
ESP32 GND     → GND (verde)
```

---

## 💻 Software e Tecnologias

### Plataforma
- **Arduino IDE 2.3.10**
- **ESP32 Arduino Core 3.3.11** (Espressif Systems)
- **Driver CP2102** (Silicon Labs) para comunicação USB-Serial

### Bibliotecas Arduino

| Biblioteca | Versão | Função |
|---|---|---|
| OneWire | 2.3.8 | Comunicação 1-Wire com DS18B20 |
| DallasTemperature | 4.0.6 | Leitura de temperatura DS18B20 |
| UniversalTelegramBot | 1.3.0 | Integração com Telegram Bot API |
| ArduinoJson | (dependência) | Parsing JSON do Telegram |
| ArduinoOTA | nativa ESP32 | Atualização OTA via rede local |
| HTTPClient | nativa ESP32 | Download de firmware via GitHub |
| Update | nativa ESP32 | Gravação de firmware OTA |
| time.h | nativa ESP32 | Sincronização de horário via NTP |

### APIs e Serviços

| Serviço | Uso | Custo |
|---|---|---|
| Telegram Bot API | Alertas e comandos | Gratuito |
| NTP (pool.ntp.org) | Sincronização de horário | Gratuito |
| GitHub | Hospedagem de firmware .bin para OTA | Gratuito |

### Configurações de rede
- **Wi-Fi:** suporte a múltiplas redes com fallback automático
- **Fuso horário:** UTC-3 (Belém/Brasília)
- **OTA local:** Arduino IDE via rede (hostname: cooler-monitor, senha: definida no código)
- **OTA remoto:** via comando Telegram + GitHub (funciona de qualquer lugar)

---

## ✨ Funcionalidades

### Controle de temperatura

| Faixa | Temperatura | Ação | PWM |
|---|---|---|---|
| Normal | < 40°C | Coolers desligados | 0% |
| Elevada | 40°C – 50°C | Velocidade baixa | 50% |
| Alta | 50°C – 55°C | Velocidade máxima | 100% |
| Crítica | > 55°C | Velocidade máxima + alerta | 100% |

### Alertas automáticos
- Mudança de faixa de temperatura (em tempo real)
- Sensor desconectado
- Relatório a cada 1h (ativado por padrão)
- Relatório completo a cada 12h
- Resumo diário automático à meia-noite

### Relatório 1h
```
⏱️ Relatorio 1h
🕐 08/08/2026 14:00:00

🌡️ Temp ambiente: 37.2C
🔥 Temp chip: 52.2C
💨 Coolers: Desligados
🔢 Faixa: Normal (desligados)
⏱️ Online ha: 6h 0min
```

### Relatório completo (12h)
```
📊 Relatorio Cooler Monitor
🕐 08/08/2026 14:00:00

🌡️ Temp ambiente: 37.2C
🔥 Temp chip: 52.2C
⚡ CPU: 240MHz
💾 RAM: 112KB/308KB (36%)
💿 Flash: 1100KB/2380KB (46%)
📡 Rede: Family Hesketh
🌐 IP: 192.168.68.200
🔌 MAC: 68:25:DD:08:45:10
🖥️ Host: cooler-monitor
⏱️ Online ha: 12h 0min

💨 Coolers: Desligados
🔢 Faixa: Normal (desligados)
⚙️ Modo: Automatico
📬 Relatorio: Ativo (1h)
```

### Resumo diário (meia-noite)
```
📅 Resumo do dia — 08/08/2026

🌡️ Temperatura ambiente:
  • Minima: 29.9C
  • Maxima: 37.9C
  • Media:  35.3C

💨 Tempo em cada faixa:
  🟢 Normal:  23h 59min
  🟡 Elevada: 0h 0min
  🟠 Alta:    0h 0min
  🔴 Critica: 0h 0min

📊 Total de leituras: 17280
🔄 Mudancas de faixa: 0
⏱️ Online ha: 24h 0min
```

---

## 📱 Comandos Telegram

### Teclado interativo
```
┌──────────────────┬──────────────────┐
│  📊 Relatório   │  🌡️ Temperatura  │
├──────────────────┼──────────────────┤
│  💨 Coolers     │     ❓ Ajuda     │
├──────────────────┼──────────────────┤
│   🟢 Ligar      │   🔴 Desligar   │
├──────────────────┼──────────────────┤
│  📅 Resumo dia  │   ⏹️ Parar 1h   │
└──────────────────┴──────────────────┘
```

### Comandos de texto

| Comando | Função |
|---|---|
| `/cooler` | Relatório completo |
| `/temp_cooler` | Temperatura atual |
| `/coolers_status` | Status dos coolers |
| `/resumo` | Resumo do dia |
| `/ligar` | Liga coolers manualmente |
| `/desligar` | Desliga coolers manualmente |
| `/auto` | Volta ao modo automático |
| `/relatorio1h` | Ativa relatório horário |
| `/parar1h` | Desativa relatório horário |
| `/atualizar` | Atualiza firmware via GitHub |
| `/ajuda_cooler` | Lista todos os comandos |

### Segurança
Todos os comandos verificam o `CHAT_ID` do remetente. Mensagens de outros usuários são ignoradas silenciosamente.

---

## 🚀 OTA via GitHub

### Como funciona
O ESP32 pode baixar e instalar um novo firmware remotamente sem necessidade de cabo USB ou acesso à mesma rede local.

```
PC → compila código → gera .bin
         ↓
    Sobe .bin no GitHub
         ↓
  Envia /atualizar no Telegram
         ↓
  ESP32 baixa .bin do GitHub
         ↓
  Instala e reinicia ✅
```

### Segurança do código-fonte
O repositório GitHub contém **apenas o arquivo .bin compilado** — nunca o código-fonte com senhas. O arquivo binário não permite extração de credenciais.

### Passo a passo para atualizar
1. Edita o código no Arduino IDE
2. **Sketch → Export Compiled Binary** → gera `.bin`
3. Sobe o `.bin` no repositório GitHub
4. Manda `/atualizar` no Telegram
5. Aguarda confirmação de reinicialização

---

## ⚙️ Instalação e Configuração

### Pré-requisitos
- Arduino IDE 2.3.10+
- Driver CP2102 (Silicon Labs)
- Pacote ESP32 by Espressif Systems 3.3.11
- Conta no Telegram com bot criado via @BotFather
- Conta no GitHub

### Configuração do código

```cpp
// Bot Telegram
#define BOT_TOKEN  "SEU_TOKEN_AQUI"
#define CHAT_ID    "SEU_CHAT_ID_AQUI"

// URL do firmware no GitHub
#define FIRMWARE_URL "https://raw.githubusercontent.com/SEU_USUARIO/cooler-monitor-ota/main/cooler_monitor.bin"

// Redes Wi-Fi (em ordem de preferência)
RedeWifi redes[] = {
  { "Rede Principal", "Senha123" },
  { "Rede Backup",    "Senha456" },
};
```

### Primeiro upload
Obrigatoriamente via cabo USB-C na primeira vez:
1. Preenche as configurações acima
2. Seleciona **ESP32 Dev Module** + porta COM
3. Upload Speed: **460800**
4. Clica Upload (segura BOOT se necessário)

### Atualizações futuras
- **Via rede local (OTA):** Arduino IDE → porta de rede `cooler-monitor`
- **Via internet (GitHub):** comando `/atualizar` no Telegram

---

## 📊 Dados de monitoramento real

Dados coletados nas primeiras 96 horas de operação em Belém, Pará:

| Data | Mínima | Máxima | Média |
|---|---|---|---|
| 01/08/2026 | 30.1°C | 37.9°C | 36.2°C |
| 02/08/2026 | 29.9°C | 37.4°C | 35.3°C |
| 03/08/2026 | 30.2°C | 37.8°C | 35.7°C |

**Observação:** A temperatura ambiente do móvel permaneceu consistentemente entre 30-38°C, sempre abaixo do limiar de 40°C para acionamento dos coolers.

---

## 🔄 Histórico de versões

| Versão | Data | Principais mudanças |
|---|---|---|
| 1.0 | 01/07/2026 | Parte 1: coolers ligados direto na fonte |
| 2.0 | 30/07/2026 | ESP32 + sensor DS18B20 + Telegram básico |
| 2.1 | 31/07/2026 | OTA via Arduino IDE, teclado Telegram |
| 2.2 | 01/08/2026 | Controle manual, relatório 1h, modo automático |
| 2.3 | 02/08/2026 | NTP (data/hora), stats sistema, resumo diário |
| 2.4 | 05/08/2026 | Multi-WiFi, correção nome rede, MAC/hostname |
| 2.5 | 08/08/2026 | OTA via Telegram + GitHub |

---

## 📍 Informações do projeto

- **Local:** Belém, Pará, Brasil
- **Equipamentos monitorados:** Modem Claro + TP-Link Deco X50
- **Ambiente:** Móvel em MDF (espaço fechado)
- **Uptime máximo registrado:** 96h+ sem reinicialização

---

*Projeto desenvolvido por Lucas Hesketh — Belém/PA*
