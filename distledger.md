Distributed Ledger
================ 

Este documento descreve o projeto da cadeira de Sistemas Distribuídos 2022/2023.

1 Introdução
------------

O objetivo do projeto de Sistemas Distribuídos (SD) é desenvolver o sistema **DistLedger**, um serviço que implementa um *ledger* distribuído, sobre o qual são suportadas trocas de uma moeda digital. O serviço é fornecido por um ou mais servidores, através de chamadas a procedimentos remotos.

O serviço pode ser acedido por dois tipos de clientes: i) os *utilizadores*, que podem ter conta no sistema e trocar moeda entre si; ii) os *administradores* que mantêm o serviço em funcionamento.

Cada utilizador pode criar uma conta, com saldo inicial nulo. Por simplificação, cada utilizador pode ter, no máximo, uma conta a cada instante.
Cada utilizador também pode apagar a sua conta, desde que o saldo da conta seja nulo nesse momento.

Adicionalmente, cada utilizador pode transferir moedas da sua conta para a conta de outro utilizador. 
Para que uma transferência seja executada, (i) a conta origem precisa existir e ter saldo 
superior ou igual ao montante a transferir e (ii) a conta destino precisa existir; caso contrário, a transferência é cancelada.

Entre os utilizadores, há um utilizador especial, chamado *broker*.
Ao contrário dos restantes utilizadores, a conta do *broker* existe sempre 
(ou seja, não é criada nem apagada pelo utilizador respetivo) e o seu saldo inicial é 1000 moedas.
Ou seja, quando o sistema **DistLedger** se inicia, existe uma única conta, com saldo 1000, cujo dono é 
o *broker*.

O utilizador *broker* corresponde a uma entidade que corre um serviço externo (não implementado no **DistLedger**) 
que permite a utilizadores comprarem moeda digital, em troca de euros. 
Sempre que uma compra de moeda digital é feita com sucesso (fora do **DistLedger**), o *broker* solicita 
ao sistema **DistLedger**
que transfira o número de moedas digitais da sua conta para a conta do utilizador em causa. 
Note-se que este último passo (a transferência de moeda digital do *broker* para o utilizador em causa) 
é o único que é observado pelo **DistLedger**.

A situação inversa também pode ocorrer, em que um utilizador transfere moedas digitais da sua conta para a do *broker*,
com o objetivo de receber o valor correspondente em euros (mais uma vez, a entrega do montante em euros acontece externamente ao **DistLedger**).

O sistema será concretizado através de um conjunto de serviços gRPC implementados na plataforma Java.

O projeto está estruturado em três fases, descritas de seguida.

2 Estado dos servidores
------------------------

O servidor (ou servidores, conforme a fase) mantém, pelo menos, o estado seguinte: 

* A lista de operações de escrita aceites, também chamada *ledger*. Inclui operações de 3 tipos: criar conta, remover conta, transferir moeda entre contas.
Inicialmente está vazia e vai crescendo à medida que lhe são acrescentadas novas operações.

* Um mapa de contas com informação sobre as contas ativas neste momento e o respetivo saldo.
Esta estrutura descreve o estado que resulta da execução ordenada de todas as operações atualmente na *ledger* e que, na 3ª parte, já estão estáveis (*stable updates*, segundo a terminologia do *gossip architecture*, o modelo de replicação que vamos usar nessa parte do projeto).
Sempre que uma nova operação é adicionada à *ledger* e estabiliza (novamente, isto só é relevante na 3ª parte), o estado do mapa de contas deve ser atualizado para refletir essa operação.


3 Arquitetura do sistema
------------------------

O sistema usa uma arquitetura cliente-servidor. O serviço é fornecido por um ou mais servidores que podem ser
contactados por processos cliente, através de chamadas a procedimentos remotos. 

Em cada fase do projeto será explorada uma variante desta arquitetura básica, tal como se descreve abaixo.


3.1 Fase 1
-------------------

Nesta fase o serviço é prestado por um único servidor, que aceita pedidos num endereço/porto fixo que é conhecido de antemão por todos os clientes.

3.2 Fase 2
-------------------

Na segunda fase, o serviço é fornecido por dois servidores: um primário e um secundário. Uma operação que altere
o estado do sistema (que passaremos a designar como 
*operação de escrita*) só pode ser realizada no primário, que
propaga a mesma para o secundário antes de responder ao cliente. 
Operações que não alterem o estado do sistema (que passaremos a designar como 
*operações de leitura*), podem ser invocadas e executadas em qualquer um dos servidores.

