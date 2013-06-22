---
layout: page
title: Documentation
---

## 1 - Introdução ##

O projeto malELFicus começou a ser desenvolvido em 2011 por Tiago Natel de
Moura com o objetivo de estudar o formato ELF (Executable and Linkable Format) e
disseminar o conhecimento de desenvolvimento e análise de malwares para Linux no
cenário nacional.

Atualmente o projeto esta passando por um refactoring para corrigir bugs
antigos, adicionar bugs novos e mudar um pouco de sua arquitetura inicial.
Basicamente o projeto malelficus está dividido em 3 partes: **libmalelf**, **malelf**
e **malelfgui**. Cada um desses projetos é apresentado separadamente, de forma
detalhada, ao logo do documento.

  O repositório do projeto está no github e pode ser acessado através dos
links:

* **libmalelf** - https://github.com/SecPlus/libmalelf
* **malelf** - https://github.com/SecPlus/malelf
* **malelfgui** - https://github.com/SecPlus/malelfgui
* **malelficus** - https://github.com/SecPlus/malelficus


Você nesse momento deve estar se perguntando para que serve o repositório
malelficus, já que todos os projetos são separados? O Malelficus é o
agregador, ele linka e faz build dos outros três projetos.


### 1.1 - O que são arquivos ELF? ###


  O objetivo desse documento não é ensinar o formato ELF, e sim apresentar o
projeto **malELFicus**. Por isso, deduzimos que o leitor tenha conhecimento prévio
sobre ELF para ler esse documento.
  Entretanto, retiramos uma parte do documento DissecandoELF.txt (escrito por
Felipe Pena (sigsegv) e publicado na Cogumelo Binário 1) explicando o que são
arquivos ELF para dar uma visão geral.

  O ELF (Executable and Linking Format) nada mais é do que um formato padrão de
arquivo executável, código objeto, objeto compartilhado, e core dumps. Em 1999
ele foi adotado como formato de arquivo binário para Unix e unix-like em x86 pe-
lo projeto 86open. [1] Sua primeira aparição foi no Solaris 2.0 (o conhecido
SunOS 5.0), que é baseado no SVR4. [2]

  Para maiores informações verificar os Links no final do documento.


## 2 - Libmalelf ##

  *"The libmalelf is an evil library that could be used for good! It was
developed with the intent to assist in the process of infecting binaries and
provide a safe way to analyze malwares".*

  Resumindo, a libmalelf, fornece uma maneira rápida e fácil para programadores
desenvolverem suas próprias ferramentas de análise e infecção de binários ELF.

  Com a libmalelf o programador poderá:

- **Analisar e infectar binários ELF;**
- **Adicionar segmentos/seções e headers ao binário;**
- **Modificar qualquer informação do ELF;**
- **Encontrar holes/gaps para inserir no código;**
- **Criar binários do zero;**

  Agora que já sabemos o que é a libmalelf, vamos descobrir como ela funciona.

*****
COMPLEMENTAR
*****


### 2.1 Build ###

  Para baixar o código fonte é necessário que você tenha o git instalado.

    $ git clone https://github.com/SecPlus/libmalelf.git

Dependências:

- NASM
- libxml2-dev
- libcunit1-dev (opcional)

Caso sua distribuição Linux seja baseada em Debian:

    $ git clone https://github.com/SecPlus/libmalelf.git
    $ sudo apt-get install nasm libxml2-dev libcunit1-dev
    $ ./configure --prefix=/usr --enable-tests
    $ make
    $ make check

Pronto! Agora você já tem a libmalelf em sua máquina e podemos começar a
programar.


### 2.2 Organização ###

  Vamos demonstrar como está organizado o código do projeto no github.

