# Kernel 6.17 é inutilizável em APUs AMD Barcelo: 15 crashes em 15 boots

**TL;DR**: O kernel 6.17.0-14-generic causa hard freeze total (sem panic, sem oops, sem watchdog) em sistemas com APU AMD Ryzen 5825U (Barcelo/Renoir). Testei 15 boots consecutivos no mesmo dia — todos travaram. Kernel 6.8 no mesmo hardware, mesmos parâmetros: zero crashes. É uma regressão no driver amdgpu.

![Taxa de crash: 100% no 6.17 vs 0% no 6.8](infografico-4-taxa-crash.png)

---

## O problema

Após atualizar para o kernel 6.17 no Ubuntu 24.04.4 LTS, meu notebook (VAIO com Ryzen 7 5825U) começou a travar completamente de forma aleatória. Sem tela azul, sem mensagem de erro, sem kernel panic — simplesmente congela. Mouse, teclado, SysRq, SSH: nada responde. Só reset forçado pelo botão de energia.

O comportamento é imprevisível: às vezes trava em 27 segundos, às vezes dura 43 minutos. A única constante é que **sempre trava**.

## Metodologia

Ao invés de simplesmente voltar ao kernel antigo, decidi investigar com rigor. Configurei uma bateria completa de instrumentação:

- **Monitoramento contínuo** a cada 5 segundos (CPU, GPU, RAM, temperaturas, dmesg, PCIe)
- **NMI watchdog** habilitado (`nmi_watchdog=1`)
- **Panic automático** em softlockup (`kernel.softlockup_panic=1`) e hung task (`kernel.hung_task_panic=1`)
- **kdump** configurado com `crashkernel` alocado para captura de crashdump
- **pstore** para persistir dados de crash entre reboots
- **Journal persistente** com sync forçado a cada 5 segundos

Controlei a variável: **mesmo hardware, mesmos parâmetros de boot** (`amdgpu.runpm=0 amd_pstate=passive`), alternando apenas o kernel entre 6.17 e 6.8. Todos os testes foram feitos no mesmo dia (2026-02-21).

Então bootei no 6.17 repetidamente e esperei travar.

---

## Resultado: 15 boots, 15 crashes

Cada barra vermelha abaixo é um boot no kernel 6.17 que terminou em hard freeze. As barras verdes são boots no kernel 6.8 — todos estáveis.

![Timeline de 17 boots: todos os 6.17 travaram, todos os 6.8 sobreviveram](infografico-1-timeline-boots.png)

Os tempos até o crash no kernel 6.17:

| Boot | Kernel | Tempo até freeze |
|:----:|--------|:----------------:|
| 1 | 6.17.0-14 | 7m 46s |
| 2 | 6.17.0-14 | 10m 03s |
| 3 | 6.17.0-14 | 14m 28s |
| 4 | 6.17.0-14 | **0m 27s** |
| 5 | 6.17.0-14 | **42m 57s** |
| 6 | 6.17.0-14 | 3m 50s |
| 7 | 6.17.0-14 | 2m 10s |
| 8 | 6.17.0-14 | 6m 50s |
| 9 | 6.17.0-14 | 2m 47s |
| 10 | 6.17.0-14 | 1m 56s |
| 11 | 6.17.0-14 | 3m 38s |
| 12 | 6.17.0-14 | 9m 08s |
| 13 | 6.17.0-14 | 9m 50s |
| 14 | 6.17.0-14 | 8m 38s |
| 15 | 6.17.0-14 | **2m 21s** |
| — | **6.8.0-100** | **estável** |
| — | **6.8.0-100** | **estável** |

**Taxa de crash: 6.17 = 100% (15/15). Kernel 6.8 = 0% (0/2).**

Tempo mediano até o freeze: **~7 minutos**. Variação de 27 segundos a 43 minutos — o crash não depende de carga ou uso, é aleatório no tempo mas inevitável.

---

## O sistema estava 100% saudável no momento do crash

O script de monitoramento capturava o estado completo a cada 5 segundos. A última leitura antes de cada freeze mostra um sistema completamente idle e saudável:

![Métricas do sistema no momento do freeze: tudo normal](infografico-2-saude-sistema.png)

- **CPU**: 42-44°C (normal, longe do limite de 95°C)
- **GPU**: 41-43°C (normal, longe do limite de 90°C)
- **SSD**: 28°C (frio)
- **RAM**: 29+ GB livres de 32 GB (92% livre, zero pressão)
- **Swap**: 0% usado
- **Load average**: 0.4-0.7 (idle, num CPU de 16 threads)
- **GPU**: 0-6% de uso, power state D0 (ativo)
- **dmesg**: Nenhum erro, nenhum warning
- **PCIe**: Nenhum erro

**Não é superaquecimento. Não é falta de RAM. Não é load alto. Não é erro de disco. Não é erro de software.** O hardware e o sistema operacional estão perfeitos — o que falha é o driver.

---

## O que é mais revelador: o que NÃO acontece

Eu habilitei todos os mecanismos de detecção de crash que o Linux oferece. **Nenhum deles detectou o freeze.**

![Todos os mecanismos de segurança do kernel falharam em detectar o freeze](infografico-3-watchdogs-falharam.png)

| Mecanismo | Configuração | Disparou? |
|-----------|-------------|:---------:|
| NMI watchdog | `nmi_watchdog=1` | **NÃO** |
| Softlockup detector | `kernel.softlockup_panic=1` | **NÃO** |
| Hung task detector | `kernel.hung_task_panic=1` | **NÃO** |
| kdump | `USE_KDUMP=1`, crashkernel alocado | **NÃO** |
| pstore | Persistent storage configurado | **VAZIO** |
| Journal | Persistente com sync a cada 5s | **CORTA ABRUPTAMENTE** |

