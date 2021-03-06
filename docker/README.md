# Rodando em Contêiner Docker

## Baixando os binários

```bash
cd bin
# Executar uma única vez o comando abaixo
./get-binaries-and-sources
cd ..
```

## Criando a imagem Docker

```bash
./build-docker-image.sh
```

## Iniciando os Conteineres

### Configuração de DNS e Network

Agora, supondo que sua rede (subnet) seja 192.168.0.0 no computador Host (macOS, por exemplo), faça o seguinte:

```bash
# limpando networks inúteis criadas anteriormente
docker network prune -f
# criando uma Network para o Spark usar entre os contêineres
# docker network create spark # Default para subrede é 172.18.0.0/16
# crio outra rede compatível com minha subrede usando a mesma subnet
docker network create --subnet=192.168.0.0/16 spark 
# listando as Networks existentes
docker network ls
```

### Iniciando o Contêiner do Master 

Abre-se uma janela de Terminal e executa-se:

```bash
docker run --rm --name spark-master -h spark-master \
           --network=spark --ip 192.168.0.170 \
           -p 19092:19092 \
           -p 7077:7077 -p 8080:8080 \
           -v $PWD/DATA:/spark/DATA -d parana/wff
```

Depois execute o comando abaixo para confirmar o endereço IP do Contêiner

```
docker exec -i -t  spark-master ip addr | grep inet
```


### Iniciando os Conteineres dos Workers 1, 2 e 3 ...

**Para cada um dos Workers** abre-se uma janela de Terminal independente e executa-se em cada uma delas os comandos:

```bash
docker run --rm --name spark-worker1 -h spark-worker1 -p 8081:8081 \
           --network=spark --ip 192.168.0.171 \
           -v $PWD/DATA:/spark/DATA -d parana/wff
```

```bash
docker run --rm --name spark-worker2 -h spark-worker2 -p 8082:8081 \
           --network=spark  --ip 192.168.0.172 \
           -v $PWD/DATA:/spark/DATA -d parana/wff
```

```bash
docker run --rm --name spark-worker3 -h spark-worker3 -p 8083:8081 \
           --network=spark  --ip 192.168.0.173 \
           -v $PWD/DATA:/spark/DATA -d parana/wff
```

### Listando os Contêineres

O comando abaixo mostra os contêineres carregados na memória.

```bash
docker ps
```


O comando abaixo mostra todos os contêineres incluindo os que não estão em execução.

```bash
docker ps -a
```

Uma shell foi criada para executar os comandos acima de forma simplicicada.
A diferença é que os conteineres são invocados no modo Background. 
Para executá-la use:

```bash
./start-all.sh
```

Como os conteineres estão executando em Background para interagis com eles
temos de executar, por exemplo:

```
docker exec -i -t spark-master bash 
```

### Inspecionando o volume /spark/DATA

O volume `/spark/DATA` é compartilhado entre todos os conteineres (master e workers) 
e pode ser inspecionado, usado e mantido por outro contêiner. Veja o exemplo simples
abaixo:

```bash
docker run -v $PWD/DATA:/spark/DATA -w /spark/DATA busybox ls -la
```

Neste exemplo listamos o conteúdo do volume `/spark/DATA` usando a imagem `busybox`
que fornece comandos de bash numa imagem de apenas 1.11 MB.

Outro exemplo:

```bash
docker run -v $PWD/DATA:/spark/DATA -w /spark/DATA busybox cat f_cmd.sh
```

### Listando ID, NAME, STATUS e SIZE de todos os conteineres que usam a rede spark

```bash
alias dl='docker ps --all --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Size}}" --size --filter network=spark'
dl
```

