# Rocksmith 2014 no Linux com Behringer UM2

Português | [English](README.en.md)

Guia de como eu deixei o Rocksmith 2014 Remastered rodando no CachyOS usando uma interface de áudio Behringer UM2 no lugar do Real Tone Cable. Latência baixa, sem ruído, com CDLC e RSMods funcionando.

Levou uma tarde inteira e umas quatro tentativas que deram errado. Este README existe pra você não precisar repetir os mesmos erros.

## Resumo rápido

Se você só quer saber o que funciona:

| Peça | O que usei |
|---|---|
| Distro | CachyOS (base Arch), PipeWire 1.6.9 |
| Camada de compatibilidade | GE-Proton11-7 |
| Driver de áudio | PipeASIO 1.10.0 (pacote `pipeasio` ou `pipeasio-bin` no AUR) |
| Ponte ASIO pro jogo | RS_ASIO 0.7.5 |
| Interface | Behringer UM2 (chip TI PCM2902) |
| CDLC | RSCDLCEnabler, build antigo de 130 KB |
| Mods | RSMods 1.2.8.4 (Fast Load e auto-enter no perfil) |

Opções de inicialização finais na Steam:

```
PROTON_USE_WOW64=1 WINEDLLPATH=<caminho que o pipeasio-manage mostrar> WINEDLLOVERRIDES="D3DX9_42=n,b;xinput1_3=n,b" %command%
```

O caminho de áudio fica assim:

```
Guitarra -> UM2 -> PipeWire -> PipeASIO -> RS_ASIO -> Rocksmith
```

Nada de JACK, nada de wineasio compilado na mão.

## Índice

