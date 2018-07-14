### Especificacao de Protocolo
*Protocolo de comunicação Serial Posijet 1*


# Protocolo I de Comunicação


Este Controlador possui duas portas seriais de comunicação, a serial 1 e a serial 2. A diferença entre elas é que a serial 1 pode ser configurada para se comunicar com a impressora através dos parâmetros ”Reversão de mensagem via serial” e “Seleção de mensagem via serial“ do menu de “Configuração da impressora“, se um desses parâmetros estiver ligado, o protocolo descrito aqui só poderá ser usado com a serial 2 a 2400Bauds, sendo que a serial 1 poderá trabalhar a 2400, 4800 e 9600 Bauds.


Parâmetros de Comunicação

Toda a comunicação realizada com o Movimentador Linear deve obedecer as seguintes condições descritas abaixo.

Tipo de Comunicação: Serial RS232C
Taxa de Transferência: Serial 1 = 9600/bauds, Serial 2 = 2400/bauds
Tamanho dos Dados: 8 bits
Paridade: Nenhuma
Delimitadores: 1 start bit e 1 stop bits


Pacotes de Comunicação

	O movimentador pode ser encarado como um dispositivo escravo, pois ele apenas pode enviar uma informação caso haja a solicitação da mesma.	
	Todas as comunicações são iniciadas pelo dispositivo mestre, sendo que a comunicação é efetuada através de blocos, que também chamamos de pacote.

	Após o dispositivo mestre enviar um pacote ele deve aguardar o dispositivo escravo enviar um pacote de resposta, este pacote pode conter o dado solicitado, pode confirmar que os dados foram aceito ou indicar que houve um erro.
	Todos os pacotes obedecem a estrutura abaixo.

```
+--------------------------------------------------------------------------------------------------------------+
|                                             Pacote de Solicitacao                                            |
+--------------------------------------------------------------------------------------------------------------+
| ESC |       |            |      CMD      |         DadoL        |        DadoH        | ESC | ETX | CheckSum |
|     | START |   DIRECAO  |               |                      |                     |     |     |          |
|     |  BYTE |      E     |               |                      |                     |     |     |          |
|     |       |    CANAL   |               |                      |                     |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
| 1Bh |   02  |  De 0 a 3F |               |                      |                     | 1Bh | 03h |    ???   |
|     |       |            |   Posicao do  |  Nao tem significado | Nao tem significado |     |     |          |
|     |       |            | primeiro byte | mas deve ser enviado |    mas deve ser     |     |     |          |
|     |       |            |   solicitado  |    qualquer coisa    |                     |     |     |          |
|     |       |            |               |                      |       enviado       |     |     |          |
|     |       |            |               |                      |    qualquer coisa   |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
|                                               Pacote de Retorno                                              |
+--------------------------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |               |                                            | 1Bh | 03h |          |
|     |  NACK |            |  Mesmo valor  |                Parametro de                |     |     |    ???   |
|     |  15h  |            |    enviado    |                  2 bytes                   |     |     |          |
|     |       |            |               |                                            |     |     |          |
|     |       |            |               |                 solicitado                 |     |     |          |
+-----+-------+------------+---------------+--------------------------------------------+-----+-----+----------+
|                                                                                                              |
|                                          Pacote de Retorno com Erro                                          |
+--------------------------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |   Indefinido  |     Byte de Erro     |       StatusL       | 1Bh | 03h |    ???   |
|     |  Nack |            |               |                      |                     |     |     |          |
|     |  15h  |            |               |                      |                     |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
							
```
#### Start byte: 
Este byte indica o inicio do pacote de comunicação e pode ter vários valores conforme a tabela abaixo.

|Valor	|Label	|Descrição|
| :---: | :---: | :--- |
|02	|`STX`	|Define como um pacote a ser enviado ao CONTROLADOR.|
|06	|`ACK`	|Define como um pacote de retorno do CONTROLADOR, indicando que a interpretação foi bem sucedida.|
|21	|`NACK`	|Define como um pacote de retorno do CONTROLADOR, e que ouve erro na interpretação do pacote, ou na recepção.|
	