libmalelf/
--+-------
  |
  +-- src/
  |    |-- binary.c
  |    |-- ehdr.c
  |    |-- phdr.c
  |    |-- shdr.c
  |    |-- debug.c
  |    |-- report.c
  |    |-- error.c
  |    |-- infect.c
  |    |-- table.c
  |    |-- util.c
  |    +-- include/
  |          |-- malelf/
  |               |-- HEADERS FILES
  |
  +-- examples/
  |    |-- CODE EXAMPLES
  |
  +-- tests/
       |-- TEST FILES


  Vamos explicar resumidamente a função de cada módulo dentro do projeto.

- Módulo binary: responsável por armazenar todas as informações do binário ELF.
- Módulo ehdr: Armazena as informações do ELF Header.
- Módulo phdr: Armazena as informações do Program Header Table.
- Módulo shdr: Armazena as informações do Section Header Table.
- Módulo debug: Implementa a parte de debug da biblioteca.
- Módulo report: Módulo responsável por gerar as informações no formato XML.
- Módulo error: Faz o mapeamento das mensagens de erro.
- Módulo infect: Implementa os métodos de infecção.
- Módulo table: Módulo responsável por gerar as informações na shell.
- Módulo util: Implementações utilitárias.


### 2.3 Módulo Binary ###

  O módulo binary é constituido por dois arquivos: binary.c e binary.h. Podemos
dizer que este é o principal módulo da biblioteca, pois ele é o responsável por
armazenar todas as informações do binário. Abaixo segue como ele está definido
dentro da biblioteca.

```c
typedef struct {
        char *fname;         /* Binary filename */
        char *bkpfile;       /* Filename of backup'ed file in case of
                                write operations */
        _i32 fd;             /* Binary file descriptor */
        _u8 *mem;            /* Binary content */
        _u32 size;           /* Binary size */
        MalelfEhdr ehdr;     /* ELF Header */
        MalelfPhdr phdr;     /* Elf Program Headers */
        MalelfShdr shdr;     /* Elf Section Headers */
        _u8 alloc_type;      /* System function used to allocate memory */
        _u32 class;          /* Binary arch */
} MalelfBinary;
```

- **fname:** Nome do binário que iremos trabalhar. Exemplo: **/bin/ls.**;
- **bkpfile:** Backup file;
- **fd**: File descriptor;
- **mem**: Conteúdo do binário;
- **size**: Tamanho do binário;
- **ehdr**: Armazena as informações relacionadas ao ELF Header;
- **phdr**: Armazena as informações relacionadas ao Program Header Table;
- **shdr**: Armazena as informações relacionadas ao Section Header Table;
- **alloc_type**: Como a memória será alocada, com mmap ou malloc.
- **class**: Arquitetura do binário;

  Os campos **ehdr**, **shdr** e **phdr** serão apresentados de forma mais detalhada ao
longo do documento.

  Para começar a utilizar a libmalelf é necessário que o programador conheça
alguns métodos básicos.

  O método malelf_binary_init() deve ser chamado antes de utilizar qualquer
outra função da biblioteca. Esse método é responsável por inicializar as
informações do objeto MalelfBinary.

  Para carregar/abrir um binário, existe o método malelf_binary_open(), que por
default utiliza a função mmap() para carregar o binário na memória. Caso o
programador deseje utilizar o malloc(), existe uma função chamada
malelf_binary_set_alloc_type() que pode ser usada passando o parâmetro
MALELF_ALLOC_MALLOC, como no exemplo abaixo.

```c
  MalelfBinary bin;
  malelf_binary_set_alloc_type(bin, MALELF_ALLOC_MALLOC);
```

  E, por último, mas não menos importante, o programador deve chamar o
método malelf_binary_close() passando o objeto MalelfBinary como parametro.

  Demonstrar todas as funcionalidades do módulo binary é uma tarefa dificil,
até porque o projeto ainda está em desenvolvimento. Demonstramos alguns
códigos de exemplo a seguir, porém a melhor maneira de conhecer o módulo
é lendo seu arquivo de header.


#### 2.3.1 - Hello libmalelf ####

  Para iniciarmos os exemplos, vamos começar com o maior clichê do mundo da
programação.

