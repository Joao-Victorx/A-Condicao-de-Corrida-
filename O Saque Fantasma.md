# Relatório de Diagnóstico e Arquitetura de Solução
## O Saque Fantasma — Análise de Concorrência e Exclusão Mútua

---

## 1. Relatório do Analista de Incidentes: Diagnóstico da Falha

### 1.1 Identificação Teórica da Falha
No ecossistema de um banco digital, a integridade das transações financeiras depende criticamente de operações atômicas e isoladas sobre os dados bancários. Contudo, em cenários de alta concorrência, a ausência de controle de acesso simultâneo resulta no fenômeno clássico conhecido como **Condição de Corrida** (*Race Condition*).

Segundo a definição clássica de Tanenbaum e Bos (2016, p. 82-83), uma condição de corrida ocorre quando dois ou mais processos ou threads realizam operações de leitura e escrita em uma área de memória ou estrutura de dados compartilhada, e o resultado final da execução depende de forma determinística do momento exato e da ordem em que os passos são executados pelo processador.

No contexto da aplicação bancária sob análise, a variável referente ao saldo da conta corrente é um dado compartilhado na memória. A sub-rotina encarregada de consultar o saldo, verificar a suficiência de fundos e efetuar o débito constitui uma **Região Crítica** (ou Seção Crítica). Conforme explicitado por Tanenbaum e Bos (2016, p. 83-84), a região crítica é o trecho de código no qual o recurso compartilhado é acessado, e que não pode, sob hipótese alguma, ser executado simultaneamente por mais de uma thread sem que haja mecanismos formais de exclusão mútua.

### 1.2 Mecânica do Incidente no Nível do Sistema Operacional
Para compreender a origem da falha em nível de infraestrutura e hardware, deve-se observar que uma operação comercial de débito (ex.: `saldo = saldo - valor`) não é executada pela CPU como uma única instrução atômica e indivisível. Em linguagem de máquina/assembly, essa instrução desdobra-se em três etapas elementares:

1. **Leitura (*Fetch*):** Transferência do valor do saldo da memória principal (RAM) para um registrador local da CPU.
2. **Modificação (*Arithmetic Operation*):** Decremento do valor no registrador local.
3. **Escrita (*Store*):** Gravação do novo valor contido no registrador de volta no endereço de memória principal.

A inconsistência temporal no milissegundo exato (o "Saque Fantasma") é desencadeada pelo **escalonamento preemptivo por chaveamento de contexto** (TANENBAUM; BOS, 2016, p. 106, 110). A cronologia detalhada da falha ocorre na seguinte sequência:

- **Passo 1 (Leitura do Saque):** A Thread A (responsável pelo saque manual de R$ 800,00) lê o saldo inicial na memória (ex.: R$ 1.000,00) e calcula localmente em seu registrador o novo valor ($1000 - 800 = 200$).
- **Passo 2 (Preempção por Interrupção de Relógio):** Antes que a Thread A execute a instrução de gravação (*store*) na memória principal, o temporizador de hardware gera uma interrupção de relógio (*clock interrupt*). O escalonador do SO suspende a Thread A e realiza o chaveamento de contexto, alocando a CPU para a Thread B (responsável pelo débito automático de R$ 50,00).
- **Passo 3 (Intercalação Inconsistente):** A Thread B é executada e lê da memória principal o saldo atual. Como a Thread A não concluiu a gravação, a memória principal ainda contém os R$ 1.000,00 originais. A Thread B calcula o abate ($1000 - 50 = 950$), grava R$ 950,00 na memória principal e finaliza seu quantum de tempo.
- **Passo 4 (Perda de Atualização / *Lost Update*):** A Thread A é reescalonada e retoma a execução exatamente a partir do registrador salvo em seu *Process Control Block* (PCB). Ela não relê a memória principal; apenas descarrega o valor pré-calculado (R$ 200,00) na memória, sobrescrevendo e destruindo completamente o débito efetuado pela Thread B.

A mecânica observada é idêntica ao exemplo clássico do diretório de *spool* de impressão descrito por Tanenbaum e Bos (2016, p. 83), onde o preenchimento de slots de arquivo falha devido à preempção no exato instante entre a leitura e a escrita do índice.

