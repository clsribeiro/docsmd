# Documentação Técnica Completa: Implantação do Assistente de IA no Terminal (ShellGPT + Groq)

Este documento constitui um guia definitivo, unificado e autossuficiente para instalação, configuração, personalização e replicação ágil de um ecossistema de Inteligência Artificial integrado diretamente à linha de comando (Shell Linux). 

A arquitetura descrita combina a flexibilidade do **ShellGPT (sgpt)** com os tempos de resposta de ultra-baixa latência da API do **Groq**, utilizando isolamento de processos por abas de terminal e persistência lógica customizada para projetos de longa duração.

---

## 1. Instalação do Ecossistema e Dependências

Para mitigar conflitos com as dependências do ecossistema Python global ou do gerenciador de pacotes nativo do sistema operacional (`apt`), a instalação do utilitário principal é realizada obrigatoriamente através do `pipx` (executável de aplicações Python em ambientes isolados).

Execute os comandos abaixo na ordem estrita indicada:

```bash
sudo apt update
```

```bash
sudo apt install python3 python3-pip pipx -y
```

```bash
pipx ensurepath
```

> **CRÍTICO:** Após a execução do comando `pipx ensurepath`, você **deve** fechar a janela atual do seu terminal e abrir uma nova janela, ou forçar a releitura imediata do perfil executando o comando abaixo, antes de prosseguir com a instalação do ShellGPT:

```bash
source ~/.bashrc
```

```bash
pipx install shell-gpt
```

---

## 2. Configuração do Arquivo de Parâmetros Operacionais (`.sgptrc`)

O ShellGPT centraliza suas configurações de comportamento, endpoints e modelos em um arquivo de texto oculto chamado `.sgptrc`. Para desviar as requisições padrão da OpenAI e adotar o ecossistema de alto desempenho do Groq, este arquivo deve ser criado ou editado por completo.

* **Caminho Físico Exato:** 
```bash
nano ~/.config/shell_gpt/.sgptrc
```

Adicione as diretivas exatas listadas abaixo dentro do arquivo:

```ini
API_BASE_URL=https://api.groq.com/openai/v1
OPENAI_API_KEY=gsk_....17
DEFAULT_MODEL=llama-3.3-70b-versatile
```

---

## 3. Customização Integral do Arquivo de Perfil (`~/.bashrc`)

Para embutir os atalhos dinâmicos, gatilhos de destruição de arquivos temporários ao encerrar o terminal e a função inteligente de gerenciamento de projetos, o bloco de código abaixo deve ser inserido de forma contínua no final do arquivo de perfil do usuário (`~/.bashrc`).

Abra o arquivo executando:

```bash
nano ~/.bashrc
```

Navegue até a **última linha** do documento e cole o bloco unificado abaixo sem nenhuma alteração:

```bash
# =================================================================
# ENVIRONMENT AI SETUP (CONFIGURAÇÃO UNIFICADA DO ASSISTENTE DE IA)
# Expires/Review: 14/08/2026
# =================================================================

# --- PROVEDOR ATIVO (SINCRO COM DIRETIVAS OPENAI/GROQ) ---
export GROQ_API_KEY="COLE_A_CHAVE_AQUI"
export OPENAI_API_KEY="$GROQ_API_KEY"

# --- CORE ALIASES PARA PRODUTIVIDADE ---

# 1. Modo Chat / Conversa Contínua (Isolado e Dinâmico por Aba)
# Utiliza o Parent Process ID ($PPID) para criar uma sessão única e isolada por aba ativa do terminal
alias ai='sgpt --chat "zorin_session_${PPID}"'

# 2. Modo Assistente de Shell (Gera comandos Linux e oferece execução/explicação em tempo real)
alias aicmd='sgpt --shell'

# 3. Modo Código Puro (Retorna exclusivamente a sintaxe limpa do código pedido, sem introduções ou textos)
alias aiexp='sgpt --code'

# 4. Listar chats salvos
alias aichats="ls -la /home/administrador/.config/shell_gpt/groq_chat_cache/"

# 5. Rotina de Limpeza Automática para Sessões Voláteis (Gatilho de Saída)
# Elimina fisicamente do disco o cache gerado pela aba atual assim que a janela do terminal for encerrada
trap 'sgpt --chat "zorin_session_${PPID}" --delete > /dev/null 2>&1' EXIT

# 6. Função de Gerenciamento de Conversas Persistentes (Fluxo de Longo Prazo)
ai_persistente() {
    # Solicita o nome identificador do projeto/conversa
    read -p "Informe o nome dessa conversa/projeto: " session_name
    
    if [ -z "$session_name" ]; then
        echo -e "\033[01;31m[-] Nome inválido. Operação cancelada.\033[00m"
        return 1
    fi
    
    # Sanitiza a string removendo espaços e caracteres especiais para segurança do sistema de arquivos
    session_name=$(echo "$session_name" | tr ' ' '_' | tr -cd 'a-zA-Z0-9_-')
    
    # Garante a existência do diretório físico de cache utilizado pela API customizada
    mkdir -p "$HOME/.config/shell_gpt/groq_chat_cache"
    
    # Define a localização exata do arquivo de histórico correspondente à sessão
    local chat_file="$HOME/.config/shell_gpt/groq_chat_cache/$session_name"
    
    if [ -f "$chat_file" ]; then
        echo -e "\n\033[01;32m[!] Carregando contexto existente da sessão: $session_name\033[00m"
    else
        echo -e "\n\033[01;34m[+] Iniciando uma nova conversa persistentemente gravada: $session_name\033[00m"
        touch "$chat_file"
    fi
    
    # Recebe a requisição inicial do usuário
    read -p "O que deseja perguntar/fazer? " user_prompt
    
    if [ -n "$user_prompt" ]; then
        # Executa a chamada do sgpt vinculada ao nome da sessão
        sgpt --chat "$session_name" "$user_prompt"
        
        # Sobrescreve temporariamente o atalho local na aba para responder continuamente ao projeto
        alias ai="sgpt --chat $session_name"
        echo -e "\n\033[00;33mℹ Nota: Nesta aba, o comando 'ai' continuará focado em '$session_name'.\033[00m"
    fi
}
```

### Ativação Imediata das Configurações

Para carregar as novas funções, aliases e chaves de API exportadas diretamente na sessão ativa do seu shell sem precisar deslogar ou reiniciar o computador, execute obrigatoriamente:

```bash
source ~/.bashrc
```

---

## 4. Guia Rápido de Uso e Casos Práticos

* **Perguntas Rápidas (Modo Volátil):** Use `ai "sua pergunta"`. O histórico funciona apenas dentro da mesma aba e se apaga sozinho quando você fecha o terminal.
* **Comandos do Sistema (Modo Shell):** Use `aicmd "o que você quer fazer no linux"`. Ele gera o comando e pergunta se você quer executar `[E]`, descrever `[D]` ou abortar `[A]`.
* **Geração de Códigos Limpos (Modo Script):** Use `aiexp "escreva um script em python..."`. Ele retorna apenas o código seco, ideal para mandar direto para um arquivo usando `> arquivo.py`.
* **Projetos de Longa Duração (Modo Persistente):** Digite `ai_persistente` para criar ou retomar um histórico gravado de forma permanente no disco.