```c
#include <stdio.h>
#include <malelf/binary.h>

int main()
{
    MalelfBinary bin;
    malelf_binary_init(&bin);

    printf("Hello Libmalelf bin[%p]\n", &bin);
    malelf_binary_close(&bin);

    return 0;
}
```

  Agora vamos compilar o nosso exemplo acima.

    $ gcc -o hello hello.c -lmalelf -Wall -Wextra -Werror
    $ ./hello
    Hello Libmalelf bin[0xbfbfd8ec]

  Não sei onde a sua libmalelf foi instalada, caso ela não seja encontrada
lembre-se de exportar a variável LD_LIBRARY_PATH para o diretório correto.

    $ export LD_LIBRARY_PATH=/home/benatto/libs/


#### 2.3.2 - Pegando o nome das seções ####


  A libmalelf fornece alguns métodos que facilitam o programador pegar uma
determinada seção, malelf_binary_get_section(), passando o objeto MalelfBinary,
a posição da seção e o objeto MalelfSection que irá armazenar as informações
da seção.
  As informações contidas na seção podem ser acessadas diretamente pelo
programador, ou através de getters, como no exemplo abaixo, utilizando o método
malelf_binary_get_section_name(); Abaixo segue o código do objeto MalelfSection
para um melhor entendimento dos seus atributos.

```c
typedef struct {
       char *name;
       _u16 type;
       _u32 offset;
       _u32 size;
       MalelfShdr *shdr;
} MalelfSection;
```

  O objeto MalelfShdr será tratado de forma mais detalhada quando entrarmos em
análise de binários ELF. O exemplo abaixo é muito simples, olhem os seguintes
passos:

1 - Chama o método init: malelf_binary_init();
2 - Carrega o binário a ser analisado: malelf_binary_open();
3 - Faz um for pelo número de seçoes do binário;
4 - Pega o nome das seções: malelf_binary_get_section_name();
5 - Imprime o nome da seção;
6 - Libera a memória chamando o método malelf_binary_close();

```c
#include <stdio.h>
#include <assert.h>

#include <malelf/binary.h>
#include <malelf/error.h>

int main()
{
    MalelfBinary bin;
    MalelfSection section;
    int error = MALELF_SUCCESS, i = 0;
    char *name = NULL;

    malelf_binary_init(&bin);

    error = malelf_binary_open("/bin/ls", &bin);
    if (MALELF_SUCCESS != error) {
            MALELF_PERROR(error);
            return 1;
    }

    /* Getting only the name of sections */
    for (i = 1; i < MALELF_ELF_FIELD(&bin.ehdr, e_shnum, error); i++) {
            error = malelf_binary_get_section_name(&bin, i, &name);
            printf("Section name: %s\n", name);
    }

    malelf_binary_close(&bin);
    return 0;
}
```

  A macro MALELF_ELF_FIELD retorna um campo do ehdr, phdr ou shdr. No caso acima
está retornando o campo e_shnum do ELF Header.


### 2.4 - Análise de binários ###


  A libmalelf fornece getters para acessar as informações do ELF Header, Program
Header Table e do Section Header Table. Porém, se o programador não gosta de
acessar os campos através de getters, o acesso poderá ser feito diretamente.

  Vamos aos exemplos. =)


#### 2.4.1 - ELF Header ####


  As informações sobre o ELF header ficam concentradas dentro do módulo ehdr,
que é constituído pelos arquivos ehdr.h e ehdr.c. O exemplo a seguir tem o
objetivo de imprimir as informações do ELF Header.

