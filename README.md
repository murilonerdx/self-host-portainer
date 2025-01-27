# Guia Completo: Deploy do Traefik e Portainer

Este guia fornece instruções detalhadas para configurar e implementar o **Traefik** e o **Portainer** no modo Swarm do Docker.  

Para qualquer modificação, atualização ou melhoria, sinta-se à vontade para retornar a esta página para revisar ou personalizar os códigos conforme necessário.

---

## Índice
1. [Traefik v2](#traefik-v2)
   - [Criar Rede para o Traefik](#criar-rede-para-o-traefik)
   - [Deploy do Traefik](#deploy-do-traefik)
2. [Portainer](#portainer)
   - [Criar Rede para o Portainer](#criar-rede-para-o-portainer)
   - [Criar Rede para o Traefik (Opcional)](#criar-rede-para-o-traefik-opcional)
   - [Stack do Portainer](#stack-do-portainer)
   - [Deploy do Portainer](#deploy-do-portainer)

---

## Traefik v2

O Traefik é um balanceador de carga dinâmico e reverso altamente eficiente. Este tutorial cobre os passos básicos para configurá-lo e utilizá-lo no Docker Swarm.

### Criar Rede para o Traefik

Se ainda não criou uma rede para trabalhar com o Traefik, execute o comando abaixo no terminal:

```bash
docker network create --driver=overlay traefik_public

```

Você pode criar uma rede com qualquer nome de sua preferência. Neste exemplo, utilizamos o nome traefik_public.

Deploy do Traefik
Copie o seguinte arquivo para fazer o deploy do Traefik:

```bash
version: '3.8'

services:
  traefik:
    image: traefik:v2.11
    command:
      - --providers.docker=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker.exposedbydefault=false
      - --providers.docker.swarmMode=true
      - --providers.docker.network=traefik_public
      - --providers.docker.endpoint=unix:///var/run/docker.sock
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.le.acme.email=seu_email@email.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_certificates:/letsencrypt
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
    networks:
      - traefik_public

volumes:
  traefik_certificates:
    external: true

networks:
  traefik_public:
    external: true


```

Esta configuração básica permite a utilização de recursos como domínio com SSL e suporte a várias aplicações.

Portainer
O Portainer é uma interface gráfica para gerenciamento do Docker, ideal para monitorar e gerenciar serviços no Swarm.

Nota: Antes de prosseguir com este tutorial, certifique-se de que o modo Swarm do Docker está ativado.

Criar Rede para o Portainer
Crie uma rede para o agente do Portainer com o comando abaixo:

```bash
docker network create --driver=overlay agent_network
docker network create --driver=overlay traefik_public
```
```bash
version: "3.8"

services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [ node.platform.os == linux ]

  portainer:
    image: portainer/portainer-ce:latest
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - 9000:9000
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
      - traefik_public
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [ node.role == manager ]
      #labels:
        #- "traefik.enable=true"
        #- "traefik.docker.network=traefik_public"
        #- "traefik.http.routers.portainer.rule=Host(`portainer.SEU_DOMINIO.com`)"
        #- "traefik.http.routers.portainer.entrypoints=websecure"
        #- "traefik.http.routers.portainer.tls.certresolver=le"
        #- "traefik.http.routers.portainer.service=portainer"
        #- "traefik.http.services.portainer.loadbalancer.server.port=9000"

networks:
  traefik_public:
    external: true
    attachable: true
  agent_network:
    external: true

volumes:
  portainer_data:
    external: true
```

## Deploy do Portainer
Para subir a stack do Portainer, utilize o comando abaixo:



```bash
docker stack deploy -c portainer.yaml portainer
docker stack deploy -c traefik.yaml traefik
```

## Exemplo aplicação spring boot

```bash
version: "3.8"

services:
  db:
    image: postgres:latest  # Usando a imagem oficial do PostgreSQL
    environment:
      POSTGRES_DB: auth
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: senha
      PGDATA: /data/postgres  # Localização dos dados dentro do container
    networks:
      - services  # Usando a rede overlay
    ports:
      - "5433:5432"  # Porta do host mapeada para o container
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Volume para persistência de dados

  ms-auth:
    image: <seu username>/servico # Nome do Dockerfile (pode ser alterado se necessário)
    ports:
      - "8084:8084"  # Porta exposta pelo serviço
    environment:
      JAVA_OPTS: "-Xms256m -Xmx1024m -XX:+UseG1GC"
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: auth
      DB_USER: postgres
      DB_PASSWORD: senha
    depends_on:
      - db  # Garante que o serviço `db` seja iniciado antes
    networks:
      - services  # Conectado à mesma rede overlay
      - traefik_public
    deploy:
      mode: replicated
      replicas: 1
      resources:
        limits:
          memory: 1G  # Limite de memória para a aplicação
          cpus: "1.0"  # Limite de 1 vCPU
        reservations:
          memory: 512M  # Memória reservada para a aplicação
          cpus: "0.5"  # Reservado 0.5 vCPU
      placement:
        constraints:
          - node.role == manager
      labels:
        - traefik.enable=true
        - traefik.http.routers.ms-auth.rule=Host(`aqui.seu.subdominio`)
        - traefik.http.routers.ms-auth.entrypoints=websecure
        - traefik.http.routers.ms-auth.tls.certresolver=le
        - traefik.http.routers.ms-auth.service=ms-auth
        - traefik.http.services.ms-auth.loadbalancer.server.port=8084

networks:
  services:
    driver: overlay
  traefik_public:
    external: true

volumes:
  postgres_data:  # Define o volume para persistência dos dados do banco

```
