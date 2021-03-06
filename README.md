# data-engineer

## Instalação : R, Python e Spark

* R Versão 3.3.3
* Python Versão 2.7.13
* Spark Versão 2.1.0 com suporte a Hadoop 2.7 usando Scala 2.11.8

Após clonar veja os maiores arquivos usando:

```bash
find  . -printf '%s %p\n'| sort -nr | head -20
```

ou no macOS

```bash
for i in G M K; do du -ah | grep [0-9]$i | sort -nr -k 1; done | head -n 11
```

## Iniciando o Master

```bash
spark/sbin/start-master.sh
```

Você verá algo como mostrado abaixo indicando onde fica o arquivo de LOG

```
starting org.apache.spark.deploy.master.Master, logging to /home/spark/spark/logs/spark-spark-org.apache.spark.deploy.master.Master-1-my-host-master.out
```




## Iniciando os Workers

```bash
spark/sbin/start-slaves.sh 
```

Você verá algo como mostrado abaixo indicando onde fica o arquivo de LOG

```
my-worker-01.acme.com: starting org.apache.spark.deploy.worker.Worker, logging to /home/spark/spark/logs/spark-spark-org.apache.spark.deploy.worker.Worker-1-my-worker-01.out

. . .

```

Este arquivo de LOG especificado fica na máquina `my-worker-01` e é um dos Slaves.

## Iniciando o Spark no restart da máquina via systemd

