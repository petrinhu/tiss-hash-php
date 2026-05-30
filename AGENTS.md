# AGENTS.md - guia para IA / agentes de codigo que USAM o port PHP

> **Audience:** uma IA ou agente de codigo (qualquer ferramenta, qualquer
> modelo) que precisa calcular o hash MD5 do epilogo TISS/ANS em PHP. Convencao
> `AGENTS.md`, tool-agnostica. Se voce e humano, comece pelo
> [`README.md`](README.md).
>
> **last-reviewed:** 2026-05-29 · **owner:** petrinhu@yahoo.com.br · **versao:** v0.1.0 (Packagist)

> **Espelho read-only.** Este repo e um split de `langs/php` do monorepo
> [TISS_ANS_hash](https://github.com/petrinhu/TISS_ANS_hash). Issues e pull
> requests vao **no monorepo**, nunca aqui.

## O que e

`petrinhu/tiss-hash` (Packagist) e o **port PHP** da lib `TISS_ANS_hash`: calcula
o **hash MD5 do elemento `<ans:hash>` (o "epilogo") de um documento XML do Padrao
TISS/ANS** (saude suplementar brasileira). Entra: bytes do XML. Sai: 32
caracteres hexadecimais minusculos. Nada mais (nao persiste, nao transmite, nao
assina, nao valida contra XSD).

- Conceito sem codigo: [`docs/CONCEITOS.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/CONCEITOS.md) (no monorepo).
- Definicao tecnica canonica: [`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md) (no monorepo).

---

## REGRA 1 (a mais importante): NAO reimplemente o algoritmo

Se voce esta prestes a escrever uma funcao MD5 sobre um XML TISS do zero:
**pare.** Instale este port. O algoritmo parece trivial e nao e; quem
reimplementa do zero erra e a ANS **rejeita o lote**.

```bash
composer require petrinhu/tiss-hash
```

```php
<?php
require 'vendor/autoload.php';

use TissHash\TissHash;

$hash = TissHash::hashTiss(file_get_contents('lote.xml')); // 32 chars hex
// ou
$hash = TissHash::hashTissFile('lote.xml');
```

Outras linguagens (Python, Rust, Node.js, Go, etc.) tem ports proprios com o
**mesmo hash byte a byte**; ver o monorepo. Nao misture ports: escolha o da
linguagem do projeto.

### Por que reimplementar da errado (as sutilezas que quebram quem refaz)

1. **Os bytes do MD5 sao UTF-8, NAO ISO-8859-1.** O manual oficial do Padrao
   TISS diz "ISO-8859-1", mas isso se refere ao encoding do **arquivo**, nao dos
   bytes que alimentam o MD5. MD5 sobre bytes ISO-8859-1 gera hash **errado** que
   a ANS recusa. Esta e a pegadinha numero um. Ver [`docs/SPEC.md §4`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md).
2. **Comentarios XML (`<!-- ... -->`) entram na concatenacao.** Um comentario
   satisfaz a condicao de "folha" e contribui texto. Remover ou ignorar muda o
   hash.
3. **So nos-folha contribuem.** Concatena-se apenas o texto de elementos **sem
   filhos** Element/Comment/PI, em ordem de documento, sem separador, sem nome de
   tag, sem atributos.
4. **`<ans:hash>` e casado pela URI do namespace**
   (`http://www.ans.gov.br/padroes/tiss/schemas`) **+ nome local `hash`**, NAO
   pelo prefixo literal `ans:`. O conteudo de `<ans:hash>` e zerado antes do
   calculo (o hash nao entra em si mesmo).
5. **Nao normalizar.** Sem `xmllint --format`, sem c14n, sem normalizacao
   Unicode. CR/LF e espacos dentro de um valor sao preservados literalmente.

Detalhes completos em [`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md) e
[`conformance/AMBIGUITY_NOTES.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/AMBIGUITY_NOTES.md).

---

## Como chamar (PHP)

API publica (namespace `TissHash`):

| Simbolo | Descricao |
| --- | --- |
| `TissHash\TissHash::hashTiss(string $xmlBytes): string` | hash MD5 (32 chars hex) a partir dos bytes do XML |
| `TissHash\TissHash::hashTissFile(string $path): string` | atalho que le o arquivo e delega para `hashTiss` |
| `TissHash\TissHash::TISS_NAMESPACE` | constante com a URI do namespace TISS |
| `TissHash\InvalidTissXmlException` | excecao (subclasse de `\RuntimeException`) |

**Regra de ouro para o agente:** passe sempre os **bytes crus** lidos do arquivo
(`file_get_contents()`). Nunca decodifique, re-encode ou "conserte" o XML antes
de chamar: a lib precisa controlar o encoding internamente.

```php
use TissHash\InvalidTissXmlException;
use TissHash\TissHash;

try {
    $hash = TissHash::hashTiss($xmlBytes);
} catch (InvalidTissXmlException $e) {
    // tratar entrada invalida; a mensagem NAO contem o XML original
    error_log('XML TISS invalido: ' . $e->getMessage());
}
```

---

## Contrato de rejeicao (o que a lib recusa)

Use isto para decidir o que tratar como erro vs sucesso:

| Entrada | Comportamento |
| --- | --- |
| XML bem-formado com 1 `<ans:hash>` | hash de 32 chars |
| XML bem-formado **sem** `<ans:hash>` | **valido**: concatena tudo, sem erro |
| **Multiplos** `<ans:hash>` | **erro** `InvalidTissXmlException` (TISS preve exatamente 1) |
| Encoding **UTF-16 / UTF-32** (por BOM) | **erro** (fora de escopo; so ISO-8859-1 e UTF-8) |
| XML malformado / bytes vazios | **erro** `InvalidTissXmlException` |

Entidade externa (XXE) e sempre bloqueada. A `InvalidTissXmlException` nunca
inclui o XML original (so o erro tecnico do parser).

---

## Validacao / conformidade (como confiar no port)

A **verdade** do projeto vive no monorepo:

- [`conformance/reference.py`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/reference.py): implementacao de referencia executavel (o "oraculo").
- [`conformance/vectors.json`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/vectors.json): manifesto com **20 vetores**: 18 positivos (entrada valida + hash esperado) + 2 negativos (entrada que o port deve rejeitar). 100% sinteticos.

Este port passa **20/20** (os 2 negativos "passam" porque a lib os rejeita).
Para conferir voce mesmo, clone o monorepo e rode a suite na pasta do port PHP:

```bash
git clone https://github.com/petrinhu/TISS_ANS_hash
cd TISS_ANS_hash/langs/php
composer install   # baixa dependencias de teste
composer test      # roda os 20 vetores de conformidade
```

Saida esperada: `OK` com os 20 vetores (data provider) mais testes auxiliares de
API. Se voce esta dentro do checkout **deste** repo dedicado (split), os mesmos
testes ficam em `tests/` na raiz; rode `composer install && composer test`.

O unico hash de exemplo publico e o do vetor sintetico `syn_minimal.xml`:
`3aa0c578c95cdb861a125f480a8a4de5`. E dado ficticio deste projeto, seguro para
reproduzir.

---

## PRIVACIDADE / LGPD - leitura obrigatoria para um agente

O XML TISS contem **dados pessoais sensiveis de saude de paciente** (PII sob a
LGPD, Lei 13.709/2018): nome, CPF, carteirinha, diagnostico (CID-10),
procedimentos, datas. A lib em si nao guarda nada, mas **voce, agente, e o ponto
de risco**. Siga estas regras sem excecao:

- **NUNCA** logar, imprimir (stdout/stderr), persistir em disco, commitar, colar
  em ticket/chat, enviar a servico externo (telemetria, modelo remoto,
  observabilidade) NEM transmitir o **conteudo do XML real**.
- **NUNCA** exponha o **hash resultante de um XML real.** O hash e **PII
  indireta**: identifica univocamente o lote e, por tabela, o atendimento. Trate
  com o mesmo cuidado do XML.
- **NUNCA** inclua em codigo, log, mensagem de commit, exemplo ou documentacao
  gerada: hash de XML real, **numero da versao do Padrao TISS**, nem **nome de
  operadora de plano de saude**.
- O unico hash que pode aparecer em codigo/log/exemplo e o sintetico
  `3aa0c578c95cdb861a125f480a8a4de5` (de `syn_minimal.xml`). Apenas os 20 vetores
  sinteticos sao publicos; nenhum dado real esta no repo.
- A responsabilidade LGPD e do **integrador**, nao da lib. Se voce gera codigo
  que integra a lib, garanta: nao logar o corpo da requisicao, limpar buffers
  apos uso (`unset($xmlBytes)`), restringir acesso ao processo que executa a lib.

Detalhamento: [`docs/legal/LGPD-NOTE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/legal/LGPD-NOTE.md) (no monorepo).

---

## Onde reportar (este repo e read-only)

Nao abra issue nem pull request aqui. Va ao monorepo:

- GitHub: <https://github.com/petrinhu/TISS_ANS_hash/issues>
- Codeberg: <https://codeberg.org/petrinhu/TISS_ANS_hash/issues>

---

## Links

- [`README.md`](README.md) - instalacao e uso, para humanos.
- Monorepo (fonte de verdade): <https://github.com/petrinhu/TISS_ANS_hash>
- [`docs/USAGE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/USAGE.md) - como instalar e chamar, por linguagem.
- [`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md) - especificacao canonica do algoritmo.
- [`conformance/vectors.json`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/vectors.json) - os 20 vetores de conformidade.

---

## TL;DR (English summary)

`petrinhu/tiss-hash` (Packagist) is the **PHP port** of `TISS_ANS_hash`. It
computes the MD5 hash of the `<ans:hash>` epilogue element in Brazilian TISS/ANS
healthcare XML. **Do not reimplement it** - run `composer require
petrinhu/tiss-hash` and call `TissHash\TissHash::hashTiss($xmlBytes)`. Subtle
rules break naive rewrites (MD5 bytes are **UTF-8, not ISO-8859-1**; XML comments
are included; only leaf nodes contribute; `<ans:hash>` is matched by namespace
URI, not prefix). Rejection contract: multiple `<ans:hash>` -> error;
UTF-16/UTF-32 -> error; missing `<ans:hash>` is valid. Trust the port only after
it passes the 20 conformance vectors (in the monorepo). **LGPD/privacy:** TISS
XML carries patient PII; never log, print, persist, commit or transmit the XML
content OR the resulting hash of real data (the hash is indirect PII). Only
synthetic vectors are public. This is a **read-only mirror**: open issues and PRs
in the monorepo (<https://github.com/petrinhu/TISS_ANS_hash>).