```c
#include <stdio.h>

#include <malelf/binary.h>
#include <malelf/ehdr.h>
#include <malelf/shdr.h>
#include <malelf/phdr.h>
#include <malelf/defines.h>


int main()
{
        /* Declarando os tipos */
        MalelfBinary binary;
        MalelfEhdr ehdr;
        MalelfEhdrTable me_type;
        MalelfEhdrTable me_machine;
        MalelfEhdrTable me_version;

        _i32 result;
        _u32 size;
        _u32 phentsize;
        _u32 shentsize;
        _u32 phnum;
        _u32 shnum;
        _u32 shstrndx;
        UNUSED(result);

        /* Chamando o metodo init */
        malelf_binary_init(&binary);

        /* Alterando o alloc_type*/
        malelf_binary_set_alloc_type(&binary, MALELF_ALLOC_MALLOC);

        /* Carregando o binario para a memoria */
        malelf_binary_open("/bin/ls", &binary);

        /* Pegando as informacoes do ELF Header */
        result = malelf_binary_get_ehdr(&binary, &ehdr);
        result = malelf_ehdr_get_version(&ehdr, &me_version);
        result = malelf_ehdr_get_type(&ehdr, &me_type);
        result = malelf_ehdr_get_machine(&ehdr, &me_machine);
        result = malelf_ehdr_get_ehsize(&ehdr, &size);
        result = malelf_ehdr_get_phentsize(&ehdr, &phentsize);
        result = malelf_ehdr_get_shentsize(&ehdr, &shentsize);
        result = malelf_ehdr_get_shnum(&ehdr, &shnum);
        result = malelf_ehdr_get_phnum(&ehdr, &phnum);
        result = malelf_ehdr_get_shstrndx(&ehdr, &shstrndx);

        printf("Version Name: %d\n", me_version.name);
        printf("Version Value: %d\n", me_version.value);
        printf("Version Description: %s\n", me_version.meaning);

        printf("Type Name: %d\n", me_type.name);
        printf("Type Value: %d\n", me_type.value);
        printf("Type Description: %s\n", me_type.meaning);

        printf("Machine Name: %d\n", me_machine.name);
        printf("Machine Value: %d\n", me_machine.value);
        printf("Machine Description: %s\n", me_machine.meaning);

        printf("Size: %d\n", size);
        printf("Program Header Table Entry Size: %d\n", phentsize);
        printf("Section Header Table Entry Size: %d\n", shentsize);
        printf("Number of Entries PHT: %d\n", phnum);
        printf("Number of Entries SHT: %d\n", shnum);
        printf("SHT index: %d\n", shstrndx);

        /* Liberando a memoria */
        malelf_binary_close(&binary);

        return 0;
}
```

  Vamos explicar como funciona o exemplo abaixo:

1 - Inicializa o objeto MalelfBinary, chamando o método init;
2 - Altera a forma de carregar o binário na memória;
3 - Carrega o binário para a memória com o método open;
4 - Salva o ELF header no objeto ehdr;
5 - Pega todos os valores com os GETTERS;
6 - Imprime as informações na tela;
7 - Libera a memória chamando o método close;

  Simples, não? =)

  Reparem que não estamos verificando o retorno das funções, isso não é uma boa
prática. Se fizéssemos todas as verificações, o texto ficaria muito longo. =)


#### 2.4.2 - Program Header Table ####


  Para demonstrar como acessar as informações do Program Header Table,
utilizaremos um código que está dentro do módulo dissect do projeto malelf. Mas
já adiantando, a idéia é muito semelhante ao exemplo anterior.

  O objeto MalelfTable será tratado quando estivermos falando de como reportar
as informações do binário, nesse momento pode ignorá-lo.

  Seguem os passos para o nosso exemplo abaixo:

1 - Salvamos o phdr;
2 - Salvamos o ehdr;
3 - Pegamos o campo e_phnum;
4 - Realizamos um loop de acordo com a quantidade de segmentos;
5 - Pegamos o offset e imprimimos;

