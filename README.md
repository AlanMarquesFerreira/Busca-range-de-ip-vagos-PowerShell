# 📡 Verificador de IPs Ativos/Inativos em Rede - PowerShell

Este script PowerShell realiza uma varredura em um intervalo de IPs da rede local, utilizando **ping** para verificar quais IPs estão **ativos** ou **inativos**, e gera relatórios organizados automaticamente.

---

## ⚙️ Como Funciona

- Converte o intervalo de IPs para inteiros para facilitar a iteração.
- Executa ping em cada IP (4 tentativas por IP).
- Classifica IPs como ativos ou inativos.
- Gera estatísticas de tempo de resposta.
- Exporta relatórios em `.TXT` e `.CSV`.
- Abre automaticamente a pasta de relatórios ao final.

---

## 🧠 Requisitos

- Windows com PowerShell
- Permissão para executar scripts (ver `Set-ExecutionPolicy`)
- Acesso à rede que será verificada

---

## 📜 Código Completo

```powershell
# Define o range de IPs
$startIP = "192.168.50.2"
$endIP = "192.168.50.255"

# Converte os IPs para inteiros para facilitar a iteração
function Convert-IPToInt64 {
    param($ip)
    $octets = $ip -split "\."
    return [int64]([int64]$octets[0]*16777216 + [int64]$octets[1]*65536 + [int64]$octets[2]*256 + [int64]$octets[3])
}

function Convert-Int64ToIP {
    param([int64]$int)
    return "$(($int -band 0xFF000000) -shr 24).$(($int -band 0x00FF0000) -shr 16).$(($int -band 0x0000FF00) -shr 8).$(($int -band 0x000000FF))"
}

$startInt = Convert-IPToInt64 $startIP
$endInt = Convert-IPToInt64 $endIP

# Listas para armazenar IPs ativos e inativos
$activeIPs = @()
$inactiveIPs = @()
$resultsFull = @()

# Executa ping para cada IP no range
for ($i = $startInt; $i -le $endInt; $i++) {
    $currentIP = Convert-Int64ToIP $i
    Write-Host ""
    Write-Host "Pingando { $currentIP } com 32 bytes de dados:" -ForegroundColor Cyan

    $results = Test-Connection -ComputerName $currentIP -Count 4 -ErrorAction SilentlyContinue

    if ($results) {
        $activeIPs += $currentIP

        foreach ($reply in $results) {
            if ($reply.ResponseTime -lt 1) {
                $time = "<1ms"
            } else {
                $time = "$($reply.ResponseTime)ms"
            }
            Write-Host "Resposta de { $currentIP }: bytes=32 tempo=$time TTL=$($reply.ResponseTimeToLive)"
        }

        $stats = $results | Measure-Object -Property ResponseTime -Average -Maximum -Minimum
        Write-Host ""
        Write-Host "Estatísticas do Ping para { $currentIP }:" -ForegroundColor Cyan
        Write-Host "    Pacotes: Enviados = 4, Recebidos = $($results.Count), Perdidos = $(4 - $results.Count) ($(100 * (4 - $results.Count)/4)% de perda),"
        Write-Host "Tempo aproximado de ida e volta em ms:" -NoNewline
        Write-Host "    Mínimo = $($stats.Minimum)ms, Máximo = $($stats.Maximum)ms, Média = $([math]::Round($stats.Average,1))ms"

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
        $inactiveIPs += $currentIP
        Write-Host "Resposta de { $currentIP }: Host de destino inacessível."
        Write-Host ""
        Write-Host "Estatísticas do Ping para { $currentIP }:" -ForegroundColor Cyan
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

# Exporta resultados
$timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
$reportFolder = "$env:USERPROFILE\Desktop\RELATORIO IP LIVRE_$timestamp"
New-Item -ItemType Directory -Path $reportFolder | Out-Null

$activeIPs | Out-File -Encoding UTF8 -FilePath "$reportFolder\IP_uso.TXT"
$inactiveIPs | Out-File -Encoding UTF8 -FilePath "$reportFolder\IP_livre.TXT"
$resultsFull | Export-Csv -Path "$reportFolder\Full_Report.csv" -NoTypeInformation -Encoding UTF8

Write-Host "`nRelatórios exportados para:" -ForegroundColor Cyan
Write-Host $reportFolder
Write-Host "- IP_uso.TXT (IPs que responderam)"
Write-Host "- IP_livre.TXT (IPs que não responderam)"
Write-Host "- Full_Report.csv (relatório detalhado com estatísticas)`n"

# Abre a pasta do relatório automaticamente
Start-Process $reportFolder
```

---

## 📂 Exemplo de Saída

- `IP_uso.TXT` → IPs que responderam ao ping ✅  
- `IP_livre.TXT` → IPs que **não** responderam ao ping ❌  
- `Full_Report.csv` → Arquivo com **estatísticas completas** 📊

---

## 🚀 Autor

Feito com 💻 e ☕ por.  
| [<img loading="lazy" src="https://avatars.githubusercontent.com/u/124633669?v=4" width=115><br><sub> Alan Marques Ferreira </sub>](https://github.com/alanmarquesferreira) |  
 | :---: |
📅 Última atualização: 15/04/2025

---

## 📬 Contribuições

Sinta-se à vontade para abrir issues, sugerir melhorias ou enviar PRs!

---
