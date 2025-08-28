[Português do Brasil](#português-do-brasil) | [English](#english)

---
<br>

## English 🇺🇸

<div id="english"></div>

# AWS Service Deployment Guide with ECS, Docker, and GitLab

This document serves as a complete guide for developers to deploy a new containerized service on AWS, using ECS (Elastic Container Service), Docker, and a CI/CD pipeline with GitLab.

---

### **Prerequisites**

Before you start, ensure you have:

- **Docker** installed on your local machine.
- **AWS CLI** installed and configured with access credentials.
- Developer access to the AWS account and the project in GitLab.

---

### **Step 1: Application Containerization**

The first step is to package your application into a Docker image. This is done with two main files: `Dockerfile` and `docker-compose.yml`.

#### **1.1. Dockerfile (For Production)**

The `Dockerfile` is the recipe for building your image. It defines the environment, copies files, installs dependencies, and specifies how the application should run.

**Example `Dockerfile` for a NestJS application:**

```dockerfile
# Base Stage: Use a lean and secure Node.js image
FROM node:20-bullseye-slim

# Set the working directory inside the container
WORKDIR /var/www

# Copy dependency files and install packages
# This leverages Docker's cache, reinstalling dependencies only if package.json changes
COPY package*.json ./
RUN npm install && \
    npm cache clean --force && \
    npm cache verify

# Copy the rest of the application code
COPY . .

# Copy the environment file from CI. In local runs,
# variables should be injected via 'docker run --env-file'
COPY .ci_env .env

# Compile the application for production
RUN npm run build

# Expose the port the application will use inside the container
EXPOSE 3000

# Command to start the application in production
CMD ["npm","run","start:prod"]
```

#### **1.2. docker-compose.yml (For Local Development)**

This file makes it easy to run your container locally, managing networks and environment variables.

**Example `docker-compose.yml`:**

```yaml
services:
  api:
    build:
      context: .
    container_name: meili-aprova-api-dev
    ports:
      - '3000:3000'
    volumes:
      - .:/usr/src/app
      - /usr/src/app/node_modules
    command: npm run start:dev
```

---

### **Step 2: CI/CD Pipeline with `.gitlab-ci.yml`**

This file automates the testing and deployment process whenever there is a push to the repository.

**Example `.gitlab-ci.yml`:**

```yaml
image: docker:20.10.16

stages:
  - test
  - deploy
  - clean

variables:
  # --- CONFIGURE THESE VARIABLES ---
  CI_AWS_ECS_SERVICE: "your-service-name-in-ecs"
  CI_ECR_REPOSITORY: "your-repository-name-in-ecr"
  CI_AWS_ECS_CLUSTER: "your-ecs-cluster-name"
  AWS_DEFAULT_REGION: "us-east-1"
  # --------------------------------

  CI_ECR_IMAGE: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$CI_ECR_REPOSITORY

# Test Job
test-job:
  image: node:20-alpine
  stage: test
  script:
    - npm install
    - npm test
  only:
    - homolog # or main/master

# Deploy Job (Build + Push + Update)
deploy-job:
  stage: deploy
  script:
    # 1. Install and Login to AWS
    - pip install awscli
    - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com

    # 2. Build and Push the Image to ECR
    - docker build --tag $CI_ECR_IMAGE:$CI_COMMIT_SHA --tag $CI_ECR_IMAGE:latest .
    - docker push $CI_ECR_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_ECR_IMAGE:latest

    # 3. Update the Service in ECS to use the new image
    - aws ecs update-service --cluster $CI_AWS_ECS_CLUSTER --service $CI_AWS_ECS_SERVICE --force-new-deployment --region $AWS_DEFAULT_REGION
  only:
    - homolog # or main/master

# Clean Job (Optional)
clean-job:
  stage: clean
  script:
    - echo "Cleaning Docker images from runner..."
    - docker rmi -f $CI_ECR_IMAGE:$CI_COMMIT_SHA
    - docker rmi -f $CI_ECR_IMAGE:latest
  allow_failure: true
  only:
    - homolog
```

**Note:** Remember to configure the environment variables (`AWS_ACCESS_KEY_ID`, etc.) in your GitLab project's CI/CD settings.

---

### **Step 3: ECR (Elastic Container Registry) Configuration**

ECR is AWS's private registry for your Docker images.

1. Access the **ECR** service in the AWS console.
2. Click **"Create repository"**.
3. Keep the visibility as `Private`.
4. Give the repository a name (e.g., `my-new-service`). This name must be the same as the one you put in the `CI_ECR_REPOSITORY` variable in your `.gitlab-ci.yml`.
5. Click **"Create repository"**.

#### **3.1. Pushing an Image Manually via AWS CLI (Optional)**

Before the CI/CD pipeline is working, it can be useful to manually push an image to ECR for testing.

**1. Authenticate Docker with ECR:**
   Run this command in your terminal to get a temporary login token.

   ```sh
   aws ecr get-login-password --region <your_region> | docker login --username AWS --password-stdin <your_account_id>.dkr.ecr.<your_region>.amazonaws.com
   ```

- Replace `<your_region>` (e.g., `us-east-1`) and `<your_account_id>`.

**2. Build your Docker Image:**
   In the project root, where the `Dockerfile` is, run:

   ```sh
   docker build -t <repository_name> .
   ```

**3. Tag the Image for ECR:**
   The image needs to be tagged with the full URI of your ECR repository.

   ```sh
   docker tag <repository_name>:latest <your_account_id>.dkr.ecr.<your_region>.amazonaws.com/<repository_name>:latest
   ```

**4. Push the Image:**
   Now, push the tagged image to the repository.

   ```sh
   docker push <your_account_id>.dkr.ecr.<your_region>.amazonaws.com/<repository_name>:latest
   ```

   After the push, you will see the `latest` image in your repository in the ECR console.

---

### **Step 4: Task Definition Configuration**

The Task Definition is the "blueprint" of your application for ECS.

1. Access the **ECS** service and go to **"Task Definitions"**.
2. Click **"Create new task definition"**.
3. **Task definition family**: Give it a name (e.g., `my-new-service-task`).
4. **Launch type**: Select **EC2**.
5. **Container details**:
    - **Name**: Give the container a name (e.g., `my-new-service-container`).
    - **Image URI**: Paste the URI of your ECR repository, including the `:latest` tag. Ex: `225399513320.dkr.ecr.us-east-1.amazonaws.com/my-new-service:latest`.
    - **Port mappings**: Add the mapping. **Container port**: `3000`, **Host port**: `0`. Using `0` for the host port allows Docker to choose a free port dynamically, avoiding conflicts.
6. **Environment variables**: Here you can add the environment variables that your service will need. It is preferable to put the environment variables in GitLab (or equivalent) and inject the variables into the pipeline at the time of the image build.
7. **Task size**: Define the resources. Start with values like **Task memory: `1024` MiB** and **Task CPU: `512` units**.
8. Click **"Create"**.

---

### **Step 5: Service Configuration in the ECS Cluster**

The service is responsible for keeping your task running in the cluster.

#### **Flow A: Use an Existing Cluster**

1. Go to your ECS cluster and click the **"Services"** tab.
2. Click **"Create"**.
3. **Capacity provider strategy**: Leave the default cluster strategy so that ECS can choose the best instance.
4. **Task Definition**: Select the "Family" and "Revision" (`LATEST`) of the task you created in the previous step.
5. **Service name**: Give the service a name (e.g., `my-new-service-service`). This name must be the same as the `CI_AWS_ECS_SERVICE` variable in your `.gitlab-ci.yml`.
6. Skip the Load Balancing section for now (we'll do that in the next step) and click **"Create"**.

#### **Flow B: Create a New Cluster**

(Usually done by the infrastructure team)

1. In the ECS console, click **"Clusters"** and **"Create Cluster"**.
2. Choose the **"EC2 Linux + Networking"** template.
3. Give the cluster a name.
4. In **"Instance configuration"**, choose the instance type (e.g., `t3.small` or larger), the number of instances, and the SSH key pair.
5. Configure the VPC and subnets where the instances will be created.
6. Click **"Create"**.

---

### **Step 6: Load Balancer and Target Group Configuration**

This securely exposes your application to the internet.

#### **Flow A: Use an Existing Load Balancer**

1. Go to **EC2 > Target Groups** and click **"Create target group"**.
    - **Target type**: `Instance`.
    - **Name**: `my-new-service-tg`.
    - **Protocol/Port**: `HTTP` / `80`.
    - **VPC**: The same as your cluster.
    - **Health checks**: Configure a health check path (e.g., `/health`). If you don't have one, use `/`.
    - Click **"Create"**.
2. Go to **EC2 > Load Balancers**, select your ALB, and go to the **"Listeners"** tab.
3. Select the listener (e.g., `HTTP: 80` or `HTTPS: 443`) and click **"View/edit rules"**.
4. Click **"Add rule"** (`+` icon).
5. **IF (Condition)**: `Host header` is `my-new-service.mydomain.com`.
6. **THEN (Action)**: `Forward to` and select the Target Group (`my-new-service-tg`) you just created.
7. Save the rule.

#### **Flow B: Create a New Load Balancer**

1. Go to **EC2 > Load Balancers** and click **"Create Load Balancer"**.
2. Choose **"Application Load Balancer"**.
3. Give it a name, choose `internet-facing`, `IPv4`.
4. Select the VPC and subnets.
5. Create or select a Security Group that allows traffic on ports 80/443.
6. Create a Target Group as in Flow A and associate it with the default listener.
7. Click **"Create"**.

---

### **Step 7: Connect the Service to the Load Balancer**

Now, let's connect everything.

1. Go back to your **Service** in ECS.
2. Click **"Update"**.
3. Navigate to the **"Load balancing"** section.
4. Select **"Application Load Balancer"**.
5. Choose the correct Load Balancer and listener.
6. In "Container to load balance", click **"Add to load balancer"** and select your **Target Group** (`my-new-service-tg`).
7. Click **"Update"**. ECS will automatically register the instances where your task is running in the Target Group.

---

### **Step 8: Domain (DNS) Configuration**

The final step is to point your domain to the Load Balancer.

#### **Flow A: Use AWS Route 53**

1. Go to the **Route 53** service.
2. Enter the **"Hosted zone"** for your domain (e.g., `mydomain.com`).
3. Click **"Create record"**.
    - **Record name**: `my-new-service`.
    - **Record type**: `A`.
    - **Alias**: Enable this option.
    - **Route traffic to**: Choose `Alias to Application and Classic Load Balancer`, select the region, and then select your Load Balancer by its DNS name.
4. Click **"Create records"**.

#### **Flow B: Use an External DNS (e.g., Cloudflare)**

If your DNS is managed by Cloudflare, the process is slightly different, but just as simple.

**1. Find your Load Balancer's Public DNS (in AWS):**

- Follow the same step as in Flow A to find and copy the **DNS name** of your Load Balancer in the EC2 console. The value will be something like `my-alb-123456789.us-east-1.elb.amazonaws.com`.

**2. Create the DNS Record in Cloudflare:**

   1. Access the **Cloudflare** dashboard and select your domain.
   2. In the left menu, go to **"DNS" > "Records"**.
   3. Click **"Add record"**.
   4. Fill out the new record form:
       - **Type**: Select `CNAME`. (This is the correct type for pointing a domain to another domain name).
       - **Name**: Enter only the subdomain (e.g., `my-new-service`).
       - **Target**: Paste the **DNS name** of your Load Balancer that you copied from AWS here.
       - **Proxy status**: Keep the status "Proxied" (orange cloud). This enables Cloudflare's CDN and security features for your subdomain.
   5. Click **"Save"**.

After a few minutes for DNS propagation, your service will be live and accessible via the configured URL, with traffic passing through Cloudflare.

<br>

---
<br>

## Português do Brasil 🇧🇷

<div id="português-do-brasil"></div>

# Guia de Deploy de Serviços na AWS com ECS, Docker e GitLab

Este documento serve como um guia completo para desenvolvedores implantarem um novo serviço containerizado na AWS, utilizando ECS (Elastic Container Service), Docker e um pipeline de CI/CD com GitLab.

---

### **Pré-requisitos**

Antes de começar, garanta que você tenha:

- **Docker** instalado na sua máquina local.
- **AWS CLI** instalado e configurado com as credenciais de acesso.
- Acesso de desenvolvedor à conta da AWS e ao projeto no GitLab.

---

### **Passo 1: Containerização da Aplicação**

O primeiro passo é empacotar sua aplicação em uma imagem Docker. Isso é feito com dois arquivos principais: `Dockerfile` e `docker-compose.yml`.

#### **1.1. Dockerfile (Para Produção)**

O `Dockerfile` é a receita para construir sua imagem. Ele define o ambiente, copia os arquivos, instala dependências e especifica como a aplicação deve ser executada.

**Exemplo de `Dockerfile` para uma aplicação NestJS:**

```dockerfile
# Estágio Base: Use uma imagem Node.js enxuta e segura
FROM node:20-bullseye-slim

# Define o diretório de trabalho dentro do container
WORKDIR /var/www

# Copia os arquivos de dependência e instala os pacotes
# Isso aproveita o cache do Docker, reinstalando dependências apenas se o package.json mudar
COPY package*.json ./
RUN npm install && \
    npm cache clean --force && \
    npm cache verify

# Copia o restante do código da aplicação
COPY . .

# Copia o arquivo de ambiente do CI. Em execuções locais,
# as variáveis devem ser injetadas via 'docker run --env-file'
COPY .ci_env .env

# Compila a aplicação para produção
RUN npm run build

# Expõe a porta que a aplicação usará dentro do container
EXPOSE 3000

# Comando para iniciar a aplicação em produção
CMD ["npm","run","start:prod"]
```

#### **1.2. docker-compose.yml (Para Desenvolvimento Local)**

Este arquivo facilita a execução do seu container localmente, gerenciando redes e variáveis de ambiente.

**Exemplo de `docker-compose.yml`:**

```yaml
services:
  api:
    build:
      context: .
    container_name: meili-aprova-api-dev
    ports:
      - '3000:3000'
    volumes:
      - .:/usr/src/app
      - /usr/src/app/node_modules
    command: npm run start:dev
```

---

### **Passo 2: Pipeline de CI/CD com `.gitlab-ci.yml`**

Este arquivo automatiza o processo de teste e deploy sempre que há um push para o repositório.

**Exemplo de `.gitlab-ci.yml`:**

```yaml
image: docker:20.10.16

stages:
  - test
  - deploy
  - clean

variables:
  # --- CONFIGURE ESTAS VARIÁVEIS ---
  CI_AWS_ECS_SERVICE: "nome-do-seu-servico-no-ecs"
  CI_ECR_REPOSITORY: "nome-do-seu-repositorio-no-ecr"
  CI_AWS_ECS_CLUSTER: "nome-do-seu-cluster-ecs"
  AWS_DEFAULT_REGION: "us-east-1"
  # --------------------------------

  CI_ECR_IMAGE: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$CI_ECR_REPOSITORY

# Job de Teste
test-job:
  image: node:20-alpine
  stage: test
  script:
    - npm install
    - npm test
  only:
    - homolog # ou main/master

# Job de Deploy (Build + Push + Update)
deploy-job:
  stage: deploy
  script:
    # 1. Instalação e Login na AWS
    - pip install awscli
    - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com

    # 2. Build e Push da Imagem para o ECR
    - docker build --tag $CI_ECR_IMAGE:$CI_COMMIT_SHA --tag $CI_ECR_IMAGE:latest .
    - docker push $CI_ECR_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_ECR_IMAGE:latest

    # 3. Atualiza o Serviço no ECS para usar a nova imagem
    - aws ecs update-service --cluster $CI_AWS_ECS_CLUSTER --service $CI_AWS_ECS_SERVICE --force-new-deployment --region $AWS_DEFAULT_REGION
  only:
    - homolog # ou main/master

# Job de Limpeza (Opcional)
clean-job:
  stage: clean
  script:
    - echo "Limpando imagens Docker do runner..."
    - docker rmi -f $CI_ECR_IMAGE:$CI_COMMIT_SHA
    - docker rmi -f $CI_ECR_IMAGE:latest
  allow_failure: true
  only:
    - homolog
```

**Observação:** Lembre-se de configurar as variáveis de ambiente (`AWS_ACCESS_KEY_ID`, etc.) nas configurações de CI/CD do seu projeto no GitLab.

---

### **Passo 3: Configuração do ECR (Elastic Container Registry)**

O ECR é o registro privado da AWS para suas imagens Docker.

1. Acesse o serviço **ECR** no console da AWS.
2. Clique em **"Create repository"**.
3. Mantenha a visibilidade como `Private`.
4. Dê um nome ao repositório (ex: `meu-novo-servico`). Este nome deve ser o mesmo que você colocou na variável `CI_ECR_REPOSITORY` do seu `.gitlab-ci.yml`.
5. Clique em **"Create repository"**.

#### **3.1. Enviando uma Imagem Manualmente via AWS CLI (Opcional)**

Antes de ter o pipeline de CI/CD funcionando, pode ser útil enviar uma imagem manualmente para o ECR para testes.

**1. Autentique o Docker no ECR:**
   Execute este comando no seu terminal para obter um token de login temporário.

   ```sh
   aws ecr get-login-password --region <sua_regiao> | docker login --username AWS --password-stdin <seu_account_id>.dkr.ecr.<sua_regiao>.amazonaws.com
   ```

- Substitua `<sua_regiao>` (ex: `us-east-1`) e `<seu_account_id>`.

**2. Construa sua Imagem Docker:**
   Na raiz do projeto, onde está o `Dockerfile`, execute:

   ```sh
   docker build -t <nome_do_repositorio> .
   ```

**3. Tagueie a Imagem para o ECR:**
   A imagem precisa ser tagueada com o endereço completo (URI) do seu repositório no ECR.

   ```sh
   docker tag <nome_do_repositorio>:latest <seu_account_id>.dkr.ecr.<sua_regiao>.amazonaws.com/<nome_do_repositorio>:latest
   ```

**4. Envie a Imagem (Push):**
   Agora, envie a imagem tagueada para o repositório.

   ```sh
   docker push <seu_account_id>.dkr.ecr.<sua_regiao>.amazonaws.com/<nome_do_repositorio>:latest
   ```

   Após o push, você verá a imagem `latest` no seu repositório no console do ECR.

---

### **Passo 4: Configuração da Task Definition**

A Task Definition é o "plano" da sua aplicação para o ECS.

1. Acesse o serviço **ECS** e vá para **"Task Definitions"**.
2. Clique em **"Create new task definition"**.
3. **Task definition family**: Dê um nome (ex: `meu-novo-servico-task`).
4. **Launch type**: Selecione **EC2**.
5. **Container details**:
    - **Name**: Dê um nome ao container (ex: `meu-novo-servico-container`).
    - **Image URI**: Cole a URI do seu repositório ECR, incluindo a tag `:latest`. Ex: `225399513320.dkr.ecr.us-east-1.amazonaws.com/meu-novo-servico:latest`.
    - **Port mappings**: Adicione o mapeamento. **Container port**: `3000`, **Host port**: `0`. Usar `0` na porta do host permite que o Docker escolha uma porta livre dinamicamente, evitando conflitos.
6. **Environment variables**: Aqui você pode adicionar as variáveis de ambiente que o seu serviço vai precisar. Preferencialmente é melhor colocar as variáveis de ambiente no gitlab (ou equivalente) e injetar as variáveis no pipeline no momento do build da imagem
7. **Task size**: Defina os recursos. Comece com valores como **Task memory: `1024` MiB** e **Task CPU: `512` units**.
8. Clique em **"Create"**.

---

### **Passo 5: Configuração do Serviço no Cluster ECS**

O serviço é responsável por manter sua task rodando no cluster.

#### **Fluxo A: Usar um Cluster Existente**

1. Vá para o seu cluster ECS e clique na aba **"Services"**.
2. Clique em **"Create"**.
3. **Capacity provider strategy**: Deixe a estratégia padrão do cluster para que o ECS possa escolher a melhor instância.
4. **Task Definition**: Selecione a "Family" e a "Revision" (`LATEST`) da task que você criou no passo anterior.
5. **Service name**: Dê um nome ao serviço (ex: `meu-novo-servico-service`). Este nome deve ser o mesmo da variável `CI_AWS_ECS_SERVICE` no seu `.gitlab-ci.yml`.
6. Pule a seção de Load Balancing por enquanto (faremos isso no próximo passo) e clique em **"Create"**.

#### **Fluxo B: Criar um Novo Cluster**

(Geralmente feito pela equipe de infraestrutura)

1. No console do ECS, clique em **"Clusters"** e **"Create Cluster"**.
2. Escolha o template **"EC2 Linux + Networking"**.
3. Dê um nome ao cluster.
4. Em **"Instance configuration"**, escolha o tipo da instância (ex: `t3.small` ou maior), o número de instâncias e o par de chaves SSH.
5. Configure a VPC e as subnets onde as instâncias serão criadas.
6. Clique em **"Create"**.

---

### **Passo 6: Configuração do Load Balancer e Target Group**

Isso expõe sua aplicação à internet de forma segura.

#### **Fluxo A: Usar um Load Balancer Existente**

1. Vá para **EC2 > Target Groups** e clique em **"Create target group"**.
    - **Target type**: `Instance`.
    - **Name**: `meu-novo-servico-tg`.
    - **Protocol/Port**: `HTTP` / `80`.
    - **VPC**: A mesma do seu cluster.
    - **Health checks**: Configure um caminho de health check (ex: `/health`). Se não tiver um, use `/`.
    - Clique em **"Create"**.
2. Vá para **EC2 > Load Balancers**, selecione seu ALB e vá na aba **"Listeners"**.
3. Selecione o listener (ex: `HTTP: 80` ou `HTTPS: 443`) e clique em **"View/edit rules"**.
4. Clique em **"Add rule"** (`+` ícone).
5. **IF (Condition)**: `Host header` é `meu-novo-servico.meudominio.com`.
6. **THEN (Action)**: `Forward to` e selecione o Target Group (`meu-novo-servico-tg`) que você acabou de criar.
7. Salve a regra.

#### **Fluxo B: Criar um Novo Load Balancer**

1. Vá para **EC2 > Load Balancers** e clique em **"Create Load Balancer"**.
2. Escolha **"Application Load Balancer"**.
3. Dê um nome, escolha `internet-facing`, `IPv4`.
4. Selecione a VPC e as subnets.
5. Crie ou selecione um Security Group que permita tráfego nas portas 80/443.
6. Crie um Target Group como no fluxo A e associe-o ao listener padrão.
7. Clique em **"Create"**.

---

### **Passo 7: Conectar o Serviço ao Load Balancer**

Agora, vamos ligar tudo.

1. Volte para o seu **Serviço** no ECS.
2. Clique em **"Update"**.
3. Navegue até a seção **"Load balancing"**.
4. Selecione **"Application Load Balancer"**.
5. Escolha o Load Balancer e o listener corretos.
6. Em "Container to load balance", clique em **"Add to load balancer"** e selecione o seu **Target Group** (`meu-novo-servico-tg`).
7. Clique em **"Update"**. O ECS irá registrar automaticamente as instâncias onde sua task está rodando no Target Group.

---

### **Passo 8: Configuração do Domínio (DNS)**

O passo final é apontar seu domínio para o Load Balancer.

#### **Fluxo A: Usar AWS Route 53**

1. Vá para o serviço **Route 53**.
2. Entre na **"Hosted zone"** do seu domínio (ex: `meudominio.com`).
3. Clique em **"Create record"**.
    - **Record name**: `meu-novo-servico`.
    - **Record type**: `A`.
    - **Alias**: Ative esta opção.
    - **Route traffic to**: Escolha `Alias to Application and Classic Load Balancer`, selecione a região, e então selecione seu Load Balancer pelo nome de DNS dele.
4. Clique em **"Create records"**.

#### **Fluxo B: Usar um DNS Externo (Ex: Cloudflare)**

Se seu DNS é gerenciado pelo Cloudflare, o processo é um pouco diferente, mas igualmente simples.

**1. Encontre o DNS Público do seu Load Balancer (na AWS):**

- Siga o mesmo passo do Fluxo A para encontrar e copiar o **DNS name** do seu Load Balancer no console do EC2. O valor será algo como `meu-alb-123456789.us-east-1.elb.amazonaws.com`.

**2. Crie o Registro DNS no Cloudflare:**

   1. Acesse o painel do **Cloudflare** e selecione o seu domínio.
   2. No menu à esquerda, vá para **"DNS" > "Records"**.
   3. Clique em **"Add record"**.
   4. Preencha o formulário do novo registro:
       - **Type**: Selecione `CNAME`. (Este é o tipo correto para apontar um domínio para outro nome de domínio).
       - **Name**: Insira apenas o subdomínio (ex: `meu-novo-servico`).
       - **Target**: Cole aqui o **DNS name** do seu Load Balancer que você copiou da AWS.
       - **Proxy status**: Mantenha o status "Proxied" (nuvem laranja). Isso ativa os recursos de CDN e segurança do Cloudflare para o seu subdomínio.
   5. Clique em **"Save"**.

Após alguns minutos para a propagação do DNS, seu serviço estará no ar e acessível pela URL configurada, com o tráfego passando pelo Cloudflare.