Em caso de indisponibilidade do primário, o sistema deve operar sem primário (embora só para operações de leitura) até que este recupere (com o estado que tinha quando ficou indisponível). 
Não se pretende nem serão valorizadas soluções que implementem a deteção da falha do primário e respetiva substituição pelo secundário (como novo primário). 

Consequentemente, só para operações de leitura é que a solução primário-secundário pretendida será capaz de continuar a responder aos clientes enquanto um servidor (seja o primário, seja o secundário) estiver indisponível. 
Há, portanto, 3 situações de erro que devem ser consideradas no caso de operações de escrita: 1) Durante a indisponibilidade do servidor primário, uma invocação de escrita feita ao primário deverá sempre retornar erro; 2) Uma invocação de escrita feita ao servidor secundário deverá retornar sempre erro, mesmo que este esteja disponível (já que esse só trata de leituras); 3) Finalmente, uma operação de escrita invocada no primário enquanto o secundário está indisponível também deve retornar erro, já que o primário não consegue propagar a operação ao secundário.

Para além disso, nesta fase nem os clientes nem os servidores sabem, quando são lançados, quais os endereços dos (outros) servidores.
Para permitir que o endereço dos servidores seja descoberto dinamicamente, a solução deverá recorrer a um servidor de nomes **(a ser desenvolvido nas aulas teórico-práticas)**.


3.3 Fase 3
-------------------

Na última fase, o serviço é fornecido por dois servidores que partilham estado usando um modelo 
de ordenação causal de operações de escrita. Como solução para ordenação causal, deve ser adotada a 
*gossip architecture*, descrita no livro principal da disciplina.

Nesta solução, um servidor, ao receber um pedido de operação de escrita e após confirmar que as respetivas dependências 
causais são satisfeitas localmente (nesse servidor), pode responder ao cliente imediatamente após
aplicar a operação localmente (confirmando ao cliente que a operação foi executada com sucesso) -- ou seja, sem 
ter de propagar a operação ao outro servidor antes de responder ao cliente. 
Consequentemente, é esperado que esta solução seja mais escalável que a anterior (fase 2) quando a grande maioria 
das operações recebidas por um servidor tiver as suas dependências causais satisfeitas nesse servidor no momento do pedido. 

Para o desenho e implementação da solução, cada grupo deve ter em conta as seguintes considerações:

* O grupo poderá alterar as estruturas de dados mantidas pelo servidor, 
assim como as mensagens definidas nos *proto*.

* Por simplificação, a operação de apagar conta *não deve ser suportada nesta fase do projeto*. 
A razão para esta simplificação prende-se com o facto desta operação exigir garantias mais fortes que a ordenação 
causal.

* A *ledger* já implementada no servidor deve ser usada como o *update log* previsto na *gossip architecture*. 
Note-se, no entanto, que será necessário adicionar um atributo a cada operação na *ledger* que distinga se a operação está estável ou 
instável (segundo a *gossip architecture*).

