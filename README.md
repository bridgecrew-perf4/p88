# p88
É como um piano de 88 teclas,
onde cada nota orquestrada soam
em harmonia. :joy: :musical_keyboard:

O objetivo desse projeto é trazer qualidade de vida
para os *devs* que tem vários projetos e não aguentam mais
ficar configurando os apps na mão. :notes:

Utilizando o conceito de *infra as code*
cada *app* vai ter seus recursos como código,
tendo controle do que tem em cada VM e possibilitando
recriar os ambientes com pouco esforço usando
[terraform](https://www.terraform.io)
e [ansible](https://www.ansible.com).

O provedor que será utilizado é o
[Digital Ocean](https://www.digitalocean.com),
com máquinas a partir de $5/mês. :moneybag:

## Requisitos
Para rodar esse projeto, vamos usar o 
[docker](https://docker.com)
com a imagem base `debian stable-slim`. :whale2:

Vamos precisar de uma conta no 
[Digital Ocean](https://www.digitalocean.com)
e criar um *token* e um *fingerprint* para
interagir via API com o [terraform](https://www.terraform.io).

## Apps
Os apps são as divisões de aplicativos/projetos
que deseja organizar, como um contexto.
Podendo separar por ambientes como:
`development`, `staging` e `production`.

No exemplo, vamos ter o `project-1`,
com dois ambientes `prod` e `staging`
e `project-2` com apenas `development`.

Veja como fica a divisão de pastas dentro do p88:
```bash
apps
├── project-1
│   ├── prod
│   │   ├── ansible
│   │   │   ├── ansible.cfg
│   │   │   ├── config.yml
│   │   │   └── playbook.yml
│   │   └── terraform
│   │       ├── locals.tf
│   │       ├── main.tf
│   │       └── variables.tf
│   └── staging
│       ├── ansible
│       │   ├── ansible.cfg
│       │   ├── config.yml
│       │   └── playbook.yml
│       └── terraform
│           ├── locals.tf
│           ├── main.tf
│           └── variables.tf
└── project-2
    └── development
        ├── ansible
        │   ├── ansible.cfg
        │   ├── config.yml
        │   └── playbook.yml
        └── terraform
            ├── locals.tf
            ├── main.tf
            └── variables.tf
```

Como exemplo ilustrativo o `project-1` será um deploy
de uma imagem do `nginx:stable-alpine`,
com a configuração de domínio.

No segundo `project-2`, será um ambiente de desenvolvimento
com a imagem do `sapk/cloud9:golang-alpine`.

### Terraform
Para criar a infraestrutura vamos utilizar o [terraform](https://www.terraform.io).

No arquivo `locals.tf` são as variáveis que
serão utilizadas para criação da infra:
```
locals {
	env = "production"

	project = "project-1"
	project_purpose = "Web Application"
	domain = "yourdomain.com.br"
	droplet_image = "ubuntu-18-04-x64"
	droplet_region = "nyc3"
	droplet_size = "s-1vcpu-1gb"
	droplet_tags = ["nginx", "docker"]
}
```

No `main.tf`, são as definições dos recursos que serão criados:
```
provider "digitalocean" {
  token = var.do_token
  version = "1.12.0"
}

module "droplet" {
  source = "/p88/lib/terraform/modules/droplet"

  name = local.project
  env = local.env
  region = "${local.droplet_region}"
  image = "${local.droplet_image}"
  size = "${local.droplet_size}"
  tags = "${local.droplet_tags}"
  user = "${local.droplet_user}"
  user_pvt_key = "${var.pvt_key}"
  ssh_fingerprint = "${var.ssh_fingerprint}"
}

module "domain" {
  source = "/p88/lib/terraform/modules/domain"

  domain = local.domain
  ipv4 = module.droplet.ipv4
}

module "project" {
  source = "/p88/lib/terraform/modules/project"

  name = local.project
  env = local.env
  purpose = local.project_purpose
  resources   = [
    "${module.droplet.urn}",
    "${module.domain.urn}",
  ]
}
```

Nesse exemplo, criamos uma *droplet*, um *domain*
e relacionamos os dois a um *projeto* no
[Digital Ocean](https://www.digitalocean.com).

### Ansible
Vamos usar o [ansible](https://www.ansible.com)
para configurar as *droplets*.

Exemplo do `config.yml`:
```yml
all:
  hosts:
    104.236.36.162
  vars:
    app: project-1
    env: production
    image: nginx:stable-alpine
```

Arquivo `playbook.yml` com exemplo de task:
```yml
tasks:
  - name: Create container
    tags: app
    docker_container:
      name: "{{ app }}"
      image: "{{ image }}"
      state: started
      ports:
        - 80:80
```

Nesse exemplo do `project-1` foram instaladas duas *roles*
do ansible [docker](https://github.com/geerlingguy/ansible-role-docker)
e [pip](https://github.com/geerlingguy/ansible-role-pip) para configurar a *droplet*.
Posteriormente baixando e criando o container
com a imagem `nginx:stable-alpine` disponível no
[Docker Hub](https://hub.docker.com/_/nginx).

## Rodando o Projeto
Para rodar o projeto vamos precisar expotar o `token` e o
`fingerprint`criado no [Digital Ocean](https://www.digitalocean.com)
como variáveis de ambiente:
```bash
export do_token=<coloque-seu-token-aqui>
export do_fingerprint=<coloque-seu-fingerprint-aqui>
```

Depois vamos construir a imagem docker do p88 com o terraform e
[ansible](https://www.ansible.com)
para rodar os provisionamentos:
```bash
make build
```

> Feito isso já é possível provisionar o ambiente usando terraform e
[ansible](https://www.ansible.com).

### Criando a infraestrutura com Terraform
Os comandos a baixo utilizam o terraform
para gerenciar a infraestrutura.

Para criar os recursos, execute:
```
make terraform-apply app=project-1 env=prod
```

Abrindo o [Digital Ocean](https://www.digitalocean.com),
podemos visualizar os recursos criados:
![alt text](docs/do-project-1.png)

Se precisar destruir os recursos, execute:
```
make terraform-destroy app=project-1 env=prod
```

### Provisionando com o Ansible
Com a *droplet* criada vamos provisionar usando
[ansible](https://www.ansible.com).

Para provisionar todo o ambiente do projeto, execute:
```
make ansible app=project-1 env=prod
```

Acessando o endereço ip da *droplet* já é possível acessar a aplicação:
![alt text](docs/project-1.png)

Se precisar especificar uma task, utilize a variável `tags`,
conforme o exemplo a baixo:
```bash
make ansible app=project-1 env=prod tags=app
```

## Testando
Para verificar o funcionamento da * droplet* , entre via `ssh <container-ip> -l root`
e liste os container em execução usando `docker ps`:
```bash
> ssh 45.55.35.147 -l root
root@project-1-production:~# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                NAMES
863f134396b5   nginx:stable-alpine   "/docker-entrypoint.…"   47 seconds ago   Up 45 seconds   0.0.0.0:80->80/tcp   project-1
```

Acessando o ambiente de dessenvolimento do `project-2`:
![alt text](docs/project-2.png)