#### Direção:
Adotando este byte como um numero binário ddccccccb onde dd indica as quatro formas de atuação do dado e cccccc pode representar até 64 canais.

|Binário	|Hexadecimal	|Função|
| :---: | :---: | :--- |
|00CCCCCC|	00h a 3Fh	|Define que esta solicitando uma informação e o pacote de retorno deverá conter a informação solicitada.|
|01CCCCCC|	40h a 7Fh	|Define que a informação enviada é uma mascara para resetar os bits a partir da posição fornecida.
(Operação AND entre a posição da memória e a mascara negada)|
|10CCCCCC|	80h a Bfh	|Define que a informação enviada é uma mascará para setar os bits a partir da posição fornecida.
Operação OR entre a posição de memória e a mascara. |
|11CCCCCC|	C0h a FFh	|Define que a informação enviada será carregada a partir da posição fornecida.
(Operação OR entre a memória e a mascara)|

O numero de canais é utilizado em sistema onde temos vários controladores conectado na mesma porta serial, cada controlador possui canal distinto, e este só interpretará e responderá ao pacotes em que o numero do canal coincidir ou for zero.
Um numero de canal zero enviado pelo mestre será atendido por qualquer controlador independente do seu numero de canal. O canal zero só pode ser usado pelo mestre quando só tiver um único controlador conectado na porta serial, caso contrario todos responderiam simultaneamente.
Um controlador com canal zero não poderá estar conectado em uma porta serial com outros controladores, pois atendera a todos pacotes recebido, e sua resposta entrará em conflito com os outro. 
	O numero do canal do controlador pode ser modificado como será descrito adiante, e se for zero responderá a todos os pacotes de comunicação, desta forma podemos se comunicar com um controlador qualquer sem termos necessidade de conhecer seu numero.
	Utiliza-se mascará para definir o conjunto de bits que se deseja zerar ou setar no controlador, pois o parâmetro alvo é um conjunto de bits e não bytes.
 
#### CMD
Indica a posição da memória do microcontrolador dos dados solicitados ou enviados. O CMD sendo de um byte e o dados de 2 bytes, se pode acessar 512 posições de memória, mas apenas algumas delas tem significado para o usuário, e algumas posição nem podem ser modificada, pois são de uso do sistema.

> INFO:
Como o tamanho do pacote é fixo, mesmo quando se solicita deve ser enviado os bytes dos dados com um valor qualquer.
 

### CheckSum:
É a soma dos bytes do pacote complementados de 2, não deverá ser somado os dois bytes ESC explicito no pacote, e os bytes ESC duplicado somente será considerado um deles.

> Observações
	O byte ESC (1Bh) é um byte de controle e ele só deve ser utilizado nas posições já especificadas, sempre que houver necessidade dele estar em qualquer outra posição deverá ser duplicado, até mesmo na posição do cheqsum.

### TIPOS DE PACOTES

Temos quatro tipos de pacotes e a cada pacote enviado temos duas possibilidades de retorno: Umas se o pacote enviado foi interpretado corretamente e outra se ouve erro na recepção do pacote.