* Ao contrário do que a *gossip architecture* prevê, as operações inseridas na *ledger*/*update log* nunca devem 
ser eliminadas. Ou seja, mesmo que uma operação já tenha chegado às *ledgers* de todos os servidores, ela deve 
persistir nas *ledgers*. Logo, a *timestamp table* definida na *gossip architecture* deixa de ser necessária no projeto. Quanto à *executed operation table*, fica ao critério de cada grupo decidir se é necessária.

* Na *gossip architecture*, as operações de escrita são propagadas em diferido (em "background") entre os servidores. 
No caso deste projeto, essa propagação deve ocorrer apenas quando solicitada pelo cliente administrador (através da operação 
*gossip*, descrita na secção 4.2).


O grupo poderá também desenvolver mecanismos extra que permitam acrescentar uma terceira réplica já com o sistema em
funcionamento. Reservamos 2 valores adicionais nesta fase para os grupos que conseguirem desenvolver corretamente um
mecanismo deste tipo (por outras palavras, os grupos podem ter uma nota superior a 20 valores na fase 3, o que poderá
compensar uma nota mais baixa noutra fase).




4 Interfaces do serviço
------------------------

Cada servidor exporta múltiplas interfaces. Cada interface está pensada para expor operações a cada tipo de cliente  
(utilizadores e administradores). Para além dessas, os servidores exportam uma terceira interface pensada para ser invocada por outros
servidores (no caso em que os servidores estão replicados, e necessitam de comunicar entre si).

4.1 Interface do utilizador
-------------------

O utilizador pode invocar as seguintes funções:

- `createAccount` -- cria uma conta associada a este utilizador
- `deleteAccount` -- apaga a conta associada ao utilizador
- `balance` -- devolve o saldo atual da conta que o utilizador tem ativa neste momento
- `transferTo` -- transfere uma quantia para outro utilizador

4.2 Interface do administrador
-------------------

- `activate` -- coloca o servidor em modo **ATIVO** (este é o comportamento por omissão), em que responde a todos os
  pedidos
- `deactivate` -- coloca o servidor em modo **INATIVO**. Neste modo o servidor responde com o erro "UNAVAILABLE" a todos os pedidos dos utilizadores
- `getLedgerState` -- apresenta o conteúdo da *ledger*
- `gossip` -- força uma réplica a fazer uma propagação diferida para a(s) outra(s) réplica(s) (só para a fase 3)


4.3 Interface entre servidores (só fases 2 e 3)
-------------------

- `propagateState` -- um servidor envia o seu estado a outra réplica.

5 Servidor de nomes (só fases 2 e 3)
------------------------

O servidor de nomes permite aos servidores registarem o seu endereço para ser conhecido por outros que estejam presentes
no sistema.

Um servidor, quando se regista, indica o nome do serviço (neste caso *DistLedger*), o seu endereço e um qualificador, que
pode assumir os valores 'A', 'B', etc. Na 2ª fase, o servidor primário é apenas o 'A'.

Este servidor está à escuta no porto **5001** e assume-se que nunca falha.

Os clientes podem obter o endereço dos servidores, fornecendo o nome do
serviço e o qualificador.

Nas fases 2 e 3, cada um dos servidores também pode usar o servidor de nomes para ficar a saber o endereço do outro servidor.

6 Processos
------------------------


6.1 Servidores
--------------

O servidor (ou servidores nas fases 2 e 3) devem ser lançados a partir da pasta `Server`, recebendo como argumentos o porto e o seu qualificador ('A', 'B', etc.). Na fase 1, o qualificador passado é ignorado.

Por exemplo, um servidor primário pode ser lançado da seguinte forma a partir da pasta `Server` (**$** representa a *shell* do sistema operativo):

`$ mvn exec:java -Dexec.args="2001 A"`

Um servidor secundário pode ser lançado da seguinte forma:

`$ mvn exec:java -Dexec.args="2002 B"`


6.2 Servidor de nomes (só fases 2 e 3)
-------------

O servidor de nomes deve ser lançado sem argumentos e ficará à escuta no porto `5001`, podendo ser lançado a partir da pasta `NamingServer` da seguinte forma:

`$ mvn exec:java`


6.3 Clientes
-------------

Ambos os tipos de processo cliente (utilizador e administrador) recebem comandos a partir da consola.
Todos os processos cliente deverão mostrar o símbolo *>* sempre que se encontrarem à espera que um comando seja
introduzido.

Para todos os comandos, caso não ocorra nenhum erro, os processos cliente devem imprimir "OK" seguido da mensagem de resposta, tal como gerada pelo método toString() da classe gerada pelo compilador `protoc`, conforme ilustrado nos exemplos abaixo. Nota: atenção que, no caso do comando 'balance' o método toString() não imprime o saldo caso este seja nulo; esse comportamento é o previsto.

No caso em que um comando origina algum erro do lado do servidor, esse erro deve ser transmitido ao cliente usando os mecanismos 
do rRPC para tratamento de erros (no caso do Java, encapsulados em exceções). Nessas situações, quando o cliente recebe uma exceção
após uma invocação remota, 
este deve simplesmente imprimir uma mensagem que descreva o erro correspondente.

Na fase 1, os programas de ambos os tipos de clientes recebem como argumentos o nome da máquina e porto onde o servidor do DistLedger pode ser encontrado. Por exemplo:

`$ mvn exec:java -Dexec.args="localhost 2001"`

A partir da fase 2, os programas cliente deixam de ter quaisquer argumentos.



### 6.3.1 Cliente *utilizador*

O cliente utilizador deve ser lançado a partir da pasta `User`. 

Para cada operação na interface utilizador do *DistLedger*, existe um comando que permite invocar essa operação.
Todos esses comandos recebem, pelo menos dois argumentos iniciais: 

- Como primeiro argumento, o qualificador do servidor que deve ser contactado ('A', 'B', etc.); na fase 1, este argumento é ignorado.
- Como segundo argumento, o nome do utilizador que está a solicitar a operação.

Adicionalmente, a operação transferTo recebe também o utilizador de destino e o valor a transferir.

Adicionalmente, existe também um comando `exit` para terminar o cliente.

Exemplo de uma interação com o cliente utilizador (que interage só com o servidor 'A'):

```
> createAccount A Alice
OK

> createAccount A Bob
OK

> transferTo A broker Alice 60
OK

> transferTo A Alice Bob 50
OK

> balance A Alice
OK
10

> balance A Bob
OK
50

> closeAccount A Alice
OK

> exit
```


### 6.3.2 Cliente *administrador*

O cliente administrador deve ser lançado a partir da pasta `Admin`. 

Todos os comandos do administrador recebem como único argumento o qualificador do servidor alvo ('A', 'B, etc.), que na fase 1 deve ser ignorado.

Exemplo:

```
> deactivate A
OK

> getLedgerState A
OK
ledgerState {
  ledger {
    type: OP_CREATE_ACCOUNT
    userId: "Alice"
  }
  ledger {
    type: OP_CREATE_ACCOUNT
    userId: "Bob"
  }
  ledger {
    type: OP_TRANSFER_TO
    userId: "broker"
    destUserId: "Alice"
    amount: 60
  }
  ledger {
    type: OP_TRANSFER_TO
    userId: "Alice"
    destUserId: "Bob"
    amount: 60
  }
  ledger {
    type: OP_DELETE_ACCOUNT
    userId: "Alice"
  }
}

> activate A
OK


> exit
```

## 6.4 Opção de *debug*

Todos os processos devem poder ser lançados com uma opção "-debug". Se esta opção for seleccionada, o processo deve
imprimir para o "stderr" mensagens que descrevam as ações que executa. O formato destas mensagens é livre, mas deve
ajudar a depurar o código. Deve também ser pensado para ajudar a perceber o fluxo das execuções durante a discussão
final.


7 Tecnologia
------------

Todos os componentes do projeto têm de ser implementados na linguagem de
programação [Java](https://docs.oracle.com/javase/specs/).

A ferramenta de construção a usar, obrigatoriamente, é o [Maven](https://maven.apache.org/).

### Invocações remotas

A invocação remota de serviços deve ser suportada por serviços [gRPC](https://grpc.io/).

Os serviços implementados devem obedecer aos *protocol buffers* fornecidos no código base disponível no repositório github do projeto.


### Persistência

Não se exige nem será valorizado o armazenamento persistente do estado dos servidores.
Caso um servidor falhe, assume-se que este retoma a sua execução com o estado que tinha aquando do momento da sua falha. 
Isto pode ser simulado usando as operações *activate* e *deactivate* a partir do cliente administrador.


### Validações

Os argumentos das operações devem ser validados obrigatoriamente e de forma estrita pelo servidor.

Os clientes podem optar por também validar, de modo a evitar pedidos desnecessários para o servidor, mas podem optar por
uma versão mais simples da validação.

# 8 Modelo de Faltas

Deve assumir-se um sistema síncrono, em que as falhas são silenciosas. Não ocorrem falhas bizantinas. 

Se durante a execução surgirem faltas, ou seja, acontecimentos inesperados, o programa deve apanhar a exceção, imprimir
informação sucinta e pode parar de executar.

Se for um servidor, o programa deve responder ao cliente com um código de erro adequado.

Se for um dos clientes, pode decidir parar com o erro recebido ou fazer novas tentativas de pedido.

Fica fora do âmbito do projeto resolver os problemas relacionados com a segurança (e.g., autenticação dos 
utilizadores ou administradores, confidencialidade ou integridade das mensagens).
Também deve ser assumido que os utilizadores são bem-comportados. Em particular, que um utilizador 
não usa dois processos cliente em simultâneo para, a partir destes, executar operações concorrentes envolvendo 
a sua conta.



# 9 Resumo
------------

Em resumo, é necessário implementar:

- Na fase 1 (1ª entrega):
  - servidor único;
  - cliente utilizador;
  - cliente administrador.

- Na fase 2 (2ª entrega):
  - servidores primário e secundário (extensão do servidor da fase 1);
  - ambos os clientes, agora a consultar o servidor de nomes.

- Na fase 3 (3ª entrega):
  - múltiplos servidores da *gossip architecture* (extensão dos servidores da fase 2);
  - cliente utilizador estendido de acordo com a *gossip architecture*;
  - cliente administrador.



10 Avaliação
------------

10.1 Fotos
---------

Cada membro da equipa tem que atualizar o Fénix com uma foto, com qualidade, tirada nos últimos 2 anos, para facilitar a
identificação e comunicação.

10.2 Identificador de grupo
--------------------------

O identificador do grupo tem o formato `GXX`, onde `G` representa o campus e `XX` representa o número do grupo de SD
atribuído pelo Fénix. Por exemplo, o grupo A22 corresponde ao grupo 22 sediado no campus Alameda; já o grupo T07
corresponde ao grupo 7 sediado no Taguspark.

O grupo deve identificar-se no documento `README.md` na pasta raiz do projeto.

Em todos os ficheiros de configuração `pom.xml` e de código-fonte, devem substituir `GXX` pelo identificador de grupo.

Esta alteração é importante para a gestão de dependências, para garantir que os programas de cada grupo utilizam sempre
os módulos desenvolvidos pelo próprio grupo.

10.3 Colaboração
---------------

O [Git](https://git-scm.com/doc) é um sistema de controlo de versões do código fonte que é uma grande ajuda para o
trabalho em equipa.

Toda a partilha de código para trabalho deve ser feita através do [GitHub](https://github.com).

O repositório de cada grupo está disponível em: https://github.com/tecnico-distsys/GXX-DistLedger/ (substituir `GXX` pelo
identificador de grupo).

A atualização do repositório deve ser feita com regularidade, correspondendo à distribuição de trabalho entre os membros
da equipa e às várias etapas de desenvolvimento.

Cada elemento do grupo deve atualizar o repositório do seu grupo à medida que vai concluindo as várias tarefas que lhe
foram atribuídas.

10.4 Entregas
------------

As entregas do projeto serão feitas também através do repositório GitHub.

A cada parte do projeto a entregar estará associada uma [*tag*](https://git-scm.com/book/en/v2/Git-Basics-Tagging).

As *tags* associadas a cada entrega são `SD_P1`, `SD_P2` e `SD_P3`, respetivamente.
Cada grupo tem que marcar o código que representa cada entrega a realizar com uma *tag* específica antes
da hora limite de entrega.

10.5 Valorização
---------------


As datas limites de entrega estão definidas no site dos laboratórios: (https://tecnico-distsys.github.io)

### Qualidade do código

A avaliação da qualidade engloba os seguintes aspetos:

- Configuração correta (POMs);

- Código legível (incluindo comentários relevantes);

- [Tratamento de exceções adequado](https://tecnico-distsys.github.io/04-rpc-error-test/index.html);

- [Sincronização correta](https://tecnico-distsys.github.io/02-tools-sockets/java-synch/index.html);

- Separação das classes geradas pelo protoc/gRPC das classes de domínio mantidas no servidor.

10.6 Instalação e demonstração
-----------------------------

As instruções de instalação e configuração de todo o sistema, de modo a que este possa ser colocado em funcionamento,
devem ser colocadas no documento `README.md`.

Este documento tem de estar localizado na raiz do projeto e tem que ser escrito em formato *[
MarkDown](https://guides.github.com/features/mastering-markdown/)*.

Cada grupo deve preparar também um mini relatório, que não deve exceder as **500** palavras, explicando a sua solução para a fase 3,
e que não deve repetir informação que já esteja no enunciado. Este documento deve estar na raiz do projeto e deve ter o nome `REPORT.md`

10.7 Discussão
-------------

As notas das várias partes são indicativas e sujeitas a confirmação na discussão final, na qual todo o trabalho
desenvolvido durante o semestre será tido em conta.

As notas a atribuir serão individuais, por isso é importante que a divisão de tarefas ao longo do trabalho seja
equilibrada pelos membros do grupo.

Todas as discussões e revisões de nota do trabalho devem contar com a participação obrigatória de todos os membros do
grupo.

10.8 Atualizações
----------------

Para acompanhar as novidades sobre o projeto, consultar regularmente
a [página Web dos laboratórios](https://tecnico-distsys.github.io).

Caso venham a surgir correções ou clarificações neste documento, podem ser consultadas no histórico (_History_).

**Bom trabalho!**
 