# Relato do grupo

Integrantes: Gabriel Felipe Kuhn Schiestl, Gabriel Favero Heller, Kauã Matheus da Silva
Data: 11/10/2026
Distribuição Linux: não usamos Linux, rodamos tudo no Windows 11 com PowerShell
Python / Docker / Minikube: Python 3.12.8 / Docker 27.3.1 (Docker Desktop) / Minikube v1.39.0

## Etapas

| Etapa | Comando e evidência | Resultado esperado | Resultado observado e explicação |
|---|---|---|---|
| Testes | `python -m unittest discover -s tests -v` (`evidencias/testes.txt`) | 8 testes OK | Rodou os 8 testes de primeira e deu tudo OK. Tem teste até pra versão quebrada, que confere se ela realmente devolve 503 - faz sentido, porque esses testes não dependem do cluster estar de pé. |
| Publicação v1 | build com `APP_VERSION=v1`, `minikube image load`, `render.py v1 --replicas 2`, apply, rollout status (`evidencias/v1-pods.txt`), smoke test (`v1-status.json`) e print (`v1-pagina.png`) | 2 Pods prontos; API v1 | Os 2 Pods subiram certinho (READY 1/1). A API devolveu version v1 e ready true, smoke test passou. No print dá pra ver a versão, a mensagem do ConfigMap e o nome do Pod que respondeu. |
| Escala | `render.py v1 --replicas 3`, apply, rollout status (`replicas-3.txt`) | 3 Pods prontos | Demorou uns segundos pro terceiro Pod ficar pronto (ficou aparecendo "2 of 3 available" até estabilizar), mas no fim os 3 ficaram OK. |
| Exclusão e recuperação | `kubectl delete pod` no pod mais antigo (nome anotado em `pod-excluido.txt`), estado depois em `replicas-apos-exclusao.txt` | Nome antigo some, Pod novo aparece | Excluímos o pod e o Kubernetes criou outro na hora pra manter as 3 réplicas. É um Pod novo mesmo, nome e IP diferentes - ele não "recupera" o pod que caiu, só cria outro igual no lugar porque o Deployment fica de olho na quantidade que a gente pediu. |
| ConfigMap | mudamos a mensagem no `k8s/configmap.yaml`, apply, testamos antes do restart (ainda veio a mensagem velha, `config-status-raw.txt`), depois `rollout restart` e testamos de novo (`config-status.json`) | Mensagem muda só depois de recriar os Pods | Confirmado: logo depois do apply a API ainda respondia com a mensagem antiga, porque os Pods que já estavam rodando tinham lido a variável de ambiente só uma vez, quando iniciaram. Só depois do restart que a mensagem nova apareceu. |
| Atualização v2 | build `APP_VERSION=v2`, `render.py v2 --replicas 3`, apply, rollout status (`v2-pods.txt`), smoke test (`v2-status.json`), revisão salva em `build/revisao-ok.txt` | 3 Pods v2 prontos, revisão salva | Os 3 Pods da v2 subiram, API respondeu v2 com a mensagem certa e ready true. A revisão salva foi a 3 (não a 2 como a gente imaginava), porque o restart do passo anterior já tinha gerado uma revisão nova mesmo sem trocar a imagem. |
| Incidente | build `APP_VERSION=quebrada`, apply, rollout status com timeout de 30s (deu erro, como esperado), describe e events salvos; achamos o pod novo e salvamos describe/logs dele; port-forward direto nele na 8081 pra testar /healthz e /readyz | Pod novo fica Running 0/1, healthz 200 e readyz 503 | Bateu exatamente com o esperado. O rollout deu timeout depois de 30s (mas isso é só o comando desistindo de esperar, o Deployment continuou tentando sozinho). Ficaram 3 Pods da v2 rodando normal e 1 Pod novo com READY 0/1. Nos eventos aparece "Readiness probe failed ... statuscode: 503". Testando direto no Pod: /healthz voltou 200 (tá vivo), /readyz voltou 503 (não tá pronto). Então o Service nunca manda tráfego pra ele, mesmo ele estando de pé. |
| Rollback | `rollout undo --to-revision=3`, depois corrigimos o `build/deployment.yaml` de volta pra v2 (senão ele reaplicaria a quebrada de novo) e reaplicamos; smoke test (`rollback-status.json`) e print (`rollback-pagina.png`) | v2 volta a funcionar | Deu certo, voltou tudo pra v2 com a mensagem certa. Importante: o rollback só mexe no "molde" dos Pods, não desfaz o resto - o ConfigMap continuou com a mensagem nova (porque rollback não toca em ConfigMap) e o arquivo gerado localmente a gente teve que corrigir na mão, porque ele continuava apontando pra versão quebrada. |
| Pipeline v3 | `python scripts/pipeline_local.py v3 3`, saída toda salva em `pipeline-v3.txt` | Smoke OK e v3 rodando com 3 réplicas | Rodou os 8 testes, buildou a imagem v3, deu load no Minikube, aplicou os manifestos, esperou o rollout e fez o smoke test sozinho, numa porta temporária (18080). Terminou com "SMOKE OK" e "Pipeline concluida: v3, replicas=3". Estado final salvo em `final-pods.txt` e `final-deployment.yaml`. |

## Respostas