### 1.3 Conclusão do Diagnóstico
A causa raiz do incidente do "Saque Fantasma" é a **ausência de exclusão mútua** sobre a região crítica de consulta e atualização do saldo. A falha ocorre devido ao intercalamento de instruções sob um escalonador preemptivo (TANENBAUM; BOS, 2016, p. 106), permitindo a leitura de estados inconsistentes ("leitura suja") e o fenômeno da atualização perdida (*lost update*).

---

## 2. Relatório do Arquiteto de Sistemas Operacionais: Solução Arquitetural

### 2.1 Objetivo da Solução e Exclusão Mútua
A solução arquitetural proposta busca impedir que duas operações financeiras alterem o saldo da mesma conta de forma concorrente e inconsistente. Para isso, aplica-se o princípio da **exclusão mútua**: enquanto uma operação estiver executando a sequência crítica de consulta, validação e atualização do saldo, outra operação que precise alterar o mesmo recurso deverá aguardar.

A fundamentação principal é o capítulo 2 de *Sistemas operacionais modernos*, de Tanenbaum e Bos (2016), especialmente as seções sobre condições de corrida, regiões críticas e mutexes.

### 2.2 Fundamentação Teórica: Condição de Corrida e Região Crítica
Tanenbaum e Bos (2016, p. 82-83) explicam que uma condição de corrida ocorre quando dois ou mais processos acessam dados compartilhados e o resultado depende da ordem ou do instante em que suas operações são executadas. Os autores apresentam um exemplo em que dois processos utilizam valores compartilhados e uma atualização de um processo acaba sobrescrevendo a atualização do outro.

O mesmo risco existe no cenário bancário: o saldo é um dado compartilhado e tanto o saque quanto o débito automático podem tentar consultá-lo e modificá-lo ao mesmo tempo. A sequência de leitura do saldo, verificação de suficiência e gravação do novo valor constitui a **região crítica**. Segundo os autores (2016, p. 83-84), a exclusão mútua busca garantir que dois processos não executem simultaneamente suas regiões críticas sobre o mesmo recurso compartilhado.

Na seção dedicada aos mutexes, Tanenbaum e Bos (2016, p. 91-93) descrevem o mutex como uma trava que pode estar livre ou ocupada. Um processo ou thread tenta adquirir a trava antes de entrar na região crítica; se ela já estiver ocupada, precisa aguardar até que seja liberada. Depois de concluir a operação protegida, o processo libera o mutex.

### 2.3 Aplicação ao Cenário “O Saque Fantasma”
Considere uma conta bancária com saldo de **R$ 150,00**, que recebe duas operações simultâneas:
- saque de R$ 100,00;
- débito automático de R$ 80,00.

O total solicitado é de R$ 180,00, superior ao saldo disponível. Portanto, as duas operações não podem ser aprovadas com base no mesmo saldo inicial.

**Sem sincronização**, pode ocorrer a seguinte sequência:
1. O processo do saque lê o saldo de R$ 150,00.
2. O processo do débito automático também lê R$ 150,00 antes da atualização do primeiro processo.
3. Cada processo verifica separadamente que sua própria operação cabe no saldo lido.
4. Uma operação grava o saldo calculado, mas a outra pode sobrescrever essa atualização com um valor calculado a partir do saldo antigo.

Nesse exemplo, ambas as operações poderiam ser aprovadas embora, juntas, somem R$ 180,00. O saldo final poderia deixar de corresponder corretamente às movimentações registradas. Esse comportamento caracteriza uma condição de corrida e um problema de atualização perdida, conforme a lógica descrita por Tanenbaum e Bos (2016, p. 82-83).

**Com exclusão mútua**, as operações são serializadas para a conta em questão. Se o saque de R$ 100,00 for processado primeiro, o saldo passa a R$ 50,00 e o débito de R$ 80,00 é rejeitado por insuficiência de saldo. Se o débito de R$ 80,00 for processado primeiro, o saldo passa a R$ 70,00 e o saque de R$ 100,00 é rejeitado. A ordem efetiva deve respeitar as regras de negócio da instituição.

### 2.4 Arquitetura Recomendada
A proposta tem duas camadas complementares:

