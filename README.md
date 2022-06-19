### Tunning FreeBSD
#### Aprendendo uns conceitos de como deixar o BSD mais performático
___
#### SWAP 
* Evitar erros de páginação:
    * Para menos de 4GB, coloque o dobro de memória SWAP;
    * Para igual 4GB de RAM, coloque a mesma quantidade de memória para SWAP;
    * Mais que 4GB, configure como desejar;
    * Tenha em mente possíveis futuras expansões para melhor do swap;

* Problemas de má gestão da SWAP:
    * Erros de páginação;
    * Aumento no tempo de processamento a longo prazo devido a dificuldades do limite definido;

* Gestão:
    * Caso tenha mais de um armazenamento, configure um swap para cada como espaço de troca entre leitura e escrita;
___
#### Sistema de arquivos

* Não é uma boa fazer partições muito grandes ou unicas, devido a gestão do Kernel sobre a partição devido ao sistema de arquivos;
___
#### Sistemas de montagem
* O mais obvio e perigo é p assíncrono (Recomendado apenas em sistemas GJOURNAL, forá isso não use em nenhum outro);
* O mais seguro e utíl é o noatime, seu funcionamento é gerar um horário de ultimo acesso sobre os arquivos, após isso, ele os usa como uma chave, seu maior problema é que com ações continuas de gravação, pode acabar gerando um buffer poluido que diminui o tempo das operações gerais;
___
#### Discos de striping
* É o ato de usar ferramentas para dividir uma unica partição em demais partições menores, fazendo com que uma partição como o /usr não sofra com consequencias de uma /tmp, já que uma tem mais utilidade escrita e outra é leitura como uma forma de exemplo;
___
#### Tunning de variaveis de sistema
* Tem 2 locais para se mexer e ter alguma melhora de performance no sistema, esses são:
    * sysctl 
        * Variaveis de sistema que o kernel utiliza;
    * rc.conf
        * Configurações de boot

<br>

