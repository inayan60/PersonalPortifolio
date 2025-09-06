## 1. Instalação do Docker Engine e Docker Compose Plugin

### 1.1. Ubuntu/Debian

1.  **Atualizar repositórios e instalar dependências:**
    ```bash
    sudo apt update
    sudo apt install -y ca-certificates curl gnupg
    ```

2.  **Adicionar chave GPG oficial do Docker:**
    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```

3.  **Adicionar repositório Docker:**
    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

4.  **Atualizar lista de pacotes e instalar componentes Docker:**
    ```bash
    sudo apt update
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

5.  **Iniciar e habilitar o serviço Docker:**
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

### 1.2. CentOS/RHEL/Fedora

1.  **Configurar repositório Docker e instalar componentes:**
    ```bash
    # Para CentOS/RHEL 8+
    sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

    # Para Fedora (descomente e use se for o caso)
    # sudo dnf install -y dnf-plugins-core
    # sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
    # sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

2.  **Iniciar e habilitar o serviço Docker:**
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

### 1.3. openSUSE

1.  **Adicionar repositório Docker e instalar componentes:**
    ```bash
    sudo zypper addrepo https://download.docker.com/linux/opensuse/docker-ce.repo
    sudo zypper refresh
    sudo zypper install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```

2.  **Iniciar e habilitar o serviço Docker:**
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

---

## 2. Configuração de Usuário para Docker

1.  **Adicionar usuário ao grupo `docker`:**
    Permite a execução de comandos `docker` sem `sudo`.
    ```bash
    sudo usermod -aG docker $USER
    ```

2.  **Aplicar mudanças no grupo:**
    Requer logout/login ou uso de `newgrp`.
    ```bash
    newgrp docker
    ```

3.  **Verificar a instalação do Docker:**
    ```bash
    docker --version
    docker compose version
    ```

---

## 3. Inicialização do Docker Swarm

1.  **Inicializar o Swarm Manager:**
    Substitua `<YOUR_SERVER_IP>` pelo endereço IP da interface de rede principal do servidor.
    ```bash
    docker swarm init --advertise-addr <YOUR_SERVER_IP>
    ```
    Exemplo:
    ```bash
    # docker swarm init --advertise-addr 192.168.1.100
    ```
    Para clusters multi-nó, siga as instruções de `docker swarm join` fornecidas após a inicialização para adicionar nós worker.

2.  **Verificar o status do Swarm:**
    ```bash
    docker node ls
    ```

---

## 4. Criação da Rede Overlay Externa

Crie uma rede overlay para a comunicação entre os serviços do stack. A flag `--attachable` permite que containers não-stack possam interagir com ela.
```bash
docker network create --driver overlay --attachable your_stack_net
```
**Nota:** `your_stack_net` é um placeholder para o nome da sua rede.

---

## 5. Estrutura de Diretórios e Permissões

1.  **Criar o diretório raiz do projeto:**
    ```bash
    mkdir -p /opt/your-stack-name
    cd /opt/your-stack-name
    ```
    **Nota:** `your-stack-name` é um placeholder para o nome do seu stack.

2.  **Criar subdiretórios para volumes persistentes:**
    Ajuste a lista de diretórios conforme os serviços a serem implantados.
    ```bash
    mkdir -p {app1-data,app2-data,app2-db-data,app2-config,app2-config/extensions}
    # Exemplo para Vaultwarden, Guacamole, Portainer:
    # mkdir -p {vw-data,guac-drive,guac-recordings,guac-db-data,guac-initdb,guac-conf,guac-conf/extensions,portainer-data}
    ```

3.  **Definir permissões de propriedade e acesso:**
    ```bash
    sudo chown -R $USER:$USER /opt/your-stack-name
    chmod -R 755 /opt/your-stack-name
    ```

---

## 6. Preparação do Banco de Dados Guacamole (Exemplo)

Para a implantação do Guacamole, é necessário um schema de banco de dados inicial.

### 6.1. Extração do Schema SQL

1.  **Navegar para o diretório do stack:**
    ```bash
    cd /opt/your-stack-name
    ```

2.  **Baixar a imagem do Guacamole:**
    ```bash
    docker pull guacamole/guacamole:latest
    ```

3.  **Extrair os scripts SQL do schema:**
    ```bash
    docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > guac-initdb/001-initdb.sql
    ```

4.  **Verificar a criação do arquivo:**
    ```bash
    ls -la guac-initdb/
    ```

---

## 7. Criação de Docker Secrets

Secrets são utilizados para armazenar informações sensíveis (senhas, tokens) de forma segura no Swarm.

1.  **Secret para o token de administração do Vaultwarden (exemplo):**
    ```bash
    echo "$(openssl rand -base64 32)" | docker secret create vw_admin_token -
    ```

2.  **Secret para a senha do usuário PostgreSQL do Guacamole (exemplo):**
    ```bash
    echo "$(openssl rand -base64 32)" | docker secret create guac_postgres_password -
    ```

3.  **Secret para a senha de administração do Portainer (exemplo):**
    ```bash
    echo "$(openssl rand -base64 32)" | docker secret create portainer_admin_password -
    ```

4.  **Secret para a senha de bind do LDAP para Guacamole (se aplicável):**
    ```bash
    echo "YOUR_LDAP_BIND_PASSWORD" | docker secret create guac_ldap_bind_password -
    # Opcional: Para gerar uma senha aleatória se o usuário de bind for novo:
    # echo "$(openssl rand -base64 24)" | docker secret create guac_ldap_bind_password -
    ```
    **IMPORTANTE:** Substitua `"YOUR_LDAP_BIND_PASSWORD"` pela senha real do usuário de bind do seu ambiente LDAP.

5.  **Listar secrets criados:**
    ```bash
    docker secret ls
    ```

**ATENÇÃO:** Para visualizar o conteúdo de um secret (apenas para depuração e com cautela, pois expõe a informação sensível):
```bash
echo "Senha PostgreSQL Guacamole:" && docker secret inspect guac_postgres_password --format='{{.Spec.Data}}' | base64 -d
echo "Admin Token Vaultwarden:" && docker secret inspect vw_admin_token --format='{{.Spec.Data}}' | base64 -d
echo "Senha Admin Portainer:" && docker secret inspect portainer_admin_password --format='{{.Spec.Data}}' | base64 -d
echo "Senha LDAP Guacamole:" && docker secret inspect guac_ldap_bind_password --format='{{.Spec.Data}}' | base64 -d
```
Recomenda-se anotar estas credenciais em um gerenciador de senhas seguro.

---

## 8. Criação do Arquivo `docker-compose.yml`

1.  **Criar o arquivo `docker-compose.yml`:**
    ```bash
    nano /opt/your-stack-name/docker-compose.yml
    ```

2.  **Insira o conteúdo YAML abaixo.**
    Este exemplo configura Vaultwarden, Guacamole (com PostgreSQL e LDAP), e Portainer. Ajuste os serviços, volumes, redes e secrets conforme suas necessidades.

    ```yaml
    version: '3.8'

    services:
      # Serviço Vaultwarden
      vaultwarden:
        image: vaultwarden/server:latest
        deploy:
          replicas: 1
          restart_policy:
            condition: on-failure
        volumes:
          - /opt/your-stack-name/vw-data:/data
        environment:
          ADMIN_TOKEN: /run/secrets/vw_admin_token # Token de administração do Vaultwarden
        secrets:
          - vw_admin_token
        networks:
          - your_stack_net
        ports: # Descomente e ajuste as portas se não usar proxy reverso
          - "8080:80"

      # Serviço de Banco de Dados PostgreSQL para Guacamole
      guacamole-db:
        image: postgres:13
        deploy:
          replicas: 1
          restart_policy:
            condition: on-failure
        environment:
          POSTGRES_DB: guacamole_db
          POSTGRES_USER: guacamole_user
          POSTGRES_PASSWORD_FILE: /run/secrets/guac_postgres_password # Senha do usuário DB
        volumes:
          - /opt/your-stack-name/guac-db-data:/var/lib/postgresql/data
          - /opt/your-stack-name/guac-initdb:/docker-entrypoint-initdb.d # Inicialização do schema
        secrets:
          - guac_postgres_password
        networks:
          - your_stack_net

      # Serviço Guacamole
      guacamole:
        image: guacamole/guacamole:latest
        deploy:
          replicas: 1
          restart_policy:
            condition: on-failure
        environment:
          GUAC_HOME: /opt/guacamole/home
          # Variável de ambiente para a senha de bind LDAP, lida do Docker Secret
          LDAP_SEARCH_BIND_PASSWORD: /run/secrets/guac_ldap_bind_password
        volumes:
          - /opt/your-stack-name/guac-conf:/opt/guacamole/home # Arquivo guacamole.properties
          - /opt/your-stack-name/guac-drive:/guacd-drive
          - /opt/your-stack-name/guac-recordings:/recordings
        secrets:
          - guac_ldap_bind_password
        networks:
          - your_stack_net
        ports: # Descomente e ajuste as portas se não usar proxy reverso
          - "8081:8080"

      # Serviço Portainer
      portainer:
        image: portainer/portainer-ce:latest
        command: -H unix:///var/run/docker.sock --tlsskipverify --admin-password-file /run/secrets/portainer_admin_password
        deploy:
          replicas: 1
          restart_policy:
            condition: on-failure
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /opt/your-stack-name/portainer-data:/data
        secrets:
          - portainer_admin_password
        networks:
          - your_stack_net
        ports: # Descomente e ajuste as portas se não usar proxy reverso
          - "9000:9000"

    secrets:
      vw_admin_token:
        external: true
      guac_postgres_password:
        external: true
      guac_ldap_bind_password:
        external: true
      portainer_admin_password:
        external: true

    networks:
      your_stack_net:
        external: true
    ```
    **Nota:** As seções `ports` estão descomentadas neste exemplo para facilitar o acesso direto. Se você for usar um proxy reverso (Seção 12), pode manter as portas comentadas no `docker-compose.yml` e o proxy fará o roteamento.

---

## 9. Configurações Específicas do Guacamole (Exemplo)

### 9.1. Configuração do `guacamole.properties`

1.  **Criar o arquivo de configuração do Guacamole:**
    Este arquivo será montado no container do Guacamole.
    ```bash
    cat > /opt/your-stack-name/guac-conf/guacamole.properties << 'EOF'
    # PostgreSQL Connection
    postgresql-hostname: guacamole-db
    postgresql-port: 5432
    postgresql-database: guacamole_db
    postgresql-username: guacamole_user

    # LDAP Configuration (AJUSTE CONFORME SEU AMBIENTE LDAP)
    ldap-hostname: ldap.yourcompany.com
    ldap-port: 389
    ldap-user-base-dn: OU=Users,DC=yourcompany,DC=com
    ldap-username-attribute: sAMAccountName
    ldap-group-base-dn: OU=Groups,DC=yourcompany,DC=com
    ldap-search-bind-dn: CN=ServiceUser,OU=ServiceAccounts,DC=yourcompany,DC=com
    # A senha é fornecida via variável de ambiente do Docker Secret
    ldap-search-bind-password: ${LDAP_SEARCH_BIND_PASSWORD}
    ldap-user-attributes: displayName,mail,memberOf

    # Recording Configuration
    recording-path: /recordings
    EOF
    ```
    **IMPORTANTE:**
    *   Substitua `ldap.yourcompany.com`, `OU=Users,DC=yourcompany,DC=com`, `OU=Groups,DC=yourcompany,DC=com`, `CN=ServiceUser,OU=ServiceAccounts,DC=yourcompany,DC=com` pelos valores reais do seu ambiente LDAP.
    *   Verifique se a montagem do volume para `/opt/guacamole/home` está correta no `docker-compose.yml` para o serviço `guacamole`.

---

## 10. Deploy do Stack no Docker Swarm

1.  **Navegar para o diretório do stack:**
    ```bash
    cd /opt/your-stack-name
    ```

2.  **Realizar o deploy do stack:**
    ```bash
    docker stack deploy -c docker-compose.yml your-stack-name
    ```
    **Nota:** `your-stack-name` deve corresponder ao nome que você deseja dar ao seu stack.

3.  **Verificar o status dos serviços do stack:**
    ```bash
    docker stack services your-stack-name
    ```

4.  **Acompanhar os logs de um serviço específico (exemplo):**
    ```bash
    docker service logs your-stack-name_vaultwarden
    ```

---

## 11. Verificação e Monitoramento dos Serviços

1.  **Listar os containers e tarefas do stack:**
    ```bash
    docker stack ps your-stack-name
    ```

2.  **Verificar o status geral dos containers em execução:**
    ```bash
    docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
    ```

3.  **Visualizar logs em tempo real para depuração:**
    ```bash
    docker service logs -f your-stack-name_guacamole
    docker service logs -f your-stack-name_vaultwarden
    docker service logs -f your-stack-name_portainer
    ```

---

## 12. Configuração de Proxy Reverso (Nginx - Opcional)

A utilização de um proxy reverso (ex: Nginx) é recomendada para roteamento de tráfego baseado em nome de domínio e para facilitação da configuração de SSL/HTTPS.

### 12.1. Instalação do Nginx

*   **Ubuntu/Debian:**
    ```bash
    sudo apt install -y nginx
    ```

*   **CentOS/RHEL/Fedora:**
    ```bash
    sudo dnf install -y nginx
    ```

*   **openSUSE:**
    ```bash
    sudo zypper install -y nginx
    ```

### 12.2. Configuração do Nginx

1.  **Criar o arquivo de configuração do Nginx:**
    ```bash
    sudo nano /etc/nginx/sites-available/your-stack-nginx-config
    ```
    Cole o conteúdo abaixo, ajustando os `server_name` para os seus domínios e as portas `proxy_pass` conforme configurado no `docker-compose.yml`.

    ```nginx
    server {
        listen 80;
        server_name vault.yourdomain.com; # Substitua pelo seu domínio
        
        location / {
            proxy_pass http://localhost:8080;  # Porta onde o Vaultwarden está exposto
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name guac.yourdomain.com; # Substitua pelo seu domínio
        
        location / {
            proxy_pass http://localhost:8081;  # Porta onde o Guacamole está exposto
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 80;
        server_name portainer.yourdomain.com; # Substitua pelo seu domínio
        
        location / {
            proxy_pass http://localhost:9000;  # Porta onde o Portainer está exposto
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
    **SEGURANÇA:** Para um ambiente de produção, é **fundamental** configurar SSL/HTTPS. Isso envolve adicionar blocos `listen 443 ssl;`, configurar certificados (ex: via Let's Encrypt com Certbot) e redirecionar o tráfego HTTP para HTTPS.

2.  **Ativar a configuração do Nginx:**
    ```bash
    sudo ln -s /etc/nginx/sites-available/your-stack-nginx-config /etc/nginx/sites-enabled/
    ```

3.  **Testar a sintaxe da configuração do Nginx:**
    ```bash
    sudo nginx -t
    ```

4.  **Reiniciar e habilitar o serviço Nginx:**
    ```bash
    sudo systemctl restart nginx
    sudo systemctl enable nginx
    ```

---

## 13. Primeiros Acessos aos Serviços

*   **Vaultwarden (exemplo)**
    *   URL: `http://<YOUR_SERVER_IP>:8080` ou `http://vault.yourdomain.com`
    *   Crie uma conta de administrador na primeira execução. O token de administração (armazenado no secret `vw_admin_token`) será necessário para configurações avançadas.

*   **Guacamole (exemplo)**
    *   URL: `http://<YOUR_SERVER_IP>:8081/guacamole` ou `http://guac.yourdomain.com/guacamole`
    *   Credenciais iniciais: `guacadmin` / `guacadmin`
    *   **Altere a senha padrão imediatamente** após o primeiro login.

*   **Portainer (exemplo)**
    *   URL: `http://<YOUR_SERVER_IP>:9000` ou `http://portainer.yourdomain.com`
    *   A senha de administrador será definida na primeira vez que você acessar a interface web, ou configurada via o secret `portainer_admin_password` conforme o `docker-compose.yml`.

---

## 14. Comandos Essenciais de Manutenção

*   **Remover o stack (parar e remover todos os serviços e recursos associados):**
    ```bash
    docker stack rm your-stack-name
    ```

*   **Realizar redeploy do stack (após alterações no `docker-compose.yml`):**
    ```bash
    docker stack deploy -c docker-compose.yml your-stack-name
    ```

*   **Verificar o uso de espaço em disco do Docker:**
    ```bash
    docker system df
    ```

*   **Limpar imagens não utilizadas, containers parados e volumes não referenciados:**
    ```bash
    docker system prune -a
    ```
    **CUIDADO:** Este comando remove *todos* os recursos não utilizados. Use com cautela em produção.

*   **Backup dos dados persistentes (exemplo):**
    ```bash
    sudo tar -czf your-stack-name-backup-$(date +%Y%m%d).tar.gz -C /opt your-stack-name
    ```

*   **Visualizar logs de um serviço em tempo real:**
    ```bash
    docker service logs -f your-stack-name_[service_name]
    # Exemplo: docker service logs -f your-stack-name_guacamole
    ```

---

## 15. Resolução de Problemas Comuns

*   **Serviços não iniciam:** Verifique os logs do serviço com `docker service logs your-stack-name_[service_name]`.
*   **Erro de conexão LDAP (Guacamole):**
    *   Confirme a conectividade de rede com o servidor LDAP (`ping ldap.yourcompany.com`).
    *   Verifique a exatidão do `ldap-search-bind-dn` e da senha no secret `guac_ldap_bind_password`.
    *   Analise os logs do serviço `guacamole` para mensagens de erro específicas.
*   **PostgreSQL não conecta (Guacamole):** Verifique se o secret `guac_postgres_password` foi criado e referenciado corretamente no `docker-compose.yml`.
*   **Volumes não persistem dados:** Confirme as permissões dos diretórios em `/opt/your-stack-name`.
*   **Acesso via Nginx não funciona:**
    *   Verifique os logs de erro do Nginx (`sudo tail -f /var/log/nginx/error.log`).
    *   Confirme que as portas dos serviços estão publicadas no `docker-compose.yml`.
    *   Verifique a resolução de DNS para os seus domínios.