```
+--------------------------------------------------------------------------------------------------------------+
|                                             Pacote de Solicitacao                                            |
+--------------------------------------------------------------------------------------------------------------+
| ESC |       |            |      CMD      |         DadoL        |        DadoH        | ESC | ETX | CheckSum |
|     | START |   DIRECAO  |               |                      |                     |     |     |          |
|     |  BYTE |      E     |               |                      |                     |     |     |          |
|     |       |    CANAL   |               |                      |                     |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
| 1Bh |   02  |  De 0 a 3F |               |                      |                     | 1Bh | 03h |    ???   |
|     |       |            |   Posicao do  |  Nao tem significado | Nao tem significado |     |     |          |
|     |       |            | primeiro byte | mas deve ser enviado |    mas deve ser     |     |     |          |
|     |       |            |   solicitado  |    qualquer coisa    |                     |     |     |          |
|     |       |            |               |                      |       enviado       |     |     |          |
|     |       |            |               |                      |    qualquer coisa   |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
|                                               Pacote de Retorno                                              |
+--------------------------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |               |                      |                     | 1Bh | 03h |          |
|     |  NACK |            |  Mesmo valor  |     Parametro de     |                     |     |     |    ???   |
|     |  15h  |            |    enviado    |       2 bytes        |                     |     |     |          |
|     |       |            |               |                      |                     |     |     |          |
|     |       |            |               |      solicitado      |                     |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
|                                                                                                              |
|                                          Pacote de Retorno com Erro                                          |
+--------------------------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |   Indefinido  |     Byte de Erro     |       StatusL       | 1Bh | 03h |    ???   |
|     |  Nack |            |               |                      |                     |     |     |          |
|     |  15h  |            |               |                      |                     |     |     |          |
+-----+-------+------------+---------------+----------------------+---------------------+-----+-----+----------+
```

```
+-----------------------------------------------------------------------------------+
|                                  Pacote de Envio                                  |
+-----------------------------------------------------------------------------------+
| ESC |       |            |     CMD     |  DadoL  |  DadoH  | ESC | ETX | CheckSum |
|     | START |   DIRECAO  |             |         |         |     |     |          |
|     |  BYTE |      E     |             |         |         |     |     |          |
|     |       |    CANAL   |             |         |         |     |     |          |
+-----+-------+------------+-------------+---------+---------+-----+-----+----------+
| 1Bh |   02  |            |             |                   | 1Bh | 03h |    ???   |
|     |       |  De C0 a   |  Posicao do |     Parametros    |     |     |          |
|     |       |            |   primeiro  |   2 bytes a ser   |     |     |          |
|     |       |     FFh    |  byte a ser |      enviado      |     |     |          |
|     |       |            |   enviado   |                   |     |     |          |
|     |       |            |             |                   |     |     |          |
+-----+-------+------------+-------------+-------------------+-----+-----+----------+
|                                 Pacote de Retorno                                 |
+-----------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |             | StatusL | StatusH | 1Bh | 03h |          |
|     |  NACK |            | Mesmo valor |         |         |     |     |    ???   |
|     |  15h  |            |   enviado   |         |         |     |     |          |
+-----+-------+------------+-------------+---------+---------+-----+-----+----------+
|                                                                                   |
|                             Pacote de Retorno com Erro                            |
+-----------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |  Indefinido |         | StatusL | 1Bh | 03h |    ???   |
|     |  Nack |            |             | Byte de |         |     |     |          |
|     |  15h  |            |             |   Erro  |         |     |     |          |
+-----+-------+------------+-------------+---------+---------+-----+-----+----------+
```

```
+-----------------------------------------------------------------------------------------+
|                                  Pacote de Resetar Bits                                 |
+-----------------------------------------------------------------------------------------+
| ESC |       |            |        CMD       |   DadoL  |  DadoH  | ESC | ETX | CheckSum |
|     | START |   DIRECAO  |                  |          |         |     |     |          |
|     |  BYTE |      E     |                  |          |         |     |     |          |
|     |       |    CANAL   |                  |          |         |     |     |          |
+-----+-------+------------+------------------+----------+---------+-----+-----+----------+
| 1Bh |   02  |            |                  |                    | 1Bh | 03h |    ???   |
|     |       |   De 40 a  |                  |     Mascara de     |     |     |          |
|     |       |     7Fh    |                  |    16 bits a ser   |     |     |          |
|     |       |            |    Posicao do    | enviada que indica |     |     |          |
|     |       |            | primeiro byte a  | quais bits a serem |     |     |          |
|     |       |            |                  |     ressetados     |     |     |          |
|     |       |            |   ser mascarado  |                    |     |     |          |
+-----+-------+------------+------------------+--------------------+-----+-----+----------+
|                                    Pacote de Retorno                                    |
+-----------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |                  |  StatusL | StatusH | 1Bh | 03h |          |
|     |  NACK |            |    Mesmo valor   |          |         |     |     |    ???   |
|     |  15h  |            |      enviado     |          |         |     |     |          |
+-----+-------+------------+------------------+----------+---------+-----+-----+----------+
|                                                                                         |
|                                Pacote de Retorno com Erro                               |
+-----------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |    Indefinido    |          | StatusL | 1Bh | 03h |    ???   |
|     |  Nack |            |                  |  Byte de |         |     |     |          |
|     |  15h  |            |                  |   Erro   |         |     |     |          |
+-----+-------+------------+------------------+----------+---------+-----+-----+----------+
```

