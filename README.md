# STR-Projeto-3

Simulação de cruzamento de trilhos de trem usando o FreeRTOS

## Vídeo Demonstração

```c

```

## Objetivo

O projeto envolve criar um sistema que simule o funcionamento de um cruzamento entre uma linha ferroviária e uma via de carros. Neste cenário, os trens e os carros podem se aproximar de qualquer das direções. Para evitar colisões, sempre que um trem se aproxima do cruzamento, uma cancela deve ser fechada e os veículos devem parar. A sincronização do acesso ao cruzamento é crucial e será gerenciada usando semáforos. Você deverá aplicar os conceitos de sincronização e exclusão mútua.


## Implementação

- Modelagem dos trens e carros: cada trem e cada carro deve ser representado por uma thread (ou processo). Eles podem vir de qualquer uma das quatro direções. O importante é que em nenhum momento exista mais de um trem no cruzamento e no mesmo sentido no mesmo instante.
- Cruzamento: Deve ser tratado como um recurso compartilhado, onde o acesso é controlado por semáforos. Note que podemos ter dois trens cruzando se eles estiverem em sentidos opostos. Os carros só podem cruzar se a cancela não estiver abaixada e a passagem do trem sempre tem prioridade.
- Semáforos: utilizar semáforos para gerenciar o acesso ao cruzamento. Deve-se garantir que os semáforos sejam usados para evitar condições de corrida e deadlocks, permitindo uma passagem segura e eficiente.
- Interface com o usuário: uma interface simples que mostre o estado atual dos trens e dos carros (aproximando-se, passando, e passou) e a situação do cruzamento. Pode ser um simples output de texto ou uma interface gráfica básica.

## Instruções de uso

Entrar na pasta 
- FreeRTOSv10.0.1\FreeRTOS\Demo\WIN32-MSVC;
- Substituir o arquivo main.c pela main.c presente no repositório;
-  Abrir o arquivo de projeto: Win32.vcxproj


## Bibliotecas Utilizadas

```c
/* Standard includes. */
#include <stdio.h>
#include <stdlib.h>
//#include <conio.h>

/* FreeRTOS kernel includes. */
#include "FreeRTOS.h"
#include "task.h"

/* Include para uso de semaforos*/
#include "semphr.h"
```

## Código

- Definimos o número de carros e trens do sistema

```c
#define NUM_CARROS 50
#define NUM_TRENS 3
```

- Declaração de variáveis globais compartilhadas entre as tasks. O gate com valor zero indica que está aberto, enquanto as variáveis de guarda em zero indicam que estão liberadas.
  
```c
int gate = 0;			
int guarda_trem = 0;	
int guarda_carro = 0;
```
- Declaração dos semáforos utilizados para garantir a sincronização e exclusão mútua entre as threads/processos. O "xSemaphoreCruzamento" é o semáforo para acesso ao cruzamento. Os outros semáforos são para bloquear a passagem de mais de um trem ou carros por sentido.

```c
SemaphoreHandle_t xSemaphoreCruzamento = NULL;
SemaphoreHandle_t xSemaphoreNorte = NULL;		
SemaphoreHandle_t xSemaphoreSul = NULL;			
SemaphoreHandle_t xSemaphoreLeste = NULL;		
SemaphoreHandle_t xSemaphoreOeste = NULL;		
SemaphoreHandle_t xSemaphoreGate = NULL;
```

