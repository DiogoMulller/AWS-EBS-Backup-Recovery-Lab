# Relatório técnico — Incidente simulado de perda de dados no Amazon EBS

**Projeto:** AWS-EBS-Backup-Recovery-Lab  
**Data da execução:** 14/10/2026  
**Ambiente:** Amazon EC2 `t3.micro` · Amazon Linux 2023 · `us-east-1` / `us-east-1c`  
**Natureza:** Simulação controlada, arquivos fictícios  
**Status:** Recuperação validada

## 1. Resumo executivo

Foi simulada a exclusão acidental de dois arquivos armazenados em um volume Amazon EBS `gp3` criptografado de 1 GiB. Um snapshot havia sido criado antes da exclusão, com o sistema de arquivos desmontado para reduzir o risco de inconsistência. A partir do snapshot, foi provisionado um novo volume, anexado à mesma EC2 e montado em modo somente leitura. Os dois arquivos foram recuperados e copiados de volta ao volume original. A comparação SHA-256 final confirmou o conteúdo íntegro dos três arquivos do conjunto de testes.

**Resultado:** 2 arquivos perdidos recuperados; 3 de 3 arquivos com hashes válidos; intervalo observado de 783 s (13 min 03 s).

## 2. Ativos e escopo

| Ativo | Configuração/função |
|---|---|
| Servidor | EC2 Amazon Linux 2023, tipo `t3.micro` |
| Volume raiz | EBS de 8 GiB, dispositivo `nvme0n1` |
| Volume de dados | EBS `gp3`, 1 GiB, dispositivo `nvme1n1`, XFS, `/mnt/ebs-lab` |
| Backup | Snapshot EBS manual, concluído em 14/10/2026 |
| Volume recuperado | EBS `gp3` de 1 GiB, dispositivo `nvme2n1`, `/mnt/ebs-restored` |
| Criptografia | EBS em repouso com chave AWS gerenciada `aws/ebs` |
| Hashes de referência | `~/hashes-originais.sha256`, no disco raiz, fora do volume de dados |

Nenhuma produção ou informação real de usuários foi utilizada.

## 3. Linha do tempo e evidências

| Ordem | Evento | Evidência |
|---|---|---|
| 1 | Volume EBS preparado e montado com XFS | [08](../screenshots/08-volume-xfs-montado.png) |
| 2 | Hashes SHA-256 originais validados | [10](../screenshots/10-hashes-originais-ok.png) |
| 3 | Snapshot concluído, 100% | [11](../screenshots/11-snapshot-concluido.png) |
| 4 | Marcador UTC de início do incidente salvo em `~/incident-start.txt` | Comandos documentados |
| 5 | Dois arquivos excluídos; verificação acusa arquivos ausentes | [12](../screenshots/12-incidente-perda-arquivos.png) |
| 6 | Novo EBS criado a partir do snapshot | [18](../screenshots/18-volume-restaurado-origem-snapshot.png) |
| 7 | Original e restaurado montados simultaneamente | [14](../screenshots/14-dois-volumes-montados.png) |
| 8 | Verificação SHA-256 e restauração dos arquivos | [15](../screenshots/15-integridade-apos-recuperacao.png), [16](../screenshots/16-validacao-final.png) |
| 9 | Marcador UTC de término salvo em `~/incident-end.txt`; intervalo calculado | [17](../screenshots/17-tempo-recuperacao.png) |
| 10 | Volumes desmontados; ambiente removido | [20](../screenshots/20-desmontagem-volumes.png), [26](../screenshots/26-ec2-encerrada-confirmada.png) |

Os horários UTC exatos dos arquivos de marcador não constam nos prints fornecidos. O intervalo calculado no terminal foi **783 segundos**, portanto não se deve atribuir uma hora absoluta de início ou término sem acesso aos marcadores.

## 4. Detecção

No EBS original, após excluir `relatorio-financeiro.txt` e `security-events.log`, `sha256sum -c ~/hashes-originais.sha256` retornou falha de leitura para esses dois arquivos e `OK` para `configuracao-servidor.conf`.

![Evidência da perda](../screenshots/12-incidente-perda-arquivos.png)

## 5. Resposta e recuperação

1. Manter o snapshot original intacto e concluído.
2. Criar volume `gp3` de 1 GiB **a partir** do snapshot, na mesma zona da EC2.
3. Anexar o EBS restaurado como segundo volume de dados.
4. Montar XFS em somente leitura com `ro,nouuid,norecovery`, evitando conflito de UUID com o original.
5. Comparar os arquivos do volume restaurado com os hashes salvos antes do incidente.
6. Copiar os dois arquivos faltantes para o volume original.
7. Validar novamente os três arquivos no volume original, registrar término e calcular o tempo observado.

![Volume restaurado e original](../screenshots/14-dois-volumes-montados.png)

## 6. Integridade e resultados

| Arquivo fictício | Após exclusão | Após recuperação |
|---|---|---|
| `relatorio-financeiro.txt` | Ausente | SHA-256 OK |
| `configuracao-servidor.conf` | Preservado | SHA-256 OK |
| `security-events.log` | Ausente | SHA-256 OK |

![Validação final](../screenshots/16-validacao-final.png)

![Hashes completos comparados](../screenshots/19-hashes-sha256-completos.png)

**Métrica:** `incident-end - incident-start = 783 s = 13 min 03 s`.

**Interpretação:** medida prática de recuperação manual neste laboratório. Não é compromisso formal de RTO, nem comprova o RPO de workloads dinâmicas. O ponto de recuperação disponível foi o snapshot anterior à exclusão, sem alterações intermediárias nos arquivos do teste.

## 7. Causa e impacto

**Causa simulada:** exclusão manual não intencional de dois arquivos em volume separado. **Impacto limitado:** indisponibilidade dos dois arquivos fictícios até o retorno ao volume original. Não foram incluídas aplicações reais, operações de negócio, perda de dados reais ou múltiplas AZs.

## 8. Controles de segurança e limitações

- Criptografia EBS/KMS em repouso e snapshots decorrentes de volume criptografado.
- Identificação dos dispositivos com `lsblk` para evitar formatação do volume raiz.
- Consistência: `sync` e desmontagem antes da criação do snapshot.
- Referência de integridade mantida fora do EBS afetado.
- Preservação do conteúdo da cópia por montagem somente leitura.
- Limites: sem automatização DLM, sem teste de recuperação de aplicações ou medição de custo, sem avaliação de escala ou throughput sob carga.

## 9. Encerramento e recomendações

As evidências mostram remoção dos dois volumes de laboratório e do snapshot, e EC2 em estado `Encerrado`. A confirmação visual vale para a região e instantes capturados; despesas anteriores e recursos de outras regiões não são abrangidos.

**Melhorias recomendadas:** agendamento e retenção via DLM, testes automáticos de restauração, alarmes de falha, relatórios de tempo de recuperação com marcações precisas e políticas de armazenamento/backup que atendam RPO e RTO definidos.

**Conclusão:** A simulação cumpriu seu objetivo: recuperar os dois arquivos perdidos a partir de um snapshot de volume EBS criptografado e validar a integridade dos três arquivos por SHA-256.
