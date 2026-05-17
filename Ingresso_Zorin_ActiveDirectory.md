# Guia: Ingressando Zorin OS no Active Directory (AD)

Este documento contém o passo a passo manual, bibliotecas e comandos necessários para ingressar uma máquina Zorin OS em um domínio Active Directory (Windows Server). Esse processo alcança o mesmo resultado da opção "Usar o Active Directory" disponível na tela de instalação do Zorin OS.

## 1. Pré-requisitos

Para que a autenticação Kerberos funcione corretamente com o Active Directory, dois fatores são fundamentais: **Resolução de Nomes (DNS)** e **Sincronização de Tempo (NTP)**.

### 1.1. Configuração de DNS
A máquina Linux precisa resolver corretamente o nome do domínio e encontrar os Controladores de Domínio (Domain Controllers).
1. Abra as **Configurações de Rede** do Zorin OS.
2. Na sua conexão ativa (IPv4), desative o DNS Automático e defina o IP do seu servidor Active Directory (`10.111.126.15`) como o servidor DNS principal.
3. Teste a resolução de nomes via terminal:
   ```bash
   ping ad.eacri.com.br
   ```

### 1.2. Sincronização de Horário (NTP)
O horário do sistema Linux não pode divergir mais do que 5 minutos do Controlador de Domínio. É recomendável sincronizar diretamente com o AD.

Instale o `chrony` para gerenciamento de tempo:
```bash
sudo apt update
sudo apt install chrony
```
Edite o arquivo de configuração do chrony (`/etc/chrony/chrony.conf`):
```bash
sudo nano /etc/chrony/chrony.conf
```
Adicione o IP do seu AD (`10.111.126.15`) ao final do arquivo:
```text
server 10.111.126.15 iburst
```
Reinicie o serviço para aplicar a mudança:
```bash
sudo systemctl restart chrony
```

---

## 2. Pacotes e Bibliotecas Necessárias

O Zorin OS (sendo baseado no Ubuntu) utiliza a ferramenta `realmd` em conjunto com o `SSSD` para facilitar e gerenciar a integração com o AD.

Abra o terminal e instale todos os componentes necessários:
```bash
sudo apt update
sudo apt install realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin oddjob oddjob-mkhomedir packagekit
```

**Resumo das ferramentas:**
* **realmd**: Fornece uma maneira simples e padronizada de descobrir e ingressar em domínios de rede.
* **sssd** *(System Security Services Daemon)*: Gerencia a autenticação, acesso e fornece cache de credenciais para permitir logins mesmo quando a máquina estiver temporariamente offline.
* **adcli**: Ferramenta que gerencia a máquina e as contas dentro do próprio Active Directory.
* **oddjob-mkhomedir**: Responsável por criar a pasta base (`/home/usuario`) do usuário automaticamente no primeiro login.

---

## 3. Ingressando no Domínio

### 3.1. Descobrir o Domínio
Verifique se a máquina consegue enxergar o domínio corretamente executando o comando `discover`:

```bash
realm discover ad.eacri.com.br
```
A saída deve exibir os pacotes necessários e informar que o tipo de domínio é `active-directory`.

### 3.2. Ingressar (Join)
Para juntar a máquina ao domínio, utilize o usuário administrador `administrator`:

```bash
sudo realm join -U administrator ad.eacri.com.br --verbose
```
O terminal solicitará a senha do usuário `administrator`. O parâmetro `--verbose` é útil para identificar problemas, caso haja alguma falha de conexão.

Para validar se a máquina ingressou com sucesso, digite:
```bash
realm list
```

---

## 4. Configurações Pós-Ingresso e Ajustes Finos

### 4.1. Habilitar Criação Automática do Diretório Home
Para que a pasta `/home/nome_do_usuario` seja criada no momento do login inicial, você deve habilitar o módulo PAM `mkhomedir`:

```bash
sudo pam-auth-update --enable mkhomedir
```

### 4.2. Simplificar o Formato de Login (Opcional)
Por padrão, o SSSD exige que o usuário digite o formato completo (FQDN) no momento do login: `usuario@ad.eacri.com.br`. Para permitir o login apenas com o nome do `usuario`:

1. Edite o arquivo do SSSD:
   ```bash
   sudo nano /etc/sssd/sssd.conf
   ```
2. Modifique as seguintes diretivas (normalmente sob a seção `[domain/ad.eacri.com.br]`):
   ```ini
   use_fully_qualified_names = False
   fallback_homedir = /home/%u
   ```
3. Salve o arquivo (`Ctrl+O`, `Enter`, `Ctrl+X`) e reinicie o serviço SSSD:
   ```bash
   sudo systemctl restart sssd
   ```

### 4.3. Conceder Permissão de Root (Sudo) aos Administradores do AD (Opcional)
Se desejar que o grupo de "Domain Admins" do Windows tenha permissão administrativa completa (sudo) na máquina Linux:

1. Abra o arquivo sudoers com segurança:
   ```bash
   sudo visudo -f /etc/sudoers.d/domain_admins
   ```
2. Adicione a seguinte linha (utilize a barra invertida `\` para escapar o espaço no nome do grupo):
   ```text
   %domain\ admins ALL=(ALL) ALL
   ```
   *(Nota: Se você não modificou a opção `use_fully_qualified_names = False`, terá que escrever o grupo completo: `%domain\ admins@ad.eacri.com.br`).*

---

## 5. Testando a Integração

Antes de reiniciar, certifique-se de que a máquina consegue ler informações dos usuários do Active Directory:

```bash
id nome_do_usuario_do_AD
```
Se o terminal retornar os IDs (uid, gid) e os grupos da rede para esse usuário, a integração está perfeita.

---

## 6. Realizando o Login (GDM)

Após reiniciar o Zorin OS, na tela de bloqueio, selecione a opção:
**"Não está listado?"** ou **"Fazer login como outro usuário"**.

Digite o seu usuário do Active Directory e a respectiva senha. O sistema irá autenticar no Windows Server, criar o ambiente do perfil e inicializar a área de trabalho do Zorin OS.