Foi criado duas funções de tarefas (threads/processos) responsáveis por simular o comportamento dos carros e trens. 
- carroTask

  
```c
void carroTask(void *pvParameters) {
	
	int g;

	for (;;) {
		srand(xTaskGetTickCount());
		int direcao = rand() % 2;
		
		if (xSemaphoreTake(xSemaphoreCruzamento, portMAX_DELAY) == pdTRUE && gate == 0) {
		
			if (guarda_carro == 0) {
				printf("Sinal - Verde\nCancela - Aberta\n");
				guarda_carro++;
			}

			if (direcao == 0) {
				if (xSemaphoreTake(xSemaphoreLeste, portMAX_DELAY) == pdTRUE) {
					printf("Carro passando pelo cruzamento - Vindo da direcao Leste\n");
					vTaskDelay(pdMS_TO_TICKS(500));
					printf("Carro passou\n");
					xSemaphoreGive(xSemaphoreLeste);
				}

			}
			else if (direcao == 1) {
				if (xSemaphoreTake(xSemaphoreOeste, portMAX_DELAY) == pdTRUE) {
					printf("Carro passando pelo cruzamento - Vindo da direcao Oeste\n");
					vTaskDelay(pdMS_TO_TICKS(500));
					printf("Carro passou\n");
					xSemaphoreGive(xSemaphoreOeste);
				}
			}
			guarda_trem = 0;
			xSemaphoreGive(xSemaphoreCruzamento);
		}
		vTaskDelay(pdMS_TO_TICKS(50));
	}
}
```
Essa função começa definindo uma direção aleatória para cada carro, representando se o carro está vindo do leste (direção 0) ou do oeste (direção 1). Em seguida, a função verifica se é possível acessar o cruzamento. Para isso, ela utiliza um semáforo "xSemaphoreCruzamento" que controla o acesso ao cruzamento. Se o semáforo estiver disponível (ou seja, nenhum outro carro ou trem está passando pelo cruzamento) e a cancela estiver aberta ("gate == 0"), o carro pode prosseguir. Antes de atravessar o cruzamento, a função verifica se a cancela está liberada para o carro ("guarda_carro == 0"). Se estiver, exibe uma mensagem indicando que o sinal está verde e a cancela está aberta. Dependendo da direção do carro, a função tenta adquirir o semáforo correspondente ("xSemaphoreLeste" ou "xSemaphoreOeste").

Se o semáforo for adquirido com sucesso, o carro atravessa o cruzamento, exibe uma mensagem indicando a direção e aguarda um curto período de tempo, simulando o tempo necessário para atravessar o cruzamento. Em seguida, libera o semáforo adquirido e atualiza a variável de guarda para indicar que o carro já passou. Por fim, o código aguarda um curto período de tempo antes de continuar para a próxima iteração do loop, simulando um intervalo de tempo entre as ações dos carros.

- tremTask
```c
  void tremTask(void *pvParameters) {

	for (;;) {

		int direcao = rand() % 2 + 2;

		if (xSemaphoreTake(xSemaphoreCruzamento, portMAX_DELAY) == pdTRUE ) {

			gate++;
			
			if (direcao == 2) {
				printf("Trem se aproximando - Vindo da direcao Norte\n");
			}
			else {
				printf("Trem se aproximando - Vindo da direcao Sul\n");
			}
			
			if (guarda_trem == 0) {
				printf("Sinal - Vermelho\nCancela - Fechada\n");
				guarda_trem++;
				guarda_carro = 0;
			}
			
			vTaskDelay(pdMS_TO_TICKS(rand()%1500 + 750));

			if (direcao == 2) {
				if (xSemaphoreTake(xSemaphoreNorte, portMAX_DELAY) == pdTRUE) {

					printf("Trem passando pelo cruzamento - Vindo da direcao Norte\n");
					vTaskDelay(pdMS_TO_TICKS(1000));
					printf("Trem passou\n");
					
					gate--;
					
					xSemaphoreGive(xSemaphoreNorte);
				}
			}
			else if (direcao == 3) {
				if (xSemaphoreTake(xSemaphoreSul, portMAX_DELAY) == pdTRUE) {

					printf("Trem passando pelo cruzamento - Vindo da direcao Sul\n");
					vTaskDelay(pdMS_TO_TICKS(1000));
					printf("Trem passou\n");
					gate--;
					
					xSemaphoreGive(xSemaphoreSul);
				}
			}
			xSemaphoreGive(xSemaphoreCruzamento);

		}
		
		vTaskDelay(pdMS_TO_TICKS(rand() % 10001 + 5000));
		
	}
}
```

A função "tremTask" é responsável por simular o comportamento dos trens que se aproximam do cruzamento. Ela começa definindo uma direção aleatória para cada trem, representando se o trem está vindo do norte (direção 2) ou do sul (direção 3). Em seguida, a função verifica se é possível acessar o cruzamento. Para isso, ela utiliza um semáforo "xSemaphoreCruzamento" que controla o acesso ao cruzamento. Se o semáforo estiver disponível (ou seja, nenhum outro carro ou trem está passando pelo cruzamento), o trem pode prosseguir.

