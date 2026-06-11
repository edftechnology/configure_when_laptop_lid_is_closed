# Como instalar o `configure_when_laptop_lid_is_closed` no `Linux Ubuntu`

## Resumo

Neste documento estao descritos os passos básicos para instalar e habilitar o programa `configure_when_laptop_lid_is_closed` no `Linux Ubuntu`.

## _Abstract_

_This document describes the basic steps to install and enable the program `configure_when_laptop_lid_is_closed` on `Linux Ubuntu`._



## Descrição

`configure_when_laptop_lid_is_closed` e um pequeno script que altera o comportamento do Ubuntu quando a tampa do laptop eh fechada. Ele permite definir a acao desejada por meio de um servico do `systemd`.



## 1. Instalar o `configure_when_laptop_lid_is_closed` no `Linux Ubuntu`

Para instalar o `configure_when_laptop_lid_is_closed`, siga os passos abaixo:

1. Abrir o `Terminal Emulator`. Você pode fazer isso pressionando:

    ```bash
    Ctrl + Alt + T
    ```

2. Certifique-se de que seu sistema esteja limpo e atualizado.

    2.1 Limpar o `cache` do gerenciador de pacotes `apt`. Especificamente, ele remove todos os arquivos de pacotes (`.deb`) baixados pelo `apt` e armazenados em `/var/cache/apt/archives/`. Digite o seguinte comando:
    ```bash
    sudo apt clean
    ```

    2.2 Remover pacotes `.deb` antigos ou duplicados do `cache` local. É útil para liberar espaço, pois remove apenas os pacotes que não podem mais ser baixados (ou seja, versões antigas de pacotes que foram atualizados). Digite o seguinte comando:
    ```bash
    sudo apt autoclean
    ```

    2.3 Remover pacotes que foram automaticamente instalados para satisfazer as dependências de outros pacotes e que não são mais necessários. Digite o seguinte comando:
    ```bash
    sudo apt autoremove -y
    ```

    2.4 Buscar as atualizações disponíveis para os pacotes que estão instalados em seu sistema. Digite o seguinte comando e pressione `Enter`:
    ```bash
    sudo apt update
    ```

    2.5 **Corrigir pacotes quebrados**: Isso atualizará a lista de pacotes disponíveis e tentará corrigir pacotes quebrados ou com dependências ausentes:
    ```bash
    sudo apt --fix-broken install
    ```

    2.6 Limpar o `cache` do gerenciador de pacotes `apt` novamente:
    ```bash
    sudo apt clean
    ```

    2.7 Para ver a lista de pacotes a serem atualizados, digite o seguinte comando e pressione `Enter`:
    ```bash
    sudo apt list --upgradable
    ```

    2.8 Realmente atualizar os pacotes instalados para as suas versões mais recentes, com base na última vez que você executou `sudo apt update`. Digite o seguinte comando e pressione `Enter`:
    ```bash
    sudo apt full-upgrade -y
    ```



### 1.1 Codigo completo para instalação

Execute o bloco abaixo para realizar toda a instalacao de uma so vez:

```bash
NÃO há.
```


## 2. Verificar e ajustar o arquivo `logind.conf`

1. Execute no `Terminal Emulator`:

    ```bash
    sudo nano /etc/systemd/logind.conf
    ```

2. E edite (ou descomente) as seguintes linhas:

    ```ini
    HandleLidSwitch=suspend
    HandleLidSwitchDocked=suspend
    HandleLidSwitchExternalPower=suspend
    ```

3. Depois, reinicie o serviço:

```bash
sudo systemctl restart systemd-logind
```

> **Importante**: isso afeta o comportamento global, inclusive antes do `login`.




## 3. Verificar se o display manager está bloqueando a suspensão

Se você usa o `lightdm`, `gdm`, `sddm` etc., eles podem bloquear a ação de suspender antes do `login`.

1. Você pode tentar habilitar a suspensão mesmo sem sessão ativa, criando (ou editando) o arquivo:

    ```bash
    sudo nano /etc/UPower/UPower.conf
    ```

2. E altere a linha:

    ```ini
    IgnoreLid=false
    ```