A lista completa de opções para o comando `docker ps` pode ser vista em
[https://docs.docker.com/engine/reference/commandline/ps/](https://docs.docker.com/engine/reference/commandline/ps/)

## Iniciando o Cluster

#### Verificando a Configuração de DNS e Network

No computador Host (macOS, por exemplo), faça:

```bash
# listando as Networks existentes
docker network ls
# inspecionando os contêineres
docker inspect --format='' spark-master | python -m json.tool | egrep "IPAddress\"|\"Gateway"
docker inspect --format='' spark-worker1 | python -m json.tool | egrep "IPAddress\"|\"Gateway"
docker inspect --format='' spark-worker2 | python -m json.tool | egrep "IPAddress\"|\"Gateway"
docker inspect --format='' spark-worker3 | python -m json.tool | egrep "IPAddress\"|\"Gateway"
```

Para testar podemos usar o `ping` dentro dos contêineres, pingando o Master e os slaves.

```bash
ping spark-master -c 3 &&  \
ping spark-worker1 -c 3 && \
ping spark-worker2 -c 3 && \
ping spark-worker3 -c 3
```


### Iniciando o Master

Na janela de Terminal do Contêiner Master, executa-se:

```bash
/usr/local/spark/sbin/start-master.sh
```

ou simplesmente `start-master.sh` pois o path `/usr/local/spark/sbin/` está no `$PATH`

O Spark Master usa o arquivo de configuração de log `org/apache/spark/log4j-defaults.properties` por padrão.


### Iniciando o Worker 3, 2 e 1 ...


Na janela de **cada um dos Workers** executa-se:

```bash
ping spark-master -c 3 # deve responder corretamente.
/usr/local/spark/sbin/start-slave.sh spark://spark-master:7077
```

### Iniciando o Driver via `spark-shell`

Como temos um ALIAS definido no Contêiner:

```bash
alias wff='spark-shell --packages "br.cefet-rj.eic:wff:0.5.0" --master spark://spark-master:7077'
```

Podemos abrir o terminal do **Contêiner Master** e executar

```bash
wff
```

Agora podemos invocar tasks pois o arquivo `spark-defaults.conf` configura,
entre outras coisas, o  `spark.master` apontando para `spark://spark-master:7077`

```bash
cat /usr/local/spark/conf/spark-defaults.conf
# conf/spark-defaults.conf

spark.master                     spark://spark-master:7077
spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.driver.memory              2g
```

## Monitorando o Master e os Workers

```bash
open http://localhost:8080
open http://localhost:8081
open http://localhost:8082
open http://localhost:8083
```

Isso abrirá 4 páginas no Browser.

## Build de Programas GO

Programas GO quando compilados são convertidos para código binário e ficam
muito eficientes. Além disso GO é uma linguagem de alto nível com suporte 
nativo a multithreading e TCP/IP.

O procedimento abaixo é feito durante o processo de build da Imagem Docker.
Esta descrição é para o caso de se fazer no computador Host.

Precisamos fazer o setup do `$GOPATH` para dentro do projeto.

```
export GOPATH=$HOME/docker/maven
# conferindo !
echo $GOPATH
```

### Instalando o GB - _builder_ baseado em projeto.

```bash
go get github.com/constabulary/gb/...
```

Mais detalhes em [https://getgb.io/examples/getting-started/](https://getgb.io/examples/getting-started/)

O `gb` fica instalado em `$GOPATH/bin`

### Fazendo o Build de um projeto


```bash
cd maven
$GOPATH/bin/gb build main/go/cmd/to_upper
```

Para testar

```bash
cat ../pom.xml | ../bin/to_upper
```

### Fonte do programa de teste

Neste caso o programa `to_upper.go` é este mostrado abaixo

```go
package main

import (
  "bufio"
  "fmt"
  "os"
  "strings"
)

func main() {
  scanner := bufio.NewScanner(os.Stdin)
  for scanner.Scan() {
    fmt.Fprintf(os.Stdout, strings.ToUpper(scanner.Text()+"\n"))
  }
}
```

### Fonte do fwd_cmd.go

O use de Bash para fazer Forward de Command para o wff-gateway não é apropriado
por questões de eficiencia. Por este motivo foi criado um programa GO para isso.


```go
package main

import (
  "bufio"
  "fmt"
  "log"
  "net"
  "os"
  "strings"
)

func main() {
  host := "127.0.0.1"
  port := "19092"
  argsWithoutProg := os.Args[1:]
  if len(argsWithoutProg) > 0 {
    host = argsWithoutProg[0]
  }
  if len(argsWithoutProg) > 1 {
    port = argsWithoutProg[1]
  }
  // connect to this socket
  conn, err := net.Dial("tcp", host+":"+port)
  if err != nil {
    log.Fatal("net.Dial(\"tcp\", \""+host+":"+port+"\") executed. ERROR: ", err)
  }
  reader := bufio.NewReader(os.Stdin)
  line, _ := reader.ReadString('\n')

  fmt.Fprintf(os.Stdout, strings.ToUpper(line+"\n"))
  scanner := bufio.NewScanner(os.Stdin)
  for scanner.Scan() {
    // send to socket
    fmt.Fprintf(conn, line+"\n")
    // listen for reply
    message, _ := bufio.NewReader(conn).ReadString('\n')
    fmt.Print("Message from server: " + message)
    line = scanner.Text()
  }
}
```


Código Scala de exemplo para download de arquivos em paralelo no Spark.

val data = List(
  "curl -O http://your-server/vra/2016_01.csv",
  "curl -O http://your-server/vra/2016_02.csv",
  "curl -O http://your-server/vra/2016_03.csv",
  "curl -O http://your-server/vra/2016_04.csv"
)
val dataRDD = sc.makeRDD(data)
val pipeRDD = dataRDD.pipe("f_cmd.sh")
val ret = pipeRDD.collect()
ret.foreach(println)


Shell `f_cmd.sh` invocada pelo Spark via método pipe()

```
#!/bin/sh
while read LINE; do
  echo ${LINE}
  echo ${LINE} | curl telnet://127.0.0.1:19092
done
```

Esta shell faz apenas o Forward do comando e pode ser substituida pelo
programa `fwd_cmd` mostrado acima, bastando substituir a linha do `pipe`
por

```scala
val pipeRDD = dataRDD.pipe("fwd_cmd")
```


### Fonte do Servidor WEB

Inicialmente obtemos as dependências do `gin` que é um framework WEB leve
e eficiente que ocupa menos de 10 MB

```
go get golang.org/x/net/context
go get golang.org/x/text/encoding
go get golang.org/x/crypto/ssh/terminal
go get golang.org/x/tools/go/buildutil
go get github.com/manucorporat/sse 
go get github.com/mattn/go-isatty
go get github.com/dustin/go-broadcast
go get github.com/manucorporat/stats
go get gopkg.in/go-playground/validator.v8
go get gopkg.in/yaml.v2 
go get gopkg.in/gin-gonic/gin.v1
```

O fonte de um exemplo está em [maven/src/main/go/cmd/web-server/web-server.go](maven/src/main/go/cmd/web-server/web-server.go)

Para fazer build de todos os programas GO use:

```bash
cd maven/src
$GOPATH/bin/gb build all
```

## Benchmark no Spark com Scala

Dado que desejamos medir o tempo de execução para rotinas específicas
podemos criar um método simples para encapsular a chamada e a medida de tempo
total para executar tais rotinas.

```scala
// Define a simple benchmark util function
def benchmark(name: String)(f: => Unit) {
  val startTime = System.nanoTime
  f
  val endTime = System.nanoTime
  println(s"Time taken in $name: " + (endTime - startTime).toDouble / 1000000000 + " seconds")
}
```

Podemos agora medir o tempo para somar 1 milhão de inteiros

```scala
benchmark("Spark million") {
  spark.range(1000L * 1000).selectExpr("sum(id)").show()
}
```

E também de um `count()` de um `join()`

```scala
benchmark("Spark Join") {
  spark.range(1000L * 1000).join(spark.range(1000L).toDF(), "id").count()
}
```

ou de um `crossJoin`

```scala
benchmark("Spark Cross Join") {
  spark.range(10000L).toDF("id").crossJoin(spark.range(10000L).toDF("xx")).count()
}
```

O Spark permite a analise de Planos de Execução. Veja um exemplo

```scala
spark.range(10000L).toDF("id").
    crossJoin(spark.range(10000L).toDF("xx")).
    selectExpr("count(*)").explain(true)
```

Outro exemplo:

```scala
spark.range(1000).filter("id > 100").selectExpr("sum(id)").explain()
```

Resultado:

```
== Physical Plan ==
*HashAggregate(keys=[], functions=[sum(id#404L)])
+- Exchange SinglePartition
   +- *HashAggregate(keys=[], functions=[partial_sum(id#404L)])
      +- *Filter (id#404L > 100)
         +- *Range (0, 1000, step=1, splits=Some(1))
```

> When an operator has a star around it (*), whole-stage code generation is enabled. In the following case, Range, Filter, and the two Aggregates are both running with whole-stage code generation. Exchange does not have whole-stage code generation because it is sending data across the network. This query plan has two "stages" (divided by Exchange). In the first stage, three operators (Range, Filter, Aggregate) are collapsed into a single function. In the second stage, there is only a single operator (Aggregate).


# Anexo I

## Dinâmica da Inicialização do Master 

A titulo de informação adicional: quando executamos `start-master.sh` a shell é expandida para:

```
/usr/local/spark/sbin/spark-daemon.sh start org.apache.spark.deploy.master.Master 1 --host spark-master --port 7077 --webui-port 8080
```

Esta shell **aguarda até o limite de 5 segundos** num loop de execução do código 
`ps -p "$newpid" -o comm=` testando o resultado contra a string `java` e em seguida
dorme por dois segundos. Ao final desse intervalo verifica se o processo morreu.
No caso do processo morrer ele mostra o final do arquivo de log como feedback ao usuário.

A shell `spark-daemon.sh` por sua vez invoca um comando tal como este abaixo:

```
 nohup -- nice -n 0 /usr/local/spark/bin/spark-class org.apache.spark.deploy.master.Master --host spark-master --port 7077 --webui-port 8080
```

`spark-class` do diretório `/usr/local/spark/bin` é uma shell que:

1. Verifica $SPARK_HOME 
2. Verifica $JAVA_HOME
3. Ajusta a variável `$LAUNCH_CLASSPATH`
4. Constroi o comando usando a classe `org.apache.spark.launcher.Main`

O fonte da classe `Main` pode ser vista em : [https://github.com/apache/spark/blob/master/launcher/src/main/java/org/apache/spark/launcher/Main.java](https://github.com/apache/spark/blob/master/launcher/src/main/java/org/apache/spark/launcher/Main.java) 
e faz parte de um pacote de funcionalidades [https://github.com/apache/spark/blob/master/launcher/src/main/java/org/apache/spark/launcher/package-info.java](https://github.com/apache/spark/blob/master/launcher/src/main/java/org/apache/spark/launcher/package-info.java)
que permite invocar o Spark programaticamente.

O comando `ps -ef --width 250` mostra o comando final executado com todos os parâmetros. Veja abaixo:

```
$JAVA_HOME/bin/java -cp /usr/local/spark/conf/:/usr/local/spark/jars/* -Xmx1g org.apache.spark.deploy.master.Master --host spark-master --port 7077 --webui-port 8080
```

## Workaround para Network

**Atenção:** este Work-around só é necessário se o procedimento com o comando `docker network` descrito anteriormente, não funcionar !

Caso a network spark criada não funcione, podemos sempre executar uma alternativa via o Work-around abaixo:

No conteiner spark-master faça:

```bash
cat /etc/hosts
```

Agora edite as ultimas linhas do arquivo `/etc/hosts` adicionando referências para os outros contêineres. Pode-se usar o comando `vi` para isso.

## Usando com Docker Swarm

Neste link [https://docs.docker.com/engine/swarm/swarm-tutorial/](https://docs.docker.com/engine/swarm/swarm-tutorial/) é possível consultar como implementar o Swarm numa rede de exemplo. 

