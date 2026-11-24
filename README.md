# M-VAVE Tank Mini — Achados de Pesquisa e Engenharia Reversa de Hardware

> **Contribuição para a comunidade:** este documento consolida em um único `README.md` todas as descobertas técnicas sobre a pedaleira **M-VAVE Tank Mini** (referência de fábrica `TANK MINI_011`) reunidas nos projetos **Tank-OS** e **Tank Rig**.
>
> **Escopo:** apenas fatos, medições em hardware, testes em RAM, protocolos, mapa de flash, áudio/DSP e alertas de segurança específicos da **Tank Mini**. Não inclui código descompilado, dumps privados de flash, chaves/MAC de unidade nem procedimentos exclusivos de outros modelos (como Tank-G, Tank-B ou FM-1).

---

## Sumário

1. [Avisos Críticos de Segurança](#1-avisos-críticos-de-segurança)
2. [Identidade de Hardware e USB](#2-identidade-de-hardware-e-usb)
3. [Mapa de Flash (1 MiB) e Formato JLFS / ENC](#3-mapa-de-flash-1-mib-e-formato-jlfs--enc)
4. [Boot, Watchdog e Recuperação](#4-boot-watchdog-e-recuperação)
5. [Footswitches e LEDs RGB (WS2812)](#5-footswitches-e-leds-rgb-ws2812)
6. [Áudio Físico (ALNK0), DMA e Clocks](#6-áudio-físico-alnk0-dma-e-clocks)
7. [DSP, Efeitos, NAM A2-Lite e Cab IR](#7-dsp-efeitos-nam-a2-lite-e-cab-ir)
8. [Protocolos de Comunicação (MIDI/SysEx e USB CDC)](#8-protocolos-de-comunicação-midisysex-e-usb-cdc)
9. [Toolchain, BSP e Metodologia de Sondas em RAM](#9-toolchain-bsp-e-metodologia-de-sondas-em-ram)
10. [Incompatibilidades Conhecidas com Outros Modelos](#10-incompatibilidades-conhecidas-com-outros-modelos)
11. [Resumo de Pendências e Limitações](#11-resumo-de-pendências-e-limitações)

---

## 1. Avisos Críticos de Segurança

- **Não existe recuperação física por botões comprovada na Tank Mini V011:** combinações de footswitches ao ligar (`A`, `B`, `C`, `A+B`, `B+C`, `A+B+C`, `Power+USB-C`, segurando por 3–5 s) **não** colocaram a unidade em modo `WL82 UBOOT1.00`. A única entrada no bootloader ROM comprovada sem abrir o hardware é via **MIDI SysEx enviado a um firmware vivo em execução**.
- **Nunca grave abaixo de `0x004000`:** a faixa `[0x000000, 0x004000)` contém o bootloader de 2º estágio e metadados iniciais.
- **Nunca grave a partir de `0x095000`:** a partir desse endereço ficam `VM`, `BTIF`, `USR` e `key/MAC` específicos da unidade.
- **Nunca use imagens ou instaladores da Tank-G / UltraRig na Tank Mini:** os instaladores de Tank-G escrevem até `0x0A4000`, sobrescrevendo **61.440 bytes** da região `VM` da Tank Mini.
- **Sempre faça backup integral (1 MiB) antes de qualquer escrita:** execute duas leituras completas em `WL82 UBOOT1.00`, exija igualdade byte a byte (e SHA-256) e mantenha o arquivo **privado** (contém Bluetooth, MAC e calibração da unidade).

---

## 2. Identidade de Hardware e USB

| Item | Valor observado | Observações |
| --- | --- | --- |
| Produto (firmware de fábrica) | `TANK MINI_011` | Identificado no pacote oficial `TANK-MINI-V011.fwsc` e no USB |
| SoC / CPU | JieLi AC791N / WL82, núcleo `PI32v2` | Arquitetura little-endian |
| SPI NOR Flash | JEDEC `0xEB6014`, **1 MiB** (`0x100000` bytes) | Medido em modo `WL82 UBOOT1.00` |
| Chip key (loader) | `0x980F` | Constante do loader/cifra ENC (não é segredo da unidade) |
| USB modo normal | `VID 4C4A`, `PID C755` (`bcdUSB 0200`) | Composto: USB Audio Class 1.0 (`0100`) + MIDIStreaming |
| USB modo download | `VID 4C4A`, `PID 8057` | Produto `WL82 UBOOT1.00` (interface SCSI/USBSTOR no Windows) |
| USB modo OTA/HID (M-UPGRADE) | `VID 4D4A`, `PID 4155` | Observado durante atualização oficial `009` → `011` |

### Descritores USB confirmados no modo normal (`4C4A:C755`)

Configuração total de **293 bytes**:

| Interface / Alt | Classe | Endpoint | Direção / Tipo | Uso |
| --- | --- | --- | --- | --- |
| `0 / 0` | AudioControl (`1/1`), UAC1 `0100` | — | — | Controle de áudio |
| `1 / 1` | AudioStreaming (`1/2`) | `0x02` | OUT, isócrono adaptativo, `288 B` | Playback (host → Mini) |
| `2 / 1` | AudioStreaming (`1/2`) | `0x83` | IN, isócrono assíncrono, `288 B` | Captura (Mini → host) |
| `3 / 0` | AudioControl (`1/1`), UAC1 `0100` | — | — | Controle de áudio |
| `4 / 0` | MIDIStreaming (`1/3`) | `0x84` IN, `0x04` OUT | Bulk, `64 B` | MIDI |

- O áudio USB declara **PCM estéreo 24 bits** empacotado em **3 bytes por amostra** a **44.100 Hz** (`288 bytes = 48 frames × 6 bytes`).
- O fato de `bcdUSB` ser `0200` não significa UAC2: `audioClassBcd` é `0100` (UAC 1.0).

---

## 3. Mapa de Flash (1 MiB) e Formato JLFS / ENC

Capacidade total: **1.048.576 bytes** (`0x000000`–`0x100000`). Confirmado por leitura integral da unidade e conferência com o pacote oficial `TANK-MINI-V011.fwsc`:

| Região | Intervalo | Tamanho | Uso e regra de preservação |
| --- | --- | ---: | --- |
| `bootloader_and_metadata` | `0x000000`–`0x003FFF` | 16.384 B | **Preservar sempre.** Bootloader V011 (14.384 B) e metadados. |
| `application_and_resources` | `0x004000`–`0x094FFF` | 593.920 B | **Única região gravável** para firmware customizado (`app.bin` + JLFS). |
| `VM` | `0x095000`–`0x0BDFFF` | 167.936 B | **Preservar.** Estado do sistema/VM (na Mini começa em `0x095000`). |
| `BTIF` | `0x0BE000`–`0x0BEFFF` | 4.096 B | **Preservar.** Dados Bluetooth da unidade. |
| `USR` | `0x0BF000`–`0x0FEFFF` | 262.144 B | **Preservar.** Estado do usuário; muda em uso normal (ex.: logs do V011). |
| `key_mac` | `0x0FF000`–`0x0FFFFF` | 4.096 B | **Preservar e nunca publicar.** Chave/MAC específicos da unidade. |

### Detalhes do layout JLFS e cifra ENC

- O pacote oficial `TANK-MINI-V011.fwsc` cobre `[0x000000, 0x095000)` (**610.304 bytes**), com `boot` em texto claro (`14.384 bytes`) e área da aplicação a partir de `0x004000`.
- A área da aplicação é formatada em **JLFS** com cifra **ENC** da JieLi:
  - Polinômio LFSR **`0x1021`** (CRC-16-CCITT);
  - Chave inicial `0x980F ^ (offset >> 2)`, aplicada em **blocos de 32 bytes** (reiniciando a cada bloco);
  - A cifra é **auto-inversa** nos 593.920 bytes da área do app.
- Os registros de diretório JLFS têm **32 bytes** com `CRC16/XMODEM` sobre os bytes `2..31`.
- O cabeçalho da aplicação declara **XIP entry point `0x02000120`** (mapeando flash `0x004000` → `0x02000000`, `app.bin` em offset `0x120`).
- **Atenção ao repacotar:** existe diretório aninhado (`cfg` / `eq_cfg_hw.bin`) com CRCs próprios de cabeçalho e dados; ferramentas que validam apenas as 3 primeiras entradas do diretório raiz deixam de verificar entradas aninhadas.
- **Observação sobre região `USR`:** durante uso normal e entradas no ROM pelo V011, o firmware de fábrica grava registros incrementais na área apagada de `USR` (ex.: registros contínuos de 18 bytes em `[0x0FCEE8, ...)`). Portanto, diferenças apenas em `USR` entre backups após boot normal são esperadas.

---

## 4. Boot, Watchdog e Recuperação

### Limite fundamental

- **Não há botão de recovery no hardware/bootloader** identificado para a Tank Mini V011.
- O **bootloader de 2º estágio não implementa contador de rollback** nem verificação de "atualização não confirmada". Durante atualizações oficiais (`M-UPGRADE`), os portões de prepare/ACK, transferência/ACK e verificação de identidade são coordenados pelo host.
- Consequência: qualquer guarda de recuperação automática (como *3-boot guard* ou pânico por footswitch) **depende de o firmware instalado conseguir iniciar e executar** até a verificação.

### Watchdog (WDT) e Reset P33

- A Boot ROM deixa o **watchdog habilitado (~4 s)** ao passar controle para código em RAM (`P3_WDT_CON=0x1C`, `P3_RST_SRC=0x40`, `P3_VLD_KEEP=0x20` observados em sonda).
- Sem alimentar o WDT em um probe em RAM, o chip reinicia automaticamente (~4,2 s–5,7 s) e retorna ao firmware instalado (`4C4A:C755`).
- O reset via **P33** (`P3_PR_PWR` bit 4) a partir de RAM também devolveu a Tank Mini ao V011 normalmente.
- Para retornar da execução em RAM/firmware ao modo `WL82 UBOOT1.00`, utiliza-se o marcador em RAM (`UPDATA_BEG` / `usb_update_mode`) seguido de reset P33 (ou `go_mask_usb_updata()` / `fm1_enter_uboot()`).

### Estratégias de recuperação testadas

| Mecanismo | Status na Tank Mini | Limite |
| --- | --- | --- |
| SysEx `F0 22 24 35 7D F7` no V011 | **Confirmado em hardware** | Requer V011 vivo e porta `USB-Midi` |
| Pânico por footswitch (PA0, PA3, PA10) | **Confirmado em firmware anterior (Tank-OS)** | Requer o firmware iniciar até ler GPIO |
| Comando serial `BOOT` (CDC-ACM) | **Confirmado em bancada (Tank-OS e Tank Rig RAM probe)** | Requer driver USB CDC operacional |
| *3-boot guard* (SRAM retida + VM) | **Implementado e testado em host/simulação** | Ainda **não disparado fisicamente** em 3 resets quentes na Mini; SRAM se perde ao desligar energia |

### Bug clássico documentado: direção de GPIO no AC791N

- No SoC AC791N, **`DIR = 1` significa entrada** (e `DIR = 0` significa saída).
- Configurar `DIR &= ~MASK` coloca os pinos dos footswitches como saída em nível baixo, fazendo o firmware ler todos os botões como pressionados e entrar em UBOOT em loop. A forma correta é `DIR |= MASK` com pull-up habilitado.

---

## 5. Footswitches e LEDs RGB (WS2812)

### Footswitches

Medido por sonda em RAM (`0x01C02000`, entradas com pull-up, amostragem de 20 s por botão):

| Footswitch | Pino | Nível ativo | Observação |
| --- | --- | --- | --- |
| Esquerdo | **PA10** | Nível baixo (`0`) | `PC10` muda junto por compartilhar rota/LED |
| Meio | **PA0** | Nível baixo (`0`) | Sem mudança em outras portas |
| Direito | **PA3** | Nível baixo (`0`) | Sem mudança em outras portas |

- Repouso observado nas portas: `PA = 0x07FF`, `PB = 0x01FE`, `PC = 0x07FF`.

### LEDs RGB

Os LEDs dos footswitches **não são GPIOs simples**: são **9 LEDs WS2812** endereçáveis ligados ao pino **PC10** via SPI/DMA (`0x11D00`), com:
- Ordem de bytes **`G, R, B`**;
- `WS divider = 9`;
- **36 bytes de reset**;
- Os LEDs retêm o último estado enviado (para apagar, é necessário enviar frame preto).

Mapeamento dos 9 pixels (3 canais por footswitch):

| Footswitch | R | G | B |
| --- | ---: | ---: | ---: |
| Esquerdo | 0 | 1 | 2 |
| Meio | 3 | 4 | 5 |
| Direito | 6 | 7 | 8 |

---

## 6. Áudio Físico (ALNK0), DMA e Clocks

### Pinagem e periférico

O áudio analógico utiliza o periférico **ALNK0** em modo mestre:

| Pino | Função |
| --- | --- |
| `PA4` | `MCLK` |
| `PA5` | `BCLK` |
| `PA6` | `LRCLK` |
| `PA7` | `TX` (canal 0) |
| `PA8` | `RX` (canal 1) |
| `PA9` | Reset / habilitação do conversor externo (manter baixo durante config e liberar após clocks estáveis) |

### Formato DMA e taxa real

- **Formato:** PCM24 com sinal nos **24 bits baixos** de palavras de 32 bits.
- **Meio-buffer DMA:** **64 quadros estéreo** = **128 palavras de 32 bits** = **512 bytes**.
- **Taxa física medida:** **~44.117 a 44.118 quadros/s** (não exatamente 44.100 Hz nem 48.000 Hz).
  - Ao interligar áudio analógico (~44.118 Hz) com USB Audio (44.100 Hz), os dois domínios precisam de controle de taxa/buffer.
- **TIMER4:** opera em base de **24 MHz**; um meio-bloco DMA de 64 quadros corresponde a um prazo de **~34.830 ticks**.

### Achados de validação em hardware

- **Pass-through digital (Tank-OS):** comprovado em hardware (`rx_peak == tx_peak = 0x7FFFFF`, `last_pending = 0x00`), exigindo ACK com `pending >> 4` (em vez de `pending & 0x03`) e timer em 24 MHz.
- **Saídas físicas:** o sinal sai **simultaneamente** no conector de fone e na saída P10 (sem roteamento independente implementado).
- **Armadilha em probes em RAM (Tank Rig):** ao entrar em `WL82 UBOOT1.00`, a Boot ROM zera/não retém `ADC_CON` e `DAC_CON` (`0x00000000`). Um probe em RAM que configura apenas `ALNK0` sem inicializar o codec/ADC/DAC completo gera TX/RX digitais funcionando a ~44.117,6 Hz, porém com **todas as amostras do RX DMA zeradas** mesmo com guitarra plugada.

---

## 7. DSP, Efeitos, NAM A2-Lite e Cab IR

Resultados medidos na série experimental do **Tank-OS** (`M0.1` → `M0.5`) em hardware real:

### Cadeia e aritmética

- Cadeia base: entrada em mono (duplicada para L/R na saída) → ganho → EQ/efeitos → `ALNK0`.
- **Bug de ganho Q15:** `GAIN_UNITY = 32768` convertido para `int16_t` virava `-32768`, invertendo a polaridade do sinal; corrigido usando `int32_t` (`mismatches = 0` em 16 blocos).
- **EQ de 6 bandas (`TONE6`):** 5 filtros passa-baixas de 1 polo em cascata (`80 Hz`, `250 Hz`, `800 Hz`, `2,5 kHz`, `6 kHz`, `> 6 kHz`), ganhos Q15 com rampa de 256 quadros (~5,8 ms) para evitar estalos no bypass.

### NAM A2-Lite (o gargalo real no AC791N)

Na Tank Mini, rodar uma captura NAM A2-Lite completa inicialmente estourava o prazo de 34.830 ticks em **10,2×**:

| Configuração testada | Pico (ticks) | Relação ao prazo (34.830) |
| --- | ---: | ---: |
| Escalar original | 356.827 | 10,2× |
| Acelerador vetorial AC79 | 283.301 | 8,1× |
| Vetorial + `-O2` / poll / recíprocos | 218.392 | 6,3× |
| Sem os 4 `sat32` de pós-processamento + gather `r6-r4` | 155.918 | 4,5× |
| Motor fundido Q24 | 90.805 | 2,6× |
| **Dual-core (CPU1 worker + CPU0)** | **sem estouro** | **dentro do prazo (`limitfx=0`)** |

- **Diagnóstico:** o acelerador vetorial representava apenas **~10%** do custo; **59%** estava no pós-processamento escalar e **23%** no gather.
- **Solução que funcionou:** rodar o motor fundido em **CPU1** como worker persistente com fila de 3 blocos na RAM dos *cache ways*, deixando DMA/USB/EQ/Cab/efeitos na **CPU0** (latência fixa adicional de ~3 blocos ≈ **4,4 ms**).
- **Cuidado ao gravar flash com CPU1 ativa:** CPU1 executa código da flash; gravar flash exige drenar e parar CPU1 antes.
- **Inline asm:** tentativa de usar `madd.s.prr` em inline assembly travou a pedaleira e foi revertida.

### Cab IR e limites de memória de efeitos

- **Cab IR:** FIR direto de 256 taps (22.050 Hz, ~11,6 ms) custava ~37.000 ticks e estourava o prazo. Otimizado com `int64` nativo, detecção dual-mono, acumuladores par/ímpar e truncamento para **128 taps** (96,95% da energia).
- **Orçamento de memória:** com Amp + Cab ativos, 9 efeitos leves consumiram **49% de CPU**, mas efeitos com linhas de atraso grandes (`CHORUS`, `DELAY`, `HALL`, `PITCH`, `ROTARY`) falharam por **falta de memória na arena DSP (~79,5 KiB–83 KiB)**, não por CPU. Histórico em 16 bits permitiu encaixar Chorus + Delay juntos; reverbs longos ainda exigem buffers reduzidos.

---

## 8. Protocolos de Comunicação (MIDI/SysEx e USB CDC)

### Firmware de fábrica (`V011`)

- **Entrada em `WL82 UBOOT1.00` via USB-MIDI:**
  ```text
  F0 22 24 35 7D F7
  ```
  **Confirmado na Tank Mini V011.** Ao enviar pela porta `USB-Midi` (`4C4A:C755`), o pedal desconecta imediatamente e reenumera como `4C4A:8057` (`WL82 UBOOT1.00`).
- **Notas oficiais de MIDI (changelog M-VAVE V011, 2025-05-16):** documentam `PC 00–20` para presets 1–21 e `CC 00–05` (valor `00`) para `FX`, `AMP`, `MOD`, `DLY`, `REV`, `CAB`. O mapa completo de parâmetros via MIDI ainda não foi capturado fisicamente no V011.
- **Dump de presets (`.atkm`):** arquivo de 1.942 bytes com cabeçalho `51 00 00 01 05 02 ...`.

### Protocolos customizados validados em bancada

1. **Tank-OS (CDC-ACM ASCII):**
   - `VID 3654`, `PID 0002`, linhas ASCII terminadas em `\n` ou `\r\n` (máx. 63 chars na base, até 180 B em `MEDDATA`).
   - Requer porta aberta com **DTR ativo**; após `BOOT`, envia `BOOT ok`, drena o anel CDC, aguarda **500 ms** e entra em `WL82 UBOOT1.00`.
   - Comandos testados em hardware: `PING`, `VER`, `HELP`, `MODEL`, `STATUS`, `UPTIME`, `BOOT`, `EQGET`/`EQSET`/`EQRESET`, `CPU`, além de painel **Web Serial** (`115200 8N1`).

2. **Tank Rig (CDC-ACM binário em RAM probe):**
   - Framing `F0 00 59 <cmd> <len_lo> <len_hi> <payload> <checksum> F7` (`checksum = (~soma(payload)) & 0xFF`).
   - Em sondas de RAM separadas (`-UsbAttachProbe`, `-UsbControlProbe`, `-UsbBootProbe`) compiladas com `-Os` na entrada: enumerou CDC (`4C4A:C756`), respondeu `GET_VERSION` e executou `ENTER_BOOTLOADER` (`F0 00 59 05 00 00 FA F7`) retornando a `WL82 UBOOT1.00`. (O perfil de entrada com `-O2` não enumerou.)

---

## 9. Toolchain, BSP e Metodologia de Sondas em RAM

- **Compilador:** `pi32v2` LLVM/Clang `23.1.5-5` (`--target=pi32v2 -mcpu=r3` ou `-mcpu=ac791n`, C11 freestanding, `-Wall -Wextra -Werror`).
- **BSP:** `felucca-bsp` (MPL-2.0) / JieLi public SDK (`apache-2.0`). O `felucca-bsp` **não** possui `boards/` para a Tank Mini e marca CPU1, flash estendida e cache RAM como beta.
- **Sondas em RAM (sem risco à flash):**
  - Executadas via `jl-uboot-tool` em `WL82 UBOOT1.00` na janela histórica `0x01C02000` (relatório em `0x01C02800`);
  - O protocolo SCSI do Windows (`win32port.py`) exige verificação explícita do status SCSI e do tamanho/resíduo retornado pelo `IOCTL` (além de `argtypes` corretos para ponteiros 64-bit), e retry curto (~50 ms) após salto em RAM para evitar `WinError 31` na leitura do resultado.

---

## 10. Incompatibilidades Conhecidas com Outros Modelos

| Aspecto | Tank Mini V011 | Outros modelos (Tank-G / FM-1 / BlackBox) |
| --- | --- | --- |
| Início da `VM` na flash | **`0x095000`** | Tank-G: `0x0A4000`; FM-1: `0x093000` |
| Área segura da aplicação | **`[0x004000, 0x095000)`** | Instaladores UltraRig Tank-G gravam até `0x0A4000` (**destrói 60 KiB da VM da Mini**) |
| SPI NOR JEDEC | **`0xEB6014`** (1 MiB) | FM-1: `0x856014` / `0xB36014` |
| Recovery por botões no boot | **Não encontrado** | BlackBox V22 usa `Home + > + encoder`; Tank-G/UltraRig dependem de *3-boot guard* no app |
| Áudio hardware | `ALNK0` (`PA4–PA9`) | FM-1 (`fm1_audio.h`) usa pinagem e conversor distintos |

---

## 11. Resumo de Pendências e Limitações

- **Recuperação sem aplicação funcional:** segue **não comprovada** na Tank Mini; qualquer gravação persistente na flash depende de aceitar esse risco ou de comprovar método de entrada em `WL82` independente do app.
- **Áudio USB (UAC1) em firmware customizado:** os descritores do V011 estão mapeados, mas nenhum firmware próprio validou streaming isócrono de áudio USB na Mini até o momento.
- **Captura MIDI completa do V011:** faltam capturas físicas de `CC`/`SysEx` de efeitos e parâmetros do firmware de fábrica.
- **Loop analógico em RAM probe pós-ROM:** pendente replicar a inicialização completa de `ADC_CON`/`DAC_CON` quando a sonda parte da Boot ROM.
- **3-boot guard em hardware:** lógica validada em host/simulação, mas ainda não exercitada com 3 resets quentes consecutivos na unidade física.