1. **Mutex na aplicação, quando necessário:** protege a região crítica entre threads que compartilham o mesmo mecanismo de sincronização dentro de um processo. Todas as rotinas da aplicação que alteram aquela conta precisam respeitar o mesmo bloqueio.
2. **Transação e bloqueio no banco de dados:** constituem o controle essencial quando existem várias instâncias da aplicação, processos separados ou servidores diferentes. Um mutex local não coordena automaticamente esses componentes. Em um banco de dados MySQL com mecanismo InnoDB, pode-se iniciar uma transação e fazer uma leitura bloqueante com `SELECT ... FOR UPDATE` sobre o registro da conta. A documentação do MySQL informa que essa leitura estabelece bloqueios exclusivos nos registros encontrados e que os bloqueios são liberados no `COMMIT` ou no `ROLLBACK` (ORACLE, [s. d.]).

Assim, em um sistema bancário distribuído, a garantia de consistência não deve depender somente de um mutex em memória de um dos servidores. O bloqueio transacional no banco de dados coordena as operações concorrentes que disputam o mesmo registro, desde que todas as rotinas relevantes sigam o protocolo de transação e bloqueio.

### 2.5 Fluxo de Execução Proposto

<img width="630" height="682" alt="Imagem1" src="https://github.com/user-attachments/assets/c809fac8-dc53-4c07-b65d-2702e977bf45" />

O diagrama apresenta o fluxo conceitual. Se ocorrer um erro durante a operação, a aplicação também deverá cancelar a transação e garantir que nenhuma atualização parcial permaneça confirmada.

### 2.6 Pseudocódigo da Solução com Controle Transacional

```text
função processarOperacaoFinanceira(contaId, valor, tipoOperacao):
    iniciar_transação_banco()
    
    tentar:
        // Bloqueio exclusivo no nível de banco de dados (Região Crítica)
        saldoAtual = executar_sql("SELECT saldo FROM contas WHERE id = :contaId FOR UPDATE")
        
        se conta_nao_encontrada():
            cancelar_transacao()
            retornar Erro("Conta não encontrada")
            
        se saldoAtual < valor:
            cancelar_transacao()
            retornar Erro("Saldo insuficiente")
            
        novoSaldo = saldoAtual - valor
        
        executar_sql("UPDATE contas SET saldo = :novoSaldo WHERE id = :contaId")
        executar_sql("INSERT INTO extrato (conta_id, valor, tipo) VALUES (:contaId, :valor, :tipoOperacao)")
        
        confirmar_transacao() // Libera o bloqueio exclusivo
        retornar Sucesso("Operação aprovada com sucesso")
        
    capturar_erro(erro):
        cancelar_transacao() // Garante ROLLBACK em caso de falha
        retornar Erro("Falha no processamento: " + erro.mensagem)
```

Em MySQL/InnoDB, uma implementação conceitual usa `START TRANSACTION`, `SELECT saldo FROM contas WHERE id = ? FOR UPDATE`, a validação do saldo, as instruções de atualização e o registro da movimentação, seguidos de `COMMIT`. Se a conta não tiver saldo suficiente ou se houver falha, a operação deve terminar com `ROLLBACK`. A leitura com `FOR UPDATE` deve ocorrer dentro de uma transação explícita; os bloqueios são mantidos até o encerramento dessa transação (ORACLE, [s. d.]).

Além da validação feita pela aplicação, é recomendável verificar se a atualização realmente afetou o registro esperado e manter o débito e o lançamento no extrato na mesma transação. Dessa forma, evita-se confirmar apenas uma parte da operação.

### 2.7 Cuidados da Implementação
Para assegurar a robustez da arquitetura proposta, os seguintes cuidados de engenharia devem ser observados na implementação prática:

- **Uniformidade nas Operações de Escrita:** Todos os fluxos que alteram o saldo devem seguir rigorosamente o mesmo protocolo de controle e bloqueio. Ignorar a transação em rotinas secundárias perpetua o risco de condições de corrida.
- **Abrangência Total da Operação Crítica:** O bloqueio transacional deve englobar a leitura, a validação de fundos e a gravação do novo saldo de forma indissociável.
- **Gerenciamento Adequado de Transações:** O término bem-sucedido deve sempre invocar `COMMIT` para consolidar as alterações, enquanto qualquer falha ou rejeição por saldo insuficiente deve acionar `ROLLBACK` para restaurar o estado consistente.
- **Tratamento de Deadlocks e Conflitos:** O sistema deve ser capaz de detectar impasses (*deadlocks*) gerados por concorrência em banco de dados, aplicando estratégias de nova tentativa (*retry*) com limites controlados.
- **Idempotência nas Requisições:** O uso de chaves de idempotência ou identificadores únicos por requisição impede que reenvios acidentais de rede gerem débitos duplicados na conta do cliente.