* Sysctl
    * Segue listado os principais parametros segundo o manual do BSD:
        * ```vm.overcommit``` -> Comportamento do overcommit do subsistema;
            * Se definido como 0, ocorrerá erros de acesso a memória do ```vm.swap_reserved```;
            * Se definido como 1, será imposto limite no ```RLIMIT_SWAP``` (controla o consumo máximo de recursos do sistema);
            * Se definido como 2, ele aloca toda a memória disponível (Exceto em uso e livre) no espaço de troca;
                * As excessões daisenção são os parametros de ```vm.stats.vm.v_free_target``` e ```vm.stats.vm.v_wire_count```;
        * ```vm.swap_total``` -> Fornece os bytes disponíveis para troca;
        * ```vm.swap_reserved``` -> Quantidade total de bytes para fazer BK da memória anonima;
        * ```kern.ipc.maxpipekva``` -> É usado para definir o tamanho do canal de buffer para o kernel, aumentar ese valor da mais espaço para o canal trabalhar;
            * Em caso de uso de todo o canal, não haverá problemas, somente levára ao reprocessamento de pipelines não finalizadas;
        * ```kern.ipc.shm_use_phys``` -> Esse por padrão é 0 e só pode ser mudado para 1, quando setado, as ações do sistema são de:
            * Ele vai paginar toda memória compartilhada para a RAM não paginada;
            * Com essa paginação duas ações podem ocorrer:
                * Toda memória compartilhada já paginada será divida em pequenos blocos de memoria compartilhada ;
                * Toda a memória paginada será divida em grandes blocos;
            * Independente de qualquer um dos dois meios, isso faz com que o kernel consiga limpar os blocos e evitando os custos de remapeamento, porém, como não há remapeamento, se torna indispensavel esse recurso, se não o sistema irá falhar;
        * ```vfs.vmiodirenable``` -> Definido como 1 por padrão, esse parametro controla o cache dos diretorios do sistema para acelerar o acesso aos mesmo (Cache bem pequeno);
            * Controlando esse parametro, pode ser alterado para utilizar toda a memoria fisica para fazer o cache de diretorios (Mesmo que não sejam do sistema);
            * As duas desvantagens do método são:
                * Sistemas com pouca RAM não é recomendado essa ação;
                * Os blocos de cache tem o padrão de 2K, mesmo com esse padrão ativado, o cache sobe ao limite de um bloco que é 4K, a velocidade é aumenta no acesso já que o mapeamento é feito de antemão, porém o desbloqueio não acelara tanto em diferença de tempo no acesso a diretorios já antes mapeados;
                * Vale comentar que pode haver disperdicio de memoria já que mesmo o mapeamento em blocos maiores não ocupem toda a área;
            * Essa alteração é recomenda em sistema de grandes quantidades de alterações de arquivos;
        * ```vfs.write_behind``` -> Esse é ativo por padrão, sua função é gravar o buffer em arquivos de saída ou em clusters, ele evita a lotar o cache de E/S, porém ele pode sofrer de erros que pode ser uma opção desabilitar ele;
        * ```vfs.hirunningspace```-> Este é usado pelo sistema de arquivos UFS, ele determina o quanto de E/S pode ser enfilerado para ações em determinados momentos;
            * Colocar muita memória não significa melhor desempenho, por motivos que toda a fila tem de ser lida, mesmo pedações em branco, causando perda de desempenho;
        * ```vfs.read_max``` -> Utilizado pelos sistemas de arquivo UFS, ext2fs e msdosfs. Ele é responsável pelo pré-leitura de arquivos do VFS e é um valor de pré-blocos a serem lidos para acelerar o E/S;
            * Alterar essa opção para maior quantidade de blocos com um sistema com poucos recursos, pode levar a problemas de desempenho geral;
        * ```vfs.ncsizefactor``` -> Controla o cachename dos arquivos, o cachename é definido por demais parametros de kernel que são utilizados em operações para definir o cachename automaticamente;
            * Dentro do cachename são definidos as permissões e os afetados arquivos no momento;
        * ```net.inet.tcp.sendspace``` e ```net.inet.tcp.recvspace``` -> Esses são os buffers de envio e recebimento de TCP, sendo esses por default 32K para envio e 64K para recebimento:
            * Alterar esses valores pode gerar melhor desempenho na rede, porém os custos são:
                * Ele retira esse memória do Kernel para cada linha de atendimento;
                * Caso uma linha caia a memória não é automaticamente realocada, levando um um tempo um esse memória não é utilizada até ser limpa, em casos de muitos acessos com erro, várias linhas com grandes quantidades de memória, podem causar lentidão no sistema;
            * Esse alteração é valida em redes com:
                * Redes GigabitEthernet;
                * Máquinas com grandes recursos (Mesmo com os possíveis problemas, se tiver uma máquina boa o suficiente para pagar os custos, é valida a alteração, já que a melhora real no desempenho dos acessos via TCP);
        * ```net.inet.tcp.always_keepalive``` -> Esse parametro é setado como 1 por padrão, sua função é manter os ```keepalives``` em conexões TCP, caso desativado, as aplicações devem se manter por conta própria, suas vantagens e desvantages são:
            * A vantagem é a não necessidade de manter a ação, levando a uma redução de uso de recursos continuos;
            * A desvantagem é que toda a conexão cortada terá de refazer uma conexão ao host, levando a um reprocessamento da requisição que é mais custoso em grandes volumes que manter a conexão;
        * ```net.inet.tcp.delayed_ack``` -> Este é referente a um tempo de delay de execução remota, seu uso é um retorno echo da ação para garantir seu cumprimento, porém sua utilização traz como desvantagem um delay tanto no envio quanto na entrega:
            * Seu desativamento leva a requisições mais rapídas, porém não a garantias de retorno por ser imediatista;
        * ```net.inet.ip.portrange.*``` -> Todos os parametros desse tipo são as faixas de socket que podem ser usadas em requisições, dividas em 3 (baixa, padrão e alta), variando do socket 49152 a 65535;
            * As aplicações normalmente utilizam o socket padrão;
            * È recomendo sempre alterar as portas baixas e padrão, já que as sobresalentes são gerenciadas automaticamente pelo sistema com as restantes para as portas altas;
        * ```kern.ipc.somaxconn``` -> Limita a quantidade de portas de entradas de conexões do socket, sendo o padrão 128, porém é recomendo aumentar esse valor para melhor desempenho do trafico em redes maiores (Ex: 1024);
        * ```kern.maxfiles``` -> Esse parametro define quantos arquivos podem ser abertos simuntaniamente no sistema, em caso de altas cargas com recursos, é bom aumentar ele;
        * ```kern.openfiles```-> Esse define a quantidade de arquivos para leitura dentro do sistema de forma simuntania;
        * ```vm.swap_idle_enabled``` ->  Essa variavel define a prioridade da memória swap em uso;
        * ```vm.swap _ idle _ threshold1``` e ```vm.swap _idle _ threshold2``` -> Responsaveis por definir a prioridade de páginas associadas a processos ociosos mais rapidamente que o deamon de pageout;
        * ```kern.ipc.nsfbufs``` -> Responsavel por controlar o buffer de serviços de envio de arquivos pela rede do ```sendfile```;
        * ```SCSI_DELAY``` -> Responsável pelo tempo de boot;

___
#### Start script pós-boot
* Altere o /etc/rc.local (Possívelmente ele não existe) para executar um comando, script, rotinada  ou demais(E/Ou um ou mais), após o boot; 
___
#### Nova compilação
* Uma das melhores formas de conseguir ganhar desempenho no FreeBSD é compilar uma versão sem os drivers desnecessários;