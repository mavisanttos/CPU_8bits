# CPU 8 bits

&emsp; Este projeto consiste no desenvolvimento e simulação de uma Unidade Central de Processamento (CPU) de 8 bits, projetada no simulador Digital. A arquitetura segue o modelo de Von Neumann, onde a memória RAM armazena tanto as instruções quanto os dados e a Unidade de Controle (UC) gerencia o ciclo de instrução através de uma máquina de estados síncrona, alternando entre as fases de busca (fetch) e execução (execute). O objetivo é demonstrar como registradores (PC, IR, AC), uma Unidade Lógica e Aritmética (ALU) e uma unidade de controle interagem para executar algoritmos programados diretamente na memória.

## 1. ALU

&emsp; A ALU utilizada na CPU é responsável por realizar todas as operações matemáticas e lógicas do processador. No contexto, ela foi otimizada para trabalhar com um barramento unificado de saída e duas entradas, permitindo que diferentes resultados sejam selecionados através de uma instrução de controle (Op).

&emsp; A ALU é composta por subcircuitos, cada um especializado em uma operação:

- **Aritmética Básica:** Somador (soma_8bit) e Subtrator (subtracao_8bit) de 8 bits.
- **Matemática Avançada:** Multiplicador (multiplicacao_8bit) e Divisor (divisor_8bit).
- **Manipulação de Bits:** Deslocadores de bits para a esquerda (shift_left) e para a direita (shift_right).

&emsp; O Multiplexador com 8 entradas recebe os resultados de cada bloco e utiliza o sinal *Op* de 3 bits para decidir qual valor será enviado para a saída *S*.

&emsp; Para operações complexas como Multiplicação e Divisão, o MUX possui entradas específicas para os bits menos significativos (AC) e mais significativos (MQ/Resto).

<div align="center">
<img src="assets\alu_cpu.png" alt="Unidade Lógica e Aritmética (ALU)" width="500">
</div>

### 1.1. Tabela de Operações (OpCodes)

&emsp; Baseando-se na lógica do circuito, estas são as operações disponíveis:

| Seletor (Op) | Operação  | Descrição                                 |
|--------------|-----------|--------------------------------------------|
| 000 (0)      | SOMA      | Adição de A + B                            |
| 001 (1)      | SUB       | Subtração de A - B                         |
| 010 (2)      | MULT_AC   | Parte baixa (LSB) da Multiplicação         |
| 011 (3)      | MULT_MQ   | Parte alta (MSB) da Multiplicação          |
| 100 (4)      | DIV_MQ    | Quociente da Divisão                       |
| 101 (5)      | DIV_AC    | Resto da Divisão                           |
| 110 (6)      | S_LEFT    | Deslocamento de bits para esquerda         |
| 111 (7)      | S_RIGHT   | Deslocamento de bits para direita          |

### 1.2. Alterações na ALU

&emsp; Para que a ALU funcionasse dentro da arquitetura da CPU, foram implementadas as seguintes alterações:
- **Unificação de Saída:** Diferente da ALU isolada, aqui todos os resultados saem por um único pino de 8 bits (S), facilitando a conexão com o barramento de dados interno.
- **Adição da entrada A:** A lógica de Acumulador foi desenvolvida fora da ALU, por isso ela foi refatorada com duas entradas (A e B).

## 2. Instruction Register

&emsp; O Instruction Register (IR) é o componente responsável por armazenar a instrução que está sendo executada no momento. Ele atua como uma ponte entre a Memória RAM e a Unidade de Controle.

### 2.1. Componentes

**1. Flip-Flops D**

&emsp; O registrador é composto por 8 Flip-Flops tipo D, assim, é possível realizar a manipulação bit a bit. Uma vez que o sinal de controle (C_IR) é ativado, o valor da instrução fica "travado" nas saídas Q, independentemente do que aconteça no barramento de dados da RAM depois disso.

**2. Splitters**

&emsp; O IR utiliza Splitters (divisores de barramento) para separar a instrução em duas partes fundamentais:

- **O OpCode (Bits de Operação):** Os 3 bits mais significativos são isolados e enviados para a saída op. Eles dizem à ALU qual operação deve ser realizada.
- **O Barramento de Dados (8 bits):** A instrução completa é mantida na saída Q para caso o valor precise ser usado como operando em outras partes do circuito.

<div align="center">
<img src="assets\IR.png" alt="Instruction Register (IR)" width="500">
</div>

### 2.2. Lógica de Funcionamento

