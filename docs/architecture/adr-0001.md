# ADR 0001 - Usar bloqueio otimista para reservar o AGV

**Status:** Aprovado

## Contexto

Quando dois operadores despacham entregas quase ao mesmo tempo, a API pode ler o mesmo AGV como disponível nas duas requisições e reservar ele para duas missões diferentes. Aí uma das missões fica esperando um AGV que nunca vai aparecer.

A gente pensou em usar `SELECT ... FOR UPDATE` (bloqueio pessimista), mas ele deixa a linha travada enquanto a API espera a resposta do Planejador de Rotas, o que pode demorar e travar outros despachos.

## Decisão

Usar **bloqueio otimista** com uma coluna `versao` na tabela do AGV no PostgreSQL. A reserva só funciona se a versão ainda for a mesma que foi lida:

```sql
UPDATE agv SET status = 'RESERVADO', versao = versao + 1
WHERE id = :agvId AND status = 'DISPONIVEL' AND versao = :versao;
```

Se nenhuma linha for atualizada, quer dizer que outro despacho pegou o AGV primeiro, e o sistema tenta o próximo AGV da lista.

## Consequências

- **Vantagens:** não trava o banco, funciona com várias instâncias da API rodando e não precisa de nenhuma ferramenta nova.
- **Desvantagens:** quando tem muito despacho ao mesmo tempo, aumentam as tentativas. Todo `UPDATE` precisa lembrar de usar a versão, senão a proteção não funciona.
