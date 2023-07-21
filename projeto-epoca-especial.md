Projeto de Época Especial
================ 

Este documento descreve o projeto de época especial da cadeira de Sistemas Distribuídos 2022/2023.

O projeto é individual e consiste em 3 requisitos, que são descritos nas secções seguintes.

A submissão é feita através do fenix até às 23:59 do dia 27/07/2023.

Na submissão, além do código do projeto, deve ser incluído um breve relatório (ficheiro de texto plano, com extensão .txt) que resuma o que foi concretizado em cada requisito. Concretamente, no relatório deve constar, para cada requisito (R1, R2, R3):

- uma auto-avaliação de qual a cotação que a solução deve receber para esse requisito;
- uma breve enumeração de quais os aspetos que foram concretizados, quais ficaram por concretizar e outras limitações da solução.

Haverá uma discussão final, a realizar em dia ser agendado posteriormente.




0 Ponto de partida
------------

O projeto consistirá numa extensão à solução da primeira parte do projeto da época normal de SD, cujo enunciado pode ser obtido [aqui](https://github.com/tecnico-distsys/DistLedger).
O resto do enunciado de época especial pressupõe que o estudante leu o enunciado mencionado acima.

Para obter a solução base, o estudante deve enviar email aos docentes de SD e estes enviarão o código de uma solução.


1 Requisito R1: Operação shareWithOthers (7 valores)
------------------------

Projeto deve ser estendido com uma operação adicional, `shareWithOthers`.
Esta operação recebe quarto argumentos: 

- O qualificador do servidor que deve ser contactado ('A', 'B', etc.). Este argumento deve ser ignorado e só será usado no requisito R2.

- O nome do utilizador que está a solicitar a operação.

- Um valor percentual (enviado como um inteiro), entre 1 e 100.

Quando invocada, a operação retira ao saldo da conta do utilizador chamador uma fatia correspondente à percentagem indicada em argumento. 
Essa fatia deve então ser transferiada de forma equitativa por todas restantes contas ativas no sistema.

Para implementar este requisito, o proto original do projeto precisa ser estendido para incluir a nova operação.
Adicionalmente, a operação deve ser invocável através da consola do cliente *utilizador*.


2 Requisito R2: Replicação com coerência fraca (7 valores)
------------------------


O serviço deve passar a ser fornecido por dois servidores, *A* e *B*, 
cada um mantendo uma réplica do estado. Cada servidor deve conhecer previamente o endereço do outro servidor (ou seja, não é preciso implementar qualquer serviço de nomes para a descoberta dinâmica de endereços).

O protocolo de replicação não precisa assegurar qualquer garantia de coerência. 
Ou seja:

- Um cliente submete uma operação a um dos servidores (escolhido aquando da invocação no cliente).

- Caso a operação seja de leitura, o servidor responde imediatamente com base no seu estado local.

- Caso a operação seja de escrita, o servidor adiciona uma entrada a descrever a operação ao seu *update log* (ou seja, à sua *ledger* local), executa localmente a operação e responde ao cliente.

- As operações nos logs de cada réplica são propagadas apenas quando solicitada pelo cliente administrador (através da operação 
*gossip*, descrita no enunciado de época normal).


3 Requisito R3: Assinatura digitais de mensagens (6 valores)
------------------------

As mensagens de pedido e resposta devem passar a levar uma assinatura de 
digital de chave pública, que deve ser devidamente validada pelo recetor da mensagem.
As chaves públicas dos clientes e servidores devem ser previamente 
conhecidas por estes interlocutores (ou seja, não se pretende implementar a distribuição de chaves dinâmica).

Para este requisito, devem ser seguidas instruções descritas no [guião laboratorial sobre segurança](https://tecnico-distsys.github.io/07-security/index.html).



**Bom trabalho!**
 