&emsp; O sistema opera de forma síncrona:
1. Enquanto o pino *C_IR* estiver em 0, as saídas *Q* e *op* permanecem inalteradas, ignorando qualquer mudança que ocorra na entrada *D*.
2. No momento em que *C_IR* recebe um pulso, o estado da entrada *D* é copiado para os Flip-Flops.
3. Esse valor capturado torna-se imediatamente disponível nas saídas e permanece lá até o próximo pulso de gravação.

## 3. Program Counter

&emsp; O Program Counter (PC), é o módulo responsável por gerenciar o endereço da próxima instrução a ser buscada. Ele funciona como um ponteiro sequencial: armazena um valor numérico e, a cada ciclo, prepara o próximo número da sequência (incremento).

### 3.1. Componentes

**1. Registrador de 8 bits**

&emsp; Uma célula de memória que guarda o endereço atual. Ele mantém o valor estável para que a memória saiba exatamente qual posição ler.

**2. Somador de 8 bits**

&emsp; Um subcircuito presente e desenvolvido para a ALU responsável por somar o valor atual do registrador com uma constante.

**3. Constante**

&emsp; Entrada fixa conectada ao somador. Ela garante que, a cada operação, o endereço seja avançado em exatamente 1 unidade.

**4. Realimentação**

&emsp; O sistema utiliza túneis (*S* e *Address_Out*) para criar um loop. O valor de saída do registrador passa pelo somador e o resultado (incrementado) volta para a entrada do registrador, aguardando o próximo comando.

<div align="center">
<img src="assets\PC.png" alt="Program Counter (PC)" width="500">
</div>

### 3.2. Lógica de Funcionamento

&emsp; O funcionamento do módulo segue um ciclo contínuo de três etapas:
1. O registrador sustenta o endereço atual na saída Ad_Out. Esse valor entra simultaneamente no somador.
2. O somador processa Ad_Out + 1 em tempo real. Este novo valor fica na porta da entrada do registrador, esperando para ser utilizado.
3. Quando o sinal de controle *C_PC* é ativado, o registrador abre sua entrada, captura o valor incrementado e o envia para a saída. O ciclo então se reinicia, preparando o próximo endereço.

## 4. Acululador

&emsp; O Acumulador (AC) tem a função de armazenar temporariamente o resultado de operações aritméticas e lógicas. Diferente de um registrador comum, o AC implementa uma lógica de realimentação seletiva, permitindo que ele mantenha seu valor atual ou capture um novo dado dependendo do sinal de controle.

### 4.1. Componentes

**1. Flip-Flops Tipo D**

&emsp; Oito unidades de memória que sustentam o estado atual do acumulador.

**2. Multiplexadores 2:1**

&emsp; Cada bit possui seu próprio multiplexador na entrada. Eles decidem a origem do dado:
- Canal 0: Recebe o valor que já está na saída do Flip-Flop (mantém o dado).
- Canal 1: Recebe o novo valor vindo da entrada externa *D_In* (carrega novo dado).

**3. Splitters de Barramento**

- Entrada: Decompõe o barramento de 8 bits em 8 linhas individuais para processamento.
- Saída: Reagrupa os 8 bits individuais processados em um único barramento de saída *D_Out*.

<div align="center">
<img src="assets\AC.png" alt="Acumulador (AC)" width="500">
</div>

### 4.2. Lógica de Funcionamento

&emsp; O AC opera com a seguinte lógica:
1. Se W_En = 0: O Multiplexador direciona a própria saída do Flip-Flop de volta para sua entrada. Mesmo que o Clock (*C_AC*) pulse, o valor não muda porque o componente está lendo ele mesmo.
2. Se W_En = 1: O Multiplexador abre o caminho para o barramento externo *D_In*. No próximo pulso de *C_AC*, o dado novo é gravado nas células de memória.
3. A saída *D_Out* está sempre ativa, fornecendo o valor armazenado para a ALU ou para o barramento de dados continuamente.

## 5. CPU

&emap; Na CPU desenvolvida temos a Unidade de Controle (UC) que utiliza uma arquitetura baseada em um Estado de Ciclo (Flip-Flop D) e um Decodificador de Instruções. Ela gerencia o fluxo de dados entre o Acumulador (AC), Contador de Programa (PC), Registro de Instrução (IR) e a RAM.

### 5.1. Fetch e Execute

&emsp; A lógica de controle é dividida em duas fases principais, alternadas pelo Flip-Flop D:
- **Fase de Busca (Fetch):** Identificada pelo sinal ¬Q (saída negada do Flip-Flop).
     - Nesta fase, o endereço do PC é enviado para a RAM.
    - A instrução lida da RAM é carregada no IR.
    - O PC é incrementado para apontar para a próxima instrução.