O link [https://linuxconfig.org/how-to-automatically-execute-shell-script-at-startup-boot-on-systemd-linux](https://linuxconfig.org/how-to-automatically-execute-shell-script-at-startup-boot-on-systemd-linux) mostra como fazer uma configuração simples quando desejamos amenas carregar um script no _start-up_ do Sistema Operacional Linux. 

Antes de fazer o procedimento para o Spark vamos testar com um exemplo mais simples. Por exemplo, considere os seguintes arquivos:

```bash
sudo cat /usr/local/bin/disk-space-check.sh
#!/bin/bash

set -e 

date > /var/disk_space_report.txt
du -sh /tmp/ >> /var/disk_space_report.txt
du -sh /desenv >> /var/disk_space_report.txt
du -sh /data >> /var/disk_space_report.txt
for a in `ls /home`
do
  TEMP=`du -sh /home/$a`
  echo "$TEMP "  >> /var/disk_space_report.txt
done
```

```
sudo cat /etc/systemd/system/disk-space-check.service 
[Unit]
After=network.target

[Service]
ExecStart=/usr/local/bin/disk-space-check.sh

[Install]
WantedBy=default.target
```

Alteramos os flags dos arquivos:

```bash
chmod 744 /usr/local/bin/disk-space-check.sh
chmod 664 /etc/systemd/system/disk-space-check.service
```

Next, install systemd service unit and enable it so it will be executed at the boot time:

```bash
systemctl daemon-reload
systemctl enable disk-space-check.service
```
O sistema responde com:

> Created symlink from /etc/systemd/system/default.target.wants/disk-space-check.service to /etc/systemd/system/disk-space-check.service.

Para testar antes do reboot use:

```bash
systemctl start disk-space-check.service
cat /var/disk_space_report.txt 
```

Se o cat funcionar, fique tranquilo que após cada reboot o processo de geração de relatório de uso do disco rígido executará novamente.

Para chamar o Spark é um processo análogo.

That's all folks ! 


## Gerenciando o ciclo de vida via systemd

Se desejarmos gerenciar o ciclo de vida completo usando o _systemd_ podemos nos basear no link [https://www.ubuntudoc.com/how-to-create-new-service-with-systemd/](https://www.ubuntudoc.com/how-to-create-new-service-with-systemd/).


## Usando o Spark

O diretório `/tmp/spark-events` deve existir

```bash
mkdir /tmp/spark-events
```


Uma sessão do Spark em Scala pode ser criada assim:

```bash
spark-shell --master spark://meu-master.acme.com:7077 --packages "br.cefet-rj.eic:wff:0.5.0"  --num-executors 2
```

Esta chamada acima carrega também a dependência definida como mostrado abaixo.

```xml
<groupId>br.cefet-rj.eic</groupId>
<artifactId>wff</artifactId>
<version>0.5.0</version>
```

As dependências são resolvidas pelo **Apache Ivy**. Os artefatos de meta informação e os JARS ficam repectivamente em `$HOME/.ivy2/cache` e `$HOME/.ivy2/jars`.

Os JARs podem ficar também em `$HOME/.m2/repository` quando tiver o Maven instalado na máquina e o Ivy copia para a área citada acima.

Você pode verificar usando:

```
find $HOME/.ivy2/cache
find $HOME/.ivy2/jars
find $HOME/.m2/repository -name "wff*.jar"
```

Um arquivo pom.xml mínimo pode ser visto em [pom.xml](pom.xml)



### Parâmetros úteis para o spark-shell

Veja esta seção da documentação: [http://spark.apache.org/docs/2.1.0/configuration.html#dynamically-loading-spark-properties](http://spark.apache.org/docs/2.1.0/configuration.html#dynamically-loading-spark-properties)

`spark-submit`  lê as configurações no arquivo `conf/spark-defaults.conf` que consiste de chave/valor separados por brancos e cada linha é uma propriedade. Por exemplo, no meu caso:

```bash
cat ~/spark/conf/spark-defaults.conf
# substitua meu-master.acme.com pelo endereço do seu servidor
spark.master                     spark://meu-master.acme.com:7077
spark.eventLog.enabled           true
# spark.eventLog.dir             hdfs://namenode:8021/directory
spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.driver.memory              2g
# spark.executor.extraJavaOptions  -XX:+PrintGCDetails -Dkey=value -Dnumbers="one two three"
```

Linhas com `#` no início são comentários


## Executando Tarefas no Cluster

Invocamos o Spark Shell com as classes do Framework

```bash
alias wff='spark-shell --packages "br.cefet-rj.eic:wff:0.5.0"'
wff
```

Carregamos o programa e em seguida saimos 

```scala
:load BasicDataFrame.sc
:quit
```

É muito importante copiar os arquivos para o diretório correto em **todas as máquinas do Cluster**. 
Só assim os **Workers** do Spark podem acessar os arquivos. Caso ele não ache ocorrerá o erro
`FileNotFoundException: File file:/desenv/DATA/people.json does not exist`


O programa de exemplo aparece abaixo

```
cat BasicDataFrame.sc
```

```scala
import org.apache.spark.sql._
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
import org.apache.spark.sql.types._

case class Person(name: String, age: Long)

// This import is needed to use the $-notation
import spark.implicits._

def runBasicDataFrameExample(spark: SparkSession): Unit = {
  // $example on:create_df$
  val df = spark.read.json("/desenv/DATA/people.json")
  // cat   /desenv/DATA/people.json
  // {"name":"Michael"}
  // {"name":"Andy", "age":30}
  // {"name":"Justin", "age":19}
  // Displays the content of the DataFrame to stdout
  df.show()

  // import spark.implicits._
  // Print the schema in a tree format
  df.printSchema()

  // Select only the "name" column
  df.select("name").show()

  // Select everybody, but increment the age by 1
  df.select($"name", $"age" + 1).show()

  // Select people older than 21
  df.filter($"age" > 21).show()

  // Count people by age
  df.groupBy("age").count().show()

  // Register the DataFrame as a SQL temporary view
  df.createOrReplaceTempView("people")

  val sqlDF = spark.sql("SELECT * FROM people")
  sqlDF.show()

  // Register the DataFrame as a global temporary view
  df.createGlobalTempView("people")

  // Global temporary view is tied to a system preserved database `global_temp`
  spark.sql("SELECT * FROM global_temp.people").show()

  // Global temporary view is cross-session
  spark.newSession().sql("SELECT * FROM global_temp.people").show()
}
runBasicDataFrameExample(spark)
```

Um script Scala para ser carregado no spark-shell pode ser visto em [AllAboutDataFrame.sc](AllAboutDataFrame.sc). Ele é mais completo e inclui o exemplo acima. 


```bash
```


```bash
```


```bash
```