3. Depois, reinicie o serviço:

    ```bash
    sudo systemctl restart upower
    ```




## 4. Verificar se o GNOME / XFCE / KDE está sobrescrevendo as ações

Mesmo com logind configurado, o ambiente gráfico (como XFCE) pode ter **preferências locais que conflitam**.

1. Vá em: `Configurações → Gerenciador de energia → Comportamento da tampa`

2. Configure para `Suspender` tanto com cabo quanto com bateria.




## 5. Teste com `loginctl`

1. Você pode verificar as sessões com:

    ```bash
    loginctl
    ```

    Se a sessão estiver inactive antes do login, isso pode explicar por que o evento de fechamento da tampa não aciona nada — pois não há sessão ativa para escutar o evento.




## 6. Teste rápido de diagnóstico

1. **Reinicie o notebook e vá até a tela de `login`** (sem digitar senha).

2. **Feche a tampa e aguarde 10 segundos**.

3. **Reabra a tampa**. Se a tela continuar ligada e o sistema ativo, a suspensão não ocorreu.

4. Agora **faça login**, feche a tampa e veja se suspende normalmente.

Esse teste valida que a ação está atrelada à sessão ativa, e não apenas ao evento do hardware.



## 7. Passo a passo para criar o serviço de suspensão via `systemd`

### 7.1 Crie o _script_ de suspensão

1. Abra um `Terminal Emulator` e crie o _script_ que vai verificar o `status` da tampa:

    ```bash
    sudo nano /usr/local/bin/lid-close-suspend.sh
    ```

2. Cole o seguinte conteúdo:

    ```bash
    #!/bin/bash

    # Verifica o status da tampa
    lid_state=$(cat /proc/acpi/button/lid/LID*/state | grep -i 'closed')

    if [[ "$lid_state" == *closed* ]]; then
        # Forca a suspensao
        /bin/systemctl suspend
    fi
    ```

    Salve com `Ctrl+O`, depois `Enter`, e saia com `Ctrl+X`.

3. Agora torne o _script_ executável:

```bash
sudo chmod +x /usr/local/bin/lid-close-suspend.sh
```




### 7.2 Crie o serviço `systemd`

1. Agora vamos criar o serviço que será iniciado em segundo plano:

```bash
sudo nano /etc/systemd/system/lid-watcher.service
```

2. Cole o conteúdo abaixo:

    ```ini
    [Unit]
    Description=Suspender automaticamente ao fechar a tampa
    After=multi-user.target

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/lid-close-suspend.sh
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```



### 7.3 Criar um timer (verifica a cada X segundos)

1. O serviço acima é executado uma vez. Para rodar periodicamente, criamos um timer:

```bash
sudo nano /etc/systemd/system/lid-watcher.timer
```

2. Cole o conteúdo abaixo:

    ```ini
    [Unit]
    Description=Verifica fechamento da tampa periodicamente

    [Timer]
    OnBootSec=10sec
    OnUnitActiveSec=15sec
    Unit=lid-watcher.service

    [Install]
    WantedBy=timers.target
    ```

    Isso fará o script rodar a cada **15 segundos**, mesmo sem usuário logado.




### 7.4 Ativar tudo

1. Ative o serviço e o timer:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now lid-watcher.timer
```

2. Verifique se está rodando:

```bash
systemctl list-timers --all | grep lid-watcher
```

**Pronto!**

Agora, mesmo na tela de `login` (sem digitar a senha), o sistema detectará o fechamento da tampa e entrará em suspensão automaticamente.


**Observações importantes**

- Esse _script_ verifica apenas se a tampa está fechada, não se foi fechada agora, então pode haver atraso de até 15 segundos.

- Você pode reduzir o tempo no campo `OnUnitActiveSec` para 5 segundos, por exemplo.

- Se o `LID*` não estiver em `/proc/acpi/button/lid/`, podemos adaptar para `/etc/acpi/events/lid ou udev`.



## Referências

[1] OPENAI. ***Como instalar configure_when_laptop_lid_is_closed no Linux Ubuntu:*** https://chatgpt.com/c/688af8ea-d364-8321-a2d3-ab76e79015e1. ChatGPT. Acessado em: 31/07/2025.


