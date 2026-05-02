---
layout: post
title:  "Como o computador descobre o que carregar ao ser ligado?"
date:   2026-05-01 18:53:56 -0300
categories: operating-system
lang: pt-br
---
**Neste artigo, exploramos o processo técnico de inicialização de um computador, detalhando o carregamento do Master Boot Record (MBR), a estrutura de setores de um disco e a implementação de um bootloader em Assembly com foco na validação da assinatura de boot.**

# Carregamento do bootloader

Ao ser ligado, o hardware executa o `Power-On Self Test (POST)`. Após a validação dos componentes, a BIOS busca por dispositivos de armazenamento inicializáveis. O primeiro setor físico (os **512 bytes iniciais, conhecido como setor de inicialização ou MBR**) é carregado na memória RAM. A BIOS valida a integridade deste setor procurando pela "**assinatura mágica**" `0xAA55` em seus dois últimos bytes; se presente, a execução é transferida para este endereço e o processo de boot prossegue.

> ⚠️ **Nota Técnica:** Introduzido em 1983, o **MBR (Master Boot Record)** foi o padrão dominante por décadas. Atualmente, sistemas modernos utilizam o **GPT (GUID Partition Table)**, que supera as limitações de particionamento e tamanho de disco do MBR. Para fins didáticos, utilizaremos o modelo MBR neste artigo.
{: .warning}

![Carregamento do setor de inicialização](/assets/images/boot-sector-loading-ptbr.png)

# Geometria de Disco e Setores

Para fins de compreensão da organização de dados, considere a estrutura de um Disco Rígido (HDD) convencional. O disco é organizado de forma hierárquica em setores, trilhas (tracks) e cabeças (heads).

![Setores de um HD](/assets/images/disk-sector-tracks-ptbr.png)

O **MBR (Master Boot Record)** localiza-se no **primeiro setor** físico do disco. Este setor possui exatamente **512 bytes** de capacidade, espaço onde é definido o **primeiro estágio (Stage 1)** do carregador de inicialização do sistema operacional.

# Implementação do Setor de Boot

O desenvolvimento de um bootloader é tipicamente realizado em **Assembly**, garantindo controle direto sobre o hardware e o mapeamento de memória. Abaixo, apresentamos um exemplo de implementação:

{% highlight assembly linenos%}
# ***********************************************************
# Setor de Boot de Exemplo
# ***********************************************************

.code16
.intel_syntax noprefix
.text
.org 0x0                                        

LOAD_SEGMENT = 0x1000                     # O carregador de 2º estágio será carregado no segmento 1000h
FAT_SEGMENT  = 0x0ee0                     # A FAT do disco de boot será carregada no segmento 0x0ee0 
                                          # (9*512 bytes abaixo do carregador de 2º estágio)

.global main

main:
    jmp short start                       # Salta para o início do código
    nop                                   # Alinhamento (nop) para o cabeçalho do setor de boot

.include "bootsector.s"
.include "macros.s"

start:
  mInitSegments                           # Inicializa os segmentos de memória
  mResetDiskSystem                        # Reinicia o subsistema de disco
  mWriteString loadmsg                    # Exibe mensagem de carregamento
  mFindFile filename, LOAD_SEGMENT        # Localiza o arquivo de 2º estágio no diretório raiz
  mReadFAT FAT_SEGMENT                    # Carrega a tabela FAT na memória
  mReadFile LOAD_SEGMENT, FAT_SEGMENT     # Transfere o 2º estágio para a memória RAM
  mStartSecondStage                       # Transfere o fluxo de execução para o 2º estágio
 
# Rotina de tratamento de falhas no processo de boot
bootFailure:
  mWriteString diskerror                  # Exibe mensagem de erro de disco
  mReboot                                 # Solicita reinicialização do sistema
  
.include "functions.s"
    
# Definição de dados e constantes
filename:    .asciz "2NDSTAGEBIN"
rebootmsg:   .asciz "Pressione qualquer tecla para reiniciar.\r\n"
diskerror:   .asciz "Erro de disco. "
loadmsg:     .asciz "Carregando DevOS...\r\n"

root_strt:   .byte 0,0      # Offset do diretório raiz
root_scts:   .byte 0,0      # Quantidade de setores do diretório raiz
file_strt:   .byte 0,0      # Offset do bootloader no disco

.fill (510-(.-main)), 1, 0  # Padding com zeros até o byte 510
BootMagic:  .int 0xAA55     # Assinatura mágica para reconhecimento pela BIOS
{% endhighlight %}

# Análise Técnica e Assinatura de Boot

A implementação utiliza diretivas específicas para garantir a conformidade com o padrão MBR. O ponto crítico reside no preenchimento do setor para atingir o tamanho exato de 512 bytes (linhas 49 e 50).

Como o volume de instruções e dados pode variar, é necessário calcular dinamicamente o preenchimento (padding) necessário. A expressão utilizada é:
`(510 - (.-main))`

Componentes da expressão:
*   `.` (ponto): Representa o contador de localização atual (location counter).
*   `main`: O endereço do ponto de entrada inicial.
*   `(.-main)`: Calcula o deslocamento (offset) total de bytes gerados até o momento.
*   `510 - (.-main)`: Determina quantos bytes restam para atingir a marca de 510 bytes.

Os dois bytes finais (511 e 512) são reservados para a assinatura **0xAA55**. Sem esta assinatura, a BIOS não reconhecerá o dispositivo como inicializável.

# Geração do Binário e Carregamento na RAM

Após a montagem do código fonte, é gerado um arquivo binário. Quando o computador identifica um dispositivo de boot, a BIOS lê o primeiro setor (512 bytes), copia seu conteúdo para o endereço físico de memória **0x7C00** e inicia a execução das instruções a partir desse endereço.

![Arquivo binário em hexadecimal](/assets/images/bootfile-ptbr.png)

# Próximas Etapas e Conclusão

A inicialização de um sistema operacional é um processo multi-estágio. O código analisado representa o **Estágio 1**, cuja função primordial é localizar e carregar o estágio subsequente na memória RAM, conforme demonstrado na lógica a partir da linha 23.

Em artigos futuros, detalharemos as fases posteriores, incluindo a transição para o Modo Protegido e o carregamento do Kernel do sistema.