Isso é **extremamente significativo**. O NMI watchdog é uma interrupção **não-mascarável** — ela deveria funcionar mesmo quando o kernel está completamente travado. Se nem o NMI dispara, significa que o freeze acontece **abaixo do nível do kernel**, num lugar que o CPU sequer consegue alcançar.

O único cenário que explica isso é um **PCIe bus hang**: a GPU trava e leva o barramento PCIe junto. Como o AMD Barcelo é uma APU (CPU e GPU no mesmo die, compartilhando o barramento), quando a GPU trava, o sistema inteiro morre.

---

## Análise de causa raiz

![Diagrama de causa raiz: da symptoma ao driver amdgpu](infografico-5-root-cause.png)

A cadeia causal:

1. **Sintoma**: Freeze total sem resposta (teclado, mouse, SSH, SysRq)
2. **Descartado**: Superaquecimento, RAM, disco, carga — monitoramento prova que está tudo normal
3. **Descartado**: Bug no kernel genérico — watchdogs habilitados não detectam nada
4. **Evidência**: 6.8 estável, 6.17 crash 100% → regressão entre essas versões
5. **Evidência**: NMI silencioso → freeze abaixo do kernel → PCIe bus hang
6. **Causa raiz**: **Regressão no driver amdgpu entre 6.8 e 6.17** causa lockup na GPU/firmware que propaga pelo PCIe e trava o sistema inteiro

### A evidência da GPU no init

Durante a inicialização, ambos os kernels fazem um `MODE2 reset` na GPU:

```
amdgpu 0000:04:00.0: amdgpu: MODE2 reset
```

Isso é normal para este hardware após power cycle. A diferença é que o **6.8 se recupera e fica estável**, enquanto o **6.17 se recupera mas morre pouco depois**. Algo mudou no driver entre essas versões que torna a GPU instável após a inicialização.

O chip é um AMD Barcelo (Renoir) — PCI ID `1002:15e7`, ATOM BIOS `113-BARCELO-004`. Família gfx9 (Vega).

---

## Bugs relacionados conhecidos

Este não é um caso isolado. A pesquisa revelou um padrão de regressões no amdgpu para APUs Renoir-family em kernels recentes:

- [**Arch Linux**](https://bbs.archlinux.org/viewtopic.php?id=306429): Freeze total desde kernel 6.15 em Ryzen 5900H (Cezanne — mesmo die que Barcelo)
- [**freedesktop.org #1318**](https://gitlab.freedesktop.org/drm/amd/-/issues/1318): Instabilidade e freeze na GPU em Renoir
- [**Launchpad #2126854**](https://bugs.launchpad.net/bugs/2126854): Firmware load timeout e freeze no amdgpu (kernel 6.14)
- [**Fedora Discussion**](https://discussion.fedoraproject.org/t/amd-apu-regression-full-halt-on-kernel-6-10-how-to-best-report/128404): Full halt em APU AMD no kernel 6.10
- [**Upstream mailing list**](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg128816.html): GPU não detectada desde 6.16.4, afeta 6.17-rc
- Patches em andamento para [reset handling em APUs Renoir](http://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg132803.html)

O que diferencia este report é a **metodologia rigorosa**: monitoramento contínuo, variáveis controladas, 15 reproduções consecutivas, e todos os mecanismos de detecção habilitados.

---

## Hardware testado

| Componente | Detalhe |
|-----------|---------|
| Notebook | VAIO VJFE69F11X-B0121H (Positivo Bahia) |
| CPU/GPU | AMD Ryzen 7 5825U (Barcelo APU) |
| GPU PCI ID | `1002:15e7` rev c1 |
| GPU Family | Renoir / gfx9 (Vega) |
| ATOM BIOS | 113-BARCELO-004 |
| RAM | 32 GB DDR4 |
| SSD | NVMe SM2P32A8-512GC1 512GB (SMART OK) |
| BIOS | V1.20.X (Out/2024) |
| Distro | Ubuntu 24.04.4 LTS (Noble Numbat) |
| Mesa | 25.2.8 |
| Display Server | GNOME / Wayland |

## Parâmetros de boot (idênticos nos dois kernels)

```
quiet splash amdgpu.runpm=0 amd_pstate=passive
```

---

## Conclusão e workaround

Isso é uma **regressão confirmada no driver amdgpu entre os kernels 6.8 e 6.17** que afeta APUs AMD Barcelo (Renoir). A regressão causa um hard freeze total que nenhum mecanismo de detecção do kernel consegue capturar — indicando um travamento no nível de hardware/firmware da GPU que se propaga pelo barramento PCIe.

**Se você tem um AMD Ryzen 5000 série U (Barcelo, Lucienne, Cezanne) e está tendo freezes no kernel 6.17: volte para o 6.8.** É a única solução até que o bug seja corrigido upstream.

### Workaround

```bash
# Fixar GRUB no kernel 6.8
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-100-generic"
sudo update-grub
```

### Bug reports

- freedesktop.org GitLab: *(link será adicionado)*
- Launchpad Ubuntu: *(link será adicionado)*

---

**Sistema**: Ubuntu 24.04.4 LTS, AMD Ryzen 7 5825U, kernel 6.17.0-14-generic vs 6.8.0-100-generic.
**Metodologia**: 17 boots totais (15 no 6.17, 2 no 6.8), monitoramento contínuo a cada 5s, NMI watchdog + softlockup + hung task panic habilitados, kdump configurado, journal persistente.
**Data dos testes**: 2026-02-21.