```c
static _u32 _malelf_dissect_table_phdr()
{
        MalelfTable table;
        MalelfPhdr phdr;
        MalelfEhdr ehdr;
        _u32 phnum;
        _u32 value;
        unsigned int i;

        char *headers[] = {"N", "Offset", NULL};

        if (MALELF_SUCCESS != malelf_table_init(&table, 60, 9, 2)) {
                return MALELF_ERROR;
        }
        malelf_table_set_title(&table, "Program Header Table (PHT)");
        malelf_table_set_headers(&table, headers);

        /* Salvando o phdr */
        malelf_binary_get_phdr(&binary, &phdr);

        /* Salvando o ehdr */
        malelf_binary_get_ehdr(&binary, &ehdr);

        /* Pegando o campo e_phnum */
        malelf_ehdr_get_phnum(&ehdr, &phnum);

        /* Percorrendo os segmentos */
        for (i = 0; i < phnum; i++) {
                malelf_table_add_value(&table, (void *)i, MALELF_TABLE_INT);
                malelf_phdr_get_offset(&phdr, &value, i);
                malelf_table_add_value(&table, (void *)value, MALELF_TABLE_HEX);
        }

        malelf_table_print(&table);
        malelf_table_finish(&table);

        return MALELF_SUCCESS;
}
```

#### 2.4.3 - Section Header Table ####


  Vamos a mais um exemplo. Agora vamos utilizar o módulo shdr para imprimir a
informação do campo offset. Novamente, podem reparar que o processo é bem
semelhante ao que já foi mostrado anteriormente.

```c
#include <stdio.h>
#include <malelf/binary.h>
#include <malelf/ehdr.h>
#include <malelf/shdr.h>
#include <malelf/phdr.h>
#include <malelf/defines.h>


int main()
{
        MalelfBinary bin;
        MalelfEhdr ehdr;
        MalelfShdr shdr;
        unsigned int i;

        _i32 result;
        _u32 shnum;
        _u32 offset;

        UNUSED(result);

        malelf_binary_init(&bin);
        malelf_binary_set_alloc_type(&bin, MALELF_ALLOC_MALLOC);
        malelf_binary_open("/bin/ls", &bin);

        result = malelf_binary_get_ehdr(&bin, &ehdr);
        result = malelf_binary_get_shdr(&bin, &shdr);
        result = malelf_ehdr_get_shnum(&ehdr, &shnum);

        printf("Number of Entries SHT: %d\n", shnum);

        for (i = 0; i < shnum; i++) {
                malelf_shdr_get_offset(&shdr, &offset, i);
                printf("Offset: 0x%08x\n", offset);
        }

        malelf_binary_close(&bin);

        return 0;
}
```

### 2.5 - Módulo Infect ###

*************************
* FIXME: Falta terminar *
*************************


### 2.6 - Reportando Informações ###

  O programador fez o seu projeto utilizando a libmalelf e gostaria de mostrar o
resultado de alguma forma. Para isso existem duas formas de gerar relatórios
utilizando a libmalef, através de arquivos xml ou stdout.


#### 2.6.1 - Arquivos XML ####


  Para gerar as informações dentro de um arquivo XML a libmalelf dispõe de um
módulo chamado report. Com isso o programador pode enviar as informações do ELF
Header, Section Program Table e Program Header Table para um arquivo no
padrão XML.

```c
#include <stdio.h>
#include <malelf/binary.h>
#include <malelf/ehdr.h>
#include <malelf/shdr.h>
#include <malelf/phdr.h>
#include <malelf/defines.h>
#include <malelf/report.h>


int main()
{
    MalelfBinary bin;
    MalelfReport report;

    malelf_binary_init(&bin);
    malelf_binary_open("/bin/ls", &bin);
    malelf_report_open(&report, "/tmp/report.xml", MALELF_OUTPUT_XML);

    malelf_report_ehdr(&report, &bin);

    malelf_report_close(&report);
    malelf_binary_close(&bin);

    return 0;
}
```

  Agora vamos ver como ficou a saída.

```xml
<?xml version="1.0" encoding="UTF8"?>
<MalelfBinary>
 <MalelfEhdr>
  <type>2</type>
  <machine>3</machine>
  <version>1</version>
  <entry>0x0804c070</entry>
  <phoff>0x00000034</phoff>
  <shoff>0x0001a444</shoff>
  <flags>0</flags>
  <phentsize>32</phentsize>
  <phnum>9</phnum>
  <shentsize>40</shentsize>
  <shnum>28</shnum>
  <shstrndx>27</shstrndx>
 </MalelfEhdr>
```

