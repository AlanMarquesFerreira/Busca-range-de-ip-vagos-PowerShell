# Busca range de ip vagos / PowerShell
# Script PowerShell para escanear um range de IPs e gerar relatórios de IPs ativos e inativos

# Define o range de IPs inicial e final
$startIP = "192.168.50.2"
$endIP = "192.168.50.255"

# Converte IPs de string para número inteiro para facilitar a iteração
function Convert-IPToInt64 {
    param($ip)
    $octets = $ip -split "\."
    return [int64]([int64]$octets[0]*16777216 + [int64]$octets[1]*65536 + [int64]$octets[2]*256 + [int64]$octets[3])
}

# Converte número inteiro de volta para formato de IP
function Convert-Int64ToIP {
    param([int64]$int)
    return "$(($int -band 0xFF000000) -shr 24).$(($int -band 0x00FF0000) -shr 16).$(($int -band 0x0000FF00) -shr 8).$(($int -band 0x000000FF))"
}

# Converte o IP inicial e final em inteiros
$startInt = Convert-IPToInt64 $startIP
$endInt = Convert-IPToInt64 $endIP

# Inicializa listas para armazenar IPs ativos, inativos e os resultados completos
$activeIPs = @()
$inactiveIPs = @()
$resultsFull = @()

# Loop para testar a conexão com cada IP dentro do intervalo
for ($i = $startInt; $i -le $endInt; $i++) {
    $currentIP = Convert-Int64ToIP $i
    Write-Host ""
    Write-Host "Pingando ${currentIP} com 32 bytes de dados:" -ForegroundColor Cyan
    
    # Realiza o ping (Test-Connection) com 4 pacotes
    $results = Test-Connection -ComputerName $currentIP -Count 4 -ErrorAction SilentlyContinue
    
    if ($results) {
        # IP respondeu ao ping
        $activeIPs += $currentIP

        foreach ($reply in $results) {
            if ($reply.ResponseTime -lt 1) {
                $time = "<1ms"
            } else {
                $time = "$($reply.ResponseTime)ms"
            }
            Write-Host "Resposta de ${currentIP}: bytes=32 tempo=$time TTL=$($reply.ResponseTimeToLive)"
        }

        # Estatísticas de tempo de resposta
        $stats = $results | Measure-Object -Property ResponseTime -Average -Maximum -Minimum
        Write-Host ""
        Write-Host "Estatísticas do Ping para ${currentIP}:" -ForegroundColor Cyan
        Write-Host "    Pacotes: Enviados = 4, Recebidos = $($results.Count), Perdidos = $(4 - $results.Count) ($(100 * (4 - $results.Count)/4)% de perda),"
        Write-Host "Tempo aproximado de ida e volta em ms:" -NoNewline
        Write-Host "    Mínimo = $($stats.Minimum)ms, Máximo = $($stats.Maximum)ms, Média = $([math]::Round($stats.Average,1))ms"

        # Armazena resultado completo
        $resultsFull += [PSCustomObject]@{
            IP = $currentIP
            Status = "Ativo"
            Perdidos = 4 - $results.Count
            TTL = ($results | Select-Object -First 1).ResponseTimeToLive
            Min = $stats.Minimum
            Max = $stats.Maximum
            Media = [math]::Round($stats.Average, 1)
        }
    } else {
        # IP não respondeu
        $inactiveIPs += $currentIP
        Write-Host "Resposta de ${currentIP}: Host de destino inacessível."
        Write-Host ""
        Write-Host "Estatísticas do Ping para ${currentIP}:" -ForegroundColor Cyan
        Write-Host "    Pacotes: Enviados = 4, Recebidos = 0, Perdidos = 4 (100% de perda),"

        $resultsFull += [PSCustomObject]@{
            IP = $currentIP
            Status = "Inativo"
            Perdidos = 4
            TTL = "N/A"
            Min = "N/A"
            Max = "N/A"
            Media = "N/A"
        }
    }
}

# Gera um timestamp para nome da pasta
$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"

# Cria a pasta de relatório na área de trabalho do usuário atual
$reportFolder = "$env:USERPROFILE\Desktop\RELATORIO IP LIVRE_$timestamp"
New-Item -ItemType Directory -Path $reportFolder | Out-Null

# Salva os IPs em arquivos de texto e relatório completo em CSV
$activeIPs | Out-File -Encoding UTF8 -FilePath "$reportFolder\IP_uso.TXT"
$inactiveIPs | Out-File -Encoding UTF8 -FilePath "$reportFolder\IP_livre.TXT"
$resultsFull | Export-Csv -Path "$reportFolder\Full_Report.csv" -NoTypeInformation -Encoding UTF8

# Mensagem final de sucesso
Write-Host "`nRelatórios exportados para:" -ForegroundColor Cyan
Write-Host $reportFolder
Write-Host "- IP_uso.TXT (IPs que responderam)"
Write-Host "- IP_livre.TXT (IPs que não responderam)"
Write-Host "- Full_Report.csv (relatório detalhado com estatísticas)`n"

# Abre a pasta do relatório automaticamente
Start-Process $reportFolder