1. [Antes de começar](#antes-de-começar)
2. [Instalação básica](#instalação-básica)
3. [CDLC](#cdlc)
4. [RSMods](#rsmods)
5. [Parte avançada](#parte-avançada)
6. [Problemas que eu tive](#problemas-que-eu-tive)
7. [Fontes](#fontes)

## Antes de começar

Você vai precisar de:

- Rocksmith 2014 Edition Remastered na Steam
- A DLC grátis "Smashing Pumpkins - Cherub Rock" (o CDLC depende dela)
- Um helper do AUR (`paru` ou `yay`)
- `protontricks` (está nos repositórios do CachyOS)
- A biblioteca da Steam numa partição Linux (ext4, btrfs). NTFS dá problema com o prefixo do Proton.

Testado em 2026-10-06 com kernel 7.2, PipeWire 1.6.9 e GE-Proton11-7. Se você está lendo isso muito depois, as versões podem ter mudado e algum passo pode não bater mais.

## Instalação básica

### 1. Instale o PipeASIO

```bash
paru -S pipeasio-bin
```

O `pipeasio-bin` vem pré-compilado. O `pipeasio` normal compila do código e puxa o `mingw-w64-gcc`, que demora bastante. Os dois instalam a mesma coisa.

### 2. Instale o GE-Proton 11

O Rocksmith é um jogo 32 bits. Pro PipeASIO de 32 bits carregar, o Proton precisa rodar no modo "new WoW64", ativado com `PROTON_USE_WOW64=1`.

O problema: o Proton 11.0-100 da Valve ignora essa variável. O GE-Proton respeita. Por isso, GE.

Pelo ProtonUp-Qt é só escolher GE-Proton11-7. Na mão:

```bash
cd "$(mktemp -d)"
N=GE-Proton11-7-x86_64
U=https://github.com/GloriousEggroll/proton-ge-custom/releases/download/GE-Proton11-7
curl -fLO "$U/$N.tar.gz"
curl -fLO "$U/$N.sha512sum"
sha512sum -c "$N.sha512sum"
tar -xzf "$N.tar.gz" -C ~/.local/share/Steam/compatibilitytools.d/
```

Atenção ao nome: a partir da 11-4 os arquivos ganharam o sufixo `-x86_64`. Guias antigos usam o nome sem sufixo e o download volta um "Not Found" de 9 bytes.

Reinicie a Steam depois pra ela enxergar o Proton novo.

### 3. Configure o jogo na Steam

Em Rocksmith 2014 > Propriedades:

- Compatibilidade: marque "Forçar" e escolha GE-Proton11-7
- Abra o jogo uma vez e feche. Isso cria o prefixo em `<SteamLibrary>/steamapps/compatdata/221680/pfx`.

### 4. Registre o PipeASIO no prefixo do jogo

O PipeASIO vem com um gerenciador que acha os jogos da Steam sozinho:

```bash
# lista os jogos e mostra o id de cada um
pipeasio-manage list

# procure o bloco com "appid": "221680" e copie o "id" (algo como steam:e474...)
pipeasio-manage --target steam:SEU_ID --include-32 install
```

O `--include-32` é obrigatório: sem ele só a versão 64 bits é registrada, e o Rocksmith não usa essa.

No final ele imprime a linha de opções de inicialização com o `WINEDLLPATH` certo pra sua máquina. Use exatamente o que ele mostrar.

Se um dia você apagar o prefixo ou trocar de Proton, rode esse comando de novo.

### 5. Instale o RS_ASIO

Baixe a versão 0.7.5 em [github.com/mdias/rs_asio/releases](https://github.com/mdias/rs_asio/releases) e copie `RS_ASIO.dll`, `avrt.dll` e `RS_ASIO.ini` pra pasta do jogo (onde fica o `Rocksmith2014.exe`).

Troque o conteúdo do `RS_ASIO.ini` por este:

```ini
[Config]
EnableWasapiOutputs=0
EnableWasapiInputs=0
EnableAsio=1

[Asio]
BufferSizeMode=driver
CustomBufferSize=

[Asio.Output]
Driver=PipeASIO
BaseChannel=0
AltBaseChannel=
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

; guitarra. Channel=0 é o lado esquerdo (L) da UM2.
; veja "Descobrir em qual canal a guitarra chega" na parte avançada
[Asio.Input.0]
Driver=PipeASIO
Channel=0
EnableSoftwareEndpointVolumeControl=1
EnableSoftwareMasterVolumeControl=1
SoftwareMasterVolumePercent=100
EnableRefCountHack=

; desligados de propósito: com Driver vazio o jogo não vê um segundo jogador fantasma
[Asio.Input.1]
Driver=
Channel=1

[Asio.Input.Mic]
Driver=
Channel=1
```

### 6. Ajuste o Rocksmith.ini

Na mesma pasta, confira estas linhas na seção `[Audio]`:

```ini
ExclusiveMode=1
Win32UltraLowLatencyMode=1
```

Guias que usam ALSA puro mandam deixar as duas em `0`. Com RS_ASIO é o contrário.

### 7. Aponte o PipeASIO pra UM2

Abra o `pipeasio-settings` (o app gráfico que vem no pacote) e escolha a UM2 como dispositivo de entrada. Ou edite `~/.config/pipeasio/config.ini` direto:

```ini
[pipeasio]
inputs = 2
outputs = 2
buffer_size = 128
fixed_buffer_size = 1
sample_rate = 48000
auto_connect = 1
input_device = alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00.analog-stereo-input
```

O nome da UM2 no PipeWire é "Burr-Brown from TI USB Audio CODEC", e não "Behringer". Pra conferir o seu:

```bash
pw-cli ls Node | grep node.name | grep -i burr
```

O guia de PipeASIO pro Real Tone Cable manda usar `inputs = 1`, porque o cabo é mono. A UM2 é estéreo, então deixe em 2 e escolha o canal no `RS_ASIO.ini`.

### 8. Opções de inicialização

Em Propriedades > Geral > Opções de inicialização:

```
PROTON_USE_WOW64=1 WINEDLLPATH=<caminho do passo 4> %command%
```

Abra o jogo e toque. Se a guitarra aparecer e soar, a parte difícil acabou.

## CDLC

### O enabler que funciona

O CDLC precisa de uma `D3DX9_42.dll` modificada na pasta do jogo. Aqui tem uma pegadinha: as versões novas do enabler do CustomsForge derrubam o jogo no Linux, uns 7 segundos depois de abrir. Testei a variante "TooManyCoresFix" (248 KB) e ela crashou com GE-Proton11.

O que funcionou foi um build antigo do RSCDLCEnabler, de 130 KB, publicado no guia do [devroom.io](https://www.devroom.io/2025/12/02/rocksmith-2014-on-cachyos-arch-linux/). Baixe de lá e coloque na pasta do jogo.

SHA-256 do arquivo que eu usei:

```
c09dacbd428a5b55db814b46401224fd7ea9c295057112334eb50c1768d3c776
```

É binário de terceiro, sem hash oficial pra comparar. As strings internas batem com o projeto RSCDLCEnabler, mas a decisão de confiar é sua.

Esse build antigo exige que a DLC Cherub Rock esteja instalada.

### Opção de inicialização

Acrescente o override pro Proton carregar a DLL da pasta do jogo:

```
WINEDLLOVERRIDES="D3DX9_42=n,b"
```

### Músicas

Baixe no [CustomsForge Ignition](https://ignition4.customsforge.com/). Pegue sempre os arquivos que terminam em `_p.psarc` (PC). Os `_m.psarc` são de Mac.

Coloque em `Rocksmith2014/dlc/`. Pode criar subpastas, tipo `dlc/cdlc/`, pra separar das DLCs oficiais.

## RSMods

O RSMods pula as intros, entra direto no perfil e tem um monte de outras opções. Ele suporta Wine/Proton oficialmente.

### Instalação

Antes de qualquer coisa, faça backup do save. Ele fica fora do prefixo:

```bash
cp -a ~/.local/share/Steam/userdata/<SEU_STEAM_ID>/221680/remote ~/rocksmith-save-backup
```

Baixe o `RS2014-Mod-Installer.exe` em [github.com/Lovrom8/RSMods/releases](https://github.com/Lovrom8/RSMods/releases) e rode dentro do prefixo do jogo:

```bash
PROTON_USE_WOW64=1 protontricks-launch --appid 221680 ~/Downloads/RS2014-Mod-Installer.exe
```

Quando ele pedir a pasta de save, use o caminho do Wine:

```
Z:\home\<seu_usuario>\.local\share\Steam\userdata\<SEU_STEAM_ID>\221680\remote
```

O `Z:` é a raiz do Linux vista de dentro do Wine.

Pra abrir o RSMods de novo depois:

```bash
cd <SteamLibrary>/steamapps/common/Rocksmith2014/RSMods
PROTON_USE_WOW64=1 protontricks-launch --cwd-app --appid 221680 ./RSMods.exe
```

### Os mods que eu uso

- Fast Load (aba Set and Forget): pula as telas de abertura. Ele edita o `cache.psarc` uma vez e guarda o original como `cache.bak`. Funciona mesmo sem a DLL do RSMods carregada.
- Force Profile Load: aperta Enter sozinho e entra no último perfil. Esse depende da DLL do RSMods.

### Opção de inicialização

A DLL do RSMods é a `xinput1_3.dll`. Sem override o Proton usa a versão interna dele e os mods de tempo de jogo não rodam:

```
WINEDLLOVERRIDES="D3DX9_42=n,b;xinput1_3=n,b"
```

Ative um mod por vez e teste. Se o jogo quebrar, você sabe quem foi.

## Parte avançada

### Como as peças se encaixam

O Rocksmith foi feito pra falar com o Real Tone Cable via WASAPI no Windows. No Linux isso vira uma cadeia:

1. A UM2 aparece no PipeWire como um dispositivo estéreo comum.
2. O PipeASIO é um driver ASIO pro Wine que fala direto com o PipeWire. Ele tem duas metades: uma DLL Windows (`pipeasio32.dll`) e uma biblioteca Linux (`pipeasio32.so`).
3. O RS_ASIO se injeta no Rocksmith e troca os dispositivos WASAPI que o jogo procura por dispositivos falsos ligados a um driver ASIO.
4. O jogo acha que está falando com o cabo oficial.

A parte frágil é o passo 2 em 32 bits. A DLL Windows de 32 bits precisa chamar a biblioteca Linux de 64 bits, e isso só funciona no new WoW64 do Wine. Daí o `PROTON_USE_WOW64=1`. O `WINEDLLPATH` diz ao Wine onde achar o `.so`.

### Descobrir em qual canal a guitarra chega

Eu errei isso. Achei que a guitarra na entrada de instrumento da UM2 chegaria no canal direito, mas ela chegou no esquerdo. Em vez de chutar, meça:

```bash
#!/usr/bin/env bash
# grava 15 s da UM2 e mostra o nível de cada canal.
# toque a guitarra sem parar enquanto grava.
# depende de: pipewire (pw-record), python3
set -euo pipefail
DIR=$(mktemp -d)
NODE=alsa_input.usb-Burr-Brown_from_TI_USB_Audio_CODEC-00.analog-stereo-input

# pw-record não tem opção de duração, então mata depois de 15 s
pw-record --target "$NODE" --channels 2 --rate 48000 --format s16 "$DIR/t.wav" &
sleep 15
kill $!

python3 -I - "$DIR/t.wav" <<'EOF'
import sys, wave, array, math
w = wave.open(sys.argv[1]); n = w.getnchannels()
a = array.array('h', w.readframes(w.getnframes()))
for c in range(n):
    s = a[c::n]
    rms = math.sqrt(sum(x*x for x in s) / max(len(s), 1))
    print(f"canal {c} ({'L' if c == 0 else 'R'}): pico={max(abs(x) for x in s)/327.67:.1f}%  rms={rms/327.67:.2f}%")
EOF
rm -rf "$DIR"
```

O canal que mostrar sinal é o número que vai no `Channel=` do `[Asio.Input.0]`.

### Ler o log do RS_ASIO

O RS_ASIO grava `RS_ASIO-log.txt` na pasta do jogo a cada execução. É o primeiro lugar pra olhar.

Funcionando, aparece algo assim:

```
AsioHelpers::FindDrivers
  PipeASIO
ASIO input channels info:
  0 - ... name: in_1
  1 - ... name: in_2
{ASIO IN 0} ... ::Start
```

Os erros que eu vi e o que significam:

| Mensagem no log | Causa |
|---|---|
| `cannot load pipeasio32.so: needs new WoW64` | Falta `PROTON_USE_WOW64=1`, ou o Proton está ignorando a variável (Valve 11.0-100) |
| `FindDrivers` sem `PipeASIO` embaixo | Driver não registrado no prefixo. Rode o `pipeasio-manage install` |
| Log não é atualizado | O jogo crashou antes do RS_ASIO carregar. Normalmente é a `D3DX9_42.dll` |

Se o log nem for atualizado, o log da Steam mostra quanto tempo o processo viveu:

```bash
grep 221680 ~/.local/share/Steam/logs/console_log.txt | tail
```

### Ver as ligações no PipeWire

Com o jogo aberto:

```bash
pw-link -l | grep -i -A4 pipeasio
```

Tem que aparecer a captura da UM2 ligada às entradas do PipeASIO. Com `auto_connect = 1` isso é automático.

### Latência

Com `buffer_size = 128` a 48 kHz fica em torno de 2,7 ms por buffer, e o RS_ASIO confirma no log (`actual buffer duration: 2ms (128 frames)`). Não senti atraso nenhum. Se der estalo no áudio, suba pra 256 no `config.ini` do PipeASIO.

## Problemas que eu tive

Em ordem cronológica:

1. Opções de inicialização sem `PROTON_USE_WOW64=1`. O jogo abria com zero dispositivos de áudio.
2. Usei o Proton padrão do sistema, que não ativou o WoW64. Troquei pro GE-Proton11-7.
3. Prefixo novo sem driver registrado. Erro "No audio output device is detected". Resolvi com `pipeasio-manage install`.
4. Guitarra no canal errado. Áudio do jogo funcionando, guitarra muda. Medi com `pw-record` e troquei pra `Channel=0`.
5. Enabler de CDLC novo crashando no boot. Troquei pelo build antigo de 130 KB.
6. A guitarra "parou de funcionar" depois de instalar o RSMods. O cabo tinha saído da UM2. Confira o óbvio primeiro.

Mais um detalhe: o `pipeasio-register --help` não mostra ajuda. Ele registra o driver no `~/.wine` padrão. Não estraga nada, mas não é o que você espera.

## Fontes

- [PipeASIO](https://github.com/M0n7y5/pipeasio)
- [RS_ASIO](https://github.com/mdias/rs_asio)
- [rocksmith-linux-pipeasio-guitar-input-fix](https://github.com/UkrainianCitizen/rocksmith-linux-pipeasio-guitar-input-fix), principalmente o [gotchas.md](https://github.com/UkrainianCitizen/rocksmith-linux-pipeasio-guitar-input-fix/blob/main/docs/gotchas.md)
- [linux-rocksmith (Nizo)](https://codeberg.org/nizo/linux-rocksmith), o guia clássico com JACK e wineasio
- [Rocksmith 2014 on CachyOS/Arch Linux (devroom.io)](https://www.devroom.io/2025/12/02/rocksmith-2014-on-cachyos-arch-linux/)
- [CustomsForge: CDLC on Steam Proton not working anymore](https://customsforge.com/topic/66510-linux-solved-rsremastered-cdlc-on-steam-proton-not-working-anymore/)
- [RSMods](https://github.com/Lovrom8/RSMods)
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom)
- [ProtonDB do Rocksmith 2014](https://www.protondb.com/app/221680)
