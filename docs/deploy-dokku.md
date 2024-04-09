# Deploy do projeto

O deployment do projeto é realizado através do [Dokku](https://dokku.com/), que
facilita a gestão de containers Docker e outros serviços, como bancos de dados,
servidores de cache etc.

## Configurações iniciais do servidor

Para a segurança do servidor, é importante que você:

- Utilize apenas chaves SSH para login (copie suas chaves usando o comando
  `ssh-copy-id` ou coloque-as em `/root/.ssh/authorized_keys`)
- Remova a opção de login via SSH usando senha (altere a configuração editando
  o arquivo `/etc/ssh/ssh_config` e depois execude `service ssh restart`)
- Se possível, crie um outro usuário que possua acesso via `sudo` e desabilite
  o login do usuário root via SSH (edite o arquivo `/etc/ssh/ssh_config`)


## Instalação de pacotes básicos de sistema

```shell
apt update && apt upgrade -y && apt install -y wget
apt install -y $(wget -O - https://raw.githubusercontent.com/turicas/dotfiles/main/server-apt-packages.txt)
apt clean
```


## Instalação do Docker

```bash
# Adicionar a chave GPG do Docker:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adicionar o repositório Docker às fontes do APT:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Instalar o Docker:
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Para testar:
docker run --rm hello-world
```

## Instalação e configuração do Dokku

```bash
# Pacotes necessários
sudo apt-get update -qq >/dev/null
sudo apt-get -qq -y --no-install-recommends install apt-transport-https

# Chave GPG do Dokku
wget -qO- https://packagecloud.io/dokku/dokku/gpgkey | sudo tee /etc/apt/trusted.gpg.d/dokku.asc

# Determinar o nome do sistema operacional
DISTRO="$(awk -F= '$1=="ID" { print tolower($2) ;}' /etc/os-release)"
OS_ID="$(awk -F= '$1=="VERSION_CODENAME" { print tolower($2) ;}' /etc/os-release)"

# Adicionar o repositório Dokku às fontes do APT
echo "deb https://packagecloud.io/dokku/dokku/${DISTRO}/ ${OS_ID} main" | sudo tee /etc/apt/sources.list.d/dokku.list
sudo apt-get update -qq >/dev/null

# Instalar o Dokku
sudo apt-get -qq -y install dokku
sudo dokku plugin:install-dependencies --core

# Configurações do Dokku
dokku config:set --global DOKKU_RM_CONTAINER=1  # don't keep `run` containers around

# Dokku plugins
sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
sudo dokku plugin:install https://github.com/dokku/dokku-maintenance.git maintenance
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git postgres
sudo dokku plugin:install https://github.com/dokku/dokku-redis.git redis

dokku letsencrypt:cron-job --add
```

Caso você não tenha colocado sua chave SSH do Dokku durante a execução do
comando `apt install dokku`, você precisa adicioná-la com o seguinte comando:

```shell
dokku ssh-keys:add admin path/to/pub_key
```

> Nota: o arquivo deve ter apenas uma chave SSH (caso o arquivo fornecido na
> interface de configuração tenha mais de uma chave, a configuração precisará
> ser feita manualmente, com o comando acima).

Dessa forma, o usuário que possuir essa chave poderá fazer deployments via git
nesse servidor.

> **Importante:** após a instalação do Dokku, caso não tenha respondido às
> perguntas durante a instalação, será necessário acessar a interface Web
> temporária para finalizar configuração (entre em `http://ip-do-servidor/` em
> seu navegador).

Ao finalizar a instalação do Dokku e acessar http://ip-do-servidor/ você deverá
ver a mensagem "Welcome to nginx!".


## Instalação da aplicação

Antes de criar a aplicação no Dokku será necessário configurar algumas
variáveis no shell, para que elas sejam adicionadas às variáveis de ambiente do
app (assim, o Dokku irá sempre carregá-las toda vez que o app for iniciado e
não precisaremos armazenar senhas em arquivos no repositório).

```shell
export ADMIN_EMAIL="admin@pythonic.cafe"
export APP_NAME="alertflow"
export DOMAIN="alertflow.pythonic.cafe"
export PG_NAME="pg_$APP_NAME"
export REDIS_NAME="redis_$APP_NAME"
```

Depois que as variáveis foram definidas, podemos criar o app e fazer as configurações iniciais:

```shell
dokku apps:create $APP_NAME
dokku checks:disable $APP_NAME
dokku letsencrypt:set $APP_NAME email $ADMIN_EMAIL
dokku domains:set $APP_NAME $DOMAIN
```

### PostgreSQL

```shell
dokku postgres:create $PG_NAME
dokku postgres:link $PG_NAME $APP_NAME
```

### Parse PostgreSQL URL

```shell
parse_url() {
  # extract the protocol
  proto="$(echo $1 | grep :// | sed -e's,^\(.*://\).*,\1,g')"
  # remove the protocol
  url="$(echo ${1/$proto/})"
  # extract the user (if any)
  user="$(echo $url | grep @ | cut -d@ -f1)"

  password="$(echo $user | grep : | cut -d: -f2)"
  user="$(echo $user | grep : | cut -d: -f1)"
  # extract the host and port
  hostport="$(echo ${url/$user@/} | cut -d/ -f1)"
  # by request host without port
  host="$(echo $url | grep @ | cut -d@ -f2)"
  host="$(echo $host | grep : | cut -d: -f1)"
  # by request - try to extract the port
  port="$(echo $hostport | sed -e 's,^.*:,:,g' -e 's,.*:\([0-9]*\).*,\1,g' -e 's,[^0-9],,g')"
  # extract the path (if any)
  path="$(echo $url | grep / | cut -d/ -f2-)"

  if [[ "$2" == "database" ]];
  then
      echo $path
  elif [[ "$2" == "host" ]];
  then
      echo $host
  elif [[ "$2" == "password" ]];
  then
      echo $password
  elif [[ "$2" == "user" ]];
  then
      echo $user
  elif [[ "$2" == "port" ]];
  then
      echo $port
  fi
}

export DATABASE_URL=$(dokku postgres:info pg_alertflow | awk '/Dsn/ {print $2}')
export psql_host=$(parse_url $DATABASE_URL host)
export psql_port=$(parse_url $DATABASE_URL port)
export psql_database=$(parse_url $DATABASE_URL database)
export psql_user=$(parse_url $DATABASE_URL user)
export psql_password=$(parse_url $DATABASE_URL password)
```

### Redis

```shell
dokku redis:create $REDIS_NAME
dokku redis:link $REDIS_NAME $APP_NAME
```

### Variáveis de ambiente

```shell
dokku config:set --no-restart $APP_NAME _AIRFLOW_WWW_USER_USERNAME=airflow
dokku config:set --no-restart $APP_NAME _AIRFLOW_WWW_USER_EMAIL=mail@example.com
dokku config:set --no-restart $APP_NAME _AIRFLOW_WWW_USER_FIRST_NAME=Airflow
dokku config:set --no-restart $APP_NAME _AIRFLOW_WWW_USER_LAST_NAME=User
dokku config:set --no-restart $APP_NAME EMAIL_MAIN=mail@example.com
dokku config:set --no-restart $APP_NAME PSQL_URI_MAIN=$DATABASE_URL
dokku config:set --no-restart $APP_NAME PSQL_USER_MAIN=$psql_user
dokku config:set --no-restart $APP_NAME PSQL_PASSWORD_MAIN=$psql_password
dokku config:set --no-restart $APP_NAME PSQL_HOST_MAIN=$psql_host
dokku config:set --no-restart $APP_NAME PSQL_PORT_MAIN=$psql_port
dokku config:set --no-restart $APP_NAME PSQL_DB_MAIN=$psql_database
dokku config:set --no-restart $APP_NAME CDSAPI_KEY=user-id:key
dokku config:set --no-restart $APP_NAME AIRFLOW_PSQL_PASSWORD_MAIN=$psql_password
dokku config:set --no-restart $APP_NAME AIRFLOW_PSQL_PORT_MAIN=$psql_port
dokku config:set --no-restart $APP_NAME AIRFLOW_PSQL_USER_MAIN=$psql_user
dokku config:set --no-restart $APP_NAME AIRFLOW_PSQL_HOST_MAIN=$psql_host
dokku config:set --no-restart $APP_NAME AIRFLOW_PSQL_DB_MAIN=$psql_database
dokku config:set --no-restart $APP_NAME EPISCANNER_HOST_DATA="/opt/airflow/episcanner_data"
```

### Dockerfile path

É necessário indicar o caminho do Dockerfile, que não está na raiz do projeto.

```shell
dokku builder-dockerfile:set $APP_NAME dockerfile-path ./docker/Dockerfile
```

### Dokku Docker build options

```shell
dokku docker-options:add $APP_NAME build '--build-arg HOST_UID=1000'
dokku docker-options:add $APP_NAME build '--build-arg HOST_GID=0'
```

### Proxy/Ports

```shell
dokku ports:set $APP_NAME http:80:8080
```

Com o app criado e configurado, agora precisamos fazer o primeiro deployment,
para então finalizar a configuração com a criação do certificado SSL via Let's
Encrypt (precisa ser feito nessa ordem).

Em sua **máquina local**, vá até a pasta do repositório e execute:

```shell
# Troque <server-ip> pelo IP do servidor e <app-name> pelo valor colocado na
# variável $APP_NAME definida no servidor
# ATENÇÃO: execute esses comandos fora do servidor (em sua máquina local)
git remote add dokku dokku@<server-ip>:<app-name>
git checkout main
git push dokku main
```

Para finalizar as configurações iniciais, conecte novamente no servidor e
execute:

```shell
dokku ps:scale web=1
dokku ps:scale scheduler=1
dokku ps:scale worker=1
dokku ps:scale triggerer=1
dokku ps:scale flower=1
dokku letsencrypt:enable $APP_NAME
```
