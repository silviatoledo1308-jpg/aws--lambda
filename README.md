Desafio: Automatização com AWS Lambda, S3 e Localstack
Entendendo o Desafio

Agora é a sua hora de brilhar e construir um perfil de destaque na DIO.
Neste desafio, você colocará em prática a automação de infraestrutura na AWS utilizando CloudFormation, Lambda e S3, além de realizar testes locais com o Localstack.
O objetivo é demonstrar a capacidade de aplicar Infraestrutura como Código (IaC), automatizando recursos e documentando todo o processo de forma clara e técnica.

Descrição do Desafio

Este laboratório tem como objetivo consolidar os conhecimentos em tarefas automatizadas usando Lambda Function e S3, aplicando um modelo do AWS CloudFormation para criar e configurar automaticamente os recursos necessários.

Ao final, você terá um ambiente totalmente automatizado e funcional contendo:

Bucket S3 configurado

Função Lambda integrada

Tabela DynamoDB para armazenamento

API Gateway para interação via HTTP

Ambiente local simulado com Localstack

Objetivos de Aprendizagem

Ao concluir este desafio, você será capaz de:

Aplicar os conceitos aprendidos em um ambiente prático

Documentar processos técnicos de forma clara e estruturada

Utilizar o GitHub como ferramenta para documentação técnica

Automatizar infraestrutura com CloudFormation e Localstack

Implementar funções Lambda e triggers automatizadas com S3

Etapas do Desafio
1. Instalação do Localstack
Opção 1 – Usando Docker
docker run -d --name localstack -p 4566:4566 -p 4571:4571 \
-e SERVICES=ALL -e DEBUG=1 \
-v /var/run/docker.sock:/var/run/docker.sock localstack/localstack

Opção 2 – Usando CLI
pip install localstack
localstack --version
localstack start


Verifique se o Localstack está ativo:

Invoke-RestMethod -Uri "http://localhost:4566/_localstack/health"

2. Configuração do AWS CLI Local

Configure o ambiente com credenciais fictícias:

aws configure


Ou diretamente:

$env:AWS_ACCESS_KEY_ID="test"
$env:AWS_SECRET_ACCESS_KEY="test"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:AWS_DEFAULT_OUTPUT="json"

3. Criação dos Recursos Locais
Criar Bucket S3
awslocal s3api create-bucket --bucket notas-fiscais-upload

Criar Tabela DynamoDB
aws dynamodb create-table --endpoint-url=http://localhost:4566 \
--table-name NotasFiscais \
--attribute-definitions AttributeName=id,AttributeType=S \
--key-schema AttributeName=id,KeyType=HASH \
--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5

Verificar Tabela
aws dynamodb list-tables --endpoint-url=http://localhost:4566

4. Criar e Configurar Função Lambda

Empacote o código:

zip lambda_function.zip grava_db.py


Crie a função:

aws lambda create-function \
--function-name ProcessarNotasFiscais \
--runtime python3.9 \
--role arn:aws:iam::000000000000:role/lambda-role \
--handler grava_db.lambda_handler \
--zip-file fileb://lambda_function.zip \
--endpoint-url=http://localhost:4566


Valide:

aws lambda list-functions --endpoint-url=http://localhost:4566

5. Configurar Trigger do S3

Conceder permissão ao S3:

aws lambda add-permission \
--function-name ProcessarNotasFiscais \
--statement-id s3-trigger-permission \
--action "lambda:InvokeFunction" \
--principal s3.amazonaws.com \
--source-arn "arn:aws:s3:::notas-fiscais-upload" \
--endpoint-url=http://localhost:4566


Criar notificação no bucket:

aws s3api put-bucket-notification-configuration \
--bucket notas-fiscais-upload \
--notification-configuration file://notification_roles.json \
--endpoint-url=http://localhost:4566

6. Testar o Fluxo

Gerar um arquivo de teste:

python gerar_dados.py


Enviar ao bucket:

aws s3 cp notas_fiscais_2025.json s3://notas-fiscais-upload --endpoint-url=http://localhost:4566


Verifique no DynamoDB:

aws dynamodb scan --table-name NotasFiscais --endpoint-url=http://localhost:4566

7. Criar API Gateway

Criar API:

aws apigateway create-rest-api --name "NotasFiscaisAPI" --endpoint-url=http://localhost:4566


Obter recurso raiz:

aws apigateway get-resources --rest-api-id <ID_API> --endpoint-url=http://localhost:4566


Criar recurso /notas:

aws apigateway create-resource \
--rest-api-id <ID_API> \
--parent-id <ID_ROOT> \
--path-part "notas" \
--endpoint-url=http://localhost:4566


Criar métodos e integração:

aws apigateway put-method \
--rest-api-id <ID_API> --resource-id <ID_NOTAS> \
--http-method POST --authorization-type "NONE" \
--endpoint-url=http://localhost:4566

aws apigateway put-integration \
--rest-api-id <ID_API> --resource-id <ID_NOTAS> \
--http-method POST --type AWS_PROXY \
--integration-http-method POST \
--uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" \
--endpoint-url=http://localhost:4566


Conceder permissão à API:

aws lambda add-permission \
--function-name ProcessarNotasFiscais \
--statement-id apigateway-access \
--action "lambda:InvokeFunction" \
--principal apigateway.amazonaws.com \
--source-arn "arn:aws:execute-api:us-east-1:000000000000:<ID_API>/*/POST/notas" \
--endpoint-url=http://localhost:4566


Criar o deployment:

aws apigateway create-deployment --rest-api-id <ID_API> --stage-name dev --endpoint-url=http://localhost:4566

8. Testar a API
POST (criar nota)
Invoke-RestMethod -Uri "http://localhost:4566/restapis/<ID_API>/dev/_user_request_/notas" `
                  -Method POST `
                  -ContentType "application/json" `
                  -Body '{"id": "NF-999", "cliente": "João Silva", "valor": 1000.0, "data_emissao": "2025-01-31"}'

GET (listar notas)
Invoke-RestMethod -Uri "http://localhost:4566/restapis/<ID_API>/dev/_user_request_/notas" -Method GET

9. Monitoramento e Logs

Verificar logs do Localstack:

docker logs localstack


Monitorar CloudWatch (se habilitado no template):

EnableCloudWatchMonitoring=true

Resumo Final

Empacotar código Lambda

Criar e configurar bucket S3

Vincular trigger do S3 à Lambda

Integrar Lambda com API Gateway

Testar chamadas HTTP localmente

Validar inserções no DynamoDB

Conclusão

Este projeto demonstra como automatizar integrações entre Lambda, S3, DynamoDB e API Gateway, validando tudo localmente com Localstack.
Com isso, você reproduz um ambiente AWS completo de forma local, documentada e versionada, consolidando boas práticas de Infraestrutura como Código (IaC).
A abordagem permite o desenvolvimento e a validação de soluções escaláveis e consistentes, reduzindo custos e riscos em ambientes de produção.

## 👩‍💻 Autora **Silvia Toledo**

## 🔗 Referências

Documentação Oficial AWS Lambda: https://docs.aws.amazon.com/lambda

Documentação do Amazon S3: https://docs.aws.amazon.com/s3

Documentação do Amazon API Gateway: https://docs.aws.amazon.com/apigateway

Documentação do Amazon DynamoDB: https://docs.aws.amazon.com/dynamodb

Documentação do AWS CloudFormation: https://docs.aws.amazon.com/cloudformation

Localstack Documentation: https://docs.localstack.cloud

AWS CLI Command Reference: https://docs.aws.amazon.com/cli/latest/reference