- **Fase de Execução (Execute):** Identificada pelo sinal Q.
    - A Unidade de Controle decodifica o opcode vindo do IR (através do túnel op).
    - Ativa os sinais específicos para realizar a operação (Soma, Subtração, etc.) no Acumulador.

### 5.2. Componentes

**1. Decoder**

&emsp; Este componente recebe 3 bits de seleção vindos do túnel op. Ele traduz o código binário da instrução (opcode) em linhas individuais de controle. Cada saída do decodificador representa uma instrução específica que a CPU pode executar.

**2. Lógica Combinacional**

&emsp; Utilização de uma porta OR e uma porta AND:
- **Porta OR:** Agrupa diferentes instruções que precisam ativar um sinal de Habilitar Escrita no AC.
- **Porta AND:** Combina o sinal de execução Q com a instrução decodificada. Isso garante que a operação só aconteça no momento exato do ciclo de execução, evitando instabilidades nos dados.

**3. Flip-Flop de Estado**

&emsp; O Flip-Flop D funciona como um divisor de frequência do clock:
- Ele alterna entre 0 e 1 a cada pulso de clock principal.
- Isso cria a alternância automática entre Busca (onde a UC ignora o opcode e foca em trazer o dado da RAM) e Execução (onde a UC foca em processar o opcode).

<div align="center">
<img src="assets\CPU.png" alt="CPU de 8 bits" width="500">
</div>

### 5.3. Lógica de Funcionamento

&emsp; Quando o sistema inicia:
- **Clock sobe:** O Flip-Flop está em ¬Q. A UC habilita a leitura da RAM no endereço do PC.
- **Clock desce/sobe:** O Flip-Flop muda para Q. O decodificador agora olha para o túnel op. Se o opcode for, por exemplo, 000 (Soma), a lógica combinacional ativa a ALU e o Acumulador para salvar o resultado.
- **Resultado:** O túnel S (saída da ALU) é conectado ao Acumulador e também exportado para o Out (pino de saída final).

### 5.4. Tabela de Conexões

| Origem (Saída)          | Destino (Entrada)            | Nome do Túnel | Função Primária                                                                 |
|--------------------------|------------------------------|----------------|----------------------------------------------------------------------------------|
| PC (Program Counter)     | RAM (Addr)                  | Ad_Out         | Envia o endereço da instrução atual para a memória.                             |
| RAM (Data Out)           | IR (Instruction Reg)        | D_RAM          | Transporta o código da instrução lida da memória para o IR.                     |
| RAM (Data Out)           | ALU (Entrada B)             | D_RAM          | Fornece operandos da memória diretamente para a Unidade Lógica e Aritmética.    |
| IR (Opcode)              | Decoder (Sel)               | op             | Envia os 3 bits de operação para serem decodificados na UC.                     |
| ALU (Resultado)          | AC (Data In)                | S              | Conduz o resultado de uma operação aritmética para o Acumulador.                |
| AC (Accumulator)         | ALU (Entrada A)             | D_AC           | Alimenta a ALU com o valor atualmente armazenado no Acumulador.                 |
| AC (Accumulator)         | RAM (Data In)               | D_AC           | Caminho para salvar o valor do Acumulador de volta na memória (Store).          |
| Clock In                 | Todos (PC, IR, AC, FF)      | CLK            | Sincroniza a transição de estados e o carregamento dos registradores.           |
| Flip-Flop D (Q)          | Lógica UC (AND)             | Q              | Sinaliza que a CPU está no Ciclo de Execução.                                   |
| Flip-Flop D (~Q)         | PC, IR, RAM                 | ¬Q             | Sinaliza que a CPU está no Ciclo de Busca (Fetch).                              |
| Lógica UC (Portas)       | Habilitação de Escrita      | UC             | Sinal mestre que autoriza registradores a salvarem novos dados.                 |
| ALU (Resultado)          | Output Pino S               | S              | Exporta o resultado final para visualização fora da CPU.                        |

### 5.5. Simulação

&emsp; Aqui é possível observar a operação da CPU: [Vídeo de Demonstração](https://youtu.be/MeV3rQZhMJs)

&emsp; A implementação desta CPU demonstra a viabilidade de construir sistemas complexos a partir de blocos lógicos fundamentais. Através da simulação apresentada no vídeo acima, é possível validar que a Unidade de Controle coordena o fluxo de dados pelo barramento, garantindo que cada instrução seja decodificada e executada no tempo correto do clock.
