# Runbook — Backup e recuperação de EBS (ambiente de laboratório)

**Atenção:** execute apenas em recursos de testes de sua propriedade ou autorizados. Os dispositivos NVMe não têm numeração garantida. **Nunca formate um dispositivo sem comprovar que é o volume vazio correto.** A exclusão proposital deve ser limitada aos arquivos fictícios deste laboratório.

## Pré-requisitos

- EC2 Amazon Linux 2023 com conexão de terminal e permissões de sudo.
- EBS `gp3` de 1 GiB criptografado, anexado **na mesma AZ** da EC2.
- Identificação do volume raiz e dos volumes de dados (`lsblk`, `file -s`, Console EC2, IDs de volume).
- Permissões IAM adequadas para criar snapshots e volumes EBS, anexá-los e removê-los.

## A. Preparação de volume novo e vazio

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo file -s /dev/nvme1n1
# Apenas após validar que /dev/nvme1n1 é o NOVO volume vazio de 1 GiB:
sudo mkfs.xfs -f /dev/nvme1n1
sudo mkdir -p /mnt/ebs-lab
sudo mount /dev/nvme1n1 /mnt/ebs-lab
df -hT /mnt/ebs-lab
sudo mkdir -p /mnt/ebs-lab/dados
sudo chown ec2-user:ec2-user /mnt/ebs-lab/dados
```

## B. Dados fictícios

```bash
cd /mnt/ebs-lab/dados
cat > relatorio-financeiro.txt <<'DATA'
RELATORIO FINANCEIRO - AMBIENTE DE TESTES
Empresa: Cloud Security Labs
Periodo: Outubro de 2026
Receita: R$ 150.000
Despesas: R$ 87.000
Status: Documento ficticio para laboratorio AWS
DATA
cat > configuracao-servidor.conf <<'DATA'
[server]
hostname=aws-ebs-recovery-lab
environment=development
backup_enabled=true
encryption=AES-256
storage=Amazon EBS gp3
DATA
cat > security-events.log <<'DATA'
2026-10-14 10:00:00 INFO - Server started
2026-10-14 10:05:00 INFO - Backup system initialized
2026-10-14 10:10:00 INFO - User authenticated
2026-10-14 10:15:00 WARNING - Suspicious login attempt simulated
2026-10-14 10:20:00 INFO - Security monitoring active
DATA
sha256sum *.txt *.conf *.log > ~/hashes-originais.sha256
sha256sum -c ~/hashes-originais.sha256
```

Esses são os **modelos de dados usados na orientação do laboratório**. Os hashes corretos são os produzidos na sua execução; veja [captura da comparação real](../screenshots/19-hashes-sha256-completos.png).

## C. Snapshot consistente

```bash
sync
cd ~
sudo umount /mnt/ebs-lab
```

No Console AWS: **EC2 → Volumes → selecionar volume de dados → Ações → Criar snapshot**. Verifique a tag `Name=EBS-Recovery-Lab-Snapshot-01` e aguarde status `Concluído`. Depois, remonte o EBS original:

```bash
sudo mount /dev/nvme1n1 /mnt/ebs-lab
cd /mnt/ebs-lab/dados
sha256sum -c ~/hashes-originais.sha256
```

## D. Simular exclusão e registrar tempo

```bash
date -u +'%Y-%m-%dT%H:%M:%SZ' | tee ~/incident-start.txt
rm /mnt/ebs-lab/dados/relatorio-financeiro.txt
rm /mnt/ebs-lab/dados/security-events.log
cd /mnt/ebs-lab/dados
sha256sum -c ~/hashes-originais.sha256  # 2 arquivos ausentes (esperado)
```

## E. Criar e montar volume de recuperação

No Console: **EC2 → Snapshots → selecionar snapshot → Ações → Criar volume a partir do snapshot**. Confirme o ID do snapshot, tipo `gp3`, 1 GiB, criptografia, AZ da EC2 e tag `EBS-Recovery-Lab-Restored`. Anexe à EC2. Verifique o dispositivo correto após anexar:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo mkdir -p /mnt/ebs-restored
# Na execução documentada, o volume restaurado foi /dev/nvme2n1.
sudo mount -t xfs -o ro,nouuid,norecovery /dev/nvme2n1 /mnt/ebs-restored
findmnt /mnt/ebs-restored
cd /mnt/ebs-restored/dados
sha256sum -c ~/hashes-originais.sha256
```

`nouuid` contorna conflito do UUID XFS entre original e snapshot; `ro,norecovery` evita gravações/replay de journal no volume recuperado. O snapshot foi realizado depois da desmontagem do sistema de arquivos. Se a montagem falhar, **não formate o volume restaurado**: confira identificação e mensagens de erro.

## F. Recuperar arquivos e verificar

```bash
cp /mnt/ebs-restored/dados/relatorio-financeiro.txt /mnt/ebs-lab/dados/
cp /mnt/ebs-restored/dados/security-events.log /mnt/ebs-lab/dados/
cd /mnt/ebs-lab/dados
sha256sum -c ~/hashes-originais.sha256
# Registre o fim apenas após todos retornarem OK.
date -u +'%Y-%m-%dT%H:%M:%SZ' | tee ~/incident-end.txt
start=$(cat ~/incident-start.txt)
end=$(cat ~/incident-end.txt)
echo "Tempo de recuperação: $(($(date -d "$end" +%s) - $(date -d "$start" +%s))) segundos"
```

**Resultado observado na execução:** 783 segundos; 3 arquivos íntegros.

## G. Limpeza e custos

Depois de salvar evidências, e **somente se os recursos não forem mais necessários**:

```bash
cd ~
sudo umount /mnt/ebs-restored
sudo umount /mnt/ebs-lab
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```

No Console, desanexe e exclua **somente** os dois volumes de 1 GiB identificados pelas tags. Encerre a EC2 e confirme a exclusão do volume raiz se `DeleteOnTermination` estiver habilitado; remova o snapshot manual somente se não precisar mais dele. Verifique também IPs elásticos alocados e recursos remanescentes. Instâncias encerradas não têm cobrança de computação, mas outros recursos podem continuar faturáveis.

## Checkpoints

| Checagem | Esperado |
|---|---|
| Antes da formatação | Raiz `/dev/nvme0n1`, volume novo distinto de 1 GiB |
| Após a montagem | EBS `xfs` em `/mnt/ebs-lab` |
| Antes do snapshot | 3/3 hashes OK |
| Snapshot | 100% concluído |
| Incidente | 2 arquivos ausentes; 1 OK |
| Volume recuperado | Sistema de arquivos XFS existente, sem reformatação |
| Verificação no restaurado | 3/3 hashes OK |
| Verificação final original | 3/3 hashes OK |
| Limpeza | Sem volumes/snapshots remanescentes do laboratório; EC2 encerrada |
