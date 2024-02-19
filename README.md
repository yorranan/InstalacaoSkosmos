Esse tutorial é um tutorial foi realizado no Debian GNU/Linux 12 (bookworm) no Windows 11 x86_64.

Caso se deseje acessar via SSH a máquina em que se está instalando a ferramenta é necessário instalar no linux da seguinte forma:

```
sudo apt install openssh-server
```
 
Para acessar basta usar `ssh -p 2222 localhost 

## Instalando Apache Jena Fuseki

### Instalando Java 11+

Um dos pré requisitos para o Fuseki é o Java 11 ou superior, para isso é necessário a JRE ou JDK

~~~~
sudo apt update
sudo apt install default-jre-headless
~~~~
Caso exista mais de uma versão do Java rodando na máquina pode se listar e *setar *a versão desejada para tal, uma vez que é necessário estar com a versão 11 ou superior do Java.

```
sudo update-java-alternatives -l
sudo update-java-alternatives --set java-1.11.0-openjdk-amd64
```

## Intalando Fuseki

Feito o processo de instalação do Java podemos instalar o Fuseki 

```
cd ~
sudo apt install wget
wget https://archive.apache.org/dist/jena/binaries/apache-jena-fuseki-4.10.0.tar.gz
cd /opt
sudo tar xzf ~/apache-jena-fuseki-4.10.0.tar.gz
sudo ln -s apache-jena-fuseki-4.10.0 fuseki
```

Verificando se tudo corretamente instalando:

```
cd /opt/fuseki/
./fuseki-server --help
./fuseki-server --version
```

Para o correto funcionamento e maior segurança é necessário criar um usuário comum denominado `fuseki`

`sudo adduser --system --home /opt/fuseki --no-create-home fuseki`

- O código Fuseki (a distribuição do servidor) entra em , como acima (na verdade, um link simbólico) `/opt/fuseki`
- as bases de dados estão em `/var/lib/fuseki`
- os arquivos de log vão em `/var/log/fuseki`
- arquivos de configuração vão em `/etc/fuseki`

```
# diretórios de banco de dados
cd /var/lib
sudo mkdir -p fuseki/{backups,databases,system,system_files}
sudo chown -R fuseki fuseki

# logs
cd /var/log
sudo mkdir fuseki
sudo chown fuseki fuseki

# configs
cd /etc
sudo mkdir fuseki
sudo chown fuseki fuseki

