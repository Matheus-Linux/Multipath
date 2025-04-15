## Multipath Servidores Proxmox 

Este documento foi elaborado para auxiliar na compreensão e documentação da funcionalidade do ambiente de virtualização com Multipath. Ele serve exclusivamente como um guia para estudantes e profissionais de TI.

---
## Função principal

- Mapear o caminho das Fibras ópticas

- Criar alias de nomes para caminhos das fribas 

- Alta disponibilidade

- Failover 

- Gerenciamento de caminhos 

## Tecnologias utilizadas 

- Linux 

- Proxmox 

- Multipathd 

- SAN Network

- Dell Unity
  
---
## Procedimento para criação de Alias 

```bash
#Encontrar o wwid mapeado para o servidor 
multipath -l 

#Caso o mesmo não esteja visível com este comando, digitar o seguinte
multipath -v3 > mapeamento.temp 

#Verificar se o mesmo já está visível
grep <nº wwid> mapeamento.temp

#Caso o mesmo seja encontrado no arquivo, adicionar manualmente
multipath -a <nº wwid> 

#Geralmente esse processo é necessário, se o multipath não
#estiver com a função de encontrar wwn de forma automatica
```

---
## Criando alias  multipath.conf

```bash
#Exemplo de um mapeamento multipath.conf 

multipaths {
    multipath { 
        wwid  36006016002f359005530c56575463c95
        alias **Unity_VMDATA01**
    }
}

#Após salvar o arquivo, realizar a reeleitura do multipath
multipath -r 
```

---
## Princais configurações do multipath.conf 

- **defaults**
  **Função**: A seção defaults define as configurações padrão que se aplicam a todos os dispositivos multipath

- **polling_interval**
  **Função**: Define o intervalo (em segundos) entre as verificações de estado dos caminhos. 
  **Valor**: 5
    - **Significado**:  O multipath verificará o estado dos caminhos a cada x segundos.

- **path_selector**
  **Função**: Define a política de seleção de caminhos para o balanceamento de carga.
  **Valor**: round-robin  
    - **Significado**: Utiliza o algoritmo de round-robin para distribuir o tráfego entre os caminhos disponíveis, com uma faixa de "0" indicando o número de caminhos a serem utilizados.

- **path_grouping_policy**
  **Função**:
  **Valor**: Failover
    - **Significado**: Caminhos são agrupados em um único grupo, e o tráfego é direcionado para o caminho primário. Se o caminho primário falhar, o tráfego é redirecionado para outro caminho disponível.

- **failback**
  **Função**: Define o comportamento do failback, ou seja, a restauração do caminho original após a falha ser resolvida.
  **Valor**: manual
    - **Significado**: O failback para o caminho original não ocorre automaticamente; deve ser feito manualmente.

- **user_friendly_names**
  **Função**: Controla se os nomes amigáveis (baseados em aliases) serão usados para dispositivos.
  **Valor**: yes
    - **Significado**: O multipath usará nomes amigáveis em vez dos nomes de dispositivo padrão (como /dev/sda).

- **find_multipaths**
  **Função**: Define se o multipath deve procurar por dispositivos multipath.
  **Valor**: no
    - **Significado**: O multipath não procurará automaticamente por dispositivos multipath; eles devem ser definidos explicitamente na seção multipaths.

- **blacklist**
  **Função**:  seção blacklist é usada para especificar dispositivos que não devem ser gerenciados pelo multipath. Isso pode ser útil para evitar que dispositivos não suportados sejam incluídos na configuração de multipath.
     1. **devnode**
        - **Descrição**: Define padrões de correspondência para dispositivos a serem excluídos com base no nome do dispositivo.
        - **valor**: "^sda$"
        - **Significado**: Exclui o dispositivo /dev/sda do gerenciamento de multipath.

     2. **wwid**
        - **Descrição**: Define identificadores de WWID (World Wide Identifier) para dispositivos a serem excluídos. 
        - **valores**:
          - 36006016002f35900af104566050c5509
          - 36000d31000abda00000000000000032f
          - 36000d31000abda000000000000000330
          - 36000d31000abda000000000000000331
          - 36000d31000abda000000000000000332
        
        **Significado**: Exclui os dispositivos com esses identificadores WWID da configuração de multipath.
          
- **multipaths**
  **Função**: Define uma configuração específica para um dispositivo multipath.
  **wwid**:
    - **Descrição**: Define uma configuração específica para um dispositivo multipath.
    - **valor**: 36006016002f359005530c56575463c95
    - **Significado**: WWID para o dispositivo que você está configurando.
  **Alias**:
    - **Descrição**: Define um alias amigável para o dispositivo multipath.
    - **Valor**: Unity_VMDATA01
    - **Significado**: O dispositivo multipath será referenciado pelo nome Unity_VMDATA01.

---
## Remoção de blocos do multipath

Essa configuração é necessária para garantir que, caso uma LUN seja removida do mapeamento do servidor, o impacto no funcionamento do cluster Proxmox seja minimizado. Se a LUN não for removida corretamente, isso pode afetar negativamente o desempenho e a estabilidade do cluster Proxmox como um todo.

# Removendo mapeamento dos blocos do servidor 

```bash
#Listar os paths das fibras
multipath -l 

#Exemplo de um mapeamento de multipath

mpathad (36000d31000abda000000000000000351) dm-16 COMPELNT,Compellent Vol
size=10T features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
`-+- policy='round-robin 0' prio=50 status=active
  |- 15:0:7:1  sdci 69:96  active ready running
  |- 16:0:6:1  sdck 69:128 active ready running
  |- 15:0:4:1  sdcj 69:112 active ready running
  `- 16:0:14:1 sdcl 69:144 active ready running

#Removendo blocos 
echo '1' > /sys/block/sdci/device/delete
echo '1' > /sys/block/sdck/device/delete
echo '1' > /sys/block/sdck/device/delete
echo '1' > /sys/block/sdcl/device/delete

#Após a remoção de todos os blocos, será necessário realizar o multipath 
systemctl restart multipathd
```

---
## Reeleitura de fibra 

Essa configuração é necessária quando uma nova LUN é apresentada aos servidores. Ela garante que a re-leitura da fibra permita que os servidores reconheçam os novos mapeamentos.

# Realizando reeleitura 

```bash 
#Realizando reeleitura em todas as firbas com estado up
echo '- - -' > /sys/class/fc_host/host15/issue_lip
echo '- - -' > /sys/class/fc_host/host16/issue_lip

#Verificando estado da fibra
cat /sys/class/fc_host/host15/port_state
cat /sys/class/fc_host/host16/port_state
```

---
## Máquinas do cluster Proxmox 

- **SRVPMXPRD11**
  - **IP**: 10.57.1.99

- **SRVPMXPRD12**
  - **IP**: 10.57.1.100

- **SRVPMXPRD13**
  - **IP**: 10.57.1.101

- **SRVPMXPRD14**
  - **IP**: 10.57.1.102

- **SRVPMXPRD15**
  - **IP**: 10.57.1.103

---
## Diagrama Multipath

![Image](https://github.com/user-attachments/assets/aa8a1f01-0c43-4751-8dfa-27d160348560)


