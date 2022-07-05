#### Criando as máquinas

- Criar duas VM de Ubuntu server 20.04, no virtual box.
  
  [Link para download](https://releases.ubuntu.com/20.04.4/ubuntu-20.04.4-live-server-amd64.iso)

#### Criando uma interface Host-Only

- No menu do virtual box, vá em Arquivo => Host Network Manager, onde a seguinte tela será apresentada:
  
  ![](/home/luiz/.var/app/com.github.marktext.marktext/config/marktext/images/2022-06-28-15-25-58-image.png)

- Caso já haja uma interface não é necessário criar outra para esse laboratório, se não existir clique em criar, e marque a caixinha para habilitar o servidor DHCP.

#### Alocando as interfaces no Firewall.

- A VM do firewall deve conter dois adaptadores de rede, um que deve estar em modo Bridge, pelo qual o Firewall terá acesso a internet. E outra em modo Host-Only, que fará a comunicação com a rede interna.

- Ao abrir o painel de configurações da VM do Firewall vá em rede, no Adaptador 1, coloque `Placa em modo Bridge`, e selecione a interface da sua máquina física que tem acesso a internet. 

![](/home/luiz/.var/app/com.github.marktext.marktext/config/marktext/images/2022-06-28-16-31-27-image.png)

- No Adaptador 2, marque a caixinha **Habilitar Placa de Rede**, após isso coloque `Place de rede exclusiva de hospedeiro (Host-Only)`, e selecione a interface que você criou no passo anterior.

![](/home/luiz/.var/app/com.github.marktext.marktext/config/marktext/images/2022-06-28-16-34-35-image.png)

#### Alocando a interface do cliente.

- A VM Cliente só necessita de uma interface de rede que lhe possibilitará a comunicação com as máquinas na rede interna.

- Ao abrir o painel de configurações da VM do Firewall vá em rede, no Adaptador 1, coloque `Place de rede exclusiva de hospedeiro (Host-Only)`, e selecione a mesma interface a qual você selecionou para o firewall.

#### "Topologia da rede".

- A rede ficará da seguinte forma, salvo os nomes das interfaces podem ter alguma mudança.
  
  ![](/home/luiz/.var/app/com.github.marktext.marktext/config/marktext/images/2022-06-29-15-03-00-rede.png)

#### Configurações básicas do Cliente e Firewall

##### Firewall

- Acesse a máquina do firewall, e executando o comando `ip a`, ao executar esse comando é para você visualizar três interfaces, porém somente nos importa no momento  a segunda e a terceira.

![](/home/luiz/.var/app/com.github.marktext.marktext/config/marktext/images/2022-06-28-16-45-45-image.png)

- A interface 2 `enp0s3`, é a interface modo Bridge, a qual você pode confirmar vendo que seu endereço ip está na mesma faixa que o endereço ip de sua máquina física. É por essa interface que os pacotes saíram do firewall, seja originados nele, ou encaminhados através dele.

- A interface 3 `enp0s8`, é a sua interface em modo Host-Only, ou seja, é a interface pela qual o firewall irá acessar a rede interna. Caso essa interface não possua um endereço ip execute os seguintes comandos para alocar um ip a essa interface:
  
  1. "Ligue" a interface de rede:
     
     ```bash
     sudo ip link set enp0s8 up
     ```
  
  2. Força a interface a solicitar um ip ao Servidor DHCP:
     
     ```bash
     sudo dhclient enp0s8
     ```

**Obs**: o nome da interface pode mudar. E caso você desligue a máquina, talvez você tenha que refazer esses passos, uma vez que eles são em tempo de execução.

- De acordo com a topologia da rede, os clientes na rede interna, não terão acesso direto a internet, sendo que para acessara internet eles deverão passar pelo Firewall, porém por padrão o encaminhamento de pacotes vem desabilitado no ubuntu, então na máquina do Firewall, para habilitar execute o seguinte comando (senão der certo execute como root):
  
  ```bash
  sudo echo “1” > /proc/sys/net/ipv4/ip_forward
  ```

##### Cliente

- O cliente só terá uma interface além da de loopback, caso essa não tenha um ip, faça o mesmo que fez no firewall.

- Configure um servidor DNS no cliente, editando o arquivo netplan
  
  ```bash
  # Fazendo um bakcup do arquivo padrão
  cd /etc/netplan/
  sudo cp 00-installer-config.yaml 00-installer-config.yaml.bkp
  
  sudo vi 00-installer-config.yaml
  ```

- Seu arquivo deve ficar da seguinte maneira.
  
  ```bash
  # This is the network config written by 'subiquity'
    network:
      ethernets:
        enp0s3:
          dhcp4: yes
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4]
      version: 2
  ```

- Além disso no cliente é necessário informar que todos os pacotes que ele queira mandar para a internet devem ser enviados para o Firewall, para isso você deve informar o ip da interface do firewall que está conectado a rede interna, pelo seguinte comando:

- ```bash
  sudo ip route add default via ip_enp0s8
  ```

- 

- Configurações do Iptables no firewall.

```bash
# Permitindo acesso SSH e ICMP e loosudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADEpback
sudo iptables -I INPUT 1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --syn --sport 22 -m state --state NEW -j ACCEPT
sudo iptables -A INPUT -p icmp -m state --state NEW -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
# Mudando a policy INPUT da tabela filter.
sudo iptables -P INPUT DROP

# Regas saída de conexões estabelecidas e ICMP e loopback.
sudo iptables -I OUTPUT 1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p icmp -m state --state NEW -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
# Mudando a policy OUTPUT da tabela filter.
sudo iptables -P OUTPUT DROP

sudo iptables -P FORWARD DROP
sudo iptables -I FORWARD 1 -i enp0s3 -o enp0s8 \
-m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -I FORWARD 2 -i enp0s8 -o enp0s3 \
-m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p icmp -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 \
-p tcp --dport 22 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p tcp --dport 22 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 \
-p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p tcp --dport 80 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 \
-p tcp --dport 443 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p tcp --dport 443 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 \
-p tcp -m multiport --dports 22,21 -m state --state NEW -j ACCEPT
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 \
-p tcp -m multiport --dports 22,21 -m state --state NEW -j ACCEPT
# Tabela nat
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

sudo iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m tcp --dport 433 -j REDIRECT --to-ports 3129
sudo iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 3129
```
