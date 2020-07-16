# Implementação de "servidor" SOAP no sistema Serveless Lambda da AWS
## Introdução e método manual

O SOAP (https://en.wikipedia.org/wiki/SOAP) é uma tecnologia um pouco antiga mas que pode ser implementada na AWS com ajuda das funções lambda (https://aws.amazon.com/pt/serverless/serverlessrepo/) , ou seja de forma serveless. 

Uma implementação de um servidor SOAP na forma precisa de um dominio teorico dos conceitos:

+ SOAP nodes
+ SOAP roles    
+ SOAP protocol binding    
+ SOAP features   
+ SOAP module

O primeiro que deve se ter na mão para um desevolvimento assim são a descrição do serviço (arquivo WSDL https://www.devmedia.com.br/wsdl-simplifique-a-integracao-de-dados-via-web-service/30066 e https://fabriciosanchez.azurewebsites.net/3/wsdl-o-que-e-pra-que-serve-onde-utilizo/) e o esquema (arquivo XSD ) os quais podem ser parseados com ajuda das ferramantas:

+ Um parser (on-line) do WSDL: https://www.wsdl-analyzer.com/
+ Um parser (on-line) do XSD: http://xsd2xml.com/

Os arquivos de teste podem se ver em:

+ **nfe-test.xml** (https://github-java-script.s3.us-east-2.amazonaws.com/cleison_paulo/nfe-test.xml)

## Detalhes de Segurança eletrônicos:

Tradicionalmente a chave eletrônica é gerada com ajuda das certificações nacionais:

https://www.iti.gov.br/icp-brasil

https://www.iti.gov.br/repositorio/84-repositorio/143-repositorio-ac-raiz

Git para gerar a nota:

https://github.com/flexait/nfse

## Considerações para Develovep:

https://medium.com/greenm/aws-lambda-or-aws-fargate-the-step-by-step-guide-to-choosing-the-right-technology-925ebcf89b7c
<!---
SOAP
    This is a set of rules formalizing and governing the format and processing rules for information exchanged between a SOAP sender and a SOAP receiver.
SOAP nodes
    These are physical/logical machines with processing units which are used to transmit/forward, receive and process SOAP messages. These are analogous to nodes in a network.
SOAP roles
    Over the path of a SOAP message, all nodes assume a specific role. The role of the node defines the action that the node performs on the message it receives. For example, a role "none" means that no node will process the SOAP header in any way and simply transmit the message along its path.
SOAP protocol binding
    A SOAP message needs to work in conjunction with other protocols to be transferred over a network. For example, a SOAP message could use TCP as a lower layer protocol to transfer messages. These bindings are defined in the SOAP protocol binding framework.[13]
SOAP features
    SOAP provides a messaging framework only. However, it can be extended to add features such as reliability, security etc. There are rules to be followed when adding features to the SOAP framework.
SOAP module
    A collection of specifications regarding the semantics of SOAP header to describe any new features being extended upon SOAP. A module needs to realize zero or more features. SOAP requires modules to adhere to prescribed rules.[14]

-->



<!---


https://www.soapui.org/soapui-projects/soapui-projects.html/

Para este fim foi construído inicialmente um sistema caraterziado por:

+ Processamentos de fluxos de dados em JSON:
	- Lembremos que a consulta desde a CLI é feito com os comandos JSON:
	```bash
	aws glacier initiate-job --job-parameters '{"Type": "inventory-retrieval"}' --account-id YOUR_ACCOUNT_ID --region YOUR_REGION --vault-name YOUR_VAULT_NAME 
	```

+ Este procedimento de inventário demora aproximadamente 4 horas, e pode ser monitorado consultando a lista de trabalhos relacionada com dito VAULT:
	```bash
	aws glacier list-jobs --account-id - --vault-name 2020_abril_06
	```
+ Sendo possível esperar como resposta perante a consulta anterior:

	```json
			{
				"JobList": [
					{
						"InventoryRetrievalParameters": {
							"Format": "JSON"
						}, 
						"VaultARN": "arn:aws:glacier:us-east-2:937852338641:vaults/2020_abril_06", 
						"Completed": false, 
						"JobId": "O0bmSJWCWIJOTojdj_BhQjbdN6jrQ1O-q3A6v79d5MI-2mHbl-1iTnZUk0vhrrL-R44A70KO3767Azzz9STA9mMknVuD", 
						"Action": "InventoryRetrieval", 
						"CreationDate": "2020-07-12T14:55:22.300Z", 
						"StatusCode": "InProgress"
					}
				]
			}
	```

+ Se a resposta for do tipo:

	```json
	{
		"JobList": [
			{
				"CompletionDate": "2020-07-12T18:40:02.958Z", 
				"VaultARN": "arn:aws:glacier:us-east-2:937852338641:vaults/2020_abril_06", 
				"InventoryRetrievalParameters": {
					"Format": "JSON"
				}, 
				"Completed": true, 
				"InventorySizeInBytes": 34120, 
				"JobId": "O0bmSJWCWIJOTojdj_BhQjbdN6jrQ1O-q3A6v79d5MI-2mHbl-1iTnZUk0vhrrL-R44A70KO3767Azzz9STA9mMknVuD", 
				"Action": "InventoryRetrieval", 
				"CreationDate": "2020-07-12T14:55:22.300Z", 
				"StatusMessage": "Succeeded", 
				"StatusCode": "Succeeded"
			}
		]
	}

	```
Estariamos perando um cenário de "trabalho pronto" ("StatusCode": "Succeeded")
+ Já com este ("StatusCode": "Succeeded") pode-se coletar o inventário do VAULT usando:
	```bash
	aws glacier get-job-output --account-id - --vault-name 2020_abril_06 --job-id  O0bmSJWCWIJOTojdj_BhQjbdN6jrQ1O-q3A6v79d5MI-2mHbl-1iTnZUk0vhrrL-R44A70KO3767Azzz9STA9mMknVuD inventario_JSON.txt
	```
	Obte-se desta linha de comando o arquivo "inventario_JSON.txt" que pode ser estudado com ajuda dos arquivos:
	- **ler_inventario_SAIDA_ArchiveId.py** (Pare gerar uma lista de ArchiveID)
	- **ler_inventario_SAIDA_DATA.py** (Para ver a data de criação de cada elemento)
+ Pode-se criar uma lista de ArchiveID com ajuda do **"ler_inventario_SAIDA_ArchiveId.py"** e assim poder criar os JOBs de recuperação de dito arquivo (identificado com o ArchiveID) em cada vault usando o comando CLI:
	```bash
	aws glacier initiate-job --account-id - --vault-name 2020_abril_06  --job-parameters '{"Type": "archive-retrieval","ArchiveId": "'$line'","Description": "'$lista' '$indice'"}'
	```
	Com o intuito de fazer este pedido de "archive-retrieval" de forma massiva foi feito um shell chamdado de **"processar.sh"** que tem a capacidade de pegar os arquivos da pasta "fazer" para ler o conteúdo de cada arquivo (de nome "x*") e posteriormente colocar dito arquivo na pasta "feitos".

+ Etapa final na qual se gera uma arquivo único com ajuda do arquivo **"get_ARQUIVOS.py"** prévia execução do comando:
  
	```bash
	aws glacier list-jobs --account-id - --vault-name 2020_abril_06	 > testar	
	```

	Cabe salientar que será preciso invocar o python na forma:

	```bash
	python get_ARQUIVOS.py
	```	
Ve-se que este processamento manual pode-se transformar numa "Máquina de Estado" com ajuda da plataforma STEP AWS.

# Usando SNS, STEP, e lambda functions:






+ Migrar o projeto anterior para consultar uma API (https://q2gp3i5m5c.execute-api.us-east-2.amazonaws.com/default/jogo_quina?numero_jogo=111) desenvolvida em AWS lambda (https://github.com/julian-gamboa-bahia/jogo_quina/blob/master/consulta_sequencia.js)
  - Para este fim será preciso apenas usar uma consulta "$.getJSON" consultando a API já mencionada.

+ Sub-lists are made by indenting 2 spaces:
  - Marker character change forces new list start:
    * Ac tristique libero volutpat at
    + Facilisis in pretium nisl aliquet
    - Nulla volutpat aliquam velit
+ Very easy!




Não se esqueça de usar o CACHE das credenciais para agilizar as operações

+ git config --global credential.helper cache

+ git config --global credential.helper 'cache --timeout=13600'

-->