### 2.6.2 - Stdout ###

  Para imprimir as informações formatadas no terminal, existe o módulo table,
responsável por criar uma tabela ascii e imprimir na shell. Com o objeto
MalelfTable, o programador consegue definir o tamanho da tabela, o título e o
número de linhas e colunas.

  Para esse exemplo, vamos novamente pegar uma função que é utilizada dentro do
projeto malelf.

  1 - Chama o método init do módulo;
  2 - Configura o título da tabela;
  3 - Configura os headers;
  4 - Pega o ELF Header;
  5 - Pega os valores desejados do ELF Header;
  6 - Imprime os valores utilizando o método  malelf_table_print();
  7 - Libera o objeto table chamando o método malelf_table_finish();

```c
#include <stdio.h>
#include <malelf/binary.h>
#include <malelf/ehdr.h>
#include <malelf/shdr.h>
#include <malelf/phdr.h>
#include <malelf/table.h>
#include <malelf/error.h>

int main()
{
        MalelfTable table;
        MalelfBinary bin;
        MalelfEhdr ehdr;
        _u32 value;

        malelf_binary_init(&bin);
        malelf_binary_open("/bin/ls", &bin);
        char *headers[] = {"Structure Member", "Description", "Value", NULL};

        /* Parameters: MalelfTable, width, rows, columns */
        if (MALELF_SUCCESS != malelf_table_init(&table, 78, 3, 3)) {
                return MALELF_ERROR;
        }

        /* Configurando o titulo da tabela */
        malelf_table_set_title(&table, "ELF Header");

        /* Salvando os headers */
        malelf_table_set_headers(&table, headers);

        malelf_binary_get_ehdr(&bin, &ehdr);

        /*  1 - Row */
        MalelfEhdrTable me_type;
        malelf_ehdr_get_type(&ehdr, &me_type);
        malelf_table_add_value(&table, (void*)"e_type", MALELF_TABLE_STR);
        malelf_table_add_value(&table, (void*)"Object Type", MALELF_TABLE_STR);
        malelf_table_add_value(&table,
                               (void*)me_type.meaning,
                               MALELF_TABLE_STR);

        /*  2 - Row */
        MalelfEhdrTable me_version;
        malelf_ehdr_get_version(&ehdr, &me_version);
        malelf_table_add_value(&table, (void*)"e_version", MALELF_TABLE_STR);
        malelf_table_add_value(&table, (void*)"Version", MALELF_TABLE_STR);
        malelf_table_add_value(&table,
                               (void*)me_version.value,
                               MALELF_TABLE_INT);

        /*  3 - Row */
        malelf_ehdr_get_entry(&ehdr, &value);
        malelf_table_add_value(&table, (void*)"e_entry", MALELF_TABLE_STR);
        malelf_table_add_value(&table, (void*)"Entry Point", MALELF_TABLE_STR);
        malelf_table_add_value(&table, (void*)value, MALELF_TABLE_HEX);

        malelf_table_print(&table);

        malelf_table_finish(&table);
        malelf_binary_close(&bin);

        return 0;
}
```

  E essa é a saída do nosso programa. =)

+-----------------------------------------------------------------------------+
|                                  ELF Header                                 |
+---------------------------+----------------------+--------------------------+
|     Structure Member      |     Description      |          Value           |
+---------------------------+----------------------+--------------------------+
|          e_type           |     Object Type      |     Executable file      |
|         e_version         |       Version        |            1             |
|          e_entry          |     Entry Point      |       0x0804c070         |
+---------------------------+----------------------+--------------------------+


### 2.7 - Debugando a libmalelf ###


  Existe a possibilidade de ver as mensagens que a libmalelf reporta. Para isso,
