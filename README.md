# Dominando o IOT HUB , Device EDGE e Device Edge Modules

> Objetivo: Criar um IOT Edge Device com módulos que trocam mensagens através do IOT HUB local. Um dos módulos irá mandar mensagens de temperatura e o outro irá receber essas mensagens e irá enviar para o IOT HUB na nuvem. 


## 💻 Pré-requisitos

Antes de começar, verifique se você atendeu aos seguintes requisitos:
* Ter uma subscription na Azure (gratuita ou paga)
* Windows 10
* Docker 
* Azure CLI
* Conta pública no DockerHUB
* .Net Core SDK 3.1

## 🚀 Configurando a Azure
* Criar um Resource Group (```az group create --location LOCATION  --name RESOURCE_GROUP_NAME```)
* Criar um IOT Hub (```az iot hub create --name IOT_HUB_NAME --resource-group RESOURCE_GROUP_NAME --sku S1```)
* Dentro do IOT Hub criado, criar um IOT Edge Device (```az iot hub device-identity create --device-id DEVICE_NAME --hub-name IOT_HUB_NAME --edge-enabled```)

Após a execução dos comandos acima, será disponibilizado um IoT Edge device que já contem um IOT EDGE Agent (gerenciador dos módulos) e um  IOT HUB (comunicação entre os módulos)
Basicamente, os módulos do device são containers Docker que serão hospedados numa VM gerenciada na própria Azure. Logo abaixo será detalhado como criar essa VM. 

## ☕ Criando a VM de deployment do IOT Edge Device

Uma VM na Azure que já terá um template pronto para acessar os módulos do device será criada a partir do comando abaixo. 
Nela é possível verificar logs dos módulos, criar deployments, gerenciar os módulos (containers), verificar estado do IOT EDGE Agent Runtime.

```
az deployment group create `
--resource-group RESOURCE_GROUP_NAME `
--template-uri "https://aka.ms/iotedge-vm-deploy" `
--parameters dnsLabelPrefix='DNS_NAME' `
--parameters adminUsername='azureuser' `
--parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id DEVICE_NAME --hub-name IOT_HUB_NAME -o tsv) `
--parameters authenticationType='password' `
--parameters adminPasswordOrKey="PASSWORD"
```


## 💻 Criando imagem dos containers 
Nesse repositório foram incluídas duas solutions em .NET Core 3.1 , cujo código fonte foi retirado dos próprios tutoriais da Microsoft. 
A solution  do diretório **MessageGenerator** corresponde ao código fonte do módulo do device que irá enviar mensagens simuladas de um sensor de temperatura.
Essas mensagens estão sendo roteadas para um output de nome **temperatureOutput**  através da linha 
	  ``` await moduleClient.SendEventAsync("temperatureOutput", eventMessage); ```

A solution  do diretório **ModuleListener**  corresponde ao código fonte do módulo do device que irá escutar as mensagens de temperatura.
Para que as mensagens sejam recebidas e tratadas, é necessário determinar um handler que irá apontar para um método de callback, onde o primeiro parâmetro corresponde ao nome do "tópico", o segundo é o método de callback e o terceiro é o client que deverá ser o Module Client previamente instanciado. 
	  ``` await ioTHubModuleClient.SetInputMessageHandlerAsync("NOME_TOPICO", MetodoCallback , context); 	``` 
	  
O método de callback deverá ser um método estático e assíncrono cujo o retorno será um MessageResponse. O primeiro parâmetro será a mensagem e o segundo é o Client do módulo.
	  ``` static async Task<MessageResponse> PipeMessage(Message message, object userContext) ```
	  
	  
É necessário criar um build de ambas imagens e enviar em seguida para o Docker HUB. 
	  ```  docker build -t DOCKER_HUB_REGISTRY_NAME/IMAGE_NAME . ``` 
	  ```  docker push DOCKER_HUB_REGISTRY_NAME/IMAGE_NAME ``` 

## 💻 Realizar o deploy dos módulos no Edge Device 
Nesse repositório você irá encontrar o arquivo de manifesto para deployment dos módulos no Edge Device **deployment.json** . 
É importante trocar os parâmetros do manifesto correspondentes às imagens dos containers ```DOCKER_HUB_REGISTRY_NAME/MESSAGE_LISTENER```  e  ``` DOCKER_HUB_REGISTRY_NAME/MESSAGE_GENERATOR```  

A configuração do roteamento do IOT HUB pode ser realizada dentro do item ``` "routes"  ``` das propriedades do "$edgeHub". 
No exemplo desse repositório, podem ser observadas 2 rotas, uma delas sendo responsável pela comunicação entre os dois módulos, e a outra que corresponde à comunicação entre o módulo de filtro e o IOT HUB na núvem. 
Para realizar uma conexão com um módulo interno, utiliza-se o **BrokeredEndpoint("/modules/MODULE_NAME/inputs/TOPIC_NAME")**, como no exemplo abaixo
	  ```   "sensorToFilter": "FROM /messages/modules/messagegenerator/outputs/temperatureOutput INTO BrokeredEndpoint("/modules/filtermodule/inputs/input1")" ``` 
 
Para realizar uma comunicação com o IOT HUB na nuvem, utiliza-se o  **$upstream** , como no exemplo abaixo
	  ```  "filterToIoTHub": "FROM /messages/modules/filtermodule/outputs/output1 INTO $upstream"  ``` 

Finalmente, para realizar o deployment do DEVICE EDGE, basta utilizar o comando abaixo, apontando para o arquivo de manifesto. 
	  ``` az iot edge set-modules --device-id DEVICE_NAME --hub-name IOT_HUB_NAME --content ./deployment.json ```
	  
	  
##  📝 Monitorar o IOT EDGE Agent
É possível conectar por SSH na VM (criada anteriormente) que orquestra os módulos do dispositivo através do comando:
	  ``` ssh azureUser@DNS_NAME.eastus.cloudapp.azure.com  ```
	 
Alguns comandos que podem ser executados nessa VM:
* Verificar o estado do Edge Agent: ``` sudo iotedge check ```
* Listar todos os módulos que estão em execução no Device: ``` sudo iotedge list ```
* Verificar logs de um módulo: ``` sudo iotedge logs MODULE_NAME ```
* Reiniciar um módulo: ``` sudo iotedge restart MODULE_NAME ```
