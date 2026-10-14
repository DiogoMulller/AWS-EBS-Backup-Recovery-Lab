# AWS EBS Backup & Recovery Lab — Evidências

**Data:** 14/10/2026 · **Região:** us-east-1 · **AZ:** us-east-1c

Este pacote contém prints organizados e recortados dos passos principais do laboratório. Os recortes removem grandes regiões vazias e barras com identificadores da conta. Não foram alteradas as saídas do terminal ou resultados técnicos. Uma faixa superior externa identifica cada evidência.

## Resumo técnico

- EC2 Amazon Linux 2023, t3.micro.
- Volume EBS original de 1 GiB, gp3, criptografado com KMS `aws/ebs`, sistema de arquivos XFS.
- Snapshot manual concluído, com restauração em segundo volume EBS na mesma zona de disponibilidade.
- Dois arquivos fictícios excluídos e recuperados: `relatorio-financeiro.txt` e `security-events.log`.
- Validação SHA-256: **3 de 3 arquivos OK** após recuperação e validação final.
- Tempo observado entre os marcadores do incidente: **783 s (13 min 03 s)**. Inclui ações manuais do laboratório; não é um SLA nem RTO garantido.

## Índice dos screenshots

| Evidência | Descrição |
|---|---|
| [01-ec2-operacional.png](../screenshots/01-ec2-operacional.png) | EC2 em execução com verificações aprovadas |
| [02-security-group-ssh.png](../screenshots/02-security-group-ssh.png) | Regra SSH restrita para acesso |
| [03-acesso-amazon-linux.png](../screenshots/03-acesso-amazon-linux.png) | Terminal Amazon Linux 2023 |
| [04-disco-raiz-antes-ebs.png](../screenshots/04-disco-raiz-antes-ebs.png) | Disco raiz antes da anexação |
| [05-configuracao-ebs-gp3.png](../screenshots/05-configuracao-ebs-gp3.png) | Configuração do volume gp3 criptografado |
| [06-tags-volume-ebs.png](../screenshots/06-tags-volume-ebs.png) | Tags de organização do EBS |
| [07-volume-ebs-anexado.png](../screenshots/07-volume-ebs-anexado.png) | Novo disco de 1 GiB detectado |
| [08-volume-xfs-montado.png](../screenshots/08-volume-xfs-montado.png) | XFS montado em /mnt/ebs-lab |
| [09-capacidade-volume.png](../screenshots/09-capacidade-volume.png) | Capacidade e espaço disponível no volume |
| [10-hashes-originais-ok.png](../screenshots/10-hashes-originais-ok.png) | Três hashes SHA-256 originais validados |
| [11-snapshot-concluido.png](../screenshots/11-snapshot-concluido.png) | Snapshot concluído 100% |
| [12-incidente-perda-arquivos.png](../screenshots/12-incidente-perda-arquivos.png) | Dois arquivos ausentes após exclusão |
| [13-ebs-restaurado-detectado.png](../screenshots/13-ebs-restaurado-detectado.png) | Novo volume restaurado detectado |
| [14-dois-volumes-montados.png](../screenshots/14-dois-volumes-montados.png) | Volumes original e recuperado montados |
| [15-integridade-apos-recuperacao.png](../screenshots/15-integridade-apos-recuperacao.png) | Verificação SHA-256 dos arquivos recuperados |
| [16-validacao-final.png](../screenshots/16-validacao-final.png) | Integridade final após retorno ao EBS original |
| [17-tempo-recuperacao.png](../screenshots/17-tempo-recuperacao.png) | Tempo de recuperação de 783 segundos |

| [18-volume-restaurado-origem-snapshot.png](../screenshots/18-volume-restaurado-origem-snapshot.png) | Criacao do EBS restaurado a partir do snapshot, configuracao gp3, KMS e tags |
| [19-hashes-sha256-completos.png](../screenshots/19-hashes-sha256-completos.png) | Hashes SHA-256 completos originais comparados com os arquivos recuperados |
| [20-desmontagem-volumes.png](../screenshots/20-desmontagem-volumes.png) | Desmontagem segura dos dois volumes de laboratorio |
| [21-volumes-adicionais-disponiveis.png](../screenshots/21-volumes-adicionais-disponiveis.png) | Volumes adicionais de 1 GiB identificados no console antes da exclusao |
| [22-volume-raiz-restante.png](../screenshots/22-volume-raiz-restante.png) | Apenas volume raiz de 8 GiB restante apos exclusao dos volumes adicionais |
| [23-encerramento-ec2.png](../screenshots/23-encerramento-ec2.png) | Acao de encerramento selecionada na instancia EC2 do laboratorio |
| [24-sem-instancias-em-execucao.png](../screenshots/24-sem-instancias-em-execucao.png) | Nenhuma instancia running encontrada com filtro aplicado |
| [25-sem-snapshots.png](../screenshots/25-sem-snapshots.png) | Nenhum snapshot proprio encontrado na regiao selecionada |

## Avaliação das evidências finais

- **Hashes completos:** demonstrados na imagem 19, com comparação dos três valores SHA-256.
- **Desmontagem:** demonstrada na imagem 20.
- **Volumes adicionais:** a imagem 21 mostra dois volumes de laboratório disponíveis; a imagem 22 mostra somente o volume raiz de 8 GiB na lista.
- **Encerramento da EC2:** imagem 23 mostra a opção de encerrar; imagem 24 comprova que não há instâncias `running` segundo o filtro selecionado. Para provar o estado `terminated` explicitamente, seria necessário remover o filtro e consultar a instância.
- **Snapshot:** imagem 25 comprova a ausência de snapshots próprios na região selecionada.
- **Origem do volume restaurado:** imagem 18 exibe a restauração a partir do snapshot.
- **Limitação:** as capturas mostram a limpeza em momentos específicos e não comprovam por si só a inexistência de todos os custos ou recursos em outras regiões.

## Observações para GitHub

- Oculte IDs de conta, IPs públicos, ARNs e qualquer nome ou credencial pessoal em capturas adicionais.
- Não publique arquivos `.pem`, segredos ou credenciais AWS.
- Algumas capturas redundantes e a tela de erro temporário do EC2 Instance Connect ficaram fora da sequência principal.
- Se uma evidência for citada no README, utilize exatamente o nome presente na pasta `screenshots/`.


## Evidência complementar 26 — encerramento confirmado

- `screenshots/26-ec2-encerrada-confirmada.png`: captura do console EC2 sem filtro de instâncias, mostrando `EBS-Recovery-Lab-Server` em estado **Encerrado**. A imagem foi recortada para retirar a barra superior com dados da conta AWS. Essa captura comprova o encerramento da instância, mas não comprova ausência de outros recursos faturáveis.
