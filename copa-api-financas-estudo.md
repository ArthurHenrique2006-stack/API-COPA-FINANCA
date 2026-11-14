# Copa da API Finanças — Estudo

**Disciplina:** Desenvolvimento Web Back-end (Node.js)
**Material:** https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html

---

## Treino 1 — Códigos de status

| # | Situação | Status | Justificativa |
|---|----------|--------|---------------|
| 1 | `POST /categorias` com nome já existente para o mesmo tipo | **409** | Conflito: nome repetido |
| 2 | `PUT /fontes-destinos/12` com `tipo: "poupanca"` | **400** | Valor de campo inválido (formato/regra violada) |
| 3 | `GET /relatorios/categorias` sem `inicio` e `fim` | **400** | Parâmetros obrigatórios ausentes |
| 4 | `POST /auth/login` com senha errada | **401** | Credenciais inválidas |
| 5 | `GET /rota-que-nao-existe` | **404** | Rota inexistente |
| 6 | `POST /lancamentos` válido | **201** | Recurso criado |

---

## Treino 2 — Caça ao bug

```javascript
// routes/fontesDestinos.js
router.delete('/:id', async (req, res) => {
  const [r] = await db.query(
    `DELETE FROM fontes_destinos WHERE id = ${req.params.id} AND usuario_id = ${req.auth.id}`
  );
  if (r.affectedRows === 0) {
    res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
  }
  res.status(204).end();
});
```

**Erros encontrados (3):**

1. **SQL montado com template string** (`${req.params.id}`): abre brecha para SQL injection. Regra violada: consultas sempre parametrizadas com `?`. *(erro principal)*
2. **Falta `return` depois da resposta de erro**: quando não encontra o registro, responde 404 e depois tenta responder 204 (duas respostas para a mesma requisição, `ERR_HTTP_HEADERS_SENT`).
3. **SQL dentro da rota**: o SQL deve ficar somente em `models/`.

**Correção:**

```javascript
// models/fontesDestinos.js
async function excluir(id, usuarioId) {
  const [r] = await db.query(
    'DELETE FROM fontes_destinos WHERE id = ? AND usuario_id = ?',
    [id, usuarioId]
  );
  return r.affectedRows > 0;
}
module.exports = { excluir };

// routes/fontesDestinos.js
router.delete('/:id', protegerRota, async (req, res) => {
  const removido = await model.excluir(req.params.id, req.auth.id);
  if (!removido) {
    return res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
  }
  return res.status(204).end();
});
```

> Observação: se a fonte/destino tiver lançamentos vinculados, a exclusão deve responder **409** (tratar o erro de chave estrangeira).

---

## Treino 3 — Parcelas em centavos

R$ 250,00 em 7×:

- `totalCentavos = 25000`
- `base = floor(25000 / 7) = 3571`
- `resto = 25000 − 3571 × 7 = 25000 − 24997 = 3`
- 1ª parcela: `3571 + 3 = 3574` → **R$ 35,74**
- Parcelas 2 a 7: **R$ 35,71** cada

**Conferência:** 35,74 + 6 × 35,71 = 35,74 + 214,26 = **R$ 250,00** ✔

---

## Treino 4 — Fatura (fechamento dia 18, vencimento dia 3)

Compra de R$ 250,00 em 2× em 18/08/2026.

- Compra **no dia** do fechamento → entra na fatura do **mês seguinte**.
- Vencimento (3) **menor** que fechamento (18) → vence no **mês seguinte** ao fechamento.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---------|-------|------------|------------|------------|
| 1/2 | R$ 125,00 | 18/09/2026 | 03/10/2026 | 2026-10 |
| 2/2 | R$ 125,00 | 18/10/2026 | 03/11/2026 | 2026-11 |

---

## Treino 5 — Fatura (fechamento dia 31, vencimento dia 8)

Compra de R$ 120,00 em 2× em 31/01/2026.

- Compra **no dia** do fechamento (31) → fatura do **mês seguinte** (fevereiro).
- Fevereiro de 2026 não tem dia 31 → fechamento vira o **último dia do mês (28/02)**.
- Vencimento (8) **menor** que fechamento (31) → vence no mês seguinte ao fechamento.

| Parcela | Valor | Fechamento | Vencimento | Referência |
|---------|-------|------------|------------|------------|
| 1/2 | R$ 60,00 | 28/02/2026 | 08/03/2026 | 2026-03 |
| 2/2 | R$ 60,00 | 31/03/2026 | 08/04/2026 | 2026-04 |

---

## Treino 6 — Por que `UNIQUE` no banco?

O `SELECT` antes do `INSERT` não é atômico: duas requisições simultâneas com o mesmo e-mail podem ambas verificar que ele não existe e ambas inserir, gerando duplicidade (condição de corrida). O `UNIQUE` faz o próprio banco garantir a regra, mesmo sob concorrência, e vale para qualquer caminho de escrita. A API captura o erro `ER_DUP_ENTRY` e responde **409**.

---

## Carta-bug (equipe)

**Arquivo:** models/lancamentos.js e routes/lancamentos.js

```javascript
// models/lancamentos.js
async function buscarPorId(id) {
  const [rows] = await db.query('SELECT * FROM lancamentos WHERE id = ?', [id]);
  return rows[0];
}

// routes/lancamentos.js
router.get('/:id', protegerRota, async (req, res) => {
  const lancamento = await model.buscarPorId(req.params.id);
  if (!lancamento) {
    return res.status(404).json({ erro: 'Lançamento não encontrado.' });
  }
  return res.status(200).json(lancamento);
});
```

**Erro (linha):** `SELECT * FROM lancamentos WHERE id = ?` não filtra por `usuario_id`.
**Regra violada:** RF3 (isolamento entre usuários): qualquer usuário autenticado consegue ler lançamento de outro.
**Correção:**

```javascript
async function buscarPorId(id, usuarioId) {
  const [rows] = await db.query(
    'SELECT * FROM lancamentos WHERE id = ? AND usuario_id = ?',
    [id, usuarioId]
  );
  return rows[0];
}
// na rota: model.buscarPorId(req.params.id, req.auth.id)
```

---

## Checklist

- [x] Tabela de códigos de status revisada
- [x] `req.auth.id` como origem do usuário; outro usuário recebe 404 (um 403 confirmaria que o registro existe)
- [x] Parcelas em centavos e fatura de cada parcela
- [x] `senha_hash` nunca na resposta; `JWT_SECRET` no `.env`
- [x] Seis treinos resolvidos
- [ ] Carta-bug enviada pelo Moodle