#link simbólico
cd /etc/fuseki
sudo ln -s /var/lib/fuseki/* .
sudo ln -s /var/log/fuseki logs
```

### Configuração para o fuseki iniciar no boot do sistema

```
sudo nano /etc/systemd/system/fuseki.service`
```
Insira nesse arquivo:

```
[Unit]
Description=Fuseki
[Service]
Environment=FUSEKI_HOME=/opt/fuseki
Environment=FUSEKI_BASE=/etc/fuseki
Environment=JVM_ARGS=-Xmx2G
User=fuseki
ExecStart=/opt/fuseki/fuseki-server
Restart=on-failure
RestartSec=15
[Install]
WantedBy=multi-user.target
```

> O argumento `Environment=JVM_ARGS=-Xmx2G` define o consumo de memória da JVM
Após isso: 
```
sudo systemctl start fuseki
sudo systemctl status fuseki
```

Se ao rodar `status` não for iniciado corretamente verifique a situação usando:

```
cat /var/log/fuseki/stderrout.log
sudo journalctl -xe
```

Sendo tudo correto:

`sudo systemctl enable fuseki`

Tudo certo acesse: http://localhost:3030/

No painel do fuseki faça:

1. Crie um dataset nomeado skosmos do tipo persistente
2. Deixe em branco o espaço de nome do grafo
3. Faça upload do RDF
4. Verifique em info se é possível carregar as tuplas

### Configurando index Fuseki

```
sudo service fuseki stop
sudo nano /etc/fuseki/configuration/skosmos.ttl
```

Dentro de skosmos.ttl copie:

```
@prefix :      <http://base/#> .
@prefix rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix tdb2:  <http://jena.apache.org/2016/tdb#> .
@prefix ja:    <http://jena.hpl.hp.com/2005/11/Assembler#> .
@prefix rdfs:  <http://www.w3.org/2000/01/rdf-schema#> .
@prefix fuseki: <http://jena.apache.org/fuseki#> .
@prefix text:  <http://jena.apache.org/text#> .
@prefix skos:  <http://www.w3.org/2004/02/skos/core#> .

ja:DatasetTxnMem  rdfs:subClassOf  ja:RDFDataset .
ja:MemoryDataset  rdfs:subClassOf  ja:RDFDataset .
ja:RDFDatasetOne  rdfs:subClassOf  ja:RDFDataset .
ja:RDFDatasetSink  rdfs:subClassOf  ja:RDFDataset .
ja:RDFDatasetZero  rdfs:subClassOf  ja:RDFDataset .

tdb2:DatasetTDB  rdfs:subClassOf  ja:RDFDataset .
tdb2:DatasetTDB2  rdfs:subClassOf  ja:RDFDataset .

tdb2:GraphTDB  rdfs:subClassOf  ja:Model .
tdb2:GraphTDB2  rdfs:subClassOf  ja:Model .

<http://jena.hpl.hp.com/2008/tdb#DatasetTDB>
    rdfs:subClassOf  ja:RDFDataset .

<http://jena.hpl.hp.com/2008/tdb#GraphTDB>
    rdfs:subClassOf  ja:Model .

text:TextDataset
    rdfs:subClassOf  ja:RDFDataset .

:service_tdb_all  a               fuseki:Service ;
    rdfs:label                    "TDB2+text skosmos" ;
    fuseki:dataset                :text_dataset ;
    fuseki:name                   "skosmos" ;
    fuseki:serviceQuery           "query" , "" , "sparql" ;
    fuseki:serviceReadGraphStore  "get" ;
    fuseki:serviceReadQuads       "" ;
    fuseki:serviceReadWriteGraphStore "data" ;
    fuseki:serviceReadWriteQuads  "" ;
    fuseki:serviceUpdate          "" , "update" ;
    fuseki:serviceUpload          "upload" .

:text_dataset a text:TextDataset ;
    text:dataset :tdb_dataset_readwrite ;
    text:index :index_lucene . 

:tdb_dataset_readwrite
    a tdb2:DatasetTDB2 ;
    # tdb2:unionDefaultGraph true ;
    tdb2:location  "/etc/fuseki/databases/skosmos" .

:index_lucene a text:TextIndexLucene ;
    text:directory <file:/etc/fuseki/databases/skosmos/text> ;
    text:entityMap :entity_map ;
    text:storeValues true .

# Text index configuration for Skosmos
:entity_map a text:EntityMap ;
    text:entityField      "uri" ;
    text:graphField       "graph" ;
    text:defaultField     "pref" ;
    text:uidField         "uid" ;
    text:langField        "lang" ;
    text:map (
         # skos:prefLabel
         [ text:field "pref" ;
           text:predicate skos:prefLabel ;
           text:analyzer [ a text:LowerCaseKeywordAnalyzer ]
         ]
         # skos:altLabel
         [ text:field "alt" ;
           text:predicate skos:altLabel ;
           text:analyzer [ a text:LowerCaseKeywordAnalyzer ]
         ]
         # skos:hiddenLabel
         [ text:field "hidden" ;
           text:predicate skos:hiddenLabel ;
           text:analyzer [ a text:LowerCaseKeywordAnalyzer ]
         ]
         # skos:notation
         [ text:field "notation" ;
           text:predicate skos:notation ;
           text:analyzer [ a text:LowerCaseKeywordAnalyzer ]
         ]
     ) . 
```

Então:

```
sudo service fuseki start
```

Se tudo correr bem, isso criará um índice de jena-text Lucene em `/var/lib/fuseki/databases/skosmos/text`, ou seja, como um subdiretório do banco de dados TDB ao qual ele está vinculado.

### Instalando requirements do SKOSMOS

É necessário instalar o Apache2, PHP e bibliotecas necessárias:

```
sudo apt install apache2 php8.2 libapache2-mod-php8.2 php8.2 php8.2-xsl php8.2-intl php8.2-mbstring php8.2-curl
```
ou 

```
sudo apt install apache2 php8.1 libapache2-mod-php8.1 php8.1 php8.1-xsl php8.1-intl php8.1-mbstring php8.1-curl
```
ou 

```
sudo apt install apache2 php7.4 libapache2-mod-php7.4 php7.4 php7.4-xsl php7.4-intl php7.4-mbstring php7.4-curl
```

Acesse: http://localhost

Se não estiver funcionando use:

```
sudo service apache2 start
sudo systemctl enable apache2
```
Para o WSL pode ser necessário ativar as opções de systemd, verifique em: https://learn.microsoft.com/pt-br/windows/wsl/wsl-config

### Configurando o apache para o SKOMOS

Edite as configurações do apache através de:

```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Dentro de `<VirtualHost *:80>` insira:

```
        <Directory /var/www/html>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                allow from all
        </Directory>
```

Você também deve ativar os módulos Apache requerido pelo SKOSMOS:

```
sudo a2enmod rewrite
sudo a2enmod expires
sudo service apache2 restart
```

### Instalando o SKOMOS

A sequência de códigos abaixo instala a versão 2.17 do SKOSMOS disponível no repositório oficial dentro da pasta `/srv/`, também instala as dependências necessárias com o composer.
> Note que se houver mensagem de erro siga as instruções indicadas.
```
cd /srv
sudo mkdir Skosmos
sudo chown `whoami` Skosmos
sudo apt install git
git clone -b v2.17-maintenance https://github.com/NatLibFi/Skosmos.git /srv/Skosmos
cd /srv/Skosmos/
curl -sS https://getcomposer.org/installer | php
php composer.phar install
sudo ln -s /srv/Skosmos /var/www/html/Skosmos
```

### Configurando o SKOMOS

No arquivo `config.ttl `da pasta `/srv/Skosmos` deve passar a conter o arquivo de configuração, para mais detalhes sobre as definições utilizadas leia a documentação de [configuração](https://github.com/NatLibFi/Skosmos/wiki/Configuration) .

```
nano config.ttl
```

Insira com: 

```
@prefix void: <http://rdfs.org/ns/void#> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl: <http://www.w3.org/2002/07/owl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
@prefix dc: <http://purl.org/dc/terms/> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix wv: <http://vocab.org/waiver/terms/norms> .
@prefix sd: <http://www.w3.org/ns/sparql-service-description#> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix skosmos: <http://purl.org/net/skosmos#> .
@prefix isothes: <http://purl.org/iso25964/skos-thes#> .
@prefix mdrtype: <http://publications.europa.eu/resource/authority/dataset-type/> .
@prefix : <#> .

# Skosmos main configuration

:config a skosmos:Configuration ;
    # SPARQL endpoint
    # a local Fuseki server is usually on localhost:3030
    # skosmos:sparqlEndpoint <http://localhost:3030/ds/sparql> ;
    # use the dev.finto.fi endpoint where the example vocabularies reside
    # skosmos:sparqlEndpoint <http://api.dev.finto.fi/sparql> ;
    skosmos:sparqlEndpoint <http://localhost:3030/skosmos/sparql> ;
    # Ative alinha abaixo apenas quando instalar o Varnish
    # skosmos:sparqlEndpoint <http://localhost:6081/skosmos/sparql> ;
    # sparql-query extension, or "Generic" for plain SPARQL 1.1
    # set to "JenaText" instead if you use Fuseki with jena-text index
    skosmos:sparqlDialect "JenaText" ;
    # whether to enable collation in sparql queries
    skosmos:sparqlCollationEnabled false ;
    # HTTP client configuration
    skosmos:sparqlTimeout 20 ;
    skosmos:httpTimeout 5 ;
    # customize the service name
    skosmos:serviceName "Skosmos" ;
    # customize the base element. Set this if the automatic base url detection doesn't work. For example setups behind a proxy.
    skosmos:baseHref "http://localhost/Skosmos/" .
    # interface languages available, and the corresponding system locales
    :config skosmos:languages (
    [ rdfs:label "en" ; rdf:value "en_GB.utf8" ]
    [ rdfs:label "es" ; rdf:value "es_ES.utf8" ]
    [ rdfs:label "pt" ; rdf:value "pt_BR.utf8" ]
    [ rdfs:label "de" ; rdf:value "de_DE.utf8" ]
    ) .
    # how many results (maximum) to load at a time on the search results page
    :config skosmos:searchResultsSize 20 ;
    # how many items (maximum) to retrieve in transitive property queries
    skosmos:transitiveLimit 1000 ;
    # whether or not to log caught exceptions
    skosmos:logCaughtExceptions false ;
    # set to TRUE to enable logging into browser console
    skosmos:logBrowserConsole false ;
    # set to a logfile path to enable logging into log file
    # skosmos:logFileName "" ;
    # a default location for Twig template rendering
    skosmos:templateCache "/tmp/skosmos-template-cache" ;
    # customize the css by adding your own stylesheet
    # skosmos:customCss "resource/css/stylesheet.css" ;
    # default email address where to send the feedback
    skosmos:feedbackAddress "" ;
    # email address to set as the sender for feedback messages
    skosmos:feedbackSender "" ;
    # email address to set as the envelope sender for feedback messages
    skosmos:feedbackEnvelopeSender "" ;
    # whether to display the ui language selection as a dropdown (useful for cases where there are more than 3 languages)
    skosmos:uiLanguageDropdown false ;
    # whether to enable the spam honey pot or not, enabled by default
    skosmos:uiHoneypotEnabled true ;
    # default time a user must wait before submitting a form
    skosmos:uiHoneypotTime 5 ;
    # plugins to activate for the whole installation (including all vocabularies)
    skosmos:globalPlugins () .

# Skosmos vocabularies

:tbcc a skosmos:Vocabulary, void:dataset ;
    dc:title "Tesauro Brasileiro de Ciência da Computação"@pt ;
    skosmos:shortName "tesaurobrasileirodeciênciadacomputação" ;
    dc:subject :cat_general ;
    dc:type mdrtype:THESAURUS ;
    void:uriSpace "http://lod.unicentro.br/sparql" ;
    skosmos:language "en", "es", "pt" ;
    skosmos:defaultLanguage "pt" ;
    skosmos:showTopConcepts true ;
    skosmos:fullAlphabeticalIndex true ;
    skosmos:groupClass isothes:ConceptGroup ;
    void:sparqlEndpoint <http://localhost:3030/skosmos/sparql> .

:categories a skos:ConceptScheme;
    skos:prefLabel "Skosmos Vocabulary Categories"@en
.

:cat_general a skos:Concept ;
    skos:topConceptOf :categories ;
    skos:inScheme :categories ;
    skos:prefLabel "Yleiskäsitteet"@fi,
        "Allmänna begrepp"@sv,
        "General concepts"@en
.

mdrtype:THESAURUS a skos:Concept ;
    skos:prefLabel "Тезаурус"@bg, "Tezaurus"@cs, "Tesaurus"@da, "Thesaurus"@de, "Θησαυρός"@el, "Thesaurus"@en, "Tesaurus"@et, "Tesaurus"@fi, "Thésaurus"@fr, "Pojmovnik"@hr, "Tezaurusz"@hu, "Tesauro"@it, "Tēzaurs"@lv, "Tezauras"@lt, "Teżawru"@mt, "Thesaurus"@nl, "Tesaurus"@no, "Tezaurus"@pl, "Tesauro"@pt, "Tezaur"@ro, "Synonymický slovník"@sk, "Tezaver"@sl, "Tesauro"@es, "Tesaurus"@sv
.

mdrtype:ONTOLOGY a skos:Concept ;
    skos:prefLabel "Онтология"@bg, "Ontologie"@cs, "Ontologi"@da, "Ontologie"@de, "Οντολογία"@el, "Ontology"@en, "Ontoloogia"@et, "Ontologia"@fi, "Ontologie"@fr, "Ontologija"@hr, "Ontológia"@hu, "Ontologia"@it, "Ontoloģija"@lv, "Ontologija"@lt, "Ontoloġija"@mt, "Ontologie"@nl, "Ontologi"@no, "Struktura pojęciowa"@pl, "Ontologia"@pt, "Ontologie"@ro, "Ontológia"@sk, "Ontologija"@sl, "Ontología"@es, "Ontologi"@sv
.
```

Salve e instale o suporte as linguagens definidas:

```
sudo apt install gettext
sudo locale-gen en_GB.utf8
sudo locale-gen es_ES.utf8
sudo locale-gen pt_BR.utf8

```
Restarte o Apache

```
sudo service apache2 restart
```

Então acesse: http://localhost/Skosmos/