```
+-----------------------------------------------------------------------------------------------+
|                                      Pacote de Setar Bits                                     |
+-----------------------------------------------------------------------------------------------+
| ESC |       |            |        CMD       |    DadoL    |    DadoH   | ESC | ETX | CheckSum |
|     | START |   DIRECAO  |                  |             |            |     |     |          |
|     |  BYTE |      E     |                  |             |            |     |     |          |
|     |       |    CANAL   |                  |             |            |     |     |          |
+-----+-------+------------+------------------+-------------+------------+-----+-----+----------+
|     |       |            |    Posicao do    |   Mascara de 16 bits a   | 1Bh | 03h |    ???   |
|     |       |   De 80 a  | primeiro byte a  |  ser enviada que indica  |     |     |          |
| 1Bh |   02  |     BFh    |                  | quais bits serao setados |     |     |          |
|     |       |            |   ser mascarado  |                          |     |     |          |
|     |       |            |                  |                          |     |     |          |
|     |       |            |                  |                          |     |     |          |
+-----+-------+------------+------------------+--------------------------+-----+-----+----------+
|                                       Pacote de Retorno                                       |
+-----------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |                  |   StatusL   |   StatusH  | 1Bh | 03h |          |
|     |  NACK |            |    Mesmo valor   |             |            |     |     |    ???   |
|     |  15h  |            |      enviado     |             |            |     |     |          |
+-----+-------+------------+------------------+-------------+------------+-----+-----+----------+
|                                                                                               |
|                                   Pacote de Retorno com Erro                                  |
+-----------------------------------------------------------------------------------------------+
| 1Bh |       | De 0 a 3Fh |    Indefinido    |             |   StatusL  | 1Bh | 03h |    ???   |
|     |  Nack |            |                  |   Byte de   |            |     |     |          |
|     |  15h  |            |                  |     Erro    |            |     |     |          |
+-----+-------+------------+------------------+-------------+------------+-----+-----+----------+
```


### Códigos de Erro de interpretação do pacote.

É armazenado no primeiro byte do dado do pacote sempre que:
O byte de start for Nack (15h)

|Cod.	|Erro|
| :---: | :--- |
|01	Start byte invalido (stx)|
|02	Estrutura do pacote de comunicação invalido|
|03	Estrutura do pacote de comunicação invalido|
|04	Estrutura do pacote de comunicação invalido|
|05	Estrutura do pacote de comunicação invalido|
|06	Estrutura do pacote de comunicação invalido|
|07	Estrutura do pacote de comunicação invalido|
|08	Estrutura do pacote de comunicação invalido|
|09	Estrutura do pacote de comunicação invalido|
|10	Não usado|
|11	End byte invalido (etx)|
|12	Timer in|
|13	Não usado|
|14	Framming|
|15	Over run|
|16	Buffer de recepção cheio|
|17	CheqSum|
|18	Buffer auxiliar ocupado|
|19	Seqüência de byte enviada muito grande|

#### StatusL e statusH

Representa uma word onde cada bits tem um significado próprio sendo o statusL temos os bits 0 a 7 e no statusH temos os bits 8 a 15, e está descrito na tabela do status (CMD = 49 =31h).

#### Parâmetros definidos como 2 bytes, ou seja, uma word.

