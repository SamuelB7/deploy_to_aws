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