> **Nota:** Esses cuidados complementam a exclusão mútua e constituem recomendações arquiteturais essenciais para sistemas bancários distribuídos, não devendo ser interpretados como regras literais presentes nas páginas citadas de Tanenbaum e Bos.

### 2.8 Resultado Esperado
A solução proposta evita que o saque e o débito automático sejam aprovados com base no mesmo saldo desatualizado. Ao serializar as alterações da conta e tratar a atualização do saldo e o registro financeiro dentro da mesma transação, o sistema reduz o risco de atualizações perdidas e de divergência entre o saldo e as movimentações.

Conclui-se que o mutex é o mecanismo de sincronização que representa a exclusão mútua exigida pelo cenário, enquanto as transações e os bloqueios do banco de dados são necessários para sustentar a consistência quando as operações são executadas por diferentes processos ou servidores.

---

## 3. Relatório do Engenheiro de Documentação e Qualidade: Instrumentos Visuais, Pesquisa Multimídia 

### 3.1 Tabela de Auditoria de Paginação e Conformidade (Tanenbaum & Bos, 4ª Edição)
Auditoria e validação rigorosa da indicação das páginas e seções consultadas no livro-base para sustentação da argumentação teórica:

| Conceito / Tópico | Seção do Livro | Páginas Citadas | Aplicação no Caso "O Saque Fantasma" |
| :--- | :--- | :--- | :--- |
| **Condições de Corrida** | Seção 2.3.1 | p. 82 – 83 | Explica a dependência do resultado com a ordem de execução das operações sobre o saldo bancário. |
| **Regiões Críticas e Exclusão Mútua** | Seção 2.3.2 | p. 83 – 84 | Define as regras para proibir que duas threads executem a leitura/escrita do saldo simultaneamente. |
| **Tentativas de Solução e Instruções Atômicas** | Seção 2.3.3 / 2.3.4 | p. 84 – 88 | Apresenta a necessidade de suporte de hardware (instruções TSL/XCHG) para exclusão mútua sem falhas. |
| **Semáforos** | Seção 2.3.5 | p. 89 – 91 | Apresenta a primitiva de controle de acesso de Dijkstra baseada nas operações atômicas `down` e `up`. |
| **Mutexes e Futexes** | Seção 2.3.6 | p. 91 – 93 | Fundamenta o uso de trava binária sem espera ocupada (*busy waiting*) como primitiva de sincronização. |
| **Escalonamento Preemptivo** | Seção 2.4.1 | p. 106 | Descreve o papel da interrupção de relógio na preempção que causa a perda de atualização (*lost update*). |
| **Chaveamento de Contexto** | Seção 2.4.2 | p. 110 | Explica a troca de registradores da CPU e salvamento do PCB durante a transição entre threads. |

---

### 3.2 Referências Bibliográficas 

CABEÇA DE LAB. **Episódio 38: Concorrência e Paralelismo**. [S. l.]: LuizaLabs, 2020. 1 podcast (42 min). Disponível em: <https://www.cabecadelab.com.br/episodios/38-concorrencia-e-paralelismo.html>. Acesso em: 27 out. 2026.

GUEDES, Dorgival. **FSPD 02c - Exclusão Mútua**. Belo Horizonte: DCC UFMG, 19 jun. 2020. 1 vídeo (15 min). Publicado pelo canal DCC UFMG. Disponível em: <https://www.youtube.com/watch?v=JOGQo6Q6y_Q>. Acesso em: 24 out. 2026.

ORACLE. **MySQL 8.4 Reference Manual: 17.7.2.4 Locking Reads**. [S. l.]: Oracle, [s. d.]. Disponível em: <https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html>. Acesso em: 22 out. 2026.

SANTOS, Fernando dos. **Programação Concorrente: Condições de Corrida (race conditions)**. [S. l.]: Canal Fernando dos Santos, 22 mar. 2021. 1 vídeo (14 min). Publicado pelo canal Fernando dos Santos. Disponível em: <https://www.youtube.com/watch?v=er1FZ8hara8>. Acesso em: 25 out. 2026.

TANENBAUM, Andrew S.; BOS, Herbert. **Sistemas operacionais modernos**. 4. ed. São Paulo: Pearson Education do Brasil, 2016.
