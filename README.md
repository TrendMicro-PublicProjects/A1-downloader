# A1-downloader
Baixar instalador Apex (Windows) via Script + instalação automática


### Objetivo
Automatizar o download e instalação do agente Trend Micro Apex One em endpoints via script PowerShell e distribuído por GPO.
Quando instalado dessa forma, após o endpoint comunicar com o V1, ele atualiza automaticamente as engines e consequentemente baixa o XBC (Basecamp). Isso facilita a distribuição para os clientes, evitando que precisem baixar o agente manualmente toda vez.

### Pré-requisitos
- Servidor de arquivos acessível por todos os endpoints (caso queira hospedar o script).
- Permissão de administrador nos endpoints.
- Opicional: GPO criada para execução de scripts no Startup ou Logon.
- Se for executar direto no endpoint, o PowerShell precisa estar habilitado e com execução de scripts permitida (Set-ExecutionPolicy RemoteSigned).

### Estrutura do Script
  1. #### Configurações

  $URL = "https://xxxxx.manage.trendmicro.com/officescan/download/agent_cloud_x64_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.msi"
  $TempFolder = Join-Path $env:TEMP "Install_Trend"
  $LogFolder = "C:\ProgramData\Install_Trend"
  $LogPath = Join-Path $LogFolder "Install_Log.txt"
  $URL: link direto para o MSI da Trend Micro. 

Essa URL é encontrada em Standard Endpoint Protection > Directories > Product Servers

Para encontrar o ID do MSI respectivo ao tenant, basta fazer um download normal do agente via Endpoint Inventory com o Endpoint Group Manager correspondente. Em uma das pastas dentro de packages do pacote baixado do vision one, vai ter uma com MSI agent_cloud_x64_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx. É exatamente o mesmo ID que precisa informar no script.

$TempFolder: pasta temporária local para download do MSI.

$LogFolder e $LogPath: local para armazenar logs da instalação.

2. #### Função de verificação da instalação


function IsInstalled {
    $ProductName = "Trend Micro Apex One Security Agent"
    $products = Get-WmiObject -Class Win32_Product | Where-Object { $_.Name -eq $ProductName }
    if ($null -eq $products) { 
        return $false 
    } else {
        return $true
    }
}
Verifica se o produto já está instalado nos endpoints.

Evita downloads e reinstalações desnecessárias.

3. #### Função de log

function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "$timestamp - $Message"
    $line | Out-File -FilePath $LogPath -Encoding Default -Append
}
Grava eventos da execução, download e instalação em um arquivo legível pelo Notepad antigo.

4. #### Fluxo principal
Criação das pastas temporárias e de log, caso não existam.

Verificação se o produto já está instalado (IsInstalled).

Se sim, registra no log e finaliza o script.

Se não estiver instalado:

Baixa o MSI da Trend Micro na pasta temporária.

Instala silenciosamente usando msiexec com /qn /norestart.

Registra conclusão no log.

Remove a pasta temporária após a instalação.

5. #### Configuração da GPO para criar a Scheduled Task
- Abra o Group Policy Management Console (gpmc.msc) no servidor de domínio.

