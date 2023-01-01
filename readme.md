Resumo do treinamento [https://app.pluralsight.com/library/courses/cassandra-developers/table-of-contents](https://app.pluralsight.com/library/courses/cassandra-developers/table-of-contents)

# Um pouco de história

* Foi desenvolvido pelo facebook em 2008
* Foi usado na camada de armazenamento de uma feature que tinha como objetivo armazenar e permitir buscar por termos em conversa do facebook. 
A feature demandava uma necessidade de lidar com uma quantidade massiva de escrita e replicação geográfica para reduzir a latência nas consultas.
* Usou como base para desenvolvimento o BigTable da Google e o DynamoDB da Amazon. Combinou o modelo de distribuição do DynamoDB e o modelo de dados do 
BigTable. Inclusive há um co-fundador em comum entre Cassandra e DynamoDB, o Avinash Lakshman
* Em 2009 passou a fazer parte de um projeto incubado da Apache
* Em 2010 passou a ser um projeto top-level da Apache
* Em 2011 Netflix havia mudado de Oracle para Cassandra
* Em 2014 até a Apple passou a utilizar Cassandra, com 70000 nodes e dezenas de pentabytes de dados
* *...???...*

# Iniciando

Diferentemente dos banco de dados RDBMS tradcionais, Cassandra é um banco de dados distribuído, ou seja, novos *nodes* podem ser somados a um *cluster* de *nodes* já existentes. Os diferentes nós podem estar em diferentes datacenters. Quanto mais nós, mais possibilidades de replicação de dados e maior tolerância a falhas isoladas. No decorrer dos tópicos entrarei em mais detalhes em como isso funciona. Por hora, apenas uma introdução para entender os comandos abaixo.

No arquivo ```/local/docker-compose.yml``` há a configuração para subir localmente três nós de cassandra (n1, n2, n3). Para iniciar um primeiro nó:
```
docker-compose up -d n1
```
Para verificar o status deste nó:
```
docker-compose exec n1 nodetool status
```
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.19.0.2  85.35 KiB  256          100.0%            a2631349-a3f9-4928-a0d0-1ffe6b46e09d  rack1
```
Para este primeiro exemplo, as duas informações mais importantes são (i) *UN* que significa que o nó está no ar e (ii) *256* são os tokens deste nó.

De forma resumida, Cassandra armazena as informações associando-as a *tokens (hash)* em nodes onde cada node possui subdivisões virtuais chamadas *vnode*. Toda vez que um novo node é adicionado no cluster, seus *vnodes* assumem um range aleatório de tokens.

Para ver no detalhe as informações dos tokens:
```
docker-compose exec n1 nodetool ring
```
```
Address     Rack        Status State   Load            Owns                Token                                       
                                                                           9133212895607654196                         
172.18.0.2  rack1       Up     Normal  70.89 KiB       100.00%             -9218040571196821503                        
172.18.0.2  rack1       Up     Normal  70.89 KiB       100.00%             -9153557735823062505  
...
```

Também é possível ver as configurações gerais do Cassandra DB:
```
docker-compose exec n1 more /etc/cassandra/cassandra.yaml
```

Para subir o segundo nó:
```
docker-compose up -d n2
```
```
[+] Running 2/2
 ⠿ Container local-n1-1  Running                                                                                                                       
 ⠿ Container local-n2-1  Started 
 ```
Verificar o status dos nós existentes:
```
docker-compose exec n2 nodetool status
```
```
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.19.0.3  76.04 KiB  256          100.0%            ea431cc9-2b8b-4d8b-b42e-fdde3800083b  rack1*
UN  172.19.0.2  75.9 KiB   256          100.0%            a2631349-a3f9-4928-a0d0-1ffe6b46e09d  rack1
```

Para subir os demais nós é o mesmo procedimento.... 

# Alguns conceitos
* Snitch - Serve como um entendedor do ambiente e seus nós. Ele quem roteia as requisições e também é consultado quando há múltiplas cópias dos dados.

  * Existem 'n' estratégias:

    * SimpleSnitch - Mais usado para ambientes de desenvolvimento e single datacenter environments. 
    * GossipingPropertyFileSnitch - Usa um arquivo de configuração deployado com cada node para descrever a localização de cada node. Esta informação é "passada/fofocada" para cada node.

Observação: Os cloud providers possuem suas próprias estratégias de snitches, que mapeam a organização dos seus datacenters e racks. Exemplo EC2Snitch or EC2MultiRegionSnitch

Para ver mais informações a respeito destas configurações:
```
docker-compose exec n1 more /etc/cassandra/cassandra-rackdc.properties
```
```
//...
# These properties are used with GossipingPropertyFileSnitch and will
# indicate the rack and dc for this node
dc=dc1
rack=rack1

# Add a suffix to a datacenter name. Used by the Ec2Snitch and Ec2MultiRegionSni
tch
# to append a string to the EC2 region name.
#dc_suffix=

# Uncomment the following line to make this snitch prefer the internal ip when possible, as the Ec2MultiRegionSnitch 
does.
# prefer_local=true
//...
```
* Keyspace - Conceito semelhante ao table space/databse no modelo relacional. Contém uma ou mais tabelas. Na criação do Keyspace é possível determinar a estratégia de replicação e a quantidade de réplicas. Exemplos:
```
// no caso de um único datacenter
create keyspace pluralsight with replication = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };
``` 
```
// no caso de multiplos datacenters
create keyspace pluralsight with replication = { 'class' : 'NetworkTopologyStrategy', 'DC1' : 3, 'DC2' , 1 };
```

* Tabelas - Semelhante ao modelo relacional
* Partition - Determina onde o dado está armazenado em um cluster, pode ter uma ou mais linhas de uma tabela


# Estratégias de consistências

* Para escritas
O node em que o client está conectado  atua como um coordinator node, responsável por inserir o registro no node correto e replicar a informação para os demais nodes. É possível configurar por comando quantos nodes devem ter os dados replicados para que a operação seja considerada sucesso. Os possíveis valores são: 1, 2, 3, Quorum (maioria), ALL, ANY(basta receber o comando de escrita, pode acontecer de não conseguir replicar em nenhum) 

* Para leitura:
Semelhante a escrita, é possível configurar de quantos nodes a informação deve ser lida e comparada para retornar a mais atual. Existe um mecanismo chamado de read repair que é responsável por replicar para os demais nodes a informação mais atualizada. Este mecanismo pode ser configurado para ocorrer com base em um percentual das operações de leitura. Quando configurado como ALL ele acaba ocorrendo automaticamente em todas as leituras.Também é possível executar um comando de repair que irá replicar as informações dos nodes mais atualizados para os nodes menos atualizados.

![image](https://user-images.githubusercontent.com/26872442/210186959-17a3fb5e-a2e6-4d53-bca6-b2a05de60932.png)

Diferença entre replication factor e write e read consistency.

Write e read consistency diz respeito a quantos nodes devem ser escritos ou lidos para considerar que uma operação obteve sucesso. Vamos supor que replication factor seja 3 e o write e read consistency seja 1, ao realizar uma escrita e logo em seguida uma leitura, pode ser que não tenha dado tempo da replicar ocorrer ao node em que está sendo feita a consulta, por este motivo não se teria um consistencia forte, apenas uma consistência eventual.

Para os cenários de multi datacenter, é possível definir estratégias específicas como por exemplo LOCAL_ONE, LOCAL_QUORUM ou EACH_QUORUM.


