---
nav_order: 12
---

# Troubleshooting

- **422 Unprocessable Entity: `size` empty**
  - Ensure effective size logic (`vEffSize`) never passes empty string.

- **500 Failed to generate embedding: empty query**
  - Guard query: trim & ensure non-empty before calling `/search`.

- **404 Not Found on `/documents`**
  - Check base URL or index name; endpoint path must be `/documents`.

- **Qlik parser: `Unexpected token` near `IsNum()`**
  - Never call `IsNum()` on an empty expansion. Use:
    - `IF Len(Trim('$(var)'))>0 THEN LET x = Alt(Num#('$(var)'), 0); END IF`

- **`Found an END IF statement, but there is no open IF`**
  - Balance `IF/END IF`. Watch for macro expansions breaking syntax.

- **Connection not found**
  - Ensure the exact connection name exists in Qlik and is accessible.
