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
7. Predição gênica / proteica
8. Anotação funcional

----------------------------------------------------------------

## Pré-requisitos

Para realizar esse tutorial, você precisa ter um computador com o [Ubuntu 18.04](https://ubuntu.com/) instalado. Caso não tenha, você pode seguir os passos abaixo para usar uma máquina virtual:

- Baixar e instalar a [VirtualBox](https://www.virtualbox.org/).
- Baixar e carregar a imagem (ISO) configurada do Ubuntu 18.04 (https://mega.nz/#!EYQhEISI!nNPEtzzITqjv9jYc8UgpkS7LbNmIebEUbPgN8HYgn_o) no VirtualBox. Depois, acesse o VirtualBox, vá em ```Arquivo > Importar appliance > Selecione o arquivo > Próximo > Modifique para 4 CPUs > Importar```. Inicie a máquina. Os dados para acesso à máquina são:

```
usuário: user
senha: user2020
```

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

### 4. Remoção de reads humanos

Essa etapa irá requerer o download do genoma humano (GRCh38), a construção de uma hash table para o alinhamento e o alinhamento propriamente dito utilizando [Bowtie2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml). Por ser bastante demorado, irei disponibilizar os arquivos resultantes no [link](https://mega.nz/#!5QBBzAJS!FHwDeEIJh4ml6nNJziW9yZf9zdpVvJG536bC7vAr_gU).

Baixar o arquivo ```SRR8180079.sam```, criar pasta em ```results/bowtie2/sam``` e copiar o arquivo para ela.

<b>-----> Você não precisa executar os comandos abaixo, pois os arquivos já foram baixados! <-----</b>

De qualquer forma, os passos para replicação dessa etapa são:

#### 4.1. Download do genoma humano já indexado

```
$ cd ..
$ mkdir -p "ref_dbs/human_db"
$ cd "ref_dbs/human_db"
$ wget ftp://ftp.ncbi.nlm.nih.gov/genomes/archive/old_genbank/Eukaryotes/vertebrates_mammals/Homo_sapiens/GRCh38/seqs_for_alignment_pipelines/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.bowtie_index.tar.gz
$ tar xvzf GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.bowtie_index.tar.gz
```

#### 4.2. Alinhamento ao genoma humano

```
$ cd ../..
$ mkdir -p results/bowtie2/sam
$ bowtie2 --threads 4 -x ref_dbs/human_db/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.bowtie_index -1 amostras/SRR8180079_1_val_1.fq -2 amostras/SRR8180079_2_val_2.fq -S results/bowtie2/sam/SRR8180079.sam
```

Será gerado o arquivo SAM, o qual armazena as sequências alinhadas à sequência de referência, bem como suas coordenadas genômicas.

OBS.: É esperado que nada (ou praticamente nada) alinhe ao genoma humano, pois os reads humanos já haviam sido retirados antes da submissão ao SRA.

<b>-----> Agora você pode voltar a executar os comandos normalmente! <-----</b>

#### 4.3. Remoção de reads humanos

Inicialmente, convertemos o arquivo SAM para um arquivo binário (BAM):

```
$ mkdir -p results/bowtie2/bam
$ samtools view -bS results/bowtie2/sam/SRR8180079.sam > results/bowtie2/bam/SRR8180079.bam
```

Depois, pegamos os reads não mapeados ao genoma humano: 

```
$ mkdir -p results/bowtie2/unmapped
$ samtools view -b -f 12 -F 256 results/bowtie2/bam/SRR8180079.bam > results/bowtie2/unmapped/SRR8180079.bam
```

Finalmente, transformamos o arquivo BAM para os dois arquivos FASTQ para fazer a classificação taxonômica posteriormente: 

```
$ mkdir -p results/bowtie2/fastq
$ bedtools bamtofastq -i results/bowtie2/unmapped/SRR8180079.bam -fq results/bowtie2/fastq/SRR8180079_1.fastq -fq2 results/bowtie2/fastq/SRR8180079_2.fastq
```

### 5. Classificação taxonômica, filtragem e visualização de abundância microbiana

Nessa etapa, os reads serão comparados contra um abrangente banco de dados utilizando o [Kraken2](https://ccb.jhu.edu/software/kraken2/), a fim de identificar os micro-organismos presentes neste metagenoma. 

Essa etapa irá requerer o download de genomas de vírus, bactérias, fungos e parasitas do NCBI, a construção de uma hash table para o alinhamento e o alinhamento propriamente dito. Por ser bastante demorado, irei disponibilizar os arquivos resultantes no [link](https://mega.nz/#F!pMYRQSZQ!7aV1MHnq0qQNub63W_hNkg).

Baixar os arquivos da pasta ```kraken2``` no link disponibilizado e inserir dentro da pasta ```results``` do seu computador.

<b>-----> Você não precisa executar os comandos abaixo, pois os arquivos já foram baixados! <-----</b>

De qualquer forma, os passos para replicação dessa etapa são:

#### 5.1. Download dos genomas de referência de micro-organismos

Inicialmente, fazemos o download dos genomas presentes no Kraken2:

```
$ kraken2-build --download-taxonomy --threads 4 --db ref_dbs/pathogens_db
$ kraken2-build --download-library viral --threads 4 --db ref_dbs/pathogens_db
$ kraken2-build --download-library archaea --threads 4 --db ref_dbs/pathogens_db
$ kraken2-build --download-library fungi --threads 4 --db ref_dbs/pathogens_db
$ kraken2-build --download-library protozoa --threads 4 --db ref_dbs/pathogens_db
$ kraken2-build --download-library bacteria --threads 4 --db ref_dbs/pathogens_db
```

Depois, fazemos o download do genoma de outros micro-organismos que já foram identificados como causadores de neuroinfecções, mas não estavam no banco de dados anterior. Para isso rodamos o script ```download_extra.sh```:

```
$ chmod +x download_extra.sh
$ ./download_extra.sh
```

#### 5.2. Construção do index

```
$ kraken2-build --build --threads 4 --max-db-size 13000000000 --db ref_dbs/pathogens_db
```

#### 5.3. Classificação taxonômica

```
$ DB_PATH="ref_dbs/pathogens_db"
$ KRAKEN2="results/kraken2"
$ mkdir -p $KRAKEN2
$ mkdir -p $KRAKEN2/classified
$ mkdir -p $KRAKEN2/unclassified
$ mkdir -p $KRAKEN2/tabular
$ mkdir -p $KRAKEN2/report
$ kraken2 -db $DB_PATH --threads 4 \
            --report $KRAKEN2/report/SRR8180079.kreport \
            --classified-out $KRAKEN2/classified/SRR8180079#.fastq \
            --unclassified-out $KRAKEN2/unclassified/SRR8180079#.fastq \
            --output $KRAKEN2/tabular/SRR8180079.txt \
            --paired results/bowtie2/fastq/SRR8180079_1.fastq results/bowtie2/fastq/SRR8180079_2.fastq
```

<b>-----> Agora você pode voltar a executar os comandos normalmente! <-----</b>

Analise os resultados gerados em ```results/kraken2/classified```, ```results/kraken2/unclassified```, ```results/kraken2/report``` e ```results/kraken2/tabular```.

#### 5.4. Filtragem de resultados para incluir somente patógenos de neuroinfecções

Para reduzir nosso escopo de análise, podemos utilizar um script em Python para filtrar os resultados para considerar somente patógenos de neuroinfecções:

```
$ python3 filtrar_patogenos.py 
```

Analise os resultados gerados em ```results/kraken_genus_species_strains``` e ```kraken_pathogens```.

#### 5.5. Estimativa de abundância (KRONA plot)

Para saber a abundância total de micro-organismos nas amostras, podemos utilizar a ferramenta [KRONA](https://github.com/marbl/Krona):

```
$ mkdir -p ferramentas/Krona-master/KronaTools/taxonomy
$ ferramentas/Krona-master/KronaTools/updateTaxonomy.sh
$ mkdir -p results/krona
$ ImportTaxonomy.pl -o results/krona/SRR8180079_krona.html -t 3 -s 4 results/kraken2/tabular/SRR8180079.txt
```

Agora podemos visualizar o gráfico gerado, que mostra a composição microbiana da amostra, no arquivo ```SRR8180079_krona.html```, localizado na pasta ```results/krona```.

### 6. Montagem de metagenoma

Para montar os reads em fragmentos maiores com o objetivo posterior de identificar genes virais, utilizamos a ferramenta [megahit](https://github.com/voutcn/megahit):

```
$ mkdir -p results/megahit
$ megahit -1 results/bowtie2/fastq/SRR8180079_1.fastq -2 results/bowtie2/fastq/SRR8180079_2.fastq -o results/megahit/SRR8180079.out
```

O resultado final da montagem pode ser encontrado nos arquivos ```final.contings.fa```, dentro da pasta ```results/megahit```.

## 7. Predição gênica / proteica

Inicialmente, vamos fazer a predição gênica utilizando o [Prokka](https://github.com/tseemann/prokka), uma ferramenta bastante utilizada para predição em procariotos, mas que também funciona bem para vírus.

```
$ prokka results/megahit/SRR8180079.out/final.contigs.fa --genus Lentivirus --kingdom Virus --norrna --notrna --metagenome --cpus 4 --outdir results/prokka_hiv
```

Podemos analisar os arquivos gerados em ```results/prokka_hiv```. Especialmente o arquivo ```PROKKA_02112020.tsv```, que mostra alguns genes virais detectados (linha 225 em diante).

Para uma análise mais específica, podemos construir um banco de dados com as proteínas do HIV-1 e alinhar as nossas sequências. Isso acaba sendo fácil pois o HIV-1 conta com apenas 10 genes e 25 variantes de proteínas.

Para isso, baixamos o arquivo ```hiv_proteins``` no [link](https://mega.nz/#!JdAjBKLD!5ExeJUBnV4fyCeiQxPjpq_rE4LHjBms7lH7ksxmT0nE).

Depois, construímos nosso banco de dados:

```
$ makeblastdb -in ref_dbs/hiv_proteins.fasta -title hiv_proteins -dbtype prot -out ref_dbs/hiv_proteins
```

Agora, usamos o BLASTX (sequência de nucleotídeos traduzida para proteína), para buscar as correspondências baseado no metagenoma montado:

```
$ mkdir -p results/blastx
$ blastx -query results/megahit/SRR8180079.out/final.contigs.fa -db ref_dbs/hiv_proteins -out results/blastx/SRR8180079.tab -evalue 1e-5 -outfmt 6
```

Para analisar os dados apresentados em ```results/blastx```, consideremos a seguinte tabela, que apresenta o significado das colunas em ordem:

Field | Description
| --- | --- |
| qseqid | query (e.g., gene) sequence id
| sseqid | subject (e.g., reference genome) sequence id
| pident | percentage of identical matches
| length | alignment length
| mismatch | number of mismatches
| gapopen | number of gap openings
| qstart | start of alignment in query
| qend | end of alignment in query
| sstart | start of alignment in subject
| send | end of alignment in subject
| evalue | expect value
| bitscore | bit score

As proteínas encontradas na busca foram:

```
$ awk '{print $2}' results/blastx/SRR8180079.tab
NP_789739.1
NP_705927.1
NP_789740.1
NP_057849.4
NP_705927.1
NP_789740.1
NP_057849.4
NP_057849.4
NP_057849.4
NP_789740.1
NP_705927.1
NP_057856.1
NP_057856.1
NP_057850.1
NP_789739.1
NP_789739.1
NP_705927.1
NP_705927.1
NP_789740.1
NP_789740.1
NP_057849.4
NP_057849.4
```

Removendo as duplicatas ficamos com:

```
NP_789739.1
NP_705927.1
NP_789740.1
NP_057849.4
NP_057856.1
NP_057850.1
```

Podemos procurar por elas no [NCBI protein](https://www.ncbi.nlm.nih.gov/protein) para ver qual sua função.


## 8. Anotação funcional

Para identificar o que essas proteínas que deram match com a nossa amostra fazem, pegamos as suas sequências de aminoácidos e submetemos ao site [Pannzer2](http://ekhidna2.biocenter.helsinki.fi/sanspanz/).

Primeiro, baixamos o arquivo FASTA já preparado com essas sequências no [link](https://mega.nz/#!YVICmAaA!ukmf-MWlkuaVYzaf60HhvrKc0Zxzc6fABXziN8_D_pE). 

Finalmente, acessamos o Pannzer (http://ekhidna2.biocenter.helsinki.fi/sanspanz/) e vamos na aba ```Annotate```. Carregamos nosso arquivo fasta chamado ```hit_proteins.fasta```.

Preencher na aba ```Optional inputs```.

```
Job title: Anotação proteínas HIV-1
Scientific name of query species: Human immunodeficiency virus 1
```

Deixar marcada a opção ```Interactive``` na aba ```Select interactive or batch processing``` e clicar em ```Submit```.

Analisar os resultados e dar ```Ctrl+S``` para salvar a página HTML.

A predição de genes nos reads de espécies de procariotos e eucariotos, bem como as vias (redes de interações entre moléculas) responsáveis pelo processo neuroinfeccioso também podem ser estudadas, mas isso é um tema para um próximo tutorial ;)

----------------------------------------

## Autores

- Vinícius Bonetti Franceschi (vinibfranc@gmail.com)
- Claudia Elizabeth Thompson (thompson.ufcspa@gmail.com)

## Agradecimentos

Agradecemos os autores do artigo [Chronic Meningitis Investigated via Metagenomic Next-Generation Sequencing](https://jamanetwork.com/journals/jamaneurology/fullarticle/2678438) pela publicação e disponibilização das amostras publicamente no SRA.