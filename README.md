# HIV-ID-GenePredict-Annotation
Tutorial de análise metagenômica para detecção de patógeno viral, predição e anotação funcional de genes em amostra de neuroinfecção. 

Nessa amostra temos um caso de encefalopatia recorrente causada pelo vírus HIV-1 publicada juntamente com outros casos de neuroinfecção na revista JAMA Neurology. O artigo completo pode ser lido [aqui](https://jamanetwork.com/journals/jamaneurology/fullarticle/2678438).

Este caso clínico é descrito da seguinte forma:

> <b>Encefalopatia recorrente com infecção conhecida pelo HIV-1:</b>

> O participante 4 era um homem de 55 anos com uma infecção por HIV-1 sem tratamento que havia sido diagnosticada oito anos antes. Ele se apresentou ao departamento de emergência 2 semanas de confusão e diminuição da produção verbal, além de vários meses de tosse, suores noturnos intermitentes e perda de peso; sua contagem de CD4 era de 6 células/μL. A ressonância magnética nuclear (RMN) do cérebro mostrou hiperintensidades em T2 confluentes e não-significativas da substância branca bilateral. Um exame no LCR mostrou que as contagens de leucócitos e hemácias eram 0/μL, seu nível de glicose era de 34 mg/dL e seu nível total de proteínas era de 93 mg/dL. Não foram observadas leituras dos vírus JC, herpesvírus ou patógenos fúngicos. Seu curso clínico foi consistente com demência por HIV-1, e nenhuma infecção oportunista no SNC foi identificada após testes extensivos.

## Visão geral do pipeline

Nosso tutorial irá conter as seguintes etapas:

1. Configuração das ferramentas de bioinformática
2. Download das amostras
3. Controle de qualidade
4. Remoção de reads humanos
5. Classificação taxonômica, filtragem e visualização de abundância microbiana
6. Montagem de metagenoma
7. Predição gênica
8. Anotação funcional de genes

----------------------------------------------------------------

## Pré-requisitos

Para realizar esse tutorial, você precisa ter um computador com o [Ubuntu 18.04](https://ubuntu.com/) instalado. Caso não tenha, você pode seguir os passos abaixo para usar uma máquina virtual:

- Baixar e instalar a [VirtualBox](https://www.virtualbox.org/).
- Baixar e carregar a imagem (ISO) do [Ubuntu 18.04](http://releases.ubuntu.com/18.04/) no VirtualBox. Para isso, você pode seguir este [tutorial](https://www.youtube.com/watch?v=zsqJhle7CXE).

## Pipeline

Inicialmente, para executar os comandos que queremos vamos precisar o Terminal. Para abrir ele, pressione as teclas ```Ctrl+Alt+T```.

Com o terminal aberto, podemos criar uma pasta chamada ```tutorial_2``` e entrar nela. Para isso, basta digitarmos:

```
$ mkdir tutorial_2
$ cd tutorial_2
```

Para que possamos ter acesso aos arquivos deste tutorial, e para tê-los na nossa máquina, iremos clonar o repositório que criei no Github.

Primeiro, instalamos o git:

```
$ sudo apt-get install git
```

Depois, clonamos o repositório e entramos na pasta principal:

```
$ git clone https://github.com/vinibfranc/HIV-ID-GenePredict-Annotation
$ cd HIV-ID-GenePredict-Annotation
```

Pronto! Agora já podemos iniciar nosso tutorial!

### 1. Configuração das ferramentas de bioinformática

Primeiramente iremos configurar as ferramentas necessárias para a execução do pipeline para não nos preocuparmos com isso mais tarde. 

O primeiro passo é dar permissão para esse script ```configuracao.sh``` ser executado:

```
$ chmod +x configuracao.sh
```

Para rodar o script, basta rodar o script passando como parâmetro a pasta que vai armazenar os arquivos das ferramentas:

```
$ ./configuracao.sh ferramentas
```

OBS.: O caminho irá variar de acordo com o computador, por isso você precisa escolher um caminho válido no seu computador.

Depois, precisamos salvar as alterações no arquivo ```~/.bashrc```, fazendo:

```
$ source ~/.bashrc
```

### 2. Download das amostras

Inicialmente, iremos criar uma pasta para colocar nossas amostras:

```
$ mkdir -p amostras
$ cd amostras
```

As amostras no formato FASTQ podem ser baixadas diretamento do [Sequence Read Archive](https://www.ncbi.nlm.nih.gov/sra) (SRA) com a ferramenta [sra-tools](https://github.com/ncbi/sra-tools) utilizando o seguinte comando:

```
$ fasterq-dump --split-3 SRR8180079
```

Nossa amostra foi separada em dois arquivos, pois se trata de um sequenciamento paired-end. Para visualizar alguns reads da amostra com código de acesso SRR8180079, podemos executar os comandos: 

```
$ head SRR8180079_1.fastq
$ head SRR8180079_2.fastq
```

Para visualizar todos, rodamos:

```
$ cat SRR8180079_1.fastq
$ cat SRR8180079_2.fastq
```

### 3. Controle de qualidade

Para visualizar como estão nossos reads após o sequenciamento utilizamos o [FASTQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/):

```
$ fastqc *.fastq -t 4
```

Para visualizar os relatórios de qualidade (QC) das amostras, podemos abrir e analisar os arquivos ```SRR8180079_1_fastqc.html``` e ```SRR8180079_2_fastqc.html``` gerados na pasta ```amostras/```.

Para remover reads de baixa qualidade e adaptadores das amostras, utilizamos as ferramentas [trim_galore](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/) e [cutadapt](https://cutadapt.readthedocs.io/en/stable/):


```
$ trim_galore --quality 30 --phred33 --fastqc_args "-t 4" --paired SRR8180079_1.fastq SRR8180079_2.fastq
```

Agora iremos visualizar os relatórios de qualidade (QC) das amostras após o controle de qualidade, acessíveis nos arquivos ```SRR8180079_1_val_1_fastqc.html``` e ```SRR8180079_2_val_2_fastqc.html``` gerados na pasta ```amostras/```.

Os arquivos com os reads que passaram no controle de qualidade e podem ser usados nas etapas posteriores são: ```SRR8180079_1_val_1.fq``` e ```SRR8180079_2_val_2.fq```.