# Encriptación de datos sensibles en reposo

Todo dato sensible en BD se protege en reposo. El mecanismo depende de si el
dato debe recuperarse en claro o no — no son intercambiables.

## Hashing (irreversible) — passwords

- Hash de una vía (BCrypt), generado/verificado en la capa Service.
- Columna: `VARCHAR(200)` (el hash, no la contraseña).
- El SP nunca compara password en claro — la verificación ocurre en el Service.
- El hash nunca se expone al frontend ni entra en un `ResponseDto`.

## Encriptación reversible — datos recuperables

Para campos que sí deben recuperarse en claro (cuenta bancaria, documento de
identidad, PII regulada):

- BD almacena el valor ya cifrado como string, nunca en claro.
- Cifrado/descifrado en la capa Service — no usar `ENCRYPTBYKEY`/`DECRYPTBYKEY` en T-SQL.
- Algoritmo por defecto: AES-256-CBC.

## IDs expuestos al frontend

Caso particular de encriptación reversible (evitar IDs enumerables en URLs, no
proteger un dato confidencial):

- PK/FK en BD siguen siendo `INT` normal — nunca se procesan como string cifrado en SQL.
- Todo `Sp_Ins` retorna `@{entity}_id INT OUTPUT`: el Service necesita el INT
  crudo para encriptarlo antes de responder al frontend.
- `Sel`/`Get` devuelven el ID como `INT` plano — se encripta al mapear al DTO.
- El frontend envía el ID cifrado; el Service lo desencripta antes de llamar al SP.

## Regla dura

La BD nunca ejecuta cifrado/hashing — ni de IDs, ni passwords, ni otros
campos. Todo mecanismo criptográfico vive en la capa Service; la BD solo
almacena el resultado (hash o cifrado) como string.
