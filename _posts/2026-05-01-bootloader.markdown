---
layout: post
title:  "Como o computador descobre o que carregar ao ser ligado?"
date:   2026-05-01 18:53:56 -0300
categories: operating-system
---
# Carregamento do bootloader

Ao ser ligado, o computador inicia o processo de inicialização com um `Power-On Self Test (POST)`. Concluído o teste com sucesso, o sistema busca por um disco inicializável. Se inserido, o primeiro setor (os **512 bytes iniciais, conhecido como setor de inicialização ou MBR**) é carregado na memória. O computador verifica a integridade desse setor procurando pela "**assinatura mágica**" de `0xAA55` nos seus dois últimos bytes; se presente, o código é executado e o processo de boot continua.

> ⚠️ **Atenção:** Criado em 1983, o **MBR (Master Boot Record)** foi amplamente utilizado por décadas, mas hoje é considerado obsoleto. Seu substituto é o **GPT (GUID Partition Table)**, que oferece suporte a discos maiores. Para efeitos didáticos iremos utilizar o MBR neste texto.
{: .warning}

![Carregamento do setor de inicialização](/assets/images/boot-sector-loading-ptbr.png)

# Setor? Que parada é essa?

Vou usar um HD **“velha guarda”** apenas para fins de exemplo 😂. Observe que há uma estrutura de organização bem definida com setores, faixas e cabeças dentro do disco rígido.

![Setores de um HD](/assets/images/disk-sector-tracks-ptbr.png)

O **MBR (Master Boot Record)**, ou Registro Mestre de Inicialização, neste contexto corresponde ao **primeiro setor** do disco rígido. Como mencionado anteriormente, ele possui **512 bytes** e é justamente nesse espaço que definimos o **primeiro estágio de carregamento do sistema operacional**.

# O código, por favor.
Usualmente, um bootloader é escrito em **Assembly**, por ser uma linguagem de baixo nível que permite controle total sobre o hardware. Veja abaixo um exemplo:

{% highlight assembly linenos%}
# ***********************************************************
# Setor de Boot de Exemplo do sistema operacional de exemplo.
# ***********************************************************

.code16
.intel_syntax noprefix
.text
.org 0x0                                        

LOAD_SEGMENT = 0x1000                     # O carregador de 2º estágio será carregado no segmento 1000h
FAT_SEGMENT  = 0x0ee0                     # A FAT do disco de boot será carregada no segmento 0x0ee0 
                                          # (9*512 bytes abaixo do carregador de 2º estágio, pois a FAT
                                          # consiste em 9 segmentos de 512 bytes).

.global main

main:
    jmp short start                       # Salta para o início do código
    nop                                   # Os dados do setor de boot começam no byte 3, por isso o nop

.include "bootsector.s"
.include "macros.s"

start:
  mInitSegments                           # Inicializa os segmentos de memória usados por este programa
  mResetDiskSystem                        # Reinicia o sistema de disco
  mWriteString loadmsg                    # Exibe "loading..."
  mFindFile filename, LOAD_SEGMENT        # Procura o arquivo de 2º estágio no diretório raiz
  mReadFAT FAT_SEGMENT                    # Carrega a tabela FAT na memória
  mReadFile LOAD_SEGMENT, FAT_SEGMENT     # Lê o arquivo de 2º estágio para a memória
  mStartSecondStage                       # Executa o arquivo de 2º estágio
 
# Falha no boot devido a erro de disco, informa o usuário e reinicia.

bootFailure:
  mWriteString diskerror                  # Mostra "Erro de disco, pressione uma tecla para reiniciar"
  mReboot                                 # Reinicia
  
.include "functions.s"
    
# Dados utilizados pelo programa
filename:    .asciz "2NDSTAGEBIN"
rebootmsg:   .asciz "Pressione qualquer tecla para reiniciar.\r\n"
diskerror:   .asciz "Erro de disco. "
loadmsg:     .asciz "Carregando DevOS...\r\n"

root_strt:   .byte 0,0      # Armazena o offset do diretório raiz no disco
root_scts:   .byte 0,0      # Armazena o número de setores do diretório raiz
file_strt:   .byte 0,0      # Armazena o offset do bootloader no disco

.fill (510-(.-main)), 1, 0  # Preenche com zeros até 510 bytes (excluindo a assinatura de boot)
BootMagic:  .int 0xAA55     # Palavra mágica para a BIOS
{% endhighlight %}

# Decifrando o código.
Tem muita coisa acontecendo por aqui 😳, mas o que nos interessa agora são as linhas 51 e 52.

Lembra da **“assinatura mágica”** **0xAA55**? É exatamente disso que essas linhas tratam.
O arquivo contém uma série de instruções e dados que formarão o setor de boot. Esse setor precisa ter exatamente 512 bytes, que é o tamanho padrão de um setor de disco.

Como o conteúdo do código pode variar de tamanho, é necessário calcular dinamicamente quantos bytes de preenchimento (zeros) devem ser adicionados para completar os 512 bytes. 

É aí que entra a expressão:
`(510 - (.-main))`

Ela funciona assim:

*   `.` representa a posição atual no código (ou seja, o tamanho já gerado até aquele ponto).
*   `main` é o ponto inicial do programa.
*   `(.-main)` calcula o tamanho atual do código.
*   `510 - (.-main)` define quantos bytes faltam para atingir 510 bytes.

Esses 510 bytes são usados porque os últimos 2 bytes (511 e 512) são reservados para a assinatura mágica `0xAA55`, exigida pela BIOS para reconhecer o setor como inicializável.

# O que acontece quando essas instruções e dados se tornam um arquivo binário?

O código de exemplo acima, quando compilado, torna-se um arquivo binário como o mostrado abaixo. Por convenção, quando o computador (BIOS) reconhece um dispositivo de boot, ele lê o **primeiro setor** (512 bytes), copia o conteúdo para o endereço de memória `0x7C00` e, em seguida, o bootloader é executado.

![Arquivo binário em hexadecimal](/assets/images/bootfile-ptbr.png)

# O que acontece depois?

Pretendo escrever um artigo de continuação detalhando as próximas fases. Um sistema operacional é formado por várias etapas de boot, cada uma com suas responsabilidades. Este primeiro estágio é responsável por carregar o restante do sistema na memória RAM, processo que está descrito a partir da linha 24 do nosso código de exemplo.

Até breve, com a continuidade dessas etapas!