basta exportarmos uma váriavel de ambiente chamada MALELF_DEBUG.

    $ export MALELF_DEBUG=1

  Lembram do nosso primeiro exemplo utilizando a libmalelf? Pois então vamos ver
o que retorna da sua execução com a opção de debug ligada.

    $ ./hello

[INFO][Fri Jun 14 00:31:47 2013][malelf_binary_init][binary.c:235] MalelfBinary
structure initialized.

Hello Libmalelf bin[0xbfc9605c]

[INFO][Fri Jun 14 00:31:47 2013][malelf_binary_close][binary.c:409] Binary
'(null)' closed

  Caso você queira receber essas informações em um arquivo de log, pode
configurar a variável de ambiente MALELF_DEBUG_FILE.

    $ export MALELF_DEBUG_FILE = /tmp/libmalelf.log


## 3 - Projeto malelf ##

  Malelf é uma ferramenta que utiliza a libmalelf para analisar e infectar
binários ELF.
  Nessa parte iremos apenas demonstrar como utilizar o binário, porque toda a
inteligência do projeto fica dentro da libmalelf que já foi explicada
anteriormente.

### 3.1 - Build do malelf ###

  O processo de build da ferramenta malelf é bem simples.

Dependências:
- libmalelf

    $ ./configure
    $ make
    $ sudo make install

  Agora que o malelf está em sua máquina podemos começar a fazer alguns
exemplos.


### 3.2 - Usando o módulo dissect

  Agora vamos utilizar a ferramenta malelf para pegar as informações do binário.
Antes de tudo, vamos ver o help do módulo dissect.

    $ malelf dissect -h

This command display information about the ELF binary.
Usage: malelf dissect <options>
         -h, --help    	Dissect Help
         -i, --input   	Binary File
         -e, --ehdr    	Display ELF Header
         -s, --shdr    	Display Section Header Table
         -p, --phdr    	Display Program Header Table
         -S, --stable  	Display Symbol Table
         -f, --format  	Output Format (XML or Stdout). Default is Stdout.
         -o, --output  	Output File.
Example: malelf dissect -i /bin/ls -f xml -o /tmp/binary.xml

  Mostrando o ELF Header na shell:

    $ malelf dissect -i /bin/ls -e

  Mostrando o Program Header Table na shell:

    $ malelf dissect -i /bin/ls -p

  Mostrando o Section Header Table na shell:

    $ malelf dissect -i /bin/ls -s

  Para jogar as informações em arquivos XML é simples:

    $ malelf dissect -i /bin/ls -f xml -o /tmp/bin.txt

### 3.3 - Usando o módulo infect ###

****************************************
* FIXME: Faltando escrever essa parte. *
****************************************

## 4 - malelfgui ##

  MalelfGUI é um front-end visual para o projeto malelf, utilizando Qt. Está em
estágio inicial de desenvolvimento e, por isso, deixaremos essa parte para
escrever em outro momento. Porém sinta-se a vontade de entrar no github e meter
a mão na massa.

- https://github.com/SecPlus/malelfgui

##
5 - Links
##

[1] - Wikipedia: Executable and Linkable Format
  http://en.wikipedia.org/wiki/Executable_and_Linkable_Format

[2] - OS Dev - ELF
  http://wiki.osdev.org/ELF

[3] - Dissecando ELF
  http://0fx66.com/files/zines/cogumelo-binario/edicoes/1/DissecandoELF.txt

##
6 - Conclusão
##

  O projeto malelficus ainda está em sua fase inicial, provavelmente com muitos
bugs. A equipe de desenvoledores do projeto ainda é pequena e com pouco tempo
livre, pois a cerveja toma muito tempo dos programadores (sim, esse projeto foi
feito por um bando de alcoólatras).
  Então sinta-se livre para ajudar de qualquer forma com o projeto, seja codando,
reportando bugs ou dando ideias. Caso não tenha gostado do projeto, pode tacar
tomate, xingar a irmã e até a mãe que está tudo beleza, mas se falar mal do
código ai tu vai me ofender. hehehe =)