A maioria desses parâmetros necessita de um *fator multiplicador/divisor e uma constante a ser somada*. Abaixo temos uma tabela indicando o numero do CMD e os valores de conversão mais usual.

Sendo que unidade de comprimento será milímetro e a unidade de tempo será segundos.



|`CMD` [Dec]      | `CMD` Hex	|Descrição	| Unidade	|Fatores de Conversão mais usual	|Valor a ser somado|
| :---: | :---: | :--- | :---: |  :---: |   :---: | 
|80	|50	|Posição inicial|	Mm	|4,92125	|512|
|81	|51	|Posição final|	Mm	|4,92125	|0|
|82	|52	|Aceleração de avanço|	Mm / S ²	|5,16031|	0|
|83	|53	|Aceleração de retorno|	Mm / S ²	|5,16031|	0|
|84	|54	|Velocidade de avanço|	Mm / S	|5,03937|	0|
|85	|55	|Velocidade de retorno|	Mm / S	|5,03937	|0|
|86	|56	|Número de mensagem no avanço|	-----------	|1	|0|
|86*|	56*	|Número de mensagem no retorno|	-----------	|1	|0|
|87	|57	|Posição da primeira impressão no avanço|	Mm	|4,92125	|512|
|88	|58	|Posição da primeira impressão no retorno|	Mm	|4,92125	|512|
|89	|59	|Posição da ultima impressão no avanço|	Mm	|4,92125|	512|
|90	|5ª	|Posição da ultima impressão no retorno|	|Mm	|4,92125	|512|
|91	|5B	|Largura do sinal de impressão|	S	|1024	|0|
|92	|5C	|Tempo para start automático|	S	|1024	|0|
|93	|5D	|Tempo para start externo|	S	|1024	|0|
|94	|5E	|Antecipação da saída de start|	Mm	|4,92125	|0
|95	|5F	|Não utilizado|	-----------|	---------------|	-----------|
|96	|60	|Descrito na tabela abaixo|	-----------	|---------------	|-----------|
|97	|61	|Janela de proteção para o giro|	Pulso|	1	|0|
|98	|62	|Numero de pulso por giro do motor|	Pulso/giro|	1	|0|
|99	|63	|Valor da posição da referencia|	Pulso|	1	|0|
|100	|64	|Aceleração de referência|	Pulso/S²|	1	|0|
|101	|65	|Velocidade de referência|	Pulso/S|	1	|0|
|102	|66	|Descrito na tabela abaixo (Controle serial)	|------------	|---------------	|------------|

Caso se deseje enviar um parâmetro ao controlador teremos que converter o valor ser enviado utilizando a tabela acima conforme a formula abaixo:

Valor a ser enviado = (Parâmetro) * (Fator de conversão) + (Valor a ser somado)

E na solicitação dos parâmetros devemos utilizar a formula abaixo

Parâmetro = [(Valor recebido) – (Valor a ser somado)] / (Fator de Conversão)

Obs.: O parâmetro deverá estar nas unidades mencionada na tabela acima e o dispositivo mecânico devera ser padrão.

Caso o equipamento não seja padrão temos a tabela abaixo para o calculo do fatore de “multiplicação / divisão”, sendo o valor a ser “somado / subtraído” o mesmo.

|Unidade desejada de trabalho	|Unidade de trabalho do controlador	
|Formula para o calculo do fator	|Fatores mais usuais
| :--- | :--- | :---: | :---: |
|Milímetro	|Pulso	|2 * NPM / NDP / PC	|4,92125|
|Milímetro / Segundo|	Pulso / T	|1,024 * 2 * NPM / NDP / PC  	|5,03937|
|Milímetro / Segundo ²|	Pulso / T ²	|1,024 ² * 2 * NPM / NDP / PC	|5,16031|
|Segundo	|1/1024 T|	1024	|1024|

|Variável	|Descrição	|Valor para um equipamento padrão|
| :---: | :---: | :--- |
|NPM	|Numero de pulso do Motor|	200|
|NDP	|Numero de dente da Polia Motora	|16|
|PC	|Passo da correia dentada	|5,08 mm|
	