- Criar nova GPO:
  - Clique com o botão direito na OU que contém os computadores alvo → Create a GPO in this domain, and Link it here…
  - Nome sugerido: Instalar Trend Micro via Scheduled Task.
  - Editar a GPO:
    - Clique com o botão direito na GPO criada → Edit.
      - Navegue até:
        - Computer Configuration → Preferences → Control Panel Settings → Scheduled Tasks
        - Criar uma nova Scheduled Task:
        - Clique com o botão direito em Scheduled Tasks → New → Scheduled Task (At least Windows 
  
       
        - Configurações da Task:
          | Configuração  | Valor         |
          | ------------- | ------------- |
          | Action  | Update  |
          | Task Name  | Install Apex  |
          | Run only when user is logged on  | Yes  |
          | When running the task, use the following user account  | NT AUTHORITY\System  |
          | Run with highest privileges  | HighestAvailable  |
          | Hidden  | Yes  |
          | Enabled  | Yes  |

        - Triggers:

          - Clique em Triggers → New…
          - Begin the task: At startup
          - Enabled: Yes
          - Actions:
          - Clique em Actions → New…
          - Action: Start a program
          - Program/script: powershell.exe
          - Arguments:
            ``` -ExecutionPolicy Bypass -File "\\SERVIDOR_ARQUIVO\Install_Trend\install_apex.ps1" ```

  - Finalize e aplique a GPO:
    - Confirme as configurações e feche o editor da GPO.
    - Ao aplicar, cada endpoint na OU receberá automaticamente a Scheduled Task que executa o script no startup.

  - Teste em um endpoint:
    - Reinicie o computador.
    - Verifique o log em:
      ``` C:\ProgramData\Install_Trend\Install_Log.txt ```
      
6. Observações importantes
- O script usa TLS 1.2 para o download seguro.
- É idempotente: evita reinstalar se já houver uma instalação.
- Mantém logs detalhados para troubleshooting.
- Limpa a pasta temporária após a instalação para evitar lixo no endpoint.
- É primordial reiniciar o endpoint para ter efeito, não coloquei vai script pois isso varia de cliente para cliente como vão seguir.

7. Script
``` # ===== CONFIGURAÇÕES =====
#substituir pelo ID do cliente
$URL = "https://vfn2wr.manage.trendmicro.com/officescan/download/agent_cloud_x64_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.msi"

# Pasta temporária para download
$TempFolder = Join-Path $env:TEMP "Install_Trend"

# Pasta e arquivo de log em ProgramData
$LogFolder = "C:\ProgramData\Install_Trend"
if (-not (Test-Path $LogFolder)) {
    New-Item -ItemType Directory -Path $LogFolder | Out-Null
}
$LogPath = Join-Path $LogFolder "Install_Log.txt"

# Criar pasta temporária
if (-not (Test-Path $TempFolder)) {
    New-Item -ItemType Directory -Path $TempFolder | Out-Null
}

# Caminho completo do MSI
$FileName = Split-Path $URL -Leaf
$LocalMSI = Join-Path $TempFolder $FileName

# ===== FUNÇÃO PARA VERIFICAR INSTALAÇÃO =====
function IsInstalled {
    $ProductName = "Trend Micro Apex One Security Agent"
    $products = Get-WmiObject -Class Win32_Product | Where-Object { $_.Name -eq $ProductName }

    # Força que seja sempre um array
    if ($null -eq $products) { 
        return $false 
    } else {
        return $true
    }
}

# ===== FUNÇÃO DE LOG =====
function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $line = "$timestamp - $Message"

    # Grava em ANSI (Default), que o Notepad antigo lê corretamente
    $line | Out-File -FilePath $LogPath -Encoding Default -Append
}



# ===== INÍCIO EXECUÇÃO =====
Write-Log "==================== INÍCIO EXECUÇÃO ===================="

# ===== VERIFICA SE JÁ ESTÁ INSTALADO ANTES DE BAIXAR =====
if (IsInstalled) {
    Write-Log "Produto já instalado. Nenhuma ação necessária."
    Write-Log "==================== FIM EXECUÇÃO ====================`n"
    exit
}

# ===== DOWNLOAD DO MSI =====
if (-not (Test-Path $LocalMSI)) {
    Write-Log "MSI não encontrado localmente. Baixando..."
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    $wc = New-Object System.Net.WebClient
    $wc.DownloadFile($URL, $LocalMSI)
    Write-Log "Download concluído."
} else {
    Write-Log "MSI já existe na pasta temporária."
}

# ===== INSTALAÇÃO SILENCIOSA =====
Write-Log "Produto não encontrado. Iniciando instalação..."
Start-Process "msiexec.exe" -ArgumentList "/i `"$LocalMSI`" /qn /norestart /l*v `"$LogPath`"" -Wait
Write-Log "Instalação concluída."

# ===== EXECUTAR PccNTMon PARA INICIALIZAR/ATUALIZAR AGENTE =====
$AgentPath = "C:\Program Files (x86)\Trend Micro\Security Agent"
$PccExe = Join-Path $AgentPath "PccNTMon.exe"

if (Test-Path $PccExe) {
    Write-Log "Executando PccNTMon.exe -us para iniciar/atualizar o agente..."
    Start-Process -FilePath $PccExe -ArgumentList "-us" -WorkingDirectory $AgentPath -Wait
    Write-Log "PccNTMon.exe executado com sucesso."
} else {
    Write-Log "PccNTMon.exe não encontrado no caminho $AgentPath"
}


# ===== LIMPAR PASTA TEMPORÁRIA =====
if (Test-Path $TempFolder) {
    Remove-Item -Path $TempFolder -Recurse -Force
    Write-Log "Pasta temporária removida."
}

Write-Log "==================== FIM EXECUÇÃO ====================`n`n`n"
```
