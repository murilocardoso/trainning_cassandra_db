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

No arquivo ```/local/docker-compose.yml``` há a configuração para subir três nós de cassandra (n1, n2, n3). Para iniciar um primeiro:
```
docker-compose up -d n1
```
Para verificar o status deste node:
```
docker-compose exec n1 nodetool status
```
```
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.18.0.2  70.89 KiB  256          100.0%            10b3f017-ede0-41e9-9d30-5018a9cc4f4c  rack1
```
Para este primeiro exemplo, as duas informações mais importantes são UN que significa que o nó está no ar e 256 que são os tokens deste nó.

De forma resumida, Cassandra armazena as informações associando-as a tokens (hash) em nodes onde cada node possui subdivisões virtuais chamadas vnode. Toda vez que um novo node é adicionado no cluster, seus vnodes assumem um range aleatório de tokens.

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

Também é possível ver as configurações presentes no arquivo de configuração do Cassandra:
```
docker-compose exec n1 more /etc/cassandra/cassandra.yaml
```