Se o cruzamento estiver disponível para acesso, o código incrementa a variável "gate", que representa a cancela. Em seguida, exibe uma mensagem indicando a direção do trem e verifica se a cancela está fechada para os trens ("guarda_trem == 0"). Se a cancela estiver fechada, exibe uma mensagem indicando que o sinal está vermelho e a cancela está fechada. Também atualiza a variável "guarda_carro" para garantir que carros não tentem atravessar o cruzamento enquanto um trem passa. Antes de atravessar o cruzamento, o trem espera por um período de tempo aleatório, simulando o tempo necessário para a aproximação do trem até o cruzamento.

O código tenta adquirir o semáforo correspondente à direção do trem ("xSemaphoreNorte" ou "xSemaphoreSul"). Se obtiver sucesso, o trem atravessa o cruzamento, exibe uma mensagem indicando a direção e espera um curto período de tempo para simular a passagem pelo cruzamento. Em seguida, decrementa a variável "gate" para indicar que o trem passou e libera o semáforo adquirido.

Após atravessar o cruzamento, o código libera o semáforo "xSemaphoreCruzamento" para permitir que outros trens ou carros possam acessá-lo. Em seguida, espera por um período de tempo aleatório antes de continuar para a próxima iteração do loop, simulando um intervalo de tempo entre as ações dos trens.

## Main()
139
338
- Função Principal (main)
```c
int main(void)
{
	prvInitialiseHeap();
	vTraceEnable(TRC_START);
	
	xSemaphoreCruzamento = xSemaphoreCreateCounting(2, 2); 
	xSemaphoreNorte = xSemaphoreCreateMutex();
	xSemaphoreSul = xSemaphoreCreateMutex();
	xSemaphoreLeste= xSemaphoreCreateMutex();
	xSemaphoreOeste = xSemaphoreCreateMutex();
	xSemaphoreGate = xSemaphoreCreateMutex();

	if (xSemaphoreCruzamento != NULL) {
		for (int i = 0; i < NUM_TRENS; i++) {
			xTaskCreate(tremTask, "TREM", configMINIMAL_STACK_SIZE, NULL, 2, NULL);
		}

		for (int i = 0; i < NUM_CARROS; i++) {
			xTaskCreate(carroTask, "CARRO", configMINIMAL_STACK_SIZE, NULL, 1, NULL);
		}

	}
	else {
		printf("erro");
	}

	vTaskStartScheduler();
	for (;;);
	return 0;
}
```

Primeiro, há a inicialização do heap, que é alocado dinamicamente para gerenciar a memória durante a execução do programa. Em seguida, é habilitado o trace recorder, que é uma ferramenta opcional utilizada para análise e depuração do código.

Após isso, são criados os semáforos necessários para controlar o acesso ao cruzamento e suas direções. O semáforo "xSemaphoreCruzamento" é configurado para permitir duas entradas, representando o cruzamento onde tanto trens quanto carros podem passar simultaneamente em diferentes direções. Além disso, são criados semáforos mutex para cada uma das direções do cruzamento, garantindo que apenas um trem ou carro possa acessar cada direção por vez.

Em seguida, são criadas as tasks "tremTask" e "carroTask" utilizando uma estrutura de repetição "for". O número de tasks criadas é determinado pelas constantes "NUM_TRENS" e "NUM_CARROS". Cada task é atribuída a uma prioridade adequada para garantir o fluxo correto no cruzamento, com as tasks de trem tendo prioridade mais alta que as tasks de carro.

Antes de criar as tasks, é verificado se o semáforo "xSemaphoreCruzamento" foi criado com sucesso. Se não, uma mensagem de erro é exibida. Por fim, o scheduler do FreeRTOS é iniciado com "vTaskStartScheduler()", responsável por gerenciar as tasks e suas execuções. O programa entra em um loop infinito, mantendo a execução contínua do sistema.

## Execução

![image](https://github.com/DouradoR/STR-Projeto-3/assets/166778388/67bc6719-c55d-4ab9-a17c-bcdd4f34d742)

![image](https://github.com/DouradoR/STR-Projeto-3/assets/166778388/1be5dd59-7d9e-4147-a4ac-001ce00276d2)

![image](https://github.com/DouradoR/STR-Projeto-3/assets/166778388/947705cd-e13b-4e7f-b651-60c3e7530ade)

