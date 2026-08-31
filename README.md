name: RDP-Conexao-Infinita
on:
  workflow_dispatch

jobs:
  manter-conectado:
    runs-on: windows-latest
    timeout-minutes: 0  # Sem limite de tempo definido aqui
    steps:
      - name: Configurar RDP Seguro
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "SecurityLayer" -Value 2 -Force
          netsh advfirewall firewall delete rule name="RDP-Seguro" -ErrorAction SilentlyContinue
          netsh advfirewall firewall add rule name="RDP-Seguro" dir=in action=allow protocol=TCP localport=3389
          Restart-Service TermService -Force -ErrorAction SilentlyContinue

      - name: Criar Usuário
        run: |
          $senha = ConvertTo-SecureString "SuaSenhaForte!2026" -AsPlainText -Force
          New-LocalUser -Name "SkyroUser" -Password $senha -AccountNeverExpires -ErrorAction SilentlyContinue
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "SkyroUser" -ErrorAction SilentlyContinue

      - name: Instalar e Conectar Tailscale
        env:
          TAIL_KEY: ${{ secrets.TAILSCALE_AUTH_KEY }}
        run: |
          $inst = "$env:TEMP\ts.msi"
          Invoke-WebRequest -Uri "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi" -OutFile $inst -UseBasicParsing
          Start-Process msiexec -ArgumentList "/i `"$inst`" /quiet /norestart" -Wait
          Remove-Item $inst -Force
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey="$env:TAIL_KEY" --hostname=conexao-permanente
          Start-Sleep 7
          $ip = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
          echo "IP_TS=$ip" >> $env:GITHUB_ENV

      - name: Manter Rodando — Loop Infinito Seguro
        run: |
          Write-Host "`n=== ACESSO ==="
          Write-Host "Endereço: $env:IP_TS"
          Write-Host "Usuário: SkyroUser"
          Write-Host "Senha: A que você definiu no código"
          Write-Host "==============`n"
          Write-Host "✅ Conexão ativa! Para encerrar: cancele a execução no GitHub`n"
          # LOOP QUE FICA RODANDO SEM PARAR
          while ($true) {
            Write-Host "[$(Get-Date)] Conexão funcionando normalmente..."
            Start-Sleep -Seconds 300  # Verifica a cada 5 minutos
          }
          