**1. Por que carregar a imagem no Minikube depois de construir no Docker?**
Porque são dois lugares de armazenamento diferentes. O `docker build` só deixa a imagem disponível no Docker do nosso PC. O nó do Minikube é uma espécie de "máquina" separada e não enxerga essas imagens automaticamente. Como o Deployment usa `imagePullPolicy: Never`, ele não vai atrás de baixar a imagem de lugar nenhum - só usa o que já estiver lá dentro. Por isso precisa do `image load`, senão o Pod fica preso em ErrImageNeverPull.

**2. Qual a função das portas 8000, 80, 30080 e 8080?**
A 8000 é onde o servidor Python escuta dentro do contêiner. A 80 é a porta do Service lá dentro do cluster. A 30080 seria a porta NodePort, pra acessar pelo nó diretamente (a gente não chegou a usar essa na prática). E a 8080 é a porta que abrimos no nosso próprio PC com o port-forward, que fica encaminhando pra porta 80 do Service.

**3. Por que o Pod excluído foi substituído? Ele tem a mesma identidade do anterior?**
Não, é um Pod diferente. O Deployment fica sempre comparando quantos Pods tem rodando com quantos a gente pediu (3, no caso). Quando excluímos um, ele percebe que ficou faltando um e sobe outro do zero - nome novo, IP novo, idade zero. Não é o mesmo Pod "ressuscitado", é uma cópia nova do mesmo modelo.

**4. Por que três Pods no mesmo nó não protegem contra a perda da máquina?**
Porque no nosso caso o Minikube só tem um nó só, então as 3 réplicas estão todas rodando dentro da mesma máquina. Se essa máquina cair, caem as 3 juntas - não importa se são 3 ou 30 réplicas. Pra isso proteger de verdade precisaria de vários nós espalhados. E de qualquer forma, réplica não é backup de dado.

**5. Por que o ConfigMap precisou de novos Pods para mudar a mensagem?**
Porque a aplicação só lê essa variável de ambiente uma vez, quando o processo inicia. Atualizar o ConfigMap no cluster não "empurra" o valor novo pra dentro de um processo que já está rodando. Por isso só depois do rollout restart (que troca os Pods) é que a mensagem nova apareceu.

**6. Tag de imagem e revisão do Deployment são a mesma coisa?**
Não. A tag (v1, v2, quebrada...) é só o nome/versão da imagem Docker. Já a revisão é um contador interno do Deployment que sobe toda vez que o modelo dos Pods muda - e isso pode acontecer sem trocar de imagem nenhuma, como aconteceu com a gente: o restart do ConfigMap gerou uma revisão nova mesmo ainda usando a v1. Foi por isso que a revisão "boa" salva pra v2 acabou sendo a 3, não a 2.

**7. Quais sinais mostram que a entrega defeituosa estava viva, mas não pronta?**
O /healthz respondendo 200 mostra que o processo está rodando normalmente. Já o /readyz respondendo 503, junto com o READY aparecendo 0/1 no kubectl get pods, mostra que ele não quer receber tráfego. Nos eventos do cluster também aparece claramente "Readiness probe failed". Então dá pra afirmar com segurança: o contêiner não travou nem reiniciou, só está dizendo pro Kubernetes "não me manda pedido agora".

**8. O que o rollback recuperou? Por que foi necessário ajustar também o arquivo gerado?**
O timeout do rollout só faz o comando parar de ficar esperando - ele não desfaz nada sozinho, o Kubernetes não reverte automaticamente por causa disso. Quem realmente reverteu foi o `rollout undo`. Ele trouxe de volta o modelo de Pods (a imagem v2), mas não voltou o resto do ambiente: o ConfigMap continuou como estava (com a mensagem nova) e o arquivo `build/deployment.yaml` que a gente tinha gerado continuava descrevendo a versão quebrada. Se aplicássemos esse arquivo de novo sem corrigir, ia pedir a versão quebrada outra vez e estragar o rollback que a gente acabou de fazer.

**9. Que verificações a pipeline faz e quais problemas ainda poderiam ocorrer mesmo após sua conclusão?**
Ela roda os testes unitários, faz o build da imagem, dá load no Minikube, aplica os manifestos, espera o rollout terminar e faz um teste HTTP rápido no final. Se qualquer uma dessas etapas falhar ela já para e não tenta consertar sozinha. O limite é que, mesmo com tudo passando, ela não pega problema que só aparece com carga real, com muita gente acessando ao mesmo tempo, ou depois de rodar por mais tempo. E como é tudo local, não tem CI de verdade, não builda a partir de um repositório remoto, nem publica em nenhum registry - é só pra praticar o fluxo mesmo.

## Sobre a automação (pipeline local)

A gente chama de "pipeline local" porque ela automatiza a sequência toda (teste, build, deploy, verificação), só que roda na nossa própria máquina e a gente que dispara ela manualmente no terminal. Não tem servidor de CI, não tem gatilho automático, não tem repositório remoto por trás. Pra virar uma esteira de CI/CD de verdade, faltaria pelo menos: disparar sozinha quando alguém faz push/merge no Git, rodar num servidor compartilhado (não na máquina de quem está codando), publicar a imagem num registry que o cluster real consiga acessar (em vez do image load local), e ter algum tipo de aprovação antes de ir pra produção. Nada disso a gente implementou, só o fluxo local mesmo, como pedido no roteiro.

## Divergências e correções

Não tivemos nenhum problema que fugisse do esperado no roteiro. A única coisa foi que, antes de começar, precisamos abrir o Docker Desktop manualmente (não estava rodando) e instalar o Minikube (não estava instalado na máquina). Depois disso resolvido, seguiu tudo normal, inclusive o timeout esperado na entrega quebrada e a recuperação via rollback.