*Parâmetro 96 =60h é definido bit a bits sendo descrito na tabela abaixo a função de cada bit.*

| BIT	|DESCRICAO	|SETADO	|RESETADO|
| :---: | :--- | :---: | :---: | 
|0	|Start automático no avanço|	Ligado	|Desligado|
|1	|Start automático no retorno|Ligado	|Desligado|
|2	|Saída de start no avanço	|Ligado	|Desligado|
|3	|Saída de start no retorno	|Ligado	|Desligado|
|4	|Entrada do start externo	|Ligado	|Desligado|
|5	|Lógica do start externo	|Aberto	|Fechado|
|6	|Entrada de start entre eixo	|Ligado|	Desligado|
|7	|Referencia pelo start esterno	|Ligado	|Desligado|
|8	|Lógica do sinal de impressão	|Aberto	|Fechado|
|9	|Lógica do sinal de reversão	|Aberto|	Fechado|
|10	|Seleção de mensagem via serial 
(Modo continuo/passo a passo)|	Ligado	|Desligado|
|11	|Reversão de mensagem via serial (Sem uso)	|Ligado|	Desligado|
|12	|Giro com função de proteção	|Ligado	|Desligado|
|13	|Giro com função de correção	|Ligado	|Desligado|
|14	|Redução da corrente em repouso	|Ligado	|Desligado|
|15	|Modo continuo/passo a passo (Sem uso)	|Paspas	|Contin|


## Algumas Variáveis importantes

#### Status ( CMD = 49 =31h )
O status é uma variável em que cada bit tem um significado próprio e só pode ser lida, caso seja feito uma escrita nesta variável o resultados do movimento serão imprevisíveis.
O status é enviado no pacote de resposta sempre que:
-	O byte de start for Nack (15h) teremos StatusL no primeiro byte
-	O byte de start for Ack (06h) e o pacote transmitido for de envio, reset ou set, teremos statusL no primeiro byte e statusH no segundo byte.
-	Quando for solicitado através do CMD=49 teremos statusL no primeiro byte e statusH no segundo byte. 

|Bit	|Descrição|
|:---:|:---|
|0|	Se 1 está referenciado
|1|	Se 1 Indica que a ultima posição a ser executada foi alcançada.
|2|	Se 1 está Referenciando
|3|	Se 1 a direção do movimento é positiva
|4|	Se 1 esta em movimento acelerado
|5|	Se 1 esta em movimento desaceleração
|6|	Reservado
|7|	Se 1 indica um evento de erro, e deve ser consultado na mascara de erro. CMD=69 =45h
|8 a 15|	Estão reservados para aplicações especiais.

*Obs.: Se os bits 4 e 5 estiverem em 1 teremos um movimento uniforme (ou seja, sem aceleração e sem desaceleração).*

#### Mascara de Erro (CMD =69 =45h)

|Bytes	|Bit	|Descrição|
|:---:|:---:|:---|
|L0	|0	|Sinal de Start esterno|
|L1	|1	|Sinal de start diversos|
|L2	|2	|Sensor de giro por falta|
|L3	|3	|Sensor de giro por excesso|
|L4	|4	|Sinal de Impressão|
|L5	|5	|Erro de comunicação ocorrido na serial 1|
|L6	|6	|Implementação Futura|
|L7	|7	|Implementação Futura|
|H0	|8	|Cheqsum da eprom2 incorreto|
|H1	|9	|O equipamento foi resetado (rebutado)|
|H2 a H7	|10 a 15|	Estão reservados para aplicações especiais.|

#### Controle Serial (CMD = 105 =69h)

|Bit	|Descrição|
|:---:|:---|
|0	Start|
|1	Stop|
|2	Pausa|
|3	Modo manual|
|4	Teste de impressão|
|5	Reservado para aplicações especiais|
|6	Salva os parâmetro na EEprom|
|7	Reservado|
|8 a 15	Estão reservados para aplicações especiais.|


