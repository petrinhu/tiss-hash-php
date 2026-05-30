<!--
  AVISO: espelho read-only. NAO edite este arquivo aqui.
  E gerado a partir de langs/php do monorepo TISS_ANS_hash.
  Pull requests e issues vao no monorepo, nao neste repo.
-->

> **Espelho read-only.** Este repositorio e um split (gerado a partir de
> `langs/php` do monorepo
> [TISS_ANS_hash](https://github.com/petrinhu/TISS_ANS_hash)).
> **Nao edite aqui.** Pull requests e issues devem ser abertos no monorepo.
> Este repo existe apenas como veiculo de publicacao no Packagist, o repositorio
> oficial de pacotes PHP, como
> [`petrinhu/tiss-hash`](https://packagist.org/packages/petrinhu/tiss-hash).

# tiss-hash (PHP)

[![Packagist Version](https://img.shields.io/packagist/v/petrinhu/tiss-hash)](https://packagist.org/packages/petrinhu/tiss-hash)
[![Packagist Downloads](https://img.shields.io/packagist/dt/petrinhu/tiss-hash)](https://packagist.org/packages/petrinhu/tiss-hash)
[![PHP Version](https://img.shields.io/packagist/php-v/petrinhu/tiss-hash)](https://packagist.org/packages/petrinhu/tiss-hash)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> Port PHP da lib `TISS_ANS_hash`: calcula o hash MD5 do epilogo de um XML do
> Padrao TISS/ANS (saude suplementar do Brasil). Voce passa os bytes do XML,
> recebe os 32 caracteres do hash.

## O que e (sem jargao)

Calcula a "impressao digital" do trecho final de um documento TISS/ANS. Os
termos antes do codigo:

- **XML**: formato de arquivo de texto que organiza dados em etiquetas (tags)
  aninhadas, como caixas dentro de caixas. O Padrao TISS e o XML que operadoras
  de saude e consultorios usam no Brasil para trocar dados de atendimento
  (regulamentado pela ANS, a Agencia Nacional de Saude Suplementar).
- **Hash**: sequencia curta e fixa de caracteres calculada a partir de um
  texto, como uma impressao digital. Mude uma letra, o hash muda inteiro.
- **MD5**: a receita (algoritmo) que gera o hash; sempre 32 caracteres
  hexadecimais (`0-9` e `a-f`).
- **Epilogo**: a parte final do documento TISS, a etiqueta `<ans:hash>`, onde o
  hash precisa ser gravado.
- **Parser**: o componente que le o texto do XML e monta a arvore de etiquetas
  na memoria. Aqui o parser e o `DOMDocument`, nativo do PHP.

Em uma frase: voce passa os bytes de um XML TISS e recebe os 32 caracteres do
hash. (Um **byte** e a menor unidade de dado do computador.) Este e o port PHP
("port" = a mesma lib reescrita em outra linguagem). Outras linguagens (Python,
Rust, Node.js, Go, etc.) seguem o mesmo contrato e os mesmos vetores de
conformidade, e produzem o **mesmo hash byte a byte**.

Para entender o problema que a lib resolve, veja
[`docs/CONCEITOS.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/CONCEITOS.md) (o que e e para que serve, sem codigo) e
[`docs/USAGE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/USAGE.md) (guia de uso).

- Repositorio principal (fonte de verdade): <https://github.com/petrinhu/TISS_ANS_hash>
- Spec canonica: [`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md)
- Implementacao de referencia: [`conformance/reference.py`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/reference.py) (Python + lxml)
- Status: **alpha**, 20/20 vetores sinteticos PASS (18 positivos + 2 negativos)

## Requisitos

- PHP **8.2+**.
- Extensoes: `ext-dom`, `ext-libxml`, `ext-mbstring` (uma **extensao** e um
  modulo que acrescenta funcoes ao PHP; estas tres costumam ja vir habilitadas
  na maioria das builds).

Se ainda nao tem PHP e Composer (o Composer e o gerenciador que baixa e instala
bibliotecas de projetos PHP):

- PHP: <https://www.php.net/manual/pt_BR/install.php> (em Linux costuma estar na
  distro, ex.: `sudo dnf install php php-dom php-mbstring`).
- Composer: <https://getcomposer.org/download/>
- Confira: `php --version` e `composer --version`.

## Instalacao

Ja publicado no Packagist. Adicione a lib ao seu projeto com:

```bash
composer require petrinhu/tiss-hash
```

## Quickstart

```php
<?php
require 'vendor/autoload.php';

use TissHash\TissHash;

// A partir dos bytes do arquivo
$hash = TissHash::hashTiss(file_get_contents('lote.xml'));
echo $hash; // ex.: "3aa0c578c95cdb861a125f480a8a4de5"

// Ou direto do caminho
$hash = TissHash::hashTissFile('lote.xml');
```

Exemplo completo com XML sintetico (nunca use XML ou hash real em exemplo ou
log; veja a secao Privacidade):

```php
<?php
require 'vendor/autoload.php';

use TissHash\TissHash;

$xml = <<<XML
<?xml version="1.0" encoding="iso-8859-1"?>
<ans:mensagemTISS xmlns:ans="http://www.ans.gov.br/padroes/tiss/schemas">
  <ans:cabecalho>
    <ans:identificacaoTransacao>
      <ans:tipoTransacao>ENVIO_LOTE_GUIAS</ans:tipoTransacao>
    </ans:identificacaoTransacao>
  </ans:cabecalho>
  <ans:epilogo>
    <ans:hash>00000000000000000000000000000000</ans:hash>
  </ans:epilogo>
</ans:mensagemTISS>
XML;

echo TissHash::hashTiss($xml); // 32 caracteres hex minusculos
```

> O conteudo de `<ans:hash>` e **zerado** antes do calculo (o hash nao entra em
> si mesmo). Tanto faz o que estiver la dentro: o resultado e o mesmo.

### Tratamento de erro

```php
use TissHash\InvalidTissXmlException;
use TissHash\TissHash;

try {
    $hash = TissHash::hashTiss($xmlBytes);
    // usar $hash
} catch (InvalidTissXmlException $e) {
    // XML vazio, malformado, encoding fora de escopo, ou multiplos <ans:hash>.
    // A mensagem NUNCA contem o XML original (apenas o erro tecnico do parser).
    error_log('XML TISS invalido: ' . $e->getMessage());
}
```

## API publica

| Simbolo | Tipo | Descricao |
| --- | --- | --- |
| `TissHash\TissHash::hashTiss(string $xmlBytes): string` | metodo estatico | Hash MD5 (hex, 32 chars) a partir dos bytes do XML. |
| `TissHash\TissHash::hashTissFile(string $path): string` | metodo estatico | Atalho que le o arquivo e delega para `hashTiss`. |
| `TissHash\TissHash::TISS_NAMESPACE` | constante | URI do namespace TISS (`http://www.ans.gov.br/padroes/tiss/schemas`). |
| `TissHash\InvalidTissXmlException` | classe | Excecao (subclasse de `\RuntimeException`) para XML malformado, vazio ou rejeitado por politica de seguranca. |

> Passe sempre os **bytes crus** do XML (o resultado de `file_get_contents()`).
> A lib controla o encoding internamente; nao "conserte" o XML antes de chamar.

## Contrato de rejeicao

O que a lib aceita e o que recusa:

| Entrada | Comportamento |
| --- | --- |
| XML bem-formado com 1 `<ans:hash>` | retorna hash de 32 chars |
| XML bem-formado **sem** `<ans:hash>` | **valido**: concatena tudo, sem erro |
| **Multiplos** `<ans:hash>` | **erro** `InvalidTissXmlException` (TISS preve exatamente 1; nao adivinhar qual zerar) |
| Encoding **UTF-16 / UTF-32** (detectado por BOM) | **erro** (fora de escopo; so ISO-8859-1 e UTF-8) |
| XML malformado / bytes vazios | **erro** `InvalidTissXmlException` |

Entidade externa (XXE) e sempre bloqueada. Detalhe das decisoes em
[`conformance/AMBIGUITY_NOTES.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/AMBIGUITY_NOTES.md).

## Algoritmo (resumo)

O que `TissHash::hashTiss` faz:

1. Parseia o XML com `DOMDocument`, endurecido contra XXE
   (`libxml_set_external_entity_loader` + `LIBXML_NONET`).
2. Zera o conteudo de `<ans:hash>` (casado pela URI do namespace
   `http://www.ans.gov.br/padroes/tiss/schemas` + nome local `hash`, nao pelo
   prefixo literal `ans:`).
3. Concatena o texto de cada no-folha (Element ou Comment sem filhos
   Element/Comment/PI) em ordem de documento, sem separador.
4. Calcula MD5 sobre os bytes **UTF-8** da string resultante.
5. Devolve o `md5()` minusculo (32 caracteres).

**Atencao (a pegadinha que quebra quem reimplementa):** os bytes alimentados ao
MD5 sao **UTF-8, NAO ISO-8859-1**. O manual oficial do Padrao TISS diz
"ISO-8859-1", mas isso se refere ao encoding do arquivo, nao dos bytes que
alimentam o MD5. Calcular MD5 sobre bytes ISO-8859-1 gera um hash que a ANS
**rejeita**. O valor validado contra goldens reais (privados, fora do repo) e
contra os vetores sinteticos publicos e UTF-8. Especificacao canonica completa:
[`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md).

## Conformidade

"Conformidade" significa provar que este port produz o mesmo hash da
implementacao de referencia em todos os casos previstos. Cada **vetor** e um par
"entrada -> resultado esperado": positivo deve produzir um hash, negativo deve
ser rejeitado (a lib precisa recusar o arquivo, em vez de inventar um hash).

20/20 vetores sinteticos do manifesto publico
([`conformance/vectors.json`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/conformance/vectors.json)):
18 positivos + 2 negativos, cobrindo acentuacao (discriminador UTF-8 vs
ISO-8859-1), CR/LF dentro de valor, CDATA, entidades XML, comentarios,
atributos, namespaces, BOM UTF-8, whitespace puro, leading zeros, simbolos
ISO-8859-1, multi-guia e documento grande (~600 KB). Os 2 negativos esperam
erro: multiplos `<ans:hash>` e UTF-16. Encodings suportados: ISO-8859-1 e UTF-8.

O hash de exemplo publico e o do vetor sintetico `syn_minimal.xml`:
`3aa0c578c95cdb861a125f480a8a4de5` (dado ficticio deste projeto, seguro para
reproduzir).

## Familia tiss-hash (outros ports e registries)

A lib e multi-linguagem (13 ports, todos com o **mesmo hash byte a byte**). Os
ports ja publicados em registry de pacotes:

| Linguagem | Pacote | Registry |
| --- | --- | --- |
| PHP | `petrinhu/tiss-hash` | [Packagist](https://packagist.org/packages/petrinhu/tiss-hash) |
| Python | `tiss-hash` | [PyPI](https://pypi.org/project/tiss-hash/) |
| Node.js | `tiss-hash` | [npm](https://www.npmjs.com/package/tiss-hash) |
| Rust | `tiss-hash` | [crates.io](https://crates.io/crates/tiss-hash) |
| Go | `github.com/petrinhu/TISS_ANS_hash` | [pkg.go.dev](https://pkg.go.dev/) |

Os demais ports (C, C++, Java, C#, Kotlin, Delphi/Object Pascal, Dart, WASM)
buildam do fonte no monorepo. Repositorios da familia:

- Monorepo (fonte de verdade), GitHub: <https://github.com/petrinhu/TISS_ANS_hash>
- Monorepo, mirror Codeberg: <https://codeberg.org/petrinhu/TISS_ANS_hash>

## Privacidade (LGPD)

Mensagens TISS contem **dados pessoais sensiveis de saude** (Lei 13.709/2018,
art. 5o, II): nome, CPF, carteirinha, diagnostico, procedimentos. Esta lib
**apenas calcula um hash em memoria**: nao transmite, nao persiste, nao registra
o conteudo. Mesmo assim, quem integra a lib e responsavel pelo tratamento:

- Nunca logar, imprimir, persistir ou transmitir o conteudo do XML real.
- Nunca expor o **hash de um XML real** (e PII indireta: identifica o lote).
- Nunca expor o numero da versao do Padrao TISS nem o nome da operadora.
- Em codigo, log ou exemplo, use apenas o hash sintetico
  `3aa0c578c95cdb861a125f480a8a4de5`.
- A `InvalidTissXmlException` nunca inclui o XML original, so o erro tecnico do
  parser (linha/coluna/sintaxe). Mantenha o mesmo cuidado nos seus logs.

Detalhamento: [`docs/legal/LGPD-NOTE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/legal/LGPD-NOTE.md).

## E uma IA / agente usando esta lib?

Leia o [`AGENTS.md`](AGENTS.md): regra no 1 (nao reimplemente o algoritmo, use
`composer require petrinhu/tiss-hash`), contrato de rejeicao, como validar
contra os vetores do monorepo e as obrigacoes de privacidade (LGPD) ao manipular
XML TISS.

## Documentacao

- [Wiki deste repo](https://github.com/petrinhu/tiss-hash-php/wiki): visao geral e como usar (Composer).
- [`AGENTS.md`](AGENTS.md): guia para IA/agente que usa a lib.
- [`docs/USAGE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/USAGE.md): guia de uso, receitas e FAQ (no monorepo).
- [`docs/SPEC.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/SPEC.md): especificacao canonica do algoritmo (no monorepo).
- [`docs/CONCEITOS.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/CONCEITOS.md): o que e e para que serve, sem codigo (no monorepo).
- [`docs/PORTING_GUIDE.md`](https://github.com/petrinhu/TISS_ANS_hash/blob/main/docs/PORTING_GUIDE.md): como portar para outra linguagem (no monorepo).

## Contribuindo

Este repo e read-only. Para reportar bug, sugerir melhoria ou imprecisao na spec
ou nos vetores, abra issue ou pull request **no monorepo**:

- GitHub: <https://github.com/petrinhu/TISS_ANS_hash/issues>
- Codeberg: <https://codeberg.org/petrinhu/TISS_ANS_hash/issues>

## Licenca

[MIT](LICENSE). Uso livre, comercial e nao-comercial, com manutencao do aviso de
copyright.